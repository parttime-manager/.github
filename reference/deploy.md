# PTManager 운영 배포

[요구사항](../spec.md) · [구현 계획](../plan.md) · [API 계약](contracts.md)

EC2·RDS에 백엔드를 수동 배포하고 두 앱을 연결하는 절차다. 프로젝트 종료로 자동 배포(CD) 워크플로는 삭제했다.

## 적용 범위

- 아래는 Ubuntu·systemd, 서비스 계정 `ptmanager`, 경로 `/opt/ptmanager`를 사용하는 제안 구성이다. 실제 서버를 확인한 기록은 아니다.
- 코드의 실행 JAR은 `build/libs/ptmanager.jar`, 서버 포트는 8080이다.
- 초기 서버 설정 → 배포 → 외부 연동·앱 연결 → 운영 점검 순서로 진행한다.

## 1. 준비

- 로컬: JDK 21, SSH 키, Backend 저장소. 빌드·테스트와 PostgreSQL 마이그레이션 CI가 통과한 커밋을 사용한다.
- EC2: `/usr/bin/java`에서 Java 21 실행 가능, `curl` 설치, SSH 접속 계정의 `sudo` 권한. 아래 예시는 `ubuntu` 계정이다.
- RDS: PostgreSQL과 빈 DB `ptmanager`, 접속 계정을 준비한다. CI는 PostgreSQL 16으로 마이그레이션을 검증한다. RDS는 비공개로 두고 5432 인바운드는 EC2 보안 그룹에만 허용한다.
- 네트워크: EC2 SSH는 관리 IP에만 허용한다. HTTPS 도메인·인증서·ALB(443 → EC2 8080)를 구성한다면 EC2 8080 인바운드는 ALB 보안 그룹에만 허용한다. 두 앱 릴리스는 HTTPS URL이 필요하다.

## 2. EC2 초기 설정

### 서비스 계정과 환경 파일

EC2에서 한 번 실행한다. 먼저 `java -version`으로 Java 21을 확인한다.

```bash
sudo useradd --system --create-home --shell /usr/sbin/nologin ptmanager
sudo install -d -o ptmanager -g ptmanager -m 0755 /opt/ptmanager
sudo install -o root -g root -m 0600 /dev/null /etc/ptmanager.env
sudoedit /etc/ptmanager.env
```

환경 파일에 아래 내용을 넣고 `<...>`를 실제 값으로 바꾼다. 비밀값·환경 파일은 저장소에 넣지 않는다.

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

`JWT_SECRET`·`QR_SECRET`은 서로 다른 값으로 설정한다. 각각 `openssl rand -hex 32`로 생성할 수 있다. `prod`는 두 값의 `dev-only-` 접두사, JWT의 32바이트 미만 길이, 두 값의 동일 여부를 검사해 어기면 기동에 실패한다. QR 키의 최소 길이는 현재 별도로 검사하지 않는다.

출근 판정과 결근 배치는 JVM 기본 시간대를 사용하므로 `JAVA_TOOL_OPTIONS`로 한국 시간을 지정한다. 기존 JVM 옵션이 있으면 `-Duser.timezone=Asia/Seoul`을 합쳐 설정한다. 이 설정은 API의 UTC 타임스탬프 형식을 바꾸지 않는다.

### systemd 서비스

