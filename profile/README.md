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

두 앱은 단일 REST API와 DB를 공유하며 `android-app/`의 `employee/`·`employer/`·`common/`으로 구성합니다. 서버가 역할·매장 소속을 검사하고 FCM·S3는 로컬에서 스텁을 씁니다. 백엔드(헥사고날)와 Android(MVVM·클린)의 경계는 [아키텍처 설계](../03-tech/01-architecture.md#아키텍처와-경계)를 따릅니다.

## 앱 화면

2026-11-11 수정본을 Pixel 3a / Android 15 에뮬레이터와 로컬 데모 데이터로 촬영했습니다(대타 요청·승인은 입력·상세 화면).

### 직원 · 출근과 대타 요청, 급여 확인

| 01. 오늘 근무·출근 | 02. 대타 요청 | 03. 근무·급여 통계 |
| :-------------------: | :-------------: | :-------------------: |
| <img src="../assets/captures/employee-home.png" width="240" alt="직원 오늘 근무·출근 화면"/> | <img src="../assets/captures/employee-swap-request.png" width="240" alt="직원 대타 요청 화면"/> | <img src="../assets/captures/employee-stats.png" width="240" alt="직원 근무·급여 통계 화면"/> |

### 사장 · 출근 인증과 근무 관리

| 01. 출근 QR | 02. 직원별 주간 스케줄 | 03. 대타 요청 승인 |
| :-----------: | :----------------------: | :------------------: |
| <img src="../assets/captures/employer-attendance-qr.png" width="240" alt="사장 출근 QR 화면"/> | <img src="../assets/captures/employer-schedule.png" width="240" alt="사장 직원별 주간 스케줄 화면"/> | <img src="../assets/captures/employer-swap-approval.png" width="240" alt="사장 대타 요청 승인 화면"/> |

## 팀 구성

<table>
  <tr><td align="center"><a href="https://github.com/ken-jeong"><img src="https://github.com/ken-jeong.png" width="60px" alt="정상겸"/></a></td><td><b>정상겸</b><br/><sub>팀장 · 풀스택</sub></td><td>양쪽 앱 화면/API 연동 전반</td></tr>
  <tr><td align="center"><a href="https://github.com/Movinggun-bit"><img src="https://github.com/Movinggun-bit.png" width="60px" alt="이동건"/></a></td><td><b>이동건</b><br/><sub>풀스택 · 인프라</sub></td><td>백엔드 코어 (인증·RBAC·REST API) · CI/CD</td></tr>
  <tr><td align="center"><a href="https://github.com/pinoba"><img src="https://github.com/pinoba.png" width="60px" alt="김병수"/></a></td><td><b>김병수</b><br/><sub>백엔드 · 문서 작성</sub></td><td>회전형 QR · 공지 첨부파일</td></tr>
  <tr><td align="center"><a href="https://github.com/dkflfkd-cmyk"><img src="https://github.com/dkflfkd-cmyk.png" width="60px" alt="정윤아"/></a></td><td><b>정윤아</b><br/><sub>프론트엔드</sub></td><td>직원용 앱 - 화면 디자인 · 인수인계 노트</td></tr>
  <tr><td align="center"><a href="https://github.com/rlacodms24-afk"><img src="https://github.com/rlacodms24-afk.png" width="60px" alt="김채은"/></a></td><td><b>김채은</b><br/><sub>프론트엔드</sub></td><td>사장용 앱 - 화면 디자인 · 인수인계 노트</td></tr>
</table>
