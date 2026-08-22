# Enrollment(소속) — 수강생 배정 슬라이스 설계

> 확정: 2026-08-20 (BE 이슈 #4, PR #5 예정). 기준 규약: [cohort/design.md](../cohort/design.md) §3·§4 — 이 문서는 그 규약에 추가되는 결정만 적는다.
> 소속 도메인의 나머지(내 분반, 명부, 운영진 지정/해제)는 Cohort 슬라이스에서 이미 구현·문서화됐다.

## 1. 무엇을 추가하나

수강생 배정/제외 API 2개. `EnrollmentService.assign/remove`는 Cohort 슬라이스 때 이미 만들어져 있어, STUDENT로 호출하는 엔드포인트만 붙인다.

| # | 메서드·경로 | 권한 | 성공 | 주요 실패 |
|---|---|---|---|---|
| 11 | `POST /api/cohorts/{cohortId}/students` `{loginIds: [..]}` | `@CohortRole(OPERATOR)` | 200 — **갱신된 명부 전체** (`GET /members`와 같은 모양) | 400(빈 목록·loginId 검증), 403, 404(분반), 409(이미 운영진인 loginId), 409 보관 |
| 12 | `DELETE /api/cohorts/{cohortId}/students/{loginId}` | `@CohortRole(OPERATOR)` | 204 | 404(미소속·운영진·모르는 loginId), 409 보관 |

## 2. 결정

1. **권한은 운영진 이상** — permissions.md §4: 수강생 배정·제외는 ADMIN(모든 반)·OPERATOR(자기 반). 운영진 지정/해제(ADMIN 전용)와 다르다.
2. **POST 응답 = 갱신된 명부 전체.** FE가 배정 직후 명부를 다시 조회할 필요가 없도록. 명부는 운영진 이상만 보는 응답이라 `MemberResponse`(loginId 포함)를 그대로 쓴다.
3. **빈 `loginIds`는 400** — "아무도 배정 안 함"은 요청할 이유가 없으니 실수로 본다(`@NotEmpty`).
4. **멱등·충돌 규칙은 `assign` 그대로**: 중복 loginId는 한 번만, 없는 사람은 MEMBER로 선등록, 이미 STUDENT면 그대로, 이미 OPERATOR면 409(역할을 몰래 바꾸지 않는다).
5. **DELETE는 STUDENT 소속만** 지운다 — 운영진을 지우려면 `/operators/{loginId}`(ADMIN). 역할 불일치는 404.
6. 보관 분반 쓰기는 항상 409 `COHORT_ARCHIVED` (`ensureActive()` 규약).

## 3. 테스트

`EnrollmentApiTest`에 역할×엔드포인트 상태코드 16개 추가. `ApiTestSupport.enrollStudent`는 리포지토리 직접 저장에서 **이 API 호출로 교체** (cohort/design.md §6에 예정돼 있던 항목).
