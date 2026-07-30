# 배포 가이드 — PTManager

[문서 안내](../README.md#관련-문서)

Ubuntu EC2(`ubuntu` 접속, 서비스 계정 `ptmanager`, `/opt/ptmanager/ptmanager.jar`, 포트 8080)와 RDS PostgreSQL 16(`ptmanager` DB, `ddl-auto=none`, `baseline-on-migrate=false`) 기준의 수동 배포 제안 구성이다(자동 배포 워크플로 없음).

## 1. 준비

- 로컬(JDK 21, SSH 키)에서 CI와 PostgreSQL 마이그레이션이 통과한 커밋을 준비한다.
- EC2(Java 21 `/usr/bin/java`, `curl`) 8080 인바운드는 ALB(HTTPS 443 → 8080)에만, SSH는 관리 IP에만, 비공개 RDS 5432는 EC2 보안 그룹에만 허용한다.

## 2. EC2 초기 설정

```bash
sudo useradd --system --create-home --shell /usr/sbin/nologin ptmanager
sudo install -d -o ptmanager -g ptmanager -m 0755 /opt/ptmanager
sudo install -o root -g root -m 0600 /dev/null /etc/ptmanager.env
sudoedit /etc/ptmanager.env
```

`/etc/ptmanager.env` 예시(`JWT_SECRET`·`QR_SECRET`은 `openssl rand -hex 32` 등으로 서로 다르게 생성; `dev-only-` 접두사·32바이트 미만 JWT·동일 키는 prod 기동 실패, `Asia/Seoul`은 출근·04:00 결근 배치 기준이며 API UTC 형식은 유지):

```ini
SPRING_PROFILES_ACTIVE=prod
JAVA_TOOL_OPTIONS=-Duser.timezone=Asia/Seoul
SPRING_DATASOURCE_URL=jdbc:postgresql://<RDS_ENDPOINT>:5432/ptmanager
SPRING_DATASOURCE_USERNAME=<DB_USER>
SPRING_DATASOURCE_PASSWORD=<DB_PASSWORD>
JWT_SECRET=<JWT_RANDOM_SECRET>
QR_SECRET=<QR_RANDOM_SECRET>
QR_MAX_AGE_SECONDS=60
FCM_ENABLED=false
S3_ENABLED=false
```

```bash
sudo tee /etc/systemd/system/ptmanager.service >/dev/null <<'UNIT'
[Unit]
Description=PTManager Backend
After=network-online.target
Wants=network-online.target

[Service]
User=ptmanager
WorkingDirectory=/opt/ptmanager
EnvironmentFile=/etc/ptmanager.env
ExecStart=/usr/bin/java -jar /opt/ptmanager/ptmanager.jar
SuccessExitStatus=143
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
UNIT
sudo systemctl daemon-reload && sudo systemctl enable ptmanager
```

## 3. 빌드·배포 및 헬스체크

DB 변경 전 RDS 스냅샷을 확인한다. 로컬 `backend/`에서 빌드·업로드 후 EC2에서 배포한다:

```bash
./gradlew clean build && \
scp -i <SSH_KEY_PATH> build/libs/ptmanager.jar ubuntu@<EC2_HOST>:/home/ubuntu/ptmanager.jar
ssh -i <SSH_KEY_PATH> ubuntu@<EC2_HOST>
```

```bash
bash <<'DEPLOY'
set -e
test -s /home/ubuntu/ptmanager.jar
sudo systemctl stop ptmanager
sudo install -o ptmanager -g ptmanager -m 0644 /home/ubuntu/ptmanager.jar /opt/ptmanager/ptmanager.jar
sudo systemctl start ptmanager
for attempt in $(seq 1 12); do
  if curl -fsS --max-time 5 http://127.0.0.1:8080/actuator/health | grep -q '"status":"UP"'; then exit 0; fi
  sleep 5
done
sudo journalctl -u ptmanager -n 100 --no-pager
exit 1
DEPLOY
```

`/actuator/health`(ALB 헬스체크 경로)는 DB 연결을 포함해 `UP`/`DOWN`을 반환한다. 롤백 시 이전 JAR이 적용된 Flyway 스키마와 호환되는지 확인한다.

## 4. FCM·S3와 앱 연결

| 대상 | 설정·확인 |
| --- | --- |
| FCM | `FCM_ENABLED=true`, `FCM_CREDENTIALS_PATH=<JSON 경로>`(비우면 `GOOGLE_APPLICATION_CREDENTIALS`), 앱별 `employee/`·`employer/google-services.json` 배치 |
| S3·첨부 | `S3_ENABLED=true`, `S3_BUCKET=<버킷>`, `S3_REGION=ap-northeast-2`, `S3_PUBLIC_BASE_URL=<배포 URL>`(비우면 S3 URL, 무서명이므로 읽기 권한 별도 구성), EC2 IAM `notices/*` `s3:PutObject` 부여(스텁 URL은 재업로드 필요, 취소·삭제 시 S3 파일 미삭제) |
| 두 앱 | `local.properties`에 `base.url=https://<API_DOMAIN>/` 설정 후 재빌드(인쇄 QR 사용 시 [QR 제약](../01-func/01-requirements.md#qr-출퇴근) 확인) |

### 앱 릴리스 서명

릴리스 빌드는 [Android 릴리스 설정](https://github.com/parttime-manager/android-app#firebase와-릴리스)을 따르며, 서명 미설정 시 무서명 빌드된다(시크릿·`.jks`·`google-services.json` 커밋 금지).

| 로컬 속성 | 빌드 환경변수 | 설명 |
| --- | --- | --- |
| `keystore.path` / `keystore.password` | `KEYSTORE_PATH` / `KEYSTORE_PASSWORD` | 키스토어 절대 경로 / 비밀번호 |
| `key.alias` / `key.password` | `KEY_ALIAS` / `KEY_PASSWORD` | 키 별칭 / 비밀번호 |

```bash
keytool -genkeypair -keystore release.jks -alias <ALIAS> -keyalg RSA -keysize 2048 -validity 10000
```

## 5. 운영 점검

```bash
sudo systemctl status ptmanager
sudo journalctl -u ptmanager -f
sudo systemctl restart ptmanager
```

`prod`는 시드 계정을 생성하지 않으므로 실계정으로 로그인·가입·QR·대타 푸시·첨부 조회를 확인한다.

| 증상 | 점검 |
| --- | --- |
| 기동 실패·헬스 `DOWN` | `journalctl -u ptmanager -n 200 --no-pager`, Java 21, `/etc/ptmanager.env`, RDS 보안 그룹·자격증명·Flyway, `DB_POOL_MAX`(20)·`DB_POOL_MIN`(5) |
| 푸시·첨부·앱 기동 실패 | FCM/S3 플래그·권한·토큰·URL 정책, 앱 HTTPS `base.url`·재빌드 |
| 출근·04:00 배치 시각 불일치 | `/etc/ptmanager.env`의 `-Duser.timezone=Asia/Seoul` 및 재시작 여부 |
