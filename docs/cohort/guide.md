# Cohort(분반) 코드 안내서 — 스프링을 몰라도 읽는 백엔드 한 바퀴

> 대상: HOJ 백엔드 코드를 처음 읽는 사람. Spring·JPA를 몰라도 따라올 수 있게 썼다.
> 목적: `cohort` 슬라이스 하나를 끝까지 이해하기. 이후 모든 도메인(과제·제출·현황판)은 이 슬라이스를 복제해서 만든다.
> 짝 문서: [design.md](design.md)는 "왜 이렇게 정했나"(결정 기록), 이 문서는 "코드가 실제로 어떻게 움직이나"(안내서).
> 기준 코드: BE `main` `d037cc4` (PR #2 머지, 2026-08-18).

---

## 0. 읽는 순서

1장에서 무엇을 만들었는지, 2장에서 데이터가 어떻게 생겼는지 본다.
3장에서 요청 하나가 들어와 나갈 때까지를 따라간다. 여기까지 읽으면 뼈대는 다 잡힌다.
4~7장은 권한·소속·로그인·설정을 각각 파고든다.
8~10장은 API 표, 테스트, 그리고 다음 도메인을 만들 때 복제하는 절차다.
11장은 파일 지도다. 코드를 열어 볼 때 옆에 두면 된다.
12장은 전체를 한 문단으로 줄인 것이다. 다 읽은 뒤 기억을 점검하는 용도다.

---

## 1. 기(起) — 무엇을 만들었나

### 1.1 문제

HOJ에는 앞으로 도메인이 여럿 붙는다. 분반, 과제, 제출, 현황판.
도메인마다 구조가 다르면 읽는 사람이 매번 새로 배워야 한다.
그래서 첫 도메인인 **분반(Cohort)** 을 만들면서 "이후 도메인이 그대로 따를 틀"까지 함께 세웠다.
분반은 "2026-2 C언어"처럼 학기·트랙 단위로 열리는 반이다. 모든 과제와 제출은 어떤 분반에 속한다. 그래서 첫 도메인으로 골랐다.

### 1.2 수직 슬라이스

한 도메인을 위에서 아래까지 세로로 한 줄 다 뚫는 것을 수직 슬라이스라 부른다.
Cohort 슬라이스에는 이 여섯 층이 다 들어 있다.

| 층 | 역할 | Cohort에서의 파일 |
|---|---|---|
| 엔티티 | DB 테이블과 1:1로 대응하는 자바 객체 | `Cohort`, `Enrollment` |
| 리포지토리 | DB에서 엔티티를 읽고 쓰는 창구 | `CohortRepository`, `EnrollmentRepository` |
| 서비스 | 업무 규칙. 트랜잭션의 경계 | `CohortService`, `EnrollmentService` |
| 컨트롤러 | HTTP 요청을 받아 서비스를 부르고 응답을 돌려줌 | `CohortController`, `EnrollmentController` |
| DTO | 요청·응답 전용 데이터 그릇 | `CohortCreateRequest`, `CohortResponse` 등 |
| 테스트 | 역할 × API → 상태코드를 검증 | `CohortApiTest`, `EnrollmentApiTest` |

여기에 "모든 도메인이 공유하는 바닥"이 함께 깔렸다.
권한 판정 장치(`auth/authorization/`), 예외를 HTTP 응답으로 바꾸는 장치(`common/error/`), 테스트 지원(`test/support/`)이다.

### 1.3 스프링을 처음 보는 사람을 위한 최소 용어

이 문서에 나오는 용어는 이것이 전부다. 모르면 여기로 돌아오면 된다.

- **어노테이션(`@Xxx`)**: 클래스·메서드·필드에 붙이는 표식. 스프링과 JPA는 이 표식을 읽고 동작을 결정한다. 예: `@RestController`가 붙은 클래스는 HTTP 요청을 받는다.
- **빈(Bean)과 주입(DI)**: `@Component`, `@Service`, `@Repository`, `@Controller` 계열이 붙은 클래스는 스프링이 객체를 하나 만들어 보관한다. 이것이 빈이다. 다른 클래스가 생성자 매개변수로 그 타입을 적어 두면 스프링이 알아서 넣어 준다. 이것이 주입이다. 우리 코드의 생성자들이 전부 그 용도다.
- **컨트롤러 / 서비스 / 리포지토리**: 위 표의 세 층. 컨트롤러는 얇고, 서비스가 규칙을 갖고, 리포지토리는 인터페이스만 선언하면 스프링 데이터 JPA가 구현을 만들어 준다.
- **JPA와 엔티티**: 자바 객체를 DB 행처럼 다루게 해 주는 표준. `@Entity`가 붙은 클래스가 테이블, 필드가 컬럼이다. 저장·조회 SQL을 직접 쓰지 않는다.
- **트랜잭션(`@Transactional`)**: "이 메서드 안의 DB 작업은 전부 성공하거나 전부 취소된다"는 묶음. 서비스 메서드에 건다. 예외가 나가면 자동으로 되돌린다(롤백).
- **영속성 컨텍스트와 지연 로딩(LAZY)**: 트랜잭션 안에서 JPA는 읽어 온 엔티티를 기억한다. 연관된 객체(예: 소속의 `user`)는 처음엔 껍데기만 두고 실제로 쓸 때 DB에서 가져온다. 이것이 지연 로딩이다. **트랜잭션 밖에서 껍데기를 건드리면 예외**가 난다.
- **인터셉터**: 컨트롤러에 도달하기 전에 요청을 가로채 검사하는 문지기. 여러 개를 순서대로 둘 수 있다.
- **DTO와 record**: 요청·응답에만 쓰는 데이터 묶음. 자바의 `record`는 필드와 생성자·getter가 자동으로 생기는 불변 클래스라 DTO에 딱 맞다.
- **세션**: 로그인 상태를 서버가 기억하는 방식. 브라우저는 쿠키로 세션 id만 들고 다닌다.

---

## 2. 승(承) — 데이터: 사람, 분반, 소속

### 2.1 테이블 세 개

```
users ──────< enrollments >────── cohorts
                (cohort_id, user_id) UNIQUE
                role: OPERATOR | STUDENT
```

- **users** — 사람. `loginId`(고유), `name`, `globalRole`(ADMIN | MEMBER), `createdAt`.
- **cohorts** — 분반. `name`, `description`, `status`(ACTIVE | ARCHIVED), `createdAt`.
- **enrollments** — 소속. "누가, 어느 분반에서, 무슨 역할인가." `cohort_id`, `user_id`, `role`(OPERATOR | STUDENT), `createdAt`. 같은 사람이 같은 분반에 두 번 들어갈 수 없다(유니크 제약).

### 2.2 역할이 두 층인 이유

역할이 두 군데 있다. 헷갈리기 쉬우니 먼저 못 박아 두자.

- `User.globalRole` — **사람에게** 붙는 전역 역할. ADMIN은 동아리 임원, MEMBER는 일반 부원.
- `Enrollment.role` — **사람–분반 관계에** 붙는 역할. 그래서 한 사람이 A반 운영진이면서 B반 수강생일 수 있다.

ADMIN은 어느 분반에도 소속되지 않아도 모든 분반을 볼 수 있고 관리할 수 있다.
반대로 MEMBER는 소속된 분반만, 그 안에서 자기 역할만큼만 할 수 있다.

### 2.3 화면용 직책 명칭 — `RoleTitle`

역할은 코드용 이름이고, 화면에는 동아리 용어를 보여 준다. 그 매핑이 `user/RoleTitle`이다.

| 누구 | 명칭 |
|---|---|
| 전역 ADMIN | 해구르르 (고정) |
| 분반 OPERATOR | 교육운영진 (고정) |
| 그 외 | 일반 수강생 (미확정 — enum의 label 한 줄만 고치면 전체 반영) |

우선순위는 ADMIN > OPERATOR > 나머지다. 임원이 어느 분반에 운영진으로 들어가 있어도 해구르르로 보인다.
API는 이 문자열을 그대로 내려 주고, 프론트는 자체 매핑 없이 표시만 한다.

### 2.4 Cohort 엔티티가 소속 목록을 갖지 않는 이유

`Cohort` 안에는 "이 분반의 소속 목록" 필드가 없다. `Enrollment` 쪽에서만 `cohort`와 `user`를 가리킨다.
소속이 필요하면 항상 `EnrollmentRepository`로 따로 조회한다.
양방향으로 엮어 두면 편해 보이지만, 조회 한 번에 소속 전체가 딸려 오거나(N+1), 서로 참조하다 꼬이는 문제가 처음부터 생긴다. 그래서 단방향만 둔다.

### 2.5 엔티티 작성 규약 (모든 도메인이 따른다)

`Cohort`를 열어 보면 이 모양이다. 이후 엔티티도 똑같이 쓴다.

- `protected` 기본 생성자 — JPA 규격이 요구한다. 외부에서 못 쓰게 `protected`.
- `private` 전체 생성자 + 정적 팩토리 `create(...)` — 만드는 방법을 한 곳으로.
- **setter 없음** — 상태 변경은 `update`, `archive`, `restore`, `promoteToOperator`처럼 이름 있는 메서드로만.
- enum은 `EnumType.STRING` — 순서 번호로 저장하면 enum 순서가 바뀔 때 데이터가 깨진다.
- 시각은 `Instant`(UTC) — KST 변환은 프론트 몫.
- 하드 삭제 없음 — 분반은 지우지 않고 `ARCHIVED`로 보낸다.

`Cohort`의 도메인 메서드는 이렇다.

- `update(name, description)` — 이름·설명 교체.
- `archive()` / `restore()` — 상태 전환. 이미 그 상태여도 예외 없음(멱등).
- `isActive()` / `isArchived()`.
- `ensureActive()` — 보관 상태면 `CohortArchivedException`을 던진다. 4.6절에서 다시 본다.

`Enrollment`의 도메인 메서드는 `promoteToOperator()`(STUDENT → OPERATOR)와 `isOperator()`뿐이다. 강등은 없다.

---

## 3. 승(承) — 요청 하나의 여행

가장 흔한 요청을 끝까지 따라가 보자.
**학생 `student1`이 자기 분반 상세를 본다: `GET /api/cohorts/1`.**

### 3.1 문지기 둘 — 인터셉터

`/api/**`로 오는 요청은 컨트롤러에 닿기 전에 인터셉터 두 개를 순서대로 지난다.
등록 위치는 `common/config/WebConfig.addInterceptors`. 예외 경로(`/api/auth/login`, `/api/auth/logout`, `/api/health`)는 `auth/AuthPaths.PUBLIC` 한 곳에만 적혀 있다.

**① `AuthInterceptor` — 로그인했나?**
세션에 `LOGIN_USER_ID`가 없으면 `UnauthenticatedException`. 하는 일은 그것뿐이다.
브라우저가 보내는 CORS 사전 요청(OPTIONS)은 세션이 없으니 통과시킨다.

**② `AuthorizationInterceptor` — 이 API에 무슨 권한이 필요하고, 이 사람이 갖고 있나?**
순서가 중요하다.

```
사전 요청/OPTIONS       → 통과 (데이터가 나가지 않는다)
컨트롤러 메서드가 아니면 → 통과 (정적 파일, 에러 페이지 등)
effective = 이 핸들러의 "유효 권한 어노테이션" 하나   ← 없으면 500 (붙이는 걸 잊은 것)
effective가 @CohortRole이면 → 경로의 {cohortId}를 먼저 읽는다 ("abc"면 400)
user = 세션의 id로 DB에서 조회
@LoginOnly  → 통과
@AdminOnly  → user.isAdmin() 아니면 403
@CohortRole → CohortAuthorizer.isAllowed(user, cohortId, 요구 역할) 아니면 403
```

우리 요청의 핸들러 `CohortController.get`에는 `@CohortRole(STUDENT)`가 붙어 있다.
그래서 cohortId=1을 읽고, student1을 로드하고, `isAllowed`를 묻는다.

`CohortAuthorizer.isAllowed`는 이렇게 판정한다.

```
ADMIN이면 true
아니면 enrollments에서 (cohort 1, student1) 한 줄을 찾는다
  있으면 → 그 role이 요구 역할을 만족하는가?
           OPERATOR.satisfies(STUDENT)=true, STUDENT.satisfies(STUDENT)=true, STUDENT.satisfies(OPERATOR)=false
  없으면 → false → 403
```

이것이 [permissions.md](../permissions.md) §2의 판정 4단계 ①로그인 ②ADMIN ③소속 ④역할 그대로다.
인터셉터는 **분반 엔티티를 읽지 않고, HTTP 메서드(GET/POST)도 보지 않는다.** 보관 여부는 권한이 아니라 서비스가 다루는 도메인 규칙이기 때문이다(4.6절).

### 3.2 컨트롤러 — 4줄 패턴

```java
@CohortRole(EnrollmentRole.STUDENT)
@GetMapping("/{cohortId}")
public CohortResponse get(@PathVariable Long cohortId, @LoginUser User me) {
    return cohortService.findOne(cohortId, me);
}
```

- `@PathVariable Long cohortId` — 경로의 "1"을 숫자로 바꿔 넣어 준다.
- `@LoginUser User me` — 우리가 만든 `LoginUserArgumentResolver`가 세션 id로 `User`를 조회해 넣어 준다. 컨트롤러가 세션을 직접 만질 일이 없다.
- 컨트롤러 규약: **권한 어노테이션 → (`@Valid`) → 서비스 호출 → 서비스가 준 DTO 반환.** 그 외 로직 없음. `CohortController`의 여섯 메서드가 전부 이 모양이다.

참고로 인터셉터도 `User`를 한 번 조회했다. 같은 요청에서 두 번 조회하는 셈인데, PK 조회라 비용은 무시할 수준이다. 알고는 있어야 한다.

### 3.3 서비스 — 트랜잭션이 열리는 곳

```java
@Transactional(readOnly = true)
public CohortResponse findOne(Long cohortId, User viewer) {
    return assembler.toResponse(requireCohort(cohortId), viewer);
}
```

- 클래스에 `@Transactional`, 조회 메서드에만 `readOnly = true`.
- `requireCohort` = `findById(...).orElseThrow(NotFoundException("분반을 찾을 수 없습니다."))`. 모든 서비스에 같은 이름의 private 메서드가 있다.
- 설정 `spring.jpa.open-in-view=false` 때문에 **트랜잭션은 서비스에서 끝난다.** 컨트롤러로 나간 엔티티의 지연 로딩 필드를 건드리면 예외다. 그래서 **서비스가 DTO까지 완성해서 돌려준다.** 이 규칙 하나가 구조 전체를 단순하게 만든다.

### 3.4 조립기 — 보는 사람에 따라 달라지는 응답

`CohortResponse`는 목록·단건·생성·수정 응답이 전부 같은 모양이다. 그런데 안에 든 값 몇 개는 **누가 보느냐**에 따라 다르다.
그 조립을 `cohort/CohortResponseAssembler`가 맡는다.

왜 서비스 안에 두지 않고 따로 뺐나. `CohortService`(분반 API)와 `EnrollmentService`(내 분반 목록) 둘 다 이 응답을 만들어야 한다. 서비스끼리 서로 주입하면 순환이 생긴다. 셋째 컴포넌트로 빼서 둘 다 그것을 쓰게 했다.

`toResponses(cohorts, viewer)`가 하는 일:

1. 분반 id 목록으로 소속을 **쿼리 한 번**에 가져온다(`findAllByCohortIdInWithUser`, user까지 함께). 분반 N개여도 쿼리는 1번이다.
2. 분반마다 채운다.
   - `operators` — OPERATOR 소속들을 `UserSummary{id, name, title}`로. loginId·globalRole은 싣지 않는다. 학생에게도 내려가는 응답이기 때문이다. 이름순 정렬.
   - `myRole` — 소속 중 `user.id == viewer.id`인 것의 role. 없으면 null(ADMIN이 남의 분반을 볼 때).
   - `myTitle` — `RoleTitle.of(viewer, myRole)`.
   - `canManage` — `CohortAuthorizer.canManage` = 분반이 ACTIVE이고 (ADMIN이거나 OPERATOR). 프론트는 이 값만 보고 운영 버튼을 보여 준다.
   - `studentCount` — ADMIN 또는 OPERATOR에게만 숫자. 학생에게는 null. 인원수도 타인 정보로 본다.

student1이 받는 응답은 대략 이렇다.

```json
{
  "id": 1, "name": "2026-2 C언어", "description": "…", "status": "ACTIVE", "createdAt": "2026-08-18T…Z",
  "operators": [{ "id": 2, "name": "operator1", "title": "교육운영진" }],
  "studentCount": null,
  "myRole": "STUDENT",
  "myTitle": "일반 수강생",
  "canManage": false
}
```

`RoleTitle`이 `"교육운영진"` 같은 문자열로 나가는 것은 enum에 `@JsonValue`를 붙였기 때문이다.

### 3.5 실패했을 때 — 예외 한 곳에서 응답으로

어디서 예외가 나든(인터셉터든 서비스든) `common/error/GlobalExceptionHandler`가 받아서 `{code, message}` 모양으로 바꾼다.
컨트롤러와 서비스는 예외를 **던지기만** 하고, 상태코드와 응답 모양은 여기서만 정한다.

| 예외 | HTTP | code | 메시지 |
|---|---|---|---|
| `UnauthenticatedException` | 401 | `UNAUTHENTICATED` | 고정 문구 |
| `ForbiddenException` | 403 | `FORBIDDEN` | 고정 문구 |
| `NotFoundException` | 404 | `NOT_FOUND` | 던진 쪽 문구 |
| `ConflictException` | 409 | `CONFLICT` | 던진 쪽 문구 |
| `CohortArchivedException` | 409 | `COHORT_ARCHIVED` | "보관된 분반은 변경할 수 없습니다…" |
| `InvalidInputException`, `@Valid` 실패, 타입 불일치, JSON 깨짐 | 400 | `INVALID_INPUT` | 필드명: 이유 |
| 지원하지 않는 메서드 | 405 | `METHOD_NOT_ALLOWED` | 고정 문구 |
| 없는 경로 | 404 | `NOT_FOUND` | "존재하지 않는 경로입니다." |
| 그 외 전부 | 500 | `INTERNAL_ERROR` | 고정 문구. 원인은 서버 로그에만 |

프론트 규칙: **403만 홈으로 보낸다.** 404는 안내 페이지, 401은 로그인으로.
401·403 메시지가 고정인 이유는 내부 사정을 노출하지 않기 위해서다.

여기까지가 뼈대다. 요청 하나가 문지기 둘 → 컨트롤러 → 서비스(트랜잭션) → 조립기 → JSON으로 흐르고, 실패하면 예외 핸들러가 받는다. 이 그림이 모든 API에 똑같이 적용된다.

---

## 4. 전(轉) — 권한 장치를 자세히

이 영역은 사고가 나면 "남의 반 과제가 새는" 곳이라 가장 신경 써서 만들었다. 수정은 PM이 맡는다.

### 4.1 어노테이션 세 개

`/api/**`의 모든 컨트롤러 메서드는 셋 중 **정확히 하나**를 단다.

| 어노테이션 | 뜻 |
|---|---|
| `@LoginOnly` | 로그인만 되어 있으면 된다. 분반과 무관한 API(내 정보, 내 분반 목록). |
| `@AdminOnly` | 전역 ADMIN만. |
| `@CohortRole(역할)` | 경로의 `{cohortId}` 분반에서 그 역할 **이상**인 사람만. ADMIN은 자동 통과. |

`@CohortRole(STUDENT)`는 "소속자 누구나", `@CohortRole(OPERATOR)`는 "운영진 이상"이다.

### 4.2 유효 어노테이션을 하나만 고르는 규칙 — `AuthorizationAnnotations`

어노테이션은 메서드에도, 클래스에도 붙일 수 있다. 둘 다 있으면 어느 것이 이기나.
규칙은 **위치 우선**이다. 메서드에 하나라도 있으면 메서드 것, 없으면 클래스 것.
같은 위치에 둘 이상 있으면 설정 오류로 본다.

이 규칙을 한 클래스에 모아 둔 이유가 있다.
초기 구현은 종류별로 따로 찾아 "`@LoginOnly`가 있으면 통과"식으로 판정했다. 그러면 클래스에 `@LoginOnly`, 메서드에 `@AdminOnly`를 단 경우 클래스 것이 먼저 걸려 **일반 부원이 관리자 API를 통과**했다. 리뷰에서 잡혀 "유효 어노테이션 하나를 먼저 고르고, 그것으로만 판정"하도록 바꿨다.
인터셉터(런타임)와 기동 검증기(부팅 시)가 같은 클래스를 써서 두 판정이 어긋날 수 없다.

### 4.3 판정 규칙의 단일 출처 — `CohortAuthorizer`

분반 권한 규칙은 이 클래스에만 있다. 두 개다.

- `isAllowed(user, cohortId, required)` — 요청을 통과시킬 것인가. ADMIN이면 참, 아니면 소속의 role이 요구 역할을 만족해야 참. 인터셉터가 쓴다.
- `canManage(user, cohort, myRoleOrNull)` — 운영 버튼을 보여 줄 것인가. 분반이 ACTIVE이고 (ADMIN 또는 OPERATOR)면 참. 조립기가 쓴다.

규칙을 바꿀 일이 생기면 여기만 고친다. 두 군데에 복제되어 서로 다르게 자라는 일을 막는다.

### 4.4 인터셉터가 보장하는 것과 보장하지 않는 것

보장하는 것: "요청자가 `{cohortId}` 분반에서 그 역할 이상이다."
보장하지 않는 것: 경로 뒤쪽의 하위 id(예: `assignmentId`)가 **그 분반 것인지**.
그래서 이후 슬라이스의 서비스는 반드시 `findByIdAndCohortId(assignmentId, cohortId)`처럼 분반으로 범위를 좁혀 조회한다. 다른 반 과제 id를 찍으면 존재 여부를 알려 주지 않고 404가 나야 한다.

### 4.5 부팅 때 잡는다 — `AuthorizationMappingValidator`

빈이 다 만들어진 직후(`SmartInitializingSingleton`) 모든 컨트롤러 매핑을 훑는다. 위반이 하나라도 있으면 **서버가 뜨지 않는다.**

- (a) `AuthPaths.PUBLIC`이 아닌 `/api/**` 핸들러에 어노테이션이 없다.
- (b) `@CohortRole`인데 경로에 `{cohortId}`가 없다.
- (c) 경로에 `{cohortId}`가 있는데 유효 어노테이션이 `@LoginOnly`다. 분반 자원을 로그인만으로 열면 안 된다.
- (d) 한 위치에 어노테이션이 둘 이상이다.
- (e) 우리 패키지(`kr.haedal.hoj`)의 컨트롤러가 `/api/` 밖에 매핑되어 있다. `/apo/...` 같은 오타로 인터셉터를 통째로 비껴가는 것을 막는다.

"어노테이션 붙이는 걸 잊는" 실수가 컴파일 다음 단계에서, 요청이 오기 전에 잡힌다. 런타임의 500은 2중 안전망이다.

### 4.6 보관은 권한이 아니라 규칙 — `ensureActive()`와 409

분반이 ARCHIVED면 누구도 바꿀 수 없다. ADMIN도 마찬가지다. 열람은 그대로 된다.
이것을 인터셉터에서 막지 않는 이유는 "권한이 없다(403)"가 아니라 "지금은 바꿀 수 없는 상태다(409)"이기 때문이다.
그래서 **분반 스코프의 쓰기 서비스 메서드는 첫 줄에서 `cohort.ensureActive()`를 부른다.**
`update`, `assign`, `promoteToOperator`, `remove`가 그렇다. `archive`/`restore` 자체는 부르지 않는다. 보관 해제를 하려면 보관 상태에서 호출해야 하니까.
프론트는 409 `COHORT_ARCHIVED`를 받으면 "보관된 분반입니다. 보관을 해제한 뒤 다시 시도하세요"를 보여 준다.

---

## 5. 전(轉) — 소속 다루기: `EnrollmentService`

소속은 별도 패키지 `enrollment/`에 있다. 서비스 메서드는 다섯 개다.

| 메서드 | 하는 일 | 규칙 |
|---|---|---|
| `findMyCohorts(me)` | 내가 소속된 분반 전부 | 보관 분반 포함. ACTIVE가 먼저, 그 안에서 최신순. 응답은 조립기로. |
| `findMembers(cohortId)` | 분반 명부 | 운영진 먼저, 그다음 등록순. 항목은 `MemberResponse{user(UserResponse), role, title, enrolledAt}` — 여기엔 loginId·globalRole이 있다. 운영진 이상만 부를 수 있어서다. |
| `assign(cohortId, loginIds, role)` | loginId 목록을 role로 일괄 소속 | `ensureActive()`. 중복 loginId 제거(입력 순서 유지). 없는 사람은 만든다(MEMBER). 같은 role로 이미 있으면 그대로(멱등). **다른 role로 있으면 409** — 역할을 몰래 바꾸지 않는다. |
| `promoteToOperator(cohortId, loginId)` | 운영진 지정 | `ensureActive()`. 미소속이면 OPERATOR로 소속시키고, STUDENT면 승격, 이미 OPERATOR면 그대로. 멱등. 승격 경로는 이것 하나. 강등 API는 없다(해제 후 다시 배정). |
| `remove(cohortId, loginId, expected)` | 소속 해제 | `ensureActive()`. `expected` 역할의 소속만 지운다. 대상이 없거나 역할이 다르면 404. 성공은 204. |

`assign`이 "없는 사람은 만든다"인 이유: 아직 HOJ에 로그인한 적 없는 부원도 loginId만으로 미리 등록할 수 있어야 한다. 그때 만들어진 `User`의 이름은 임시로 loginId와 같고, 나중에 홈페이지 연동 시 실제 이름으로 바뀐다. 이 find-or-create는 `UserService.findOrCreateMember`에 있고, 스텁 로그인과 소속 배정이 같이 쓴다. loginId가 비었거나 50자를 넘으면 400.

`CohortService.create`는 분반을 저장한 뒤 `enrollmentService.assign(..., OPERATOR)`를 부른다. 두 호출은 같은 트랜잭션 안이다. 운영진 지정이 409로 실패하면 **분반 생성도 함께 되돌아간다.**

다음 슬라이스인 수강생 배정은 이 서비스에 손댈 필요가 없다. `POST /api/cohorts/{cohortId}/students {loginIds}` → `assign(..., STUDENT)`, `DELETE .../students/{loginId}` → `remove(..., STUDENT)`. 컨트롤러 메서드 두 개면 끝난다.

리포지토리 이름 규약도 여기서 정해졌다. 연관을 함께 가져오는(fetch join) 조회는 이름 끝에 `WithXxx`를 붙이고 반드시 `@Query`를 단다. `@Query` 없이 이 이름을 쓰면 스프링 데이터가 "With"를 속성으로 오해해 기동에 실패한다.

---

## 6. 전(轉) — 로그인은 지금 스텁이다: `auth/`

P1의 인증 방식은 미확정이다(홈페이지 연동이 유력). 그래서 지금은 **가짜 문**을 달아 두고 나머지 흐름은 실물과 같게 만들었다.

- `AuthService` — 인터페이스. `User login(String loginId)`. "이 loginId가 진짜 그 사람인가"를 확인하는 경계다. **이 인터페이스 뒤편만 교체**하면 실제 인증으로 바뀐다. 컨트롤러·세션·나머지는 그대로다.
- `StubAuthService` — 지금의 구현. 검증 없이 `findOrCreateMember(loginId)`. 없으면 MEMBER로 만든다.
- `AuthController`
  - `POST /api/auth/login {loginId}` — 공개 경로. 로그인 전 세션이 있으면 버리고 새로 만든다(세션 고정 공격 방지). 세션에 **User 엔티티가 아니라 id만** 넣는다. 엔티티를 넣으면 나중에 이름·역할이 바뀌어도 낡은 사본을 믿게 되기 때문이다. 응답은 `UserResponse{id, loginId, name, globalRole}`.
  - `GET /api/auth/me` — `@LoginOnly`. 프론트가 앱 시작 시 로그인 상태·역할을 확인한다.
  - `POST /api/auth/logout` — 공개 경로. 세션이 이미 없어도 조용히 성공한다. 만료된 사용자가 로그아웃을 눌렀을 때 401을 보지 않게.
- `AuthPaths.PUBLIC` — 로그인 없이 되는 `/api` 경로의 **단 하나의 목록.** 인터셉터 제외 목록과 기동 검증기가 둘 다 이것을 본다.
- `SessionConst.LOGIN_USER_ID` — 세션 키 이름.
- `@LoginUser` + `LoginUserArgumentResolver` — 컨트롤러 매개변수에 붙이면 세션 id로 조회한 **최신** `User`가 들어온다. 세션은 살아 있는데 DB에서 사용자가 지워졌으면 401.

세션은 서버 메모리에 있고 12시간 유지된다. 서버를 재시작하면 전원 로그아웃되는데, P1에서는 감수하기로 했다.

---

## 7. 전(轉) — 설정, CORS, 시더, 문서화

`src/main/resources/application.yml`의 요점.

- `spring.profiles.active: local` — 아무 지정 없이 실행하면 local. 테스트는 `@ActiveProfiles("test")`로 덮어쓴다.
- `jpa.hibernate.ddl-auto: update` — 엔티티를 보고 테이블을 자동으로 맞춘다. 개발용이며, 첫 운영 배포 전에 Flyway 마이그레이션으로 바꾼다(알려진 부채).
- `jpa.open-in-view: false` — 3.3절의 그 설정.
- `jackson.time-zone: UTC` — 응답의 시각은 UTC ISO-8601.
- `server.servlet.session` — `timeout: 12h`, 쿠키 `same-site: lax`(다른 사이트의 form POST에 세션 쿠키가 실리지 않게, CSRF 1차 방어), `http-only: true`.
- `logging.level.org.hibernate.SQL: debug` — 실행되는 SQL이 콘솔에 보인다. 학습용.
- local 프로필의 datasource — docker compose의 PostgreSQL(`localhost:5432/hoj`). 진짜 비밀은 `.gitignore`된 `application-local.yml`에.

`common/config/WebConfig`는 세 가지를 한다.

- CORS — 개발 중 프론트(5173)와 백엔드(8080)의 출처가 다르다. `hoj.cors.allowed-origins`(기본 `http://localhost:5173`)를 허용하고 `allowCredentials(true)`. 세션 쿠키가 오가려면 자격증명 허용과 명시적 origin이 필요하다. 운영에서는 같은 도메인 아래 두면 CORS 자체가 사라진다.
- 인터셉터 2개 등록(3.1절).
- `LoginUserArgumentResolver` 등록.

`common/config/LocalDataSeeder`는 local 프로필에서만 돈다. `admin`(ADMIN) 계정이 없으면 만들고, 분반이 하나도 없으면 샘플 둘을 만든다.
"2026-2 C언어"(ACTIVE: operator1 + student1~3), "2026-1 파이썬"(ARCHIVED: student1). 계정은 전부 스텁 로그인으로 바로 들어갈 수 있다. 테스트에서는 돌지 않는다.

`springdoc-openapi`가 붙어 있다. 서버를 띄우고 `/swagger-ui/index.html`을 열면 API 목록과 DTO 설명(`@Schema`, `@Operation`)이 보인다. **프론트와의 계약은 이 화면이 기준**이다.

---

## 8. 결(結) — API 한 장

| 메서드·경로 | 누가 | 하는 일 | 성공 | 주요 실패 |
|---|---|---|---|---|
| `POST /api/auth/login` | 누구나 | 스텁 로그인 | 200 UserResponse | 400 |
| `POST /api/auth/logout` | 누구나 | 세션 파기 | 200 | — |
| `GET /api/auth/me` | 로그인 | 내 정보 | 200 UserResponse | 401 |
| `GET /api/me/cohorts` | 로그인 | 내 분반 목록(보관 포함, ACTIVE 먼저) | 200 [CohortResponse] | 401 |
| `GET /api/cohorts?status=ACTIVE` | ADMIN | 분반 목록(상태별, 기본 ACTIVE, 최신순) | 200 [CohortResponse] | 403 |
| `POST /api/cohorts` | ADMIN | 생성 + 운영진 동시 지정 | 201 + Location | 400, 409(운영진이 다른 역할로 소속) |
| `GET /api/cohorts/{cohortId}` | 소속자·ADMIN | 상세 | 200 CohortResponse | 403, 404 |
| `PUT /api/cohorts/{cohortId}` | ADMIN | 이름·설명 전체 교체 | 200 | 400, 404, 409 보관 |
| `POST /api/cohorts/{cohortId}/archive` | ADMIN | 보관(멱등) | 200 | 404 |
| `POST /api/cohorts/{cohortId}/restore` | ADMIN | 보관 해제(멱등) | 200 | 404 |
| `GET /api/cohorts/{cohortId}/members` | 운영진 이상·ADMIN | 명부 | 200 [MemberResponse] | 403(학생), 404 |
| `PUT /api/cohorts/{cohortId}/operators/{loginId}` | ADMIN | 운영진 지정(멱등, 승격 포함) | 200 MemberResponse | 400, 404, 409 보관 |
| `DELETE /api/cohorts/{cohortId}/operators/{loginId}` | ADMIN | 운영진 해제 | 204 | 404(미소속·수강생), 409 보관 |

DTO 요약.

- `CohortCreateRequest{name(필수, ≤100), description(≤2000), operatorLoginIds[](각 ≤50)}` — 검증 어노테이션은 요청 DTO 필드에만 단다.
- `CohortUpdateRequest{name, description}` — PUT 전체 교체.
- `CohortResponse{id, name, description, status, createdAt, operators[UserSummary], studentCount|null, myRole|null, myTitle, canManage}` — 3.4절.
- `UserSummary{id, name, title}` — 타인에게 보여도 되는 최소 정보.
- `UserResponse{id, loginId, name, globalRole}` — 본인, 또는 운영진 이상이 보는 명부에만.
- `MemberResponse{user: UserResponse, role, title, enrolledAt}`.

---

## 9. 결(結) — 테스트: 왜 실제 DB로 도는가

테스트는 53개, 전부 실제 PostgreSQL 위에서 돈다. `test/support/`의 네 파일이 바닥이다.

- `PostgresContainerConfig` — Testcontainers로 `postgres:16` 컨테이너를 테스트 전체에서 한 번 띄운다. `@ServiceConnection`이 접속 정보를 자동으로 채워서 test용 yml이 필요 없다.
- `ApiTestSupport` — 모든 API 테스트가 `extends` 하는 베이스. `@SpringBootTest` + MockMvc라 인터셉터·리졸버·예외 핸들러까지 실제 스택이 다 돈다. 픽스처 헬퍼(`createCohort`, `enrollStudent`, `archiveCohort`, `restoreCohort`)와 JSON 유틸이 있다.
- `LoginHelper` — "이 사람으로 로그인된 세션"을 만든다. 로그인 API를 부르지 않고 세션 속성을 직접 넣는다. 인증 방식이 바뀌어도 이 헬퍼는 그대로다. `login.admin()`, `login.member("student1")`.
- `DatabaseCleaner` — 매 테스트 후 모든 테이블을 `TRUNCATE … RESTART IDENTITY CASCADE`. 테스트 클래스에 `@Transactional`을 걸어 롤백하는 흔한 방식을 **일부러 쓰지 않는다.** 그렇게 하면 요청 전체가 테스트 트랜잭션 안에서 돌아 `open-in-view=false`가 드러내야 할 지연 로딩 예외가 테스트에서는 안 보이고 실제 서버에서만 터진다. 요청은 실제와 같이 자기 트랜잭션으로 돌게 두고 뒷정리만 한다.

테스트 파일은 넷이다.

- `cohort/CohortApiTest` — 분반 API. `@Nested`로 엔드포인트별 묶음, 메서드명은 한국어 시나리오(`미로그인이면_401`, `일반부원이면_403`, …). **역할 × 엔드포인트 → 상태코드**가 검증의 축이다.
- `enrollment/EnrollmentApiTest` — 내 분반, 명부, 운영진 지정/해제.
- `auth/AuthApiTest` — 실제 로그인/로그아웃/me/health 흐름.
- `auth/authorization/AuthorizationMappingValidatorTest` — 검증기 규칙 a~e를 가짜 매핑으로 단위 테스트.

이후 슬라이스의 테스트는 `CohortApiTest`의 구조를 그대로 복제한다.

---

## 10. 결(結) — 다음 도메인을 만들 때

과제(Assignment)를 예로 들면 절차는 이렇다. 권한 코드는 한 줄도 쓰지 않는다.

1. `docs/assignment/design.md`를 먼저 쓴다. 도메인마다 `docs/<domain>/` 폴더다.
2. 엔티티 `Assignment` — `Cohort`의 규약대로. `cohort`를 `@ManyToOne(LAZY)`로 가진다.
3. 리포지토리 — 분반 스코프 조회 `findByIdAndCohortId`를 반드시 둔다.
4. 서비스 — 클래스 `@Transactional`, `requireXxx`, 쓰기 첫 줄 `cohort.ensureActive()`, DTO 반환.
5. 컨트롤러 — 경로는 `/api/cohorts/{cohortId}/assignments/...` 로 **반드시 `{cohortId}` 아래에** 둔다. 메서드마다 어노테이션 하나. 학생이 보는 건 `@CohortRole(STUDENT)`, 운영진이 만드는 건 `@CohortRole(OPERATOR)`.
6. DTO — record, 요청 검증은 필드에, 응답은 `from()`/`of()`.
7. 테스트 — `extends ApiTestSupport`, 역할 × 엔드포인트 → 상태코드.
8. 서버를 띄운다. 검증기가 규약 위반을 알려 주면 고친다.

지켜야 할 규약을 한 번 더 모으면 이렇다.

- `/api/**` 핸들러에는 권한 어노테이션이 정확히 하나.
- 분반 스코프 자원은 `/api/cohorts/{cohortId}/...` 아래. 변수 이름은 `cohortId` 고정.
- 하위 자원 조회는 `findByIdAndCohortId`. 다른 반 것이면 404.
- 분반 스코프 쓰기는 `ensureActive()` 먼저. 보관이면 409.
- 서비스가 DTO를 돌려준다. 컨트롤러는 엔티티를 받지 않는다.
- 학생에게 타인 정보를 내려주지 않는다. 명부는 운영진 이상, 인원수도 마찬가지.
- 직책 명칭은 `RoleTitle` 한 곳. 프론트는 매핑을 갖지 않는다.
- Lombok 없음. record와 정적 팩토리로.
- 검증 어노테이션은 요청 DTO 필드에만.
- fetch join 조회는 `WithXxx` + `@Query`.
- 시각은 UTC `Instant`, enum은 STRING, 하드 삭제 없음.

---

## 11. 파일 지도

```
src/main/java/kr/haedal/hoj/
├─ HojApplication.java                    부팅 진입점
├─ user/
│  ├─ User.java                           사람. globalRole ADMIN|MEMBER. member()/admin() 팩토리, isAdmin()
│  ├─ GlobalRole.java                     ADMIN | MEMBER
│  ├─ RoleTitle.java                      화면 직책 명칭 해구르르/교육운영진/일반 수강생. @JsonValue
│  ├─ UserRepository.java                 findByLoginId
│  ├─ UserService.java                    findOrCreateMember(loginId) — 스텁 로그인·소속 배정 공용
│  └─ dto/UserResponse.java, UserSummary.java
├─ cohort/
│  ├─ Cohort.java                         엔티티. create/update/archive/restore/ensureActive
│  ├─ CohortStatus.java                   ACTIVE | ARCHIVED
│  ├─ CohortRepository.java               findAllByStatusOrderByCreatedAtDesc
│  ├─ CohortService.java                  findAll/findOne/create/update/archive/restore
│  ├─ CohortResponseAssembler.java        뷰어별 CohortResponse 조립(쿼리 1번)
│  ├─ CohortController.java               /api/cohorts
│  └─ dto/CohortCreateRequest, CohortUpdateRequest, CohortResponse
├─ enrollment/
│  ├─ Enrollment.java                     소속. unique(cohort_id,user_id). promoteToOperator/isOperator
│  ├─ EnrollmentRole.java                 OPERATOR | STUDENT. satisfies()
│  ├─ EnrollmentRepository.java           findByCohortIdAndUserId, …WithCohort, …WithUser, …InWithUser
│  ├─ EnrollmentService.java              findMyCohorts/findMembers/assign/promoteToOperator/remove
│  ├─ EnrollmentController.java           /api/me/cohorts, /api/cohorts/{cohortId}/members|operators
│  └─ dto/MemberResponse.java
├─ auth/
│  ├─ AuthService.java                    인터페이스 — 실제 인증으로 교체할 지점
│  ├─ StubAuthService.java                지금의 구현(검증 없음)
│  ├─ AuthController.java                 login/logout/me
│  ├─ AuthInterceptor.java                ① 로그인 여부
│  ├─ AuthPaths.java                      공개 경로 목록(단일 출처)
│  ├─ SessionConst.java                   세션 키
│  ├─ LoginUser.java, LoginUserArgumentResolver.java   @LoginUser 주입
│  ├─ dto/LoginRequest.java
│  └─ authorization/
│     ├─ LoginOnly.java, AdminOnly.java, CohortRole.java   어노테이션 3종
│     ├─ AuthorizationAnnotations.java    유효 어노테이션 하나 고르기(위치 우선)
│     ├─ AuthorizationInterceptor.java    ②③④ 판정
│     ├─ CohortAuthorizer.java            isAllowed / canManage — 규칙의 단일 출처
│     └─ AuthorizationMappingValidator.java   기동 시 규칙 a~e 검증
└─ common/
   ├─ HealthController.java               /api/health
   ├─ config/WebConfig.java               CORS, 인터셉터 2개, 리졸버
   ├─ config/LocalDataSeeder.java         local 전용 admin + 샘플 분반
   └─ error/                              ErrorResponse{code,message}, 예외 6종, GlobalExceptionHandler

src/test/java/kr/haedal/hoj/
├─ support/ApiTestSupport, LoginHelper, DatabaseCleaner, PostgresContainerConfig
├─ cohort/CohortApiTest
├─ enrollment/EnrollmentApiTest
└─ auth/AuthApiTest, auth/authorization/AuthorizationMappingValidatorTest
```

---

## 12. 한 문단 요약

사람(`users`)과 분반(`cohorts`)이 있고, 둘을 잇는 소속(`enrollments`)에 분반 안 역할이 붙는다.
`/api/**` 요청은 문지기 둘을 지난다. 첫째는 로그인 여부, 둘째는 핸들러에 붙은 어노테이션 하나(`@LoginOnly`/`@AdminOnly`/`@CohortRole`)를 보고 ADMIN·소속·역할 순으로 판정한다.
컨트롤러는 서비스를 부르기만 하고, 서비스는 트랜잭션 안에서 규칙(없으면 404, 보관이면 409, 충돌이면 409)을 적용해 DTO를 완성해 돌려준다.
응답은 보는 사람에 따라 달라지고, 학생에게는 타인 정보를 내려주지 않는다.
예외는 한 곳에서 `{code, message}`가 된다.
어노테이션을 잊으면 서버가 뜨지 않는다.
다음 도메인은 이 슬라이스를 복제해서 만들고, 권한 코드는 쓰지 않는다.
