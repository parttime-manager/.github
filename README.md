# PTManager 문서

PTManager의 요구사항·설계·배포 안내입니다. 원격 저장소는 조직 프로필과 문서를 함께 관리하는 `.github`입니다.

## 시작하기

[요구사항](spec.md) → [구현 계획](plan.md) 순서로 확인합니다. 설치·실행·테스트 명령은 아래 코드 저장소의 README를 따릅니다.

현재 구현의 제약은 [API 계약](reference/contracts.md), 운영에서 별도로 확인할 항목은 [배포 가이드](reference/deploy.md)에 기록합니다. 실제 배포·실기기 검증 완료를 뜻하지 않습니다.

## 관련 문서

| 문서 | 내용 |
| --- | --- |
| [요구사항](spec.md) | 제품 범위·동작 규칙·완료 기준 |
| [구현 계획](plan.md) | 아키텍처·구현 방향·검증 전략 |
| [API 계약](reference/contracts.md) | 인증·요청·응답·오류 규약 |
| [데이터 모델](reference/data-model.md) | 관계·테이블·무결성 |
| [배포 가이드](reference/deploy.md) | 환경 설정·배포·운영 점검 |
| [작업 지침](AGENTS.md) | 문서별 역할·변경 원칙 |
| [프로젝트 소개](profile/README.md) | 핵심 기능·아키텍처·앱 화면·팀 구성 |

## 관련 저장소

| 저장소 | 내용 |
| --- | --- |
| [백엔드](https://github.com/parttime-manager/backend) | 서버 실행·데모 계정·테스트 |
| [Android](https://github.com/parttime-manager/android-app) | 직원·사장 앱 설정·빌드·사용. `employee/`·`employer/` 모듈과 공통 `common/` 관리 |