아래 서비스 파일을 등록한다. 환경 파일은 systemd가 읽고, 서버 프로세스는 `ptmanager` 계정으로 실행한다.

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
sudo systemctl daemon-reload
sudo systemctl enable ptmanager
```

`prod`는 PostgreSQL·Flyway를 사용하고 JPA는 스키마를 생성·검증하지 않는다(`ddl-auto=none`). 시작 시 Flyway V1~V5가 적용되며 H2 콘솔과 샘플 데이터 생성은 비활성화된다. 기존 테이블이 있는 DB는 `baseline-on-migrate=false`이므로 이력을 확인한 뒤 별도 이관해야 한다.

## 3. 빌드·배포

### 수동 배포

로컬 `backend/`에서 실행한다. `<...>`는 실제 키 경로·호스트로 바꾼다. PR CI의 빌드·테스트와 PostgreSQL 마이그레이션 검사가 통과한 커밋을 사용하고, 빌드가 성공한 뒤 업로드한다.

```bash
./gradlew clean build && \
scp -i <SSH_KEY_PATH> build/libs/ptmanager.jar ubuntu@<EC2_HOST>:/home/ubuntu/ptmanager.jar
ssh -i <SSH_KEY_PATH> ubuntu@<EC2_HOST>
```

EC2에서 아래 블록을 실행한다. 서비스 중지부터 재기동까지 잠시 중단된다. DB 변경이 포함되면 먼저 RDS 백업·스냅샷과 복원 방법을 확인한다.

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

### 헬스체크와 롤백

`/actuator/health`는 인증 없이 DB 연결까지 포함한 상태를 반환한다(DB가 죽으면 `DOWN`). 세부 컴포넌트는 노출하지 않는다.

```bash
curl -fsS --max-time 10 http://127.0.0.1:8080/actuator/health
```

ALB 헬스체크 경로도 `/actuator/health`로 설정한다. 이전 버전으로 돌아갈 때는 해당 JAR을 같은 절차로 배포하되, 이미 적용된 Flyway 변경과 호환되는지 먼저 확인한다. JAR 교체로 DB 변경이 되돌아가지는 않는다.

## 4. FCM·S3와 앱 연결

기본 `false` 설정은 서버 기동 점검용이다. 푸시·첨부파일까지 운영하려면 실제 연동을 준비하고 환경 파일을 변경한 뒤 서비스를 재시작한다.

| 대상 | 설정·확인 |
| --- | --- |
| FCM | `FCM_ENABLED=true`, `FCM_CREDENTIALS_PATH=<서비스 계정 JSON 절대 경로>`. 경로를 비우면 `GOOGLE_APPLICATION_CREDENTIALS` 등 기본 자격증명을 사용한다. JSON은 `ptmanager`만 읽을 수 있게 배치하고 통합 저장소의 `employee/google-services.json`·`employer/google-services.json`에 각 앱의 Firebase 설정을 둔다. |
| S3 | `S3_ENABLED=true`, `S3_BUCKET=<버킷>`, `S3_REGION=ap-northeast-2`(실제 리전으로 변경). EC2 IAM Role에 대상 버킷의 `notices/*`에 대한 `s3:PutObject` 권한을 부여한다. |
| 첨부 조회 | `S3_PUBLIC_BASE_URL=<파일 배포 URL>`을 지정한다. 비우면 S3 객체 URL을 반환한다. 현재 반환 URL에는 서명이 없으므로 앱이 파일을 읽을 수 있는 경로·접근 정책을 별도로 구성하고 실제 조회를 확인한다. |
| 두 앱 | 통합 저장소 루트 또는 앱별 `local.properties`에 `base.url=https://<API_DOMAIN>/`을 설정하고 다시 빌드한다. 릴리스 서명 키는 [앱 릴리스 서명](#앱-릴리스-서명)을 따른다. |

FCM 비활성 상태에서는 로그만 남고 푸시가 발송되지 않는다. S3 비활성 상태에서는 파일을 저장하지 않고 가짜 URL을 반환하므로 기존 스텁 URL의 파일을 이관할 수 없다. 실연동 후 재업로드해야 한다.

첨부 업로드는 공지 작성보다 먼저 수행하며 공지 삭제·작성 취소 시 S3 객체를 자동 삭제하지 않는다. 저장 비용과 보존 정책을 정할 때 이 동작을 반영한다. 정적 QR 인쇄물을 쓰는 매장에서는 사장 앱 QR 화면의 주기적 재발급이 인쇄물을 무효화할 수 있으므로 [QR 모드와 제약](contracts.md#검증과-모드)을 확인한다.

### 앱 릴리스 서명

통합 저장소 루트 또는 앱별 `local.properties`에 다음 값을 설정하면 `./gradlew assembleRelease`로 서명된 APK를 만든다. 경로는 절대 경로를 권장한다. 서명 설정이 없으면 로컬 릴리스는 서명 없이 빌드된다.

| 로컬 속성 | 빌드 환경변수 | 값 |
| --- | --- | --- |
| `keystore.path` | `KEYSTORE_PATH` | 키스토어 절대 경로 |
| `keystore.password` | `KEYSTORE_PASSWORD` | 키스토어 비밀번호 |
| `key.alias` | `KEY_ALIAS` | 키 별칭 |
| `key.password` | `KEY_PASSWORD` | 키 비밀번호 |

키스토어는 다음 명령으로 생성한다. `<ALIAS>`는 실제 별칭으로 바꾼다.

```bash
keytool -genkeypair -keystore release.jks -alias <ALIAS> -keyalg RSA -keysize 2048 -validity 10000
```

앱 CI는 `v*` 태그에서 `KEYSTORE_BASE64` 시크릿을 임시 키스토어 파일로 복원하고 `KEYSTORE_PATH`를 설정한다. 통합 저장소에 `KEYSTORE_BASE64`·`KEYSTORE_PASSWORD`·`KEY_ALIAS`·`KEY_PASSWORD` 시크릿과 `/`로 끝나는 HTTPS API 주소인 Repository variable `BASE_URL`을 등록한다. CI는 이 값으로 루트 `local.properties`를 생성한다. 앱별 설정과 CI 요구사항은 [Android README](https://github.com/parttime-manager/android-app#ci)를 따른다.

키스토어·`local.properties`·자격증명은 커밋하지 않는다. CI는 추적된 `google-services.json`·`.jks`·`.keystore` 파일을 차단한다.

현재 앱 CI는 `google-services.json`을 주입하지 않는다. FCM이 필요한 릴리스는 각 모듈의 Firebase 설정을 빌드 전에 제공하는 CI 단계가 추가로 필요하다. 파일 없이 APK 빌드가 성공해도 실제 푸시 수신까지 검증된 것은 아니다.

## 5. 운영 점검

```bash
sudo systemctl status ptmanager
sudo journalctl -u ptmanager -f
sudo systemctl restart ptmanager
```

첫 배포 후에는 실제 계정으로 로그인·매장 가입·QR 출퇴근·대타 승인 푸시·첨부 업로드와 재접속 후 조회를 확인한다. `prod`에는 시드 계정이 자동 생성되지 않는다. RDS 자동 백업·보존 기간과 복원 절차도 운영 전에 정한다.

| 증상 | 점검 |
| --- | --- |
| 서비스 시작 실패 | `journalctl -u ptmanager -n 200 --no-pager`: Java 버전, 환경 파일, DB 접속, Flyway 오류 |
| DB 연결 실패·풀 대기 시간 초과 | RDS 엔드포인트·계정·보안 그룹, RDS 연결 한도와 `DB_POOL_MAX`(기본 20)·`DB_POOL_MIN`(기본 5) |
| `/actuator/health` `DOWN` | RDS 접속·보안 그룹·자격증명. 응답이 아예 없으면 프로세스·포트를 먼저 확인 |
| 푸시 미수신·첨부 조회 실패 | FCM/S3 플래그, 서비스 계정 파일 권한, 기기 토큰, IAM·파일 URL 접근 정책 |
| 릴리스 앱 기동 실패 | HTTPS `base.url`·인증서와 앱 재빌드 여부 |
| 출근 판정·04:00 결근 배치 시각 불일치 | `/etc/ptmanager.env`의 JVM 시간대 옵션·서비스 재시작 여부. 서버 코드가 JVM 기본 시간대를 사용함 |
