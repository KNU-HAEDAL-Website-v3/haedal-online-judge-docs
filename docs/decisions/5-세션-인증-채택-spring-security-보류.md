# 5. P1 인증 — 세션 + 인터셉터 채택, Spring Security 보류

- 날짜: 2026-08-14 · 상태: 확정

## 결정

- P1 인증: **HttpSession 쿠키 + HandlerInterceptor(default-deny) + @LoginUser 리졸버** 직접 구현
- Spring Security·JWT: P1 미사용
- 신원 확인: `AuthService` 인터페이스 뒤에 격리
  - 개발 중: `StubAuthService`(무조건 신뢰)
  - 홈페이지 연동 확정 시: 구현체만 교체

## 근거

1. **팀 학습 부하**
   - Spring Security의 필터체인·인증객체 모델: 학습 곡선 가파름
   - 백엔드 주니어가 코드를 "읽고 복제"할 수 있어야 함 — 시큐리티 도입 시 마법 상자 하나 추가
2. **HOJ 권한 모델과 불일치**
   - HOJ 권한의 핵심 = "분반 범위 역할"(Enrollment.role) — 시큐리티의 전역 Role 모델로 표현 불가
   - 커스텀 판정 컴포넌트가 어차피 필요 → 그 밑의 프레임워크는 단순할수록 유리
3. **JWT 대비 세션의 이점**
   - 단일 서버 → 세션 공유 문제 없음
   - 서버 측 무효화(강제 로그아웃) 즉시 가능
   - 토큰 만료/갱신 로직 통째로 불필요
   - localhost는 포트가 달라도 same-site → 개발 중 쿠키 문제 없음

## 감수하는 것

- 서버 재시작 시 전원 로그아웃(인메모리 세션) — 동아리 도구 규모에서 수용
- 새 경로 면제: WebConfig excludePathPatterns 한 곳에서만 관리

## 재검토 조건

- 서버 수평 확장
- 모바일 네이티브 클라이언트 등장
- 홈페이지 SSO가 JWT 발급 방식으로 확정
  - 대응: 토큰 검증 AuthService 구현 + 필요 시 세션 → 토큰 전환
