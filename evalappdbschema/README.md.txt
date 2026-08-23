# Employee evaluation app — database schema

Default assumptions used below (swap out if your actual requirements differ):
- **Login**: admin pre-creates employee accounts (no public self-registration)
- **Auth**: session-based (Spring Security), not JWT
- **Visibility**: role-based (`EMPLOYEE` sees own evals, `MANAGER`/`ADMIN` see everyone's)

## Full schema

```sql
-- ========================================
-- EMPLOYEES / AUTH
-- ========================================
CREATE TABLE M_Employee (
    EmployeeId    SERIAL PRIMARY KEY,
    EmployeeName  VARCHAR(100) NOT NULL,
    Email         VARCHAR(150) NOT NULL UNIQUE,
    PasswordHash  VARCHAR(255) NOT NULL,
    Role          VARCHAR(20)  NOT NULL DEFAULT 'EMPLOYEE',  -- EMPLOYEE, MANAGER, ADMIN
    IsActive      BOOLEAN      NOT NULL DEFAULT TRUE,
    CreatedAt     TIMESTAMP    NOT NULL DEFAULT now()
);

-- ========================================
-- EVAL PERIODS (unchanged from original)
-- ========================================
CREATE TABLE M_EvalPeriod (
    EvalPeriodId    SERIAL PRIMARY KEY,
    EvalPeriodName  VARCHAR(100) NOT NULL,   -- "2026 Summer Bonus", "2026 Annual"
    StartDate       DATE,
    EndDate         DATE,
    IsActive        BOOLEAN NOT NULL DEFAULT TRUE
);

-- ========================================
-- EVAL SECTIONS (new — the "blocks" within a period)
-- ========================================
CREATE TABLE M_EvalSection (
    EvalSectionId   SERIAL PRIMARY KEY,
    EvalSectionName VARCHAR(100) NOT NULL,   -- "Overall Self Evaluation", "Soft Skill", etc.
    DisplayOrder    INT NOT NULL DEFAULT 0
);

-- ========================================
-- THE ACTUAL EVALUATION CONTENT
-- ========================================
CREATE TABLE T_Eval (
    EvalId        SERIAL PRIMARY KEY,
    EvalContent   TEXT,
    EvalPeriodId  INT NOT NULL REFERENCES M_EvalPeriod(EvalPeriodId),
    EvalSectionId INT NOT NULL REFERENCES M_EvalSection(EvalSectionId),
    EmployeeId    INT NOT NULL REFERENCES M_Employee(EmployeeId),
    CreatedAt     TIMESTAMP NOT NULL DEFAULT now(),
    UpdatedAt     TIMESTAMP NOT NULL DEFAULT now(),
    UNIQUE (EvalPeriodId, EvalSectionId, EmployeeId)
);

CREATE INDEX idx_teval_period_employee ON T_Eval(EvalPeriodId, EmployeeId);
```

## Table purposes

| Table | Purpose |
|---|---|
| `M_Employee` | Who can log in, and their role (controls what they can view) |
| `M_EvalPeriod` | Original entity — Summer Bonus, Winter Bonus, Annual, etc. |
| `M_EvalSection` | New — the repeatable content blocks per period (Overall / Soft Skill / Technical) |
| `T_Eval` | The actual written content: one row per (period, section, employee) |

## Entity relationship diagram

```mermaid
erDiagram
    M_EMPLOYEE ||--o{ T_EVAL : writes
    M_EVALPERIOD ||--o{ T_EVAL : contains
    M_EVALSECTION ||--o{ T_EVAL : contains

    M_EMPLOYEE {
        int EmployeeId PK
        string EmployeeName
        string Email
        string PasswordHash
        string Role
        boolean IsActive
    }
    M_EVALPERIOD {
        int EvalPeriodId PK
        string EvalPeriodName
        date StartDate
        date EndDate
        boolean IsActive
    }
    M_EVALSECTION {
        int EvalSectionId PK
        string EvalSectionName
        int DisplayOrder
    }
    T_EVAL {
        int EvalId PK
        int EvalPeriodId FK
        int EvalSectionId FK
        int EmployeeId FK
        text EvalContent
        timestamp CreatedAt
        timestamp UpdatedAt
    }
```

GitHub renders `mermaid` code blocks natively in markdown, so this diagram will display as-is in the repo.

## Design notes

**Why a separate `M_EvalSection` table instead of fixed columns on `T_Eval`:**
- Adding a new section later (e.g. "Leadership Evaluation") is a data insert, not a schema migration.
- The unique constraint `(EvalPeriodId, EvalSectionId, EmployeeId)` guarantees one row per person per section per period — a clean upsert target.
- Room to grow: if sections ever need to differ by period, a join table `M_EvalPeriodSection` can be added later without touching `T_Eval`.

**Why no permissions table yet:**
A `Role` column on `M_Employee` is enough for now — `EMPLOYEE` sees only their own rows, `MANAGER`/`ADMIN` sees everyone's, enforced in application logic. If org-hierarchy-based visibility (e.g. "only your direct manager can see yours") is needed later, add a self-referencing `ManagerId` FK on `M_Employee` at that point — no need to model it before it's needed.

## Fetching a period's evaluations (all sections, filled or empty)

Start the query from `M_EvalSection` and left-join `T_Eval`, so every section shows up even if the employee hasn't written it yet:

```sql
SELECT s.EvalSectionId, s.EvalSectionName, e.EvalId, e.EvalContent
FROM M_EvalSection s
LEFT JOIN T_Eval e
    ON e.EvalSectionId = s.EvalSectionId
   AND e.EvalPeriodId = :periodId
   AND e.EmployeeId = :employeeId
ORDER BY s.DisplayOrder;
```

Example API response shape:

```json
{
  "evalPeriodId": 1,
  "evalPeriodName": "2026 Summer Bonus",
  "employeeId": 5,
  "sections": [
    { "evalSectionId": 1, "evalSectionName": "Overall Self Evaluation", "evalId": 10, "content": "..." },
    { "evalSectionId": 2, "evalSectionName": "Soft Skill Evaluation", "evalId": null, "content": null },
    { "evalSectionId": 3, "evalSectionName": "Technical Skill Evaluation", "evalId": 12, "content": "..." }
  ]
}
```

## Saving: batch upsert

Since a period now has multiple content blocks, save them together instead of one PUT per section:

```
PUT /api/eval-periods/{periodId}/evaluations?employeeId={employeeId}
Body:
[
  { "evalSectionId": 1, "content": "..." },
  { "evalSectionId": 2, "content": "..." }
]
```

Postgres upsert per item, using the unique constraint as the conflict target:

```sql
INSERT INTO T_Eval (EvalContent, EvalPeriodId, EvalSectionId, EmployeeId, UpdatedAt)
VALUES (:content, :periodId, :sectionId, :employeeId, now())
ON CONFLICT (EvalPeriodId, EvalSectionId, EmployeeId)
DO UPDATE SET EvalContent = EXCLUDED.EvalContent, UpdatedAt = now();
```