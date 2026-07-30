# PTManager 문서

PTManager의 요구사항·설계·배포 안내입니다(원격 저장소 `.github`).

## 시작하기

[요구사항](01-func/01-requirements.md) → [기술 구조](03-tech/01-architecture.md) 순서로 읽고, 실행·테스트는 아래 코드 저장소 README를 따릅니다. 구현 제약은 [요구사항](01-func/01-requirements.md#현재-구현과의-차이), 운영 점검은 [배포 가이드](05-ops/01-deploy.md)에 적으며 실배포·실기기 검증 완료를 뜻하지 않습니다.

## 관련 문서

- [작업 지침](AGENTS.md): 작업 순서·문서 관리.
- [프로젝트 소개](profile/README.md): 핵심 기능·아키텍처·앱 화면·팀 구성.

### 문서 분류 기준

<!-- doc-standard:start — doc-standard에서 동기화한다. 이 구간은 직접 고치지 않는다. -->
| 위치 | 담는 내용 |
| --- | --- |
| [`01-func/`](01-func/01-requirements.md) | 범위, 역할별 기능, 업무 정책, 완료 기준 |
| [`02-uxui/`](02-uxui/01-design.md) | 화면 배치·조작·디자인 기준·시각 자산 |
| [`03-tech/`](03-tech/README.md) | 아키텍처, 소스 구조, 데이터 모델, REST/IPC 계약 |
| [`04-test/`](04-test/01-verification.md) | 검증 방법, 판정 기준, 최신 판정과 증거 |
| [`05-ops/`](05-ops/01-deploy.md) | 배포·인프라 구성·운영 점검·장애 대응 |
| 이 저장소 루트·주제 폴더 | 비명세 자료(과제 안내, 보고서, 발표, 협업 규칙, 자료 수집 기록) |
| 코드 저장소 루트 README | 로컬 설치·실행·테스트 명령 |

API 상세 원본은 OpenAPI·라우터·DTO이며 명세에는 원본 위치와 한계만 적는다. 배포 절차는 `05-ops/`(없으면 기술 명세)에 두고 로컬 실행 명령은 README를 링크한다.
<!-- doc-standard:end -->

## 관련 저장소

| 저장소 | 내용 |
| --- | --- |
| [백엔드](https://github.com/parttime-manager/backend) | 서버 실행·데모 계정·테스트 |
| [Android](https://github.com/parttime-manager/android-app) | 직원·사장 앱 설정·빌드·사용. `employee/`·`employer/` 모듈과 공통 `common/` 관리 |
