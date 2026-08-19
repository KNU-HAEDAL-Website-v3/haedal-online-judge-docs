# HOJ P1 DB 스키마 (확정)

> 확정일: 2026-08-19 (PM 결정). 기준 코드: BE `main` `d037cc4` (users·cohorts·enrollments는 구현 완료, assignments·submissions는 이 문서가 설계 원본).
> 이 문서가 P1 스키마의 단일 출처다. 바꾸려면 이 문서에 결정을 기록하고, 코드·ERD(ERDCloud)를 함께 갱신한다.
> P1 바깥(자동 채점, QnA, 공지, 출석부)은 그리지 않는다 — 스코프에서 제외됐거나 미결이라 지금 그리면 추측이 된다.

## 1. 확정 결정 (2026-08-19)

| # | 결정 | 내용 |
|---|---|---|
| 1 | 재제출 = 이력 | 백준처럼 제출마다 행이 쌓인다. 수정·삭제 없음(append-only). 추후에도 모든 제출을 확인 가능. |
| 2 | 제출 상태 4가지 | 저장하지 않고 이력에서 계산한다. 미제출 / **제출**(초록: 마감 내 제출 있음) / **제출(추가)**(초록: 마감 내 제출 + 마감 후 재제출도 있음) / **지각**(주황: 마감 후 제출만 있음). |
| 3 | 제출 형식 = 본문 + 링크 | 본문(코드 붙여넣기 **또는** zip 업로드, 탭으로 택1) + 링크(선택, GitHub·배포 URL). **본문 또는 링크 최소 1개**여야 제출 가능. 과제별 제출 타입 지정은 폐기 — 모든 과제가 세 방식을 상시 허용. |
| 4 | 과제 삭제 = 경고 후 연쇄 | 운영진이 삭제 가능. "제출물 N건이 함께 삭제됩니다" 경고 → 제출 이력 + 서버의 zip 파일까지 삭제. |
| 5 | 분반 = 보관 기본 + 삭제 공존 | 학기 종료는 **보관**(기본 경로, 열람 유지 — 홈 "지난 소속"이 이 데이터를 쓴다). **영구 삭제**는 ADMIN 전용 최후 수단 — 소속·과제·제출물·파일 전부 연쇄 삭제, 분반 이름을 입력해야 확인되는 강한 경고. **users는 어떤 삭제에도 지워지지 않는다.** |

파일 저장은 P1은 서버 로컬 디스크. S3 등으로 바꿔도 `stored_path`에 키만 넣으면 되도록 열을 잡아 뒀다.

## 2. ERD — ERDCloud 가져오기용 SQL

ERDCloud의 SQL 가져오기는 MySQL 문법 기준이라 그 문법으로 쓴다. 실제 DB는 PostgreSQL 16 — 타입 대응은 §4.

