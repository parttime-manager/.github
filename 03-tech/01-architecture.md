# 기술 구조 및 설계 — PTManager

[요구사항](../01-func/01-requirements.md)을 구현하는 아키텍처·소스 구조와 남은 작업을 다룬다(동작: [업무 규칙](../01-func/01-requirements.md#업무-규칙), 저장: [데이터 모델](02-data-model.md), 검증: [테스트 명세](../04-test/01-verification.md), 실행: [관련 저장소](../README.md#관련-저장소)).

## 아키텍처와 경계

API·DB·앱 ID·저장소 키를 유지하며 백엔드(`com.ptmanager.backend`)는 헥사고날, Android는 MVVM·클린 아키텍처로 패키지를 분리한다.

### 백엔드 경계

| 패키지 | 책임 |
| --- | --- |
| `domain` | 업무 모델·상태, JPA 매핑(`Long` FK) |
| `application` | 기능별 서비스·DTO, `port/out`(`EntityStore`, `PageQuery`/`PageResult`, `TokenIssuer`, `CurrentUser`, `PushSender`, `StorageService`) |
| `adapter/inbound/web` | HTTP 입력 검증·변환, 서비스 직접 호출 |
| `adapter/out/{persistence,...}` | Spring Data JPA·페이지 변환 및 인증·푸시·파일 외부 연동 |
| `configuration`, `common` | 빈 조립·시드, `ApiExceptionHandler`·`WorkplaceAccessGuard` 등 공통 지원 |

의존성은 `web → application → domain`, `adapter/out → application/port/out`이다. 포트에 Spring Data 타입을 노출하지 않으며 도메인 JPA와 서비스의 Spring DI·트랜잭션은 유지한다.

### Android 경계

| 패키지 | 책임 |
| --- | --- |
| `domain/{model,repository}` | 도메인 모델, suspend 저장소 계약 |
| `data` | Retrofit·OkHttp, 저장소·업로드·세션 구현 |
| `presentation/{view,viewmodel}` | 화면 표시·입력·탐색, `AppViewModel`·StateFlow 상태 조정(`NotificationsViewModel` 등) |
| 루트 `AppGraph` | 저장소 주입과 ViewModel factory 조립 |

의존성은 `View → ViewModel → domain`, `data → domain`이다. ViewModel은 View·Context·ViewBinding을 보관하지 않고 Kotlin 인터페이스 위임으로 저장소를 호출한다(일부 View 세션 접근과 루트 진입점은 유지). `employee`·`employer`는 독립 앱 모듈이고 `common`은 소스 템플릿이다(`common/android.gradle`이 `build/generated/ptmCommon/{main,test,androidTest}`로 복사하며 `com.example.ptmanagercommon`을 앱 namespace로 치환).

## 소스 구조

- **백엔드**([backend](https://github.com/parttime-manager/backend), JDK 21): 운영은 PostgreSQL·Flyway(`ddl-auto=none`), 로컬·테스트는 H2(`create-drop`)를 쓴다. 공지 미확인은 `last_read_notice_at`, QR은 무상태 서명 검증, 인건비는 조회 시 실시간 계산([기준](../01-func/01-requirements.md#근무와-인건비))으로 처리하며 FCM·S3는 플래그로 전환한다.
- **Android**([android-app](https://github.com/parttime-manager/android-app), minSdk 29 / targetSdk 36): Kotlin·XML·Retrofit·OkHttp·Coroutines 기반이며 401 시 토큰을 갱신해 재시도한다. `google-services.json`이 있을 때만 Firebase 플러그인을 적용한다([공통 코드](https://github.com/parttime-manager/android-app/blob/main/common/README.md), [Android README](https://github.com/parttime-manager/android-app#시작하기)).

## 남은 작업과 리스크

자동 배포(CD)는 없으며 운영 재개 시 [배포 가이드](../05-ops/01-deploy.md)에 따라 아래 항목을 준비·확인한다(구현 차이는 [요구사항](../01-func/01-requirements.md#현재-구현과의-차이) 참고).

| 우선순위 | 구현된 흐름 | 준비·확인할 항목 |
| --- | --- | --- |
| P1 서버 배포 | 수동 빌드·테스트 → JAR 전송·재기동 → DB 헬스체크 | RDS·HTTPS·EC2 환경 파일·SSH·백업 절차 확정 |
| P2 외부 연동 | FCM·S3 스텁 및 활성화 플래그 전환 | Firebase 계정·앱 파일, S3 버킷·IAM·첨부 조회 경로 |
| P3 앱 릴리스 | 서명 설정·HTTPS 강제·수동 릴리스 빌드(코드 축소 비활성) | 키스토어 서명(미설정 시 무서명 빌드), HTTPS API, 실기기 검증 |

통신 계약: [REST API 명세](api.md).
