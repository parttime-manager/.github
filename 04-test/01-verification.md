# 테스트 명세 — PTManager

완료 기준: [요구사항](../01-func/01-requirements.md) · 구조·남은 작업: [기술 구조](../03-tech/01-architecture.md) · 실행: [각 코드 저장소 README](../README.md#관련-저장소)

## 검증 전략

| 대상 | 검증 |
| --- | --- |
| 백엔드 | [백엔드 테스트](https://github.com/parttime-manager/backend#테스트): H2 테스트(`BaselineApiTests`, `ProdSecretGuardTest`의 대타 409·QR 60초·prod 시크릿 검사) 및 CI PostgreSQL 마이그레이션 |
| 두 앱 | [Android 빌드](https://github.com/parttime-manager/android-app#시작하기)·[테스트](https://github.com/parttime-manager/android-app#테스트): 두 앱 컴파일·단위 테스트(`NotificationsViewModelTest`)·Lint |
| 통합·운영 | [로컬 서버](https://github.com/parttime-manager/backend#시작하기) 시드로 [기능 검증](../01-func/01-requirements.md#기능-검증), 실계정으로 [운영 검증](../01-func/01-requirements.md#운영-검증) 재현 |

초안 노출·시간 충돌 검사 범위는 [현재 구현과의 차이](../01-func/01-requirements.md#현재-구현과의-차이)를 따르며, H2 통과를 PostgreSQL 제약·실제 FCM/S3 검증으로 간주하지 않는다.

## 아키텍처 경계 및 API 계약 검증

- 백엔드 `CleanArchitectureTest`(패키지·포트·트랜잭션 경계)와 Android `common/test/CleanArchitectureTest`(`common/src`의 domain·ViewModel 경계 및 `Network.api` 직접 호출 금지; 앱별 모듈은 범위 외)를 실행한다.
- API 변경 시 [REST API 명세](../03-tech/api.md)의 정상·권한 거부·오류 응답을 OpenAPI·컨트롤러·DTO와 대조한다.
