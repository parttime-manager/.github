<h1 align="center">PTManager</h1>

<p align="center">
  <b>소규모 매장의 근무 관리를 단톡방과 종이 근무표에서 앱 하나로 옮깁니다</b><br/>
  대타·근태·스케줄을 직원 앱 · 사장 앱이 공유하는 풀스택 팀 프로젝트
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Kotlin-7F52FF?style=for-the-badge&logo=kotlin&logoColor=white" alt="Kotlin"/>
  <img src="https://img.shields.io/badge/Android-3DDC84?style=for-the-badge&logo=android&logoColor=white" alt="Android"/>
  <img src="https://img.shields.io/badge/Spring Boot-6DB33F?style=for-the-badge&logo=springboot&logoColor=white" alt="Spring Boot"/>
  <img src="https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white" alt="PostgreSQL"/>
</p>

<p align="center">
  <a href="https://github.com/parttime-manager/backend">Backend</a> /
  <a href="https://github.com/parttime-manager/android-app">Android</a>
</p>

---

## 핵심 기능

<p align="center">
  <img src="../assets/key-features.svg" width="100%" alt="PTManager 핵심 기능"/>
</p>

## 시스템 아키텍처

<p align="center">
  <img src="../assets/architecture.svg" width="100%" alt="PTManager 시스템 아키텍처"/>
</p>

두 앱은 하나의 REST API와 데이터를 공유합니다. 서버가 직원·사장 역할과 매장 소속을 검사하고, 직원·사장 앱은 `android-app/` 저장소의 `employee/`·`employer/` 모듈로 관리하며 공통 소스는 `common/`에 둡니다. FCM·S3는 설정으로 활성화하며, 기본 로컬 실행에서는 스텁을 사용합니다.

백엔드는 헥사고날·클린 아키텍처의 유스케이스·출력 포트·어댑터로, Android는 MVVM·클린 아키텍처의 View·ViewModel·도메인·데이터 구현으로 분리합니다. JPA 매핑과 일부 기존 화면 렌더링은 유지합니다. 정확한 의존성 방향과 검사 범위는 [아키텍처 설계](../plan.md#아키텍처)를 참고하세요.

## 앱 화면

2026-10-28 현재 소스에서 직원·사장 Debug APK를 캐시 재사용 없이 다시 빌드·설치한 뒤 Android 에뮬레이터에서 실제 앱 화면 8장을 새로 촬영했습니다. 로컬 서버의 데모 계정과 샘플 데이터를 사용했습니다.

탭 목록 대신 직원의 **근무 확인 → 대타 요청 → 급여 확인**, 사장의 **근무 편성 → 대타 승인 → QR 근태 → 인건비 확인**을 선정했습니다. 대타는 요청·승인 화면을 촬영했으며 실제 요청 제출이나 승인은 하지 않았습니다.

### 직원용 앱

| 오늘 근무·소식 | 주간 스케줄 | 대타 요청 | 월 급여 |
| --- | --- | --- | --- |
| <img src="../assets/EmployeeApp/01_home.png" width="220" alt="오늘 근무 시간과 출근 버튼 및 매장 소식"/> | <img src="../assets/EmployeeApp/02_schedule.png" width="220" alt="주간 달력과 선택한 날짜의 내 근무"/> | <img src="../assets/EmployeeApp/03_swap_request.png" width="220" alt="근무를 선택한 대타 요청의 공개 범위와 사유 입력"/> | <img src="../assets/EmployeeApp/04_stats.png" width="220" alt="근무 시간과 시급을 반영한 월 급여와 주차별 추이"/> |

### 사장용 앱

| 주간 근무 편성 | 대타 지원자·승인 | 출근 QR | 매장 인건비 |
| --- | --- | --- | --- |
| <img src="../assets/EmployerApp/02_schedule.png" width="220" alt="직원별 근무 시간과 예상 인건비를 포함한 주간 근무표"/> | <img src="../assets/EmployerApp/03_swap_approval.png" width="220" alt="대타 근무와 사유 및 지원자 2명을 확인하는 승인 상세"/> | <img src="../assets/EmployerApp/01_attendance_qr.png" width="220" alt="직원 출근용 회전 QR과 자동 갱신 안내"/> | <img src="../assets/EmployerApp/04_stats.png" width="220" alt="월별 총 인건비와 직원별 급여 분석"/> |

## 팀 구성

<table>
  <tr><td align="center"><a href="https://github.com/ken-jeong"><img src="https://github.com/ken-jeong.png" width="60px" alt="정상겸"/></a></td><td><b>정상겸</b><br/><sub>팀장 · 풀스택</sub></td><td>양쪽 앱 화면/API 연동 전반</td></tr>
  <tr><td align="center"><a href="https://github.com/Movinggun-bit"><img src="https://github.com/Movinggun-bit.png" width="60px" alt="이동건"/></a></td><td><b>이동건</b><br/><sub>풀스택 · 인프라</sub></td><td>백엔드 코어 (인증·RBAC·REST API) · CI/CD</td></tr>
  <tr><td align="center"><a href="https://github.com/pinoba"><img src="https://github.com/pinoba.png" width="60px" alt="김병수"/></a></td><td><b>김병수</b><br/><sub>백엔드 · 문서 작성</sub></td><td>회전형 QR · 공지 첨부파일</td></tr>
  <tr><td align="center"><a href="https://github.com/dkflfkd-cmyk"><img src="https://github.com/dkflfkd-cmyk.png" width="60px" alt="정윤아"/></a></td><td><b>정윤아</b><br/><sub>프론트엔드</sub></td><td>직원용 앱 - 화면 디자인 · 인수인계 노트</td></tr>
  <tr><td align="center"><a href="https://github.com/rlacodms24-afk"><img src="https://github.com/rlacodms24-afk.png" width="60px" alt="김채은"/></a></td><td><b>김채은</b><br/><sub>프론트엔드</sub></td><td>사장용 앱 - 화면 디자인 · 인수인계 노트</td></tr>
</table>
