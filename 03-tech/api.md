# REST API 명세 — PTManager

HTTP 경로·인증·응답 계약을 다룬다(규칙: [기능 명세](../01-func/01-requirements.md), 검증: [테스트 명세](../04-test/01-verification.md)).

## 진입점과 원본

- Scalar(`/docs`)와 OpenAPI(`/v3/api-docs`)는 [백엔드 README](https://github.com/parttime-manager/backend#사용-방법)에 따라 접속하며, 원본은 [컨트롤러](https://github.com/parttime-manager/backend/tree/main/src/main/kotlin/com/ptmanager/backend/adapter/inbound/web)·[DTO](https://github.com/parttime-manager/backend/tree/main/src/main/kotlin/com/ptmanager/backend/application)·[보안/예외 처리기](https://github.com/parttime-manager/backend/tree/main/src/main/kotlin/com/ptmanager/backend)다.

## 인증과 응답

- 기본 경로는 `/api`다(`POST /api/auth/{signup,login,refresh}`, 가입 201·로그아웃 204). 보호 요청은 무세션 `Authorization: Bearer <accessToken>`을 쓰며 서버가 역할·매장을 검사한다.
- 성공 응답은 DTO·목록·페이지·빈 본문이고, 오류는 `ApiError(code, message, timestamp, fields)` 또는 인증 필터의 빈 본문이다.
- QR 발급은 `POST /api/workplaces/{workplaceId}/qr-token`만 허용한다(GET·HEAD는 405, 규칙은 [QR 출퇴근](../01-func/01-requirements.md#qr-출퇴근)).

## 클라이언트·시간 표현

- Android는 동시 401 갱신을 직렬화해 토큰을 재사용하며 600ms 간격 최대 3회 시도한다. 본문 있는 400·401은 세션을 종료하고, 네트워크·5xx·빈 400/401은 유지하며, 재시도 후 401이면 중단한다.
- 타임스탬프는 DB `TIMESTAMPTZ`에 저장하고 API는 UTC로 전달한다.
