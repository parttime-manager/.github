# PTManager 구현 계획

[요구사항](spec.md) · [API 계약](reference/contracts.md) · [데이터 모델](reference/data-model.md)

## 아키텍처

백엔드는 헥사고날·클린 아키텍처, Android는 MVVM·클린 아키텍처를 기준으로 패키지 의존성을 분리한다. API URL·JSON 계약, DB 스키마, 앱 ID와 저장소 키를 유지한 현재 구현을 설명한다.

### 백엔드 경계

| 패키지 | 책임 |
| --- | --- |
| `domain` | 업무 모델·상태와 JPA 매핑 |
| `application` | 유스케이스 서비스·DTO와 `port/out`의 저장소·외부 연동 계약 |
| `adapter/inbound/web` | HTTP 입력 검증·변환과 서비스 호출 |
| `adapter/out/persistence` | Spring Data JPA 구현과 페이지 타입 변환 |
| `adapter/out`의 나머지 패키지 | 인증·푸시·파일 등 외부 연동 구현 |
| `configuration` | Spring 빈 설정·조립과 초기 데이터 |
| `common` | 오류 처리 등 공통 지원 코드 |

소스 의존성은 `web → application → domain`, `출력 어댑터 → application/port/out` 방향이다. 실행 시 서비스는 출력 포트를 호출하고 Spring이 연결한 어댑터가 DB·외부 시스템에 접근한다. 컨트롤러는 별도 입력 포트 인터페이스 없이 애플리케이션 서비스를 직접 호출한다. 계층은 단일 백엔드 Gradle 프로젝트 안의 패키지 경계이며 독립 배포 서비스가 아니다.

저장소 포트는 `EntityStore`와 업무별 계약을 사용한다. Spring Data 저장소가 포트를 구현하며, 페이지 변환이 필요한 경로는 별도 영속성 어댑터를 사용한다. 포트에는 Spring Data 타입을 노출하지 않는다. 도메인의 JPA 매핑과 서비스의 Spring DI·트랜잭션 참조는 유지하므로 모든 코어가 프레임워크와 독립된 구조는 아니다.

패키지 기준은 `com.ptmanager.backend`다. 서비스와 DTO는 `application` 아래 auth·shift·notice 등 기능별로 배치한다. 페이지 요청·응답은 `PageQuery`·`PageResult`, 인증은 `TokenIssuer`·`CurrentUser`, 푸시·파일은 `PushSender`·`StorageService` 포트를 사용한다. `common/access/WorkplaceAccessGuard`와 비밀번호 인코더·이벤트·트랜잭션 콜백 등 Spring 기반 지원 코드도 유지한다.

### Android 경계

| 패키지 | 책임 |
| --- | --- |
| `domain/model`, `domain/repository` | 모델과 suspend 저장소 계약 |
| `data` | Retrofit·OkHttp, 저장소 구현·업로드·세션 저장 |
| `presentation/view` | 화면 표시·입력·탐색 |
| `presentation/viewmodel` | 요청 조정과 화면 상태 |
| 루트 `AppGraph` | 구체 저장소 연결과 ViewModel factory |

소스 의존성은 `View → ViewModel → domain`, `data → domain`이며, `AppGraph`에서 구현을 연결한다. ViewModel은 View·Context·ViewBinding을 보관하지 않고 `ViewModelProvider`로 얻는다. 단순 요청은 공통 `AppViewModel`의 suspend 호출과 요청 상태를 사용한다. 이 호출은 호출자 코루틴에서 실행되며, 모든 요청이 `viewModelScope`로 이동한 것은 아니다. 전용 ViewModel의 StateFlow와 기존 화면의 suspend 결과 렌더링을 함께 사용한다. View의 세션·설정 접근과 루트 Application·메시징 진입점은 남아 있다.

`employee`·`employer`는 별도 앱 모듈이며 `common`은 컴파일되는 라이브러리 모듈이 아닌 소스 템플릿이다. `common/android.gradle`이 `common/src`, `common/test`, `common/androidTest`를 각 앱의 `build/generated/ptmCommon/{main,test,androidTest}`로 복사하고 패키지 `com.example.ptmanagercommon`을 앱 namespace로 치환한다. 생성 파일 대신 템플릿을 수정한다. 공통 domain은 모델·저장소 계약 중심이며 별도 기능별 유스케이스 계층은 없다. 역할별 화면도 각 앱의 `presentation/view`에 둔다. `NotificationsViewModel`은 알림 목록·로딩·오류를 StateFlow로 제공하고 전체 읽음 성공 후 상태를 갱신한다.

### 경계 검증과 적용 범위

백엔드 `CleanArchitectureTest`는 domain/application의 어댑터 참조, domain의 application 참조, application의 JPA 직접 참조, 포트의 Spring 참조, 웹 어댑터의 영속성 구현 참조·트랜잭션 선언을 검사한다. 소스 검사이므로 전체 런타임 의존성이나 업무 동작을 보장하지 않는다.

Android `common/test/CleanArchitectureTest`는 `common/src`의 domain·ViewModel에서 data·Android UI·통신 라이브러리 참조와 공통 화면의 `Network.api` 직접 호출을 검사한다. 역할별 앱 소스는 이 검사의 탐색 범위에 포함되지 않는다. `NotificationsViewModelTest`와 두 앱의 빌드·단위 테스트·Lint를 함께 실행한다.

