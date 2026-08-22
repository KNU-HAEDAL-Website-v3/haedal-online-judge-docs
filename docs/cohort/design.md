# Cohort(분반) 수직 슬라이스 설계

> 상태: 확정 (2026-08-17) · 구현 완료 (BE `feat/cohort-slice`, 2026-08-17 - 구현 리뷰 반영분으로 1절·2절·3절·6절·7절 갱신) · 구현 레포: [haedal-online-judge-BE](https://github.com/KNU-HAEDAL-Website-v3/haedal-online-judge-BE)
> 기준 문서: [mvp-scope.md](../mvp-scope.md) · [permissions.md](../permissions.md) · [flows-and-usecases.md](../flows-and-usecases.md) · [decisions/3](../decisions/3-권한-모델-2층-구조.md) · [decisions/5](../decisions/5-세션-인증-채택-spring-security-보류.md) · BE `CLAUDE.md`
> 검증 기준 버전: Spring Boot 4.1.0 / Spring Framework 7.0 / springdoc 3.1.0 / Testcontainers 2.0.5

**이 문서를 읽는 법**

- BE의 첫 도메인 API
- 이후 모든 도메인(수강생 배정 → 과제 → 제출 → 대시보드): 이 문서의 4절 규약을 그대로 따르고, 코드는 `cohort/`·`enrollment/` 패키지를 복제해 작성
- 처음 합류 시 읽는 순서
  - 문서: 0절 → 4절 → 3절
  - 코드: `Cohort.java` → `CohortService.java` → `CohortController.java` → `AuthorizationInterceptor.java`

---

## 0. 이 슬라이스가 세우는 것

| # | 산출물 | 이후 슬라이스에서의 역할 |
|---|---|---|
| 1 | `cohort`, `enrollment` 패키지 - 첫 도메인 API | 모든 도메인이 복제할 **기준 패턴** |
| 2 | 권한 판정 공통 컴포넌트 (`@LoginOnly` / `@AdminOnly` / `@CohortRole`) + 기동 시 검증 | 이후 API는 어노테이션 한 줄 + 하위 리소스 스코프 조회 규약(4절) |
| 3 | springdoc(OpenAPI) | FE 계약의 기준 |
| 4 | 테스트 기반 (`support/`) | 이후 슬라이스는 `extends ApiTestSupport` 한 줄 |

- 기능 범위: 분반 CRUD(하드 삭제 없음, 보관) + 운영진 지정/해제 + 내 분반 목록
- **수강생 배정(UC-O2)은 다음 슬라이스** - Enrollment 엔티티·리포지토리·서비스 메서드는 이번에 작성, students 엔드포인트만 다음에 추가 (틀 복제 연습 겸)

## 1. 도메인 모델

### Cohort (분반)
| 필드 | 타입 | 제약 |
|---|---|---|
| id | Long | PK, IDENTITY |
| name | String | not null, ≤100. 자유 텍스트("2026-2 C언어"), unique 아님 |
| description | String | nullable, TEXT |
| status | `CohortStatus {ACTIVE, ARCHIVED}` | not null, `@Enumerated(STRING)` |
| createdAt | Instant | not null, updatable=false |

- 생성: `Cohort.create(name, description)`. 상태 변경은 도메인 메서드만 - `update(name, description)`, `archive()`, `restore()`, `isArchived()`, `ensureActive()`. setter 없음
- **보관(ARCHIVED) = 얼어붙은 분반**
  - 소속자 열람은 유지, 쓰기(운영진 지정·과제 등록·제출 등 분반 스코프의 모든 변경)는 누구도 불가 → `409 COHORT_ARCHIVED`
  - ADMIN은 `restore` 후 변경
  - permissions.md 1절 "보관 시 OPERATOR 효력 상실"의 구현 (8절 결정 3)
- 넣지 않는 것: startDate/endDate/주차 수(UI 시안 "진행 2주차/총 8주"는 시안 미확정 + Assignment.week 도입 시 재검토), term 필드(name에 포함), 하드 삭제

### Enrollment (소속) - 별도 패키지 `enrollment`
| 필드 | 타입 | 제약 |
|---|---|---|
| id | Long | PK |
| cohort | `@ManyToOne(LAZY)` Cohort | not null |
| user | `@ManyToOne(LAZY)` User | not null |
| role | `EnrollmentRole {OPERATOR, STUDENT}` | not null, STRING |
| createdAt | Instant | not null |

- **unique (cohort_id, user_id)** - 한 사람은 한 분반에서 역할 하나
- `Enrollment.create(cohort, user, role)`, `promoteToOperator()`. `EnrollmentRole.satisfies(required)`: OPERATOR ⊇ STUDENT
- Enrollment = **하드 삭제되는 관계 테이블, 다른 엔티티의 FK 대상 아님** - Submission은 (assignment_id, user_id)로 사용자 직접 참조, 대시보드의 행은 현재 Enrollment(STUDENT)에서 생성
- Repository
  - `findByCohortIdAndUserId`(권한 판정용, 연관 안 건드림) · `findAllByUserIdWithCohort`(fetch join cohort) · `findAllByCohortIdWithUser`(fetch join user, 운영진 먼저·등록순) · `findAllByCohortIdInWithUser`(fetch join user - 목록 응답 조립용 1회 쿼리)
  - fetch join 조회는 이름 끝에 `WithXxx` + 반드시 `@Query`
  - `/me/cohorts` 정렬(ACTIVE 먼저 → 최신순): 서비스에서 `Comparator.comparing(Cohort::isArchived).thenComparing(createdAt desc)`

### User 보강
- `UserService.findOrCreateMember(loginId)` 추출 → `StubAuthService`와 `EnrollmentService`가 공용
  - 이유: 아직 로그인 안 한 부원도 loginId로 배정 가능해야 함. 이름은 loginId로 임시, 홈페이지 연동 시 갱신
  - `StubAuthService`는 UserRepository 대신 UserService만 주입
  - loginId는 1~50자(`users.login_id varchar(50)`) - 요청 DTO는 `@Size(max=50)`, 경로 변수로 들어오는 loginId는 `findOrCreateMember`가 한 번 더 차단(400)
- 기존 `auth/dto/UserResponse` → `user/dto/UserResponse`로 이동(본인 정보·명부용, loginId·globalRole 포함)
- 타인에게 공개 가능한 최소 정보 = `user/dto/UserSummary {id, name}` - 학생 화면의 운영진 목록이 사용

## 2. 권한 판정 공통 컴포넌트 - `auth/authorization/`

- permissions.md 2절의 4단계를 **어노테이션 + 인터셉터**로 구현 - AOP 의존성 없음
- 기존 `AuthInterceptor`(로그인 여부, default-deny)와 같은 메커니즘, 그다음 순서로 등록
- 고위험 영역 - PM 담당

### 어노테이션 3종 - `/api/**` 핸들러는 반드시 하나 부착
```java
@LoginOnly                              // 로그인만 (예: GET /api/me/cohorts, GET /api/auth/me)
@AdminOnly                              // 전역 ADMIN만
@CohortRole(EnrollmentRole.OPERATOR)    // 경로 {cohortId} 분반의 OPERATOR 이상 (ADMIN 자동 통과)
@CohortRole(EnrollmentRole.STUDENT)     // 그 분반 소속이면 누구나 (ADMIN 자동 통과)
```
- 공개 경로(`/api/auth/login`, `/api/auth/logout`, `/api/health`): `AuthPaths.PUBLIC` 상수 하나에 집약 - WebConfig의 `excludePathPatterns`와 아래 검증기가 이 상수를 공유

**유효 어노테이션은 하나 (`AuthorizationAnnotations.resolve`)**

- 위치 우선: 메서드에 있으면 메서드 것, 없으면 클래스 것 - 클래스 `@LoginOnly` + 메서드 `@AdminOnly`면 AdminOnly
- 한 위치에 둘 이상 부착 → 기동 실패
- 이유: 종류별로 따로 찾아 "LoginOnly가 있으면 통과" 식으로 판정하면 클래스 LoginOnly가 메서드 AdminOnly를 덮어 권한 누수 → 인터셉터와 검증기가 같은 규칙 하나를 사용

### `AuthorizationInterceptor.preHandle`
1. CORS preflight와 일반 OPTIONS(Spring의 투명 처리, Allow 헤더만 응답) → 통과. `handler`가 `HandlerMethod`가 아니면(정적/에러) → 통과
2. 유효 어노테이션 하나 선택 (`AuthorizationAnnotations.resolve`) - **없거나 한 위치에 둘 이상이면 `IllegalStateException`(500)**: 기동 검증을 못 거친 컨텍스트에서도 fail-closed
3. `@CohortRole`이면 URI 템플릿 변수 `cohortId`를 **ADMIN 판정보다 먼저** 읽음 (`HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE`) - 없으면 설정 오류 → 500. `Long.parseLong` 실패 → `InvalidInputException` → 400 `INVALID_INPUT` (인터셉터가 `@PathVariable` 바인딩보다 먼저 돌므로 직접 처리)
4. 세션 userId로 User 로드 (없으면 401)
5. `@LoginOnly` → 통과. `@AdminOnly` → `user.isAdmin()` 아니면 403
6. `@CohortRole(required)` → `CohortAuthorizer.isAllowed(user, cohortId, required)`가 false면 403 (ADMIN은 안에서 통과)

- 인터셉터는 **Cohort 미로드·HTTP 메서드 미확인** - 4단계 판정만 수행. 보관 여부는 도메인 규칙(1절)이지 권한이 아님
- request attribute로 User를 넘기는 최적화는 미채택 (`@LoginUser` 리졸버는 현행대로 재조회)

### `CohortAuthorizer` - 판정 규칙의 단일 출처
```java
Optional<EnrollmentRole> roleOf(User user, Long cohortId)              // 소속 역할. 비소속(ADMIN 포함)이면 empty
boolean isAllowed(User user, Long cohortId, EnrollmentRole required)   // 인터셉터가 씀: ADMIN || roleOf.satisfies(required)
boolean canManage(User user, Cohort cohort, EnrollmentRole myRoleOrNull) // 응답 조립이 씀: cohort ACTIVE && (ADMIN || OPERATOR)
```
- 인터셉터가 Cohort를 로드하지 않으므로(순수 4단계) `canManage`는 이미 로드된 Cohort를 받는 별도 메서드
- 두 규칙 모두 이 클래스에만 존재 - 어느 쪽을 바꿔도 이 클래스만 수정
- `canManage` = FE의 "운영 기능 진입 버튼" 판정값

### `AuthorizationMappingValidator` - 기동 시 검증 (`SmartInitializingSingleton`)
`RequestMappingHandlerMapping.getHandlerMethods()`를 순회해 `/api/**` 핸들러 검사:
- (a) `AuthPaths.PUBLIC`이 아닌데 3종 어노테이션 없음 → 실패
- (b) `@CohortRole`인데 경로 패턴에 `{cohortId}` 없음 → 실패
- (c) 경로에 `{cohortId}`가 있는데 유효 어노테이션이 `@CohortRole`도 `@AdminOnly`도 아님 → 실패
- (d) 한 위치(메서드 또는 클래스)에 3종 중 둘 이상 → 실패
- (e) 우리 패키지(`kr.haedal.hoj`)의 컨트롤러가 `/api/` 밖에 매핑 → 실패 (prefix 오타로 인터셉터 2개를 통째로 비껴가는 것을 방지)

위반 처리: 위반 목록을 모아 `IllegalStateException` → 부팅 중단. "어노테이션 붙이는 걸 잊는" 실수를 컴파일 다음 단계에서 차단.

### 인터셉터가 보장하는 것 / 보장하지 않는 것
- 보장: 요청자가 `{cohortId}` 분반에서 요구 역할 이상
- **보장하지 않음: 경로의 하위 id(assignmentId 등)가 그 분반 소속인지 여부** - 서비스의 스코프 조회 규약(4절)이 담당
- OJ(P3, cohort_id NULL) 엔드포인트는 `@CohortRole` 미사용 - 풀이는 `@LoginOnly`, 출제는 `@AdminOnly`
- 경로에 cohortId가 없는 분반 스코프 리소스가 필요해지면 "assignmentId → cohort 조회" 리졸버를 같은 인터셉터에 추가하는 방식으로 확장 (P3까지 예약만)

## 3. API

공통: 응답 시각 UTC ISO-8601. 에러 `{code, message}`.

| code | HTTP | 언제 | FE 처리 |
|---|---|---|---|
| UNAUTHENTICATED | 401 | 기존 | 재로그인 유도 + 작성 내용 보존 |
| FORBIDDEN | 403 | 기존 | **홈 리다이렉트는 이 코드에만** |
| INVALID_INPUT | 400 | 기존(@Valid) + `@CohortRole` 경로의 `{cohortId}` 파싱 실패(인터셉터) + 그 외 경로/쿼리 파라미터 타입 불일치(`MethodArgumentTypeMismatchException`) + 깨진 JSON 본문 | 입력 오류 표시 |
| NOT_FOUND | 404 | 신규. `NotFoundException(message)` - 자원별 한국어 문구를 인자로. 하위 리소스 소속 불일치도 404(존재 비노출). 매핑되지 않은 경로도 404 | "찾을 수 없음" 안내 페이지 (홈으로 보내지 않음) |
| METHOD_NOT_ALLOWED | 405 | 지원하지 않는 HTTP 메서드 | 안내 |
| CONFLICT | 409 | 신규. `ConflictException(message)` - 역할 충돌 등 | 안내 |
| COHORT_ARCHIVED | 409 | 신규. `CohortArchivedException` - "보관된 분반은 변경할 수 없습니다. 보관을 해제한 뒤 다시 시도하세요." | 안내 (홈으로 보내지 않음). FE는 `status == ARCHIVED`면 쓰기 UI를 사전 비활성 |

| # | 메서드·경로 | 권한 | 컨트롤러 | 요청 | 응답 |
|---|---|---|---|---|---|
| 1 | `GET /api/me/cohorts` | @LoginOnly | Enrollment | - | `[CohortResponse]` 소속된 모든 분반(ACTIVE·ARCHIVED 모두, ACTIVE 먼저). 빈 배열 = Enrollment 없음 |
| 2 | `GET /api/cohorts?status=ACTIVE` | @AdminOnly | Cohort | status 선택(기본 ACTIVE) | `[CohortResponse]` |
| 3 | `POST /api/cohorts` | @AdminOnly | Cohort | `CohortCreateRequest {name, description?, operatorLoginIds: []}` | 201 + Location + `CohortResponse` |
| 4 | `GET /api/cohorts/{cohortId}` | @CohortRole(STUDENT) | Cohort | - | `CohortResponse` |
| 5 | `PUT /api/cohorts/{cohortId}` | @AdminOnly | Cohort | `CohortUpdateRequest {name, description?}` | 200 `CohortResponse`. 보관 중 → 409 COHORT_ARCHIVED |
| 6 | `POST /api/cohorts/{cohortId}/archive` | @AdminOnly | Cohort | - | 200 `CohortResponse` (멱등) |
| 7 | `POST /api/cohorts/{cohortId}/restore` | @AdminOnly | Cohort | - | 200 `CohortResponse` (멱등) |
| 8 | `GET /api/cohorts/{cohortId}/members` | @CohortRole(OPERATOR) | Enrollment | - | `[MemberResponse]` 전체 (역할 필터 없음). **STUDENT는 못 본다** - 학생에게 다른 사람 정보 비노출 |
| 9 | `PUT /api/cohorts/{cohortId}/operators/{loginId}` | @AdminOnly | Enrollment | - | 200 `MemberResponse`. 미소속 → OPERATOR로 생성(User find-or-create), STUDENT → 승격, 이미 OPERATOR → 그대로(멱등). 보관 중 → 409 |
| 10 | `DELETE /api/cohorts/{cohortId}/operators/{loginId}` | @AdminOnly | Enrollment | - | 204. **OPERATOR Enrollment만 삭제.** STUDENT·미소속·User 없음 → 404. 마지막 운영진 해제도 허용(ADMIN은 항상 운영자 이상). 보관 중 → 409 |

다음 슬라이스: `POST /api/cohorts/{cohortId}/students {loginIds: []}` (명단 붙여넣기, @CohortRole(OPERATOR)), `DELETE .../students/{loginId}`. 이미 OPERATOR인 loginId가 섞이면 → 409 CONFLICT (역할을 바꾸지 않음).

### DTO (record)
- `CohortResponse {id, name, description, status, createdAt, operators: [UserSummary{id, name, title}], studentCount: Integer|null, myRole: OPERATOR|STUDENT|null, myTitle, canManage: boolean}` - 목록·단건·생성·수정 응답 **전부 이 하나**
  - `operators`: **id·이름·직책 명칭만** - 학생이 볼 수 있는 유일한 타인 정보이므로 loginId·globalRole 미포함. FE: 운영진 이름은 다른 색으로 표시, 클릭 시 작은 팝업에 이름 + `title`. (PM 결정 2026-08-17)
  - **직책 명칭(`title`, `myTitle`) = 서버의 `RoleTitle` 한 곳에서 정한 문자열**: 전역 ADMIN → **해구르르**(임원단, 고정) / 분반 OPERATOR → **교육운영진**(고정) / 그 외 → **일반 수강생**(미확정, enum의 label 한 줄만 바꾸면 전체 반영). 판정 우선순위: ADMIN > OPERATOR > 나머지 - 임원이 운영진으로 소속돼 있어도 해구르르. FE는 문자열 그대로 표시, 자체 매핑 없음
  - `studentCount`: **요청자가 OPERATOR/ADMIN일 때만 값, STUDENT면 null**
  - `myRole`: 요청자의 Enrollment 역할(비소속 ADMIN만 null). `canManage`: 2절 정의. 두 필드는 `@Schema(description)`으로 판정 규칙을 OpenAPI에 명시
- `MemberResponse {user: UserResponse, role, title, enrolledAt}`
- `CohortCreateRequest {@NotBlank @Size(max=100) name, @Size(max=2000) description, List<@NotBlank String> operatorLoginIds}` / `CohortUpdateRequest {name, description}` (검증 어노테이션은 DTO 필드에만)
- 조립: `cohort/CohortResponseAssembler.toResponses(List<Cohort>, User viewer)` 하나
  - `findAllByCohortIdInWithUser(ids)` 1회로 operators/studentCount/myRole 채움, `CohortAuthorizer.canManage`로 canManage
  - 단건도 리스트 1개짜리로 같은 경로
  - 별도 컴포넌트인 이유: `CohortService`(생성 시 운영진 지정 → `EnrollmentService.assign`)와 `EnrollmentService`(`/api/me/cohorts` 응답)가 둘 다 사용 → 서비스끼리 주입하면 순환 발생
  - 자체 `@Transactional` 없음 - 호출한 서비스의 트랜잭션 안에서 실행
- 생성 응답의 계약 = **본문의 `id`**. `Location` 헤더는 REST 관례상 덧붙이는 것 (CORS `exposedHeaders` 없이는 FE에서 읽히지 않음 - 의존 금지)

### FE 화면 계약 메모
- **학생 홈**: `GET /api/me/cohorts` 결과를 status로 나눠 "현재 소속(ACTIVE)" / "지난 소속(ARCHIVED)" 두 섹션. 각 섹션은 꺾쇠로 접고 펼침, 현재 소속이 위. 펼쳤을 때 비어 있으면 표시 없음(별도 미소속 안내 문구 없음)
- **관리자 분반 관리**: `GET /api/cohorts`(기본 ACTIVE)와 `?status=ARCHIVED`로 "진행 중" / "보관" 두 섹션. 학생 홈과 같은 꺾쇠 패턴
- 학생 화면의 운영진 이름: 다른 색으로 표시, 클릭 시 작은 팝업에 이름 + 직책(`title`: 해구르르 / 교육운영진). 홈 카드의 내 배지는 `myTitle`. 그 이상의 정보(loginId 등)는 API가 미제공
- 학생 화면의 운영 기능 진입 버튼: `canManage`로만 분기. ADMIN 전용 액션(수정·보관·운영진 지정)은 관리자 화면 소관 - `/api/auth/me`의 `globalRole`로 판단

## 4. 정한 규약 - 이후 슬라이스가 그대로 따른다

**패키지·의존**
- 도메인 = 패키지 하나: `Entity, XxxStatus/Role, Repository, Service, Controller, dto/`. 예외 클래스는 `common/error`, 권한은 `auth/authorization`
- 컨트롤러는 **자기 패키지의 서비스만** 주입. 패키지 간 의존은 서비스/리포지토리 계층에서 단방향(cohort→enrollment: 응답 조립용 읽기 / enrollment→cohort: 존재·보관 확인). 순환 발생 시 설계 재검토
- URL 매핑: 공통 prefix가 있으면 class-level `@RequestMapping("/api/cohorts")`, 아니면 메서드에 전체 경로. `/api`처럼 넓은 prefix로 서로 다른 리소스를 한 컨트롤러에 모으지 않음

**URL·권한**
- 분반 스코프 리소스(수강생·과제·제출·현황판)는 **항상 `/api/cohorts/{cohortId}/...` 아래에 중첩** - `@CohortRole`이 이 경로변수를 읽으므로 flat URL(`/api/assignments/{id}`)로는 분반 권한 판정 불가. 사용자 스코프 목록은 `/api/me/...`. (flows 1.2절의 `POST /submissions`는 플로우차트 라벨이지 API 스펙이 아님 - 계약은 springdoc)
- **하위 리소스 스코프 조회**: 하위 id가 있으면 서비스에서 반드시 `findByIdAndCohortId(id, cohortId)`(손자는 체인 전부)로 경로의 cohortId와 함께 조회. `findById` 단독 호출 금지. 불일치·부재 → `NotFoundException` 404 (존재 비노출). 서비스 메서드는 `(Long cohortId, Long childId, ...)`로 cohortId를 첫 인자로 수신
- **학생 권한은 최소** - 학생은 자기 소속 분반의 과제 관련(+ 향후 Q&A·분반 공지)만 접근, 다른 사람의 정보(명부·타인 제출물·타인 상태)는 어떤 API로도 비노출. 새 API에 `@CohortRole(STUDENT)` 부착 시 응답에 타인 정보가 섞이지 않는지 확인
- `/{role}s/{loginId}` 하위 자원은 해당 role의 Enrollment만 처리: 삭제 시 role 불일치 → 404, 배정 시 다른 role로 이미 소속 → 409. 역할 변경(승격)은 ADMIN 전용 #9 한 경로뿐, 강등 경로 없음. `EnrollmentService.remove(cohortId, loginId, EnrollmentRole expected)` / `assign(cohortId, loginIds, EnrollmentRole role)` 시그니처가 이를 강제
- 분반 스코프 쓰기 서비스는 첫 줄에서 `cohort.ensureActive()`(보관이면 `CohortArchivedException`) - 권한 컴포넌트가 아니라 도메인 규칙
- 세션 쿠키: `SameSite=Lax`, `HttpOnly` (application.yml) - 다른 사이트의 `<form>` POST에 세션이 실리지 않아 CSRF 1차 방어. CSRF 토큰은 P1 미도입

**계층·트랜잭션**
- 컨트롤러 순서: 권한(어노테이션) → 검증(`@Valid`) → 서비스 호출 → **서비스가 돌려준 응답 DTO를 그대로 반환**
  - 컨트롤러는 엔티티 수신·DTO 변환 없음
  - 이유: `open-in-view: false`라 트랜잭션(=서비스) 밖에서 LAZY를 건드리면 `LazyInitializationException`; 여러 조회를 합치는 응답은 컨트롤러가 조립 불가
  - (기존 `AuthController`의 `UserResponse.from(user)`는 연관 없는 단일 엔티티라 예외 - 주석으로 명시)
- 서비스: 클래스 레벨 `@Transactional`, 조회 메서드만 `@Transactional(readOnly = true)`. 목록 조회는 fetch join으로 LAZY를 트랜잭션 안에서 종결
- 리포지토리: 연관을 fetch join 하는 조회는 이름 끝에 `WithXxx` + 반드시 `@Query`(없으면 Spring Data가 `With`를 속성으로 파싱해 기동 실패). enum 컬럼 `order by`는 STRING 매핑이라 알파벳순 - 의도한 순서인지 주석으로 명시
- 응답 DTO 정적 팩토리: 엔티티 하나면 `from(entity)`, 여러 값 조합이면 `of(...)`
- 컨트롤러 반환: 200은 DTO 직접 반환, 201은 `ResponseEntity.created(uri).body(dto)`, 204는 `void` + `@ResponseStatus(NO_CONTENT)`. 그 외 ResponseEntity 금지
- 상태 전이 = `POST /{id}/{동사}` 멱등. 수정 = PUT 전체 교체. 삭제 = 204, 대상 없으면 404. 목록 페이징 없음(P1), 정렬 createdAt desc (`/me/cohorts`만 status 우선)

**코드 스타일**
- Lombok 미사용 - 생성자 주입·getter·로거(`private static final Logger log = LoggerFactory.getLogger(X.class)`) 직접 작성
- 엔티티는 `User.java` 패턴: `protected` 기본 생성자, `private` 전체 생성자에서 `createdAt = Instant.now()`, 정적 팩토리로만 생성, setter 없음, enum은 `@Enumerated(STRING)`
- 예외: `NotFoundException/ConflictException`은 한국어 문구를 생성자 인자로, 핸들러가 `e.getMessage()`를 그대로 message에. Unauthenticated/Forbidden은 기존대로 핸들러 고정 문구

## 5. springdoc

- `implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:3.1.0'` (Boot BOM 관리 대상이 아니라 버전 명시. 3.x = Boot 4.x 라인)
- `/v3/api-docs`, `/swagger-ui/index.html` - `/api/**` 밖이라 인터셉터·검증기 미적용. 별도 `OpenApiConfig` 없음
- 컨트롤러 `@Tag("Cohort")`/`@Tag("Enrollment")`, 메서드 `@Operation(summary)`, 규칙이 있는 필드만 `@Schema(description)`. 최소한만 - 코드가 문서
- README: 기술 스택 표를 "Spring Boot 4.1 (Java 21)"로 갱신 + swagger 접속 방법 추가

## 6. 테스트

- 의존성 (버전 생략 - Boot 4.1 BOM이 관리): `spring-boot-testcontainers`, `org.testcontainers:testcontainers-postgresql` (**Testcontainers 2.x** - 패키지 `org.testcontainers.postgresql.PostgreSQLContainer`, 1.x의 `containers.*`는 deprecated). `testcontainers-junit-jupiter`는 불필요(@Bean 방식)
- `application.yml` 정리: jpa/jackson/session/logging은 프로필 없는 공통 문서로, `local`에는 datasource만. 테스트는 `@ActiveProfiles("test")`(→ LocalDataSeeder 미실행), datasource는 `@ServiceConnection`이 채움. `application-test.yml` 불필요
- `src/test/java/kr/haedal/hoj/support/` - 한 번만 만드는 파일(PM 담당, 주니어는 수정 안 함):
  - `PostgresContainerConfig` - `@TestConfiguration(proxyBeanMethods=false)` + `@Bean @ServiceConnection PostgreSQLContainer("postgres:16")`
  - `ApiTestSupport` - 추상 베이스: `@SpringBootTest @AutoConfigureMockMvc @ActiveProfiles("test") @Import(PostgresContainerConfig.class)`, `MockMvc`, `ObjectMapper`(Jackson 3 `tools.jackson`), `LoginHelper`, 리포지토리, `@AfterEach` `DatabaseCleaner.clean()`, 그리고 여러 슬라이스가 공용으로 쓰는 픽스처 `createCohort(name, operators...)`, `enrollStudent(cohortId, loginId)`(수강생 API 나오면 API 호출로 교체), `archiveCohort`, `restoreCohort`, JSON 유틸. **슬라이스 고유 픽스처(createAssignment 등)는 그 슬라이스 테스트 클래스의 private 헬퍼로** - support/는 PM 파일
  - `DatabaseCleaner` - `TRUNCATE ... RESTART IDENTITY CASCADE`. 테스트 `@Transactional` 롤백은 미채택(요청이 테스트 트랜잭션 안에서 돌아 open-in-view=false가 드러내야 할 LAZY 문제를 가림)
  - `LoginHelper` - `MockHttpSession as(User)` : `session.setAttribute(SessionConst.LOGIN_USER_ID, user.getId())` 직접 세팅 (실제 로그인 API 호출 없음 - 인증 교체와 무관). `admin()`, `member(loginId)`는 User 저장 후 세션 반환. 로그인 API 자체(세션 고정 방지 등)는 `AuthApiTest` 1개로 분리
- `HojApplicationTests` 삭제 (남기면 DB 없이 `./gradlew build`가 깨짐)
- MockMvc(`@AutoConfigureMockMvc`, 패키지 `org.springframework.boot.webmvc.test.autoconfigure`) 채택. RestTestClient/MockMvcTester는 이 슬라이스에서 미사용
- 테스트 파일은 컨트롤러와 1:1: `cohort/CohortApiTest`(#2~#7), `enrollment/EnrollmentApiTest`(#1, #8~#10), `auth/AuthApiTest`, `auth/authorization/AuthorizationMappingValidatorTest`(가짜 매핑으로 규칙 a~e + 위치 우선 해석 + 위반 시 기동 중단을 단위 테스트; 실제 컨텍스트 위반은 모든 API 테스트가 기동 실패로 알림)
- 케이스 (역할 × 엔드포인트 대표):
  - 미로그인 → 401 / MEMBER `POST /cohorts` → 403 / ADMIN → 201 + Location + operators 채워짐
  - 비소속 `GET /cohorts/{id}` → 403 / STUDENT → 200 myRole=STUDENT canManage=false **studentCount=null** / OPERATOR → 200 canManage=true studentCount 값 / 비소속 ADMIN → 200 myRole=null canManage=true
  - STUDENT `GET members` → 403 / OPERATOR → 200
  - `/api/cohorts/abc` → 400 INVALID_INPUT (MEMBER·ADMIN 세션 모두, `@AdminOnly` 경로도) / 없는 id: ADMIN → 404, MEMBER → 403 / **교차 분반**: 다른 반의 STUDENT·OPERATOR가 이 반 GET·members → 403, `/me/cohorts`는 남의 소속이 섞이지 않음
  - archive → ARCHIVED, restore → ACTIVE (멱등) / 보관 후 OPERATOR GET → 200 canManage=false / 보관 후 ADMIN `PUT operators`·`DELETE operators`·`PUT cohort` → 409 COHORT_ARCHIVED / restore 후 → 200 / PUT 수정은 재조회로 flush 확인
  - `PUT operators` 멱등 두 번 200 / STUDENT였던 사람 → OPERATOR 승격 / `DELETE operators` STUDENT loginId → 404, Enrollment 유지 / 없는 분반 → 404 / loginId 51자 → 400 / `assign` 다른 역할 충돌 → 409(서비스 레벨, API 도달은 다음 슬라이스) / 세션은 있는데 사용자 삭제 → 401 / 깨진 JSON → 400
  - `GET /me/cohorts`: 미소속 → [] / 소속 → myRole·operators 포함 / 보관 분반은 ARCHIVED로 뒤에 정렬
  - 다음 슬라이스 필수 케이스(지금은 문서에만): 분반 A OPERATOR가 `/api/cohorts/A/assignments/{B의 과제}` → 404
- 시더(`LocalDataSeeder`, local 전용) 확장: 샘플 분반 1개(ACTIVE, operator1 + student1~3) + 보관 분반 1개(student1 소속) → FE가 바로 붙을 수 있게. **테스트는 시더 비의존** - 픽스처는 각 테스트가 LoginHelper/서비스로 생성

## 7. 파일 목록

```
src/main/java/kr/haedal/hoj/
├─ cohort/      Cohort, CohortStatus, CohortRepository, CohortService, CohortController, CohortResponseAssembler
│               dto/ CohortCreateRequest, CohortUpdateRequest, CohortResponse
├─ enrollment/  Enrollment, EnrollmentRole, EnrollmentRepository, EnrollmentService, EnrollmentController
│               dto/ MemberResponse
├─ user/        UserService (신규: findOrCreateMember), dto/UserResponse (auth에서 이동)
├─ auth/        AuthPaths (신규), StubAuthService(UserService 사용으로 수정), AuthController(@LoginOnly 부착)
│  └─ authorization/  LoginOnly, AdminOnly, CohortRole, AuthorizationAnnotations(유효 어노테이션 해석), CohortAuthorizer, AuthorizationInterceptor, AuthorizationMappingValidator
├─ common/error/  NotFoundException, ConflictException, CohortArchivedException, InvalidInputException (+ GlobalExceptionHandler: 위 4개 + 타입 불일치·깨진 JSON·405·미매핑 경로 핸들러)
├─ common/config/ WebConfig(인터셉터 등록·AuthPaths 사용), LocalDataSeeder(샘플 분반)
src/main/resources/application.yml (공통/local 분리, 세션 쿠키 SameSite=Lax·HttpOnly)
src/test/java/kr/haedal/hoj/
├─ support/     PostgresContainerConfig, ApiTestSupport, DatabaseCleaner, LoginHelper
├─ cohort/CohortApiTest, enrollment/EnrollmentApiTest, auth/AuthApiTest, auth/authorization/AuthorizationMappingValidatorTest
build.gradle: springdoc 3.1.0, spring-boot-testcontainers, testcontainers-postgresql
README.md: 스택 표 갱신, swagger 안내
```

## 8. 결정 사항 (BE PR 본문에 그대로 옮긴다)

1. **Enrollment은 별도 패키지.** 다음 슬라이스(학생 배정)의 홈이며, 권한 컴포넌트의 의존 대상이 명확해짐.
2. **권한은 어노테이션 + 인터셉터, fail-closed + 기동 검증.** AOP 없음. `/api/**` 핸들러는 3종 중 하나 필수. `{cohortId}`와 `@CohortRole`의 대응을 부팅 시 검증.
3. **보관 분반 = 얼어붙음(도메인 규칙, 409 COHORT_ARCHIVED).** 인터셉터는 permissions.md 4단계만 구현, 보관은 모름. 소속자 열람은 유지, 변경은 restore 후. permissions.md 1절·2절에 동기화(같은 커밋). 기각 대안: ①인터셉터에서 GET 외 403 - HTTP 동사에 인가를 결합, FE 403→홈 규칙과 충돌 / ②소속자에게 완전 비노출 - 과거 과제·제출물 열람 필요.
4. **운영진 지정은 loginId + find-or-create User.** 개강 전 세팅 시 아직 로그인 안 한 사람도 배정 가능해야 함. 홈페이지 명부 API 연동 시 존재 검증 추가.
5. **분반 하드 삭제 API 없음.** flows 3절-8 소프트 삭제 원칙과 일관. *(갱신 2026-08-19: 보관이 기본인 것은 유지하되, ADMIN 전용 영구 삭제를 최후 수단으로 추가하기로 결정 - 소속·과제·제출물·파일 연쇄 삭제, 분반 이름 입력 확인. 별도 슬라이스에서 구현. docs/db/schema.md 1절 결정 5)*
6. **분반 스코프 리소스는 URL 중첩 + 스코프 조회 필수** (4절). 이번 슬라이스 코드에는 하위 리소스가 없지만 다음 슬라이스가 IDOR를 복제하지 않도록 지금 규약으로 못 박음.
7. **역할별 하위 자원은 그 역할만 처리.** DELETE operators는 OPERATOR만, 승격은 #9 한 경로.
8. **응답 DTO는 `CohortResponse` 하나 + `canManage`.** 상세 전용 필드가 생기는 슬라이스에서 `CohortDetailResponse`를 분리.
9. **Enrollment은 FK 대상 아님** - Submission은 user_id 직접 참조.
10. **학생 권한 최소** - 명부 불가, `studentCount` 비노출, 운영진 이름만 공개. (PM 결정 2026-08-17)
11. **404 vs 403** - 하위 리소스 소속 불일치는 404(존재 비노출). FE는 403만 홈 리다이렉트, 404는 안내 페이지. FE CLAUDE.md 규칙 3에 반영 필요. (PM 결정 2026-08-17)
12. **관리자 목록 기본 ACTIVE**, 보관은 `?status=ARCHIVED`로 별도 섹션. (PM 결정 2026-08-17)
13. 목록 페이징 없음. 시더 확장(local 전용).
14. **권한 어노테이션은 위치 우선 단일 유효** - 메서드 > 클래스, 한 위치에 둘 이상 금지(기동 실패). 우리 컨트롤러는 `/api/` 아래에만(기동 검증). 종류별 OR 판정은 클래스 `@LoginOnly`가 메서드 `@AdminOnly`를 덮는 fail-open이라 기각. (구현 리뷰 2026-08-17)
15. 세션 쿠키 `SameSite=Lax`(CSRF 1차 방어), CSRF 토큰은 P1 미도입.
16. **학생에게 내려가는 운영진은 `UserSummary {id, name, title}`** - 학생이 운영진의 loginId·전역 역할을 알 필요가 없으며 보안상으로도 비노출(PM, 2026-08-17). FE는 이름을 다른 색으로, 클릭 시 팝업(이름 + title).
17. **직책 명칭은 서버 `RoleTitle` 단일 출처** - 해구르르(임원단)·교육운영진은 고정, 그 외("일반 수강생")는 미확정이라 enum label 한 줄로 변경 가능. API는 문자열을 내려주고 FE는 매핑 미보유. (PM, 2026-08-17)

## 9. 후속 작업

- [x] BE: `feat/cohort-slice` 브랜치에서 구현 → PR (본문에 8절 결정 사항)
- [x] docs: permissions.md 1절·2절 보관 규칙 문구 동기화
- [ ] FE: CLAUDE.md 규칙 3에 "404는 안내 페이지, 403만 홈" 보강 - FE 로그인/홈 슬라이스에서
- [ ] 다음 BE 슬라이스: 수강생 배정 (`students` 엔드포인트, 이 문서 3절·4절 규약대로)
