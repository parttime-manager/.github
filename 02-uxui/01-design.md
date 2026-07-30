# 디자인 기준 — PTManager Android

[문서 안내](../README.md#관련-문서)

Opaque Liquid / Soft Relief 스타일의 native Android XML 디자인 기준이다.

- 표면·깊이: 불투명 배경 `#F2F4F6`, 카드 `#FFFFFF`(블러·투과·전체 alpha 축소 없음). 상/좌 밝은 가장자리와 하/우 음영(오프셋 1dp/2dp, inset 0.5dp/하단 1dp, elevation 1dp, 라이트 `#EBEEF2` / 강조 `#E1E6EC`).
- 형태·간격: 카드 28dp, 입력·주요 버튼 18dp, 중간 컴포넌트 20dp, 탐색 필(pill) 형태. 4dp 기준, 화면 여백 20dp, 섹션 간격 24dp, 터치 최소 48dp(주요 버튼 52dp).
- 텍스트·브랜드: 시스템 sans-serif(제목 22sp, 본문 `#191F28`, 보조 `#333D4B`, 설명 `#596579`). 두 앱 공통 Action Blue `#0066CC`(배경 `#F5F9FF`, 강조 카드 `#EDF4FD`, 선택 칩 `#EAF2FE`, 테두리 `#CFE2F8`, 브랜드 위 보조 `#E8F1FF`) 및 성공/경고/오류(삭제 빨강) 의미색 유지.
- 토큰·공통 리소스: 앱별 `res/values/colors.xml`, 공통 `common/res/layout/view_*` 및 `common/res/values/{styles,dimens}.xml`·`drawable`. 기존 ripple·disabled·loading을 유지한다.

## 공통 UX/UI와 탐색

- 5탭(홈·스케줄·소통·급여·내 정보, 상단 여백 28dp)과 화면 구조를 공유한다: 스케줄(주 이동·오늘 버튼; 직원 조회 / 사장 편성), 급여(월 선택 → 요약 카드 → 주차 차트), 소통(공지 → 인수인계 → 대타요청), 근무 상세(뒤로·제목 → 날짜·시간 → 상세 → 주요 작업), 목록(공통 카드·빈 상태; 멤버 관리는 사장 전용).
- 하단 탐색·고정 작업(`FloatingActionBar`, `FloatingActionPrimary`/`Secondary`): 최소 높이 80dp, 좌우 16dp, 하단 12dp + 시스템 안전 영역, 필 형태. 탐색은 흰색→`#F9FAFC`·elevation 2dp이며 스크롤 뷰가 바 뒤까지 확장된다. 고정 작업 바(대타 승인·지원, 프로필·시급 저장, 공지·인수인계 등록)는 패딩 12dp·elevation 8dp(내부 버튼 2dp)이며 작업 숨김 시 바도 숨긴다.
- 우선순위·복구: 요약 카드와 작업 진입을 분리하고, 빈 목록·검색 없음·실패·로딩마다 다음 행동을 제공한다. 동사형 버튼 라벨을 일관되게 쓰고 오류 시 입력값을 보존하며 중복 저장을 막는다.

## 검증

[Android 빌드](https://github.com/parttime-manager/android-app#시작하기)와 [테스트](https://github.com/parttime-manager/android-app#테스트)로 컴파일·Debug APK·회귀 검사를 확인하고, 간격·대비·안전 영역은 에뮬레이터·기기에서 확인한다.