## 구현 방향

### 저장소와 계약

서버는 [backend](https://github.com/parttime-manager/backend), 직원·사장 앱은 [android-app](https://github.com/parttime-manager/android-app) 저장소로 관리한다. [API 계약](reference/contracts.md)과 서버 Swagger를 기준으로 연동하고, 데이터 관계와 저장 의미는 [데이터 모델](reference/data-model.md)을 따른다.

### 백엔드

- Kotlin·Spring Boot에 위 아키텍처 경계를 적용한다.
- 공통 오류는 `ApiExceptionHandler`, 매장 접근은 `WorkplaceAccessGuard`에서 처리한다. JPA 연관 객체 대신 `Long` FK를 저장한다.
- 운영은 PostgreSQL·Flyway와 `ddl-auto=none`, 로컬·테스트는 H2와 `create-drop`을 사용한다.
- 공지 미확인은 `last_read_notice_at`으로 판단한다. QR 토큰은 서명으로 검증하고 DB에 저장하지 않는다.
- 인건비는 별도 집계 테이블 없이 조회 시 근무 기록과 현재 시급으로 계산한다. 계산 조건은 [API 계약](reference/contracts.md#근무근태인건비)을 따른다.
- FCM·S3는 인터페이스와 활성화 플래그로 전환한다. WebSocket·Redis는 실제 병목이 확인되면 검토한다.

### Android 앱

Kotlin·XML View·Retrofit·OkHttp·Coroutines를 사용하고 401 응답 시 토큰을 갱신해 재시도한다. 공통 소스·리소스·빌드 설정은 통합 저장소의 `common/`에서 관리한다. 직원 앱은 `:employee`, 사장 앱은 `:employer` Gradle 모듈이며 별도 APK를 빌드한다. 앱별 화면·색상·문구는 각 모듈에 둔다.

공통 코드 생성 방식과 빌드 설정은 [공통 코드 안내](https://github.com/parttime-manager/android-app/blob/main/common/README.md), 실행·환경 설정·CI는 [Android README](https://github.com/parttime-manager/android-app#시작하기)를 따른다. 한 번의 체크아웃으로 두 앱을 검증한다.

Firebase 설정은 모듈별 파일이 있을 때만 Google Services 플러그인을 적용한다.

## 배포 준비

프로젝트 종료로 백엔드 자동 배포(CD) 워크플로는 삭제했다. 아래는 남아 있는 흐름과 운영에서 준비할 항목을 구분한 계획이다. 실제 배포 완료를 뜻하지 않는다. 서버 초기 설정·환경변수·운영 점검은 [배포 가이드](reference/deploy.md)에서 관리한다.

| 우선순위 | 구현된 흐름 | 준비·확인할 항목 |
| --- | --- | --- |
| P1 서버 배포 | 수동 빌드·테스트 → JAR 전송 → 재기동 → DB 포함 헬스체크 | RDS·HTTPS, EC2 환경 파일, SSH 접속 설정 |
| P2 외부 연동 | FCM·S3의 스텁과 실연동 전환 | Firebase 서비스 계정·앱 설정, S3 버킷·IAM·첨부 조회 경로 |
| P3 앱 릴리스 | 로컬·환경변수 서명 설정, `v*` 태그의 CI 릴리스 빌드 | 키스토어·CI 서명 시크릿, HTTPS API 주소, 실제 기기 검증 |

## 검증 전략

| 대상 | 검증 |
| --- | --- |
| 백엔드 | `./gradlew test`, CI의 H2 테스트·PostgreSQL 마이그레이션 검사 |
| 두 앱 | 통합 저장소에서 `./gradlew assembleDebug testDebugUnitTest lintDebug` |
| 통합 흐름 | 로컬 `./gradlew bootRun`의 시드로 [기능 완료 기준](spec.md#기능-검증) 재현 |
| 운영 | 실제 계정으로 [운영 완료 기준](spec.md#운영-검증) 재현 |

공통 Android 코드를 바꾸면 두 앱을 모두 검증한다. 대타 중복 승인 409·근무자 유지, QR 60초 만료 경계, prod 비밀값 검사는 `BaselineApiTests`·`ProdSecretGuardTest`에서 확인한다. 실행·데모 계정은 [Backend README](https://github.com/parttime-manager/backend#시작하기)를 따른다.

초안의 목록·상세·출퇴근·급여 노출 차이, 일반 편성과 대타의 시간 충돌 검사 범위는 [현재 API 계약](reference/contracts.md#근무근태인건비)을 기준으로 확인한다. H2 테스트 통과만으로 PostgreSQL 제약이나 실제 FCM·S3 연동이 검증된 것으로 보지 않는다.

## 제약과 미결정

- FCM·S3는 기본적으로 스텁이다. 운영 푸시와 첨부파일은 실연동 설정이 필요하다.
- 두 앱 릴리스는 HTTPS가 아니면 기동하지 못한다.
- 제품 요구사항과 현재 구현의 차이는 [요구사항의 차이 목록](spec.md#현재-구현과의-차이)을 따른다.
- 로컬 앱 릴리스는 키스토어 설정이 없으면 서명 없이 빌드된다. 코드 축소는 비활성화되어 있다.
- 배포 가이드의 EC2 경로·systemd 서비스는 제안 구성이다. 실제 서버와 운영 백업·복원 절차는 배포 시 확정한다.