```sql
-- =========================================================================
-- HOJ P1 DB 스키마 — 확정본 2026-08-19 (docs/db/schema.md)
-- ERDCloud 가져오기용 MySQL 문법. 실제 DB는 PostgreSQL 16.
-- =========================================================================

CREATE TABLE users (
    id          BIGINT      NOT NULL AUTO_INCREMENT COMMENT 'PK',
    login_id    VARCHAR(50) NOT NULL COMMENT '로그인 ID — 홈페이지 계정과 매칭하는 키. 아직 로그인 안 한 부원도 이 값으로 선등록됨',
    name        VARCHAR(50) NOT NULL COMMENT '표시 이름 — 스텁 로그인에서는 login_id와 동일, 홈페이지 연동 후 실제 이름으로 갱신',
    global_role VARCHAR(20) NOT NULL COMMENT '전역 역할 ADMIN(임원단=해구르르) | MEMBER(일반 부원) — enum은 STRING 저장',
    created_at  DATETIME(6) NOT NULL COMMENT '생성 시각(UTC) — KST 변환은 프론트 몫',
    PRIMARY KEY (id),
    UNIQUE KEY uk_users_login_id (login_id)
) COMMENT='사용자 — 신원의 원본은 홈페이지, HOJ는 login_id로 매칭하는 로컬 사본. 하드 삭제 없음. 분반 삭제에도 지워지지 않는다';

CREATE TABLE cohorts (
    id          BIGINT       NOT NULL AUTO_INCREMENT COMMENT 'PK',
    name        VARCHAR(100) NOT NULL COMMENT '분반 이름 (예: 2026-2 C언어)',
    description TEXT         NULL     COMMENT '설명 (선택)',
    status      VARCHAR(20)  NOT NULL COMMENT 'ACTIVE(진행 중) | ARCHIVED(보관 — 열람만 가능, 모든 변경은 409)',
    created_at  DATETIME(6)  NOT NULL COMMENT '생성 시각(UTC)',
    PRIMARY KEY (id)
) COMMENT='분반 — 학기·트랙 단위 반. 학기 종료는 보관(기본, 지난 소속에서 열람). ADMIN 전용 영구 삭제는 최후 수단 — 소속·과제·제출물·파일 연쇄 삭제, 분반 이름 입력 확인';

CREATE TABLE enrollments (
    id         BIGINT      NOT NULL AUTO_INCREMENT COMMENT 'PK',
    cohort_id  BIGINT      NOT NULL COMMENT 'FK → cohorts.id',
    user_id    BIGINT      NOT NULL COMMENT 'FK → users.id',
    role       VARCHAR(20) NOT NULL COMMENT '분반 안 역할 OPERATOR(교육운영진) | STUDENT(수강생). STUDENT→OPERATOR 승격만 있고 강등 없음',
    created_at DATETIME(6) NOT NULL COMMENT '소속 등록 시각(UTC)',
    PRIMARY KEY (id),
    UNIQUE KEY uk_enrollment_cohort_user (cohort_id, user_id),
    CONSTRAINT fk_enrollments_cohort FOREIGN KEY (cohort_id) REFERENCES cohorts (id),
    CONSTRAINT fk_enrollments_user   FOREIGN KEY (user_id)   REFERENCES users (id)
) COMMENT='소속 — 누가 어느 분반에서 무슨 역할인가. 한 사람은 한 분반에 역할 하나. 해제 시 하드 삭제되는 관계 테이블 — 다른 테이블이 FK로 참조하지 않는다';

CREATE TABLE assignments (
    id          BIGINT       NOT NULL AUTO_INCREMENT COMMENT 'PK',
    cohort_id   BIGINT       NOT NULL COMMENT 'FK → cohorts.id — 과제는 분반에 속한다. 조회는 항상 (id, cohort_id) 스코프(다른 반 과제면 404)',
    title       VARCHAR(200) NOT NULL COMMENT '과제 제목',
    description TEXT         NULL     COMMENT '과제 내용 — 문제 링크 포함 자유 텍스트',
    due_at      DATETIME(6)  NOT NULL COMMENT '마감 시각(UTC) — 운영진이 생성 시 설정. 마감 후 제출은 지각 표시(차단 아님). 마감을 수정하면 지각 판정도 새 마감 기준으로 다시 계산된다',
    created_at  DATETIME(6)  NOT NULL COMMENT '등록 시각(UTC)',
    PRIMARY KEY (id),
    KEY idx_assignments_cohort (cohort_id),
    CONSTRAINT fk_assignments_cohort FOREIGN KEY (cohort_id) REFERENCES cohorts (id)
) COMMENT='과제 — 등록·수정·삭제는 운영진 이상. 보관 분반에서는 쓰기 409. 삭제는 경고(제출물 N건 함께 삭제) 후 연쇄. 제출 타입 열 없음 — 모든 과제가 코드/zip/링크를 받는다';

CREATE TABLE submissions (
    id             BIGINT        NOT NULL AUTO_INCREMENT COMMENT 'PK',
    assignment_id  BIGINT        NOT NULL COMMENT 'FK → assignments.id',
    user_id        BIGINT        NOT NULL COMMENT 'FK → users.id — enrollment이 아니라 사용자 직접 참조. 소속이 해제돼도 제출물은 남는다 (design.md §1)',
    code_text      TEXT          NULL COMMENT '본문·코드 — 붙여넣은 코드 텍스트. 본문은 코드/파일 중 택1(서비스 강제)',
    file_name      VARCHAR(255)  NULL COMMENT '본문·파일 — 업로드한 zip의 원본 파일명',
    stored_path    VARCHAR(500)  NULL COMMENT '본문·파일 — 저장 위치 키. P1은 서버 로컬 디스크, S3 전환 시에도 이 열에 키만',
    file_size      BIGINT        NULL COMMENT '본문·파일 — 크기(byte). 용량 관리·삭제 경고 문구용',
    link_url       VARCHAR(2048) NULL COMMENT '링크 — GitHub·배포 URL 등(선택). 본문 또는 링크 최소 1개 NOT NULL (서비스 + CHECK 강제)',
    submitted_at   DATETIME(6)   NOT NULL COMMENT '제출 시각(UTC). due_at과 비교해 지각을 계산한다 — 저장하는 지각 플래그 없음',
    score          INT           NULL COMMENT '[P2 준비] 점수 — P1에서는 항상 NULL (mvp-scope §5)',
    mentor_comment TEXT          NULL COMMENT '[P2 준비] 멘토 코멘트 — P1에서는 항상 NULL',
    PRIMARY KEY (id),
    KEY idx_submissions_assignment_user (assignment_id, user_id, submitted_at),
    KEY idx_submissions_user (user_id),
    CONSTRAINT fk_submissions_assignment FOREIGN KEY (assignment_id) REFERENCES assignments (id),
    CONSTRAINT fk_submissions_user       FOREIGN KEY (user_id)       REFERENCES users (id)
) COMMENT='제출 이력 — 백준처럼 재제출마다 행 추가(수정·삭제 없음). 상태는 계산: 미제출/제출/제출(추가)/지각. 최신 제출 = submitted_at 최대 행';
```

