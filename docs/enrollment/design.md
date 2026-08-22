# Enrollment(소속) - 수강생 배정 슬라이스 설계

> 확정: 2026-08-20 (BE 이슈 #4, PR #5 예정)
> 기준 규약: [cohort/design.md](../cohort/design.md) 3절·4절 - 이 문서는 그 규약에 추가되는 결정만 기록
> 소속 도메인의 나머지(내 분반, 명부, 운영진 지정/해제): Cohort 슬라이스에서 구현·문서화 완료

## 1. 추가 범위

- 수강생 배정/제외 API 2개
- `EnrollmentService.assign/remove`: Cohort 슬라이스에서 구현 완료 → STUDENT로 호출하는 엔드포인트만 추가

| # | 메서드·경로 | 권한 | 성공 | 주요 실패 |
|---|---|---|---|---|
| 11 | `POST /api/cohorts/{cohortId}/students` `{loginIds: [..]}` | `@CohortRole(OPERATOR)` | 200 - **갱신된 명부 전체** (`GET /members`와 같은 모양) | 400(빈 목록·loginId 검증), 403, 404(분반), 409(이미 운영진인 loginId), 409 보관 |
| 12 | `DELETE /api/cohorts/{cohortId}/students/{loginId}` | `@CohortRole(OPERATOR)` | 204 | 404(미소속·운영진·모르는 loginId), 409 보관 |

## 2. 결정

1. **권한 = 운영진 이상** - permissions.md 4절: 수강생 배정·제외는 ADMIN(모든 반)·OPERATOR(자기 반). 운영진 지정/해제(ADMIN 전용)와 구분.
2. **POST 응답 = 갱신된 명부 전체** - 배정 직후 FE의 명부 재조회 불필요. 명부는 운영진 이상만 보는 응답이므로 `MemberResponse`(loginId 포함) 그대로 사용.
3. **빈 `loginIds` = 400** - "아무도 배정 안 함"은 요청할 이유가 없는 실수로 간주(`@NotEmpty`).
4. **멱등·충돌 규칙 = `assign` 그대로**: 중복 loginId는 한 번만, 없는 사람은 MEMBER로 선등록, 이미 STUDENT면 그대로, 이미 OPERATOR면 409(역할을 몰래 바꾸지 않음).
5. **DELETE = STUDENT 소속만 삭제** - 운영진 삭제는 `/operators/{loginId}`(ADMIN). 역할 불일치 → 404.
6. 보관 분반 쓰기 → 항상 409 `COHORT_ARCHIVED` (`ensureActive()` 규약).

## 3. 테스트

- `EnrollmentApiTest`: 역할×엔드포인트 상태코드 16개 추가
- `ApiTestSupport.enrollStudent`: 리포지토리 직접 저장 → **이 API 호출로 교체** (cohort/design.md 6절 예정 항목)