## 3. 계산 규칙 (열로 저장하지 않는 것)

한 학생의 한 과제 제출 상태는 제출 이력과 `due_at`만으로 계산한다.

```
onTime = submitted_at ≤ due_at 인 제출 존재 여부
late   = submitted_at >  due_at 인 제출 존재 여부

없음            → 미제출
onTime && !late → 제출          (초록)
onTime && late  → 제출(추가)    (초록 계열 — 마감 내 제출 후 추가 제출함)
!onTime && late → 지각          (주황)
```

- 마감(`due_at`)을 수정하면 위 계산이 새 마감 기준으로 다시 이뤄진다. 마감 연장 시 지각이 제출로 바뀌는 것은 의도된 동작이다.
- 현황판의 미제출자 = 현재 `Enrollment(STUDENT)` 명단 − 제출자. 소속이 해제된 학생은 명단에서 빠지지만 제출 데이터는 남는다.
- "학생의 최신 제출" = 그 (assignment, user)에서 `submitted_at`이 가장 큰 행. `idx_submissions_assignment_user`가 이 조회를 받친다.

## 4. 실제 PostgreSQL 대응과 구현 메모

| ERD(MySQL 표기) | 실제 PostgreSQL (Hibernate 생성) |
|---|---|
| `BIGINT AUTO_INCREMENT` | `bigint GENERATED BY DEFAULT AS IDENTITY` |
| `DATETIME(6)` | `timestamptz(6)` — 자바 `Instant`(UTC) |
| `VARCHAR(n)` / `TEXT` / `INT` / `BIGINT` | 동일 |

구현 시 잊지 말 것:

- **연쇄 삭제는 DB가 아니라 서비스가 한다.** zip 파일이 DB 밖(디스크)에 있어서 DB `ON DELETE CASCADE`만으로는 파일이 남는다. FK는 기본(RESTRICT)으로 두고, 서비스가 파일 → submissions → assignments → enrollments → cohort 순서로 지운다. RESTRICT는 순서를 빼먹으면 에러로 알려 주는 안전망이 된다.
- **PostgreSQL은 FK에 인덱스를 자동으로 만들지 않는다** (MySQL과 다름). `idx_assignments_cohort`, `idx_submissions_*`는 엔티티에 `@Table(indexes = ...)`로 명시한다.
- **"본문 또는 링크 최소 1개"**: 서비스 검증 + DB CHECK(`code_text IS NOT NULL OR stored_path IS NOT NULL OR link_url IS NOT NULL`) 이중으로. CHECK는 Flyway 전환 때 넣는다. "본문은 코드/파일 중 택1"은 서비스 규칙으로만 강제한다(정책이 바뀔 수 있어 DB에 굳히지 않는다).
- 제약 이름 중 코드에 실제 명시된 것은 `uk_enrollment_cohort_user`뿐이다. 나머지는 지금 Hibernate가 무작위 이름으로 만든다 — Flyway 전환 때 이 문서의 이름으로 고정한다.
- `ddl-auto: update`는 열 삭제를 하지 않는다. 기존 개발 DB에 만들어졌던 열이 남아 있으면 무시하면 된다(운영 전 Flyway로 정리).

## 5. 관련 문서

- [cohort/design.md](../cohort/design.md) — Cohort 슬라이스 결정 기록 (구현 완료 부분의 원본)
- [cohort/cohort guide.md](../cohort/cohort%20guide.md) — 코드 안내서
- [permissions.md](../permissions.md) — 권한 판정 4단계
- [mvp-scope.md](../mvp-scope.md) — P1 기능 목록 (이 문서의 결정 3이 §4·§5·§6에 반영됨)
