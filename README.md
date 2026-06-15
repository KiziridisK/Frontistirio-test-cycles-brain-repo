# Frontistirio — Test Cycles Brain

> **Purpose:** Documents how test cycles and individual tests are created, managed, and used to evaluate students. Also covers how test result PDFs are generated.

---

## Overview

**Test Cycles** are groupings of individual **Tests** (exam sessions). A test cycle might represent a term's examination period, grouping multiple tests across courses and grades.

```
TestCycle (e.g., "Winter Exams 2025")
  ├── Test (Math, Grade A, 15 Jan 2026, 10:00–12:00)
  ├── Test (English, Grade B, 16 Jan 2026, 10:00–12:00)
  └── Test (Physics, Grade A, 17 Jan 2026, 14:00–16:00)
```

---

## Models

### TestCycle (`models/testCycle.js`)

```
TestCycle {
  description   String
  store_id      → Store
  period_id     → TeachingPeriod
  grade_ids     → [Grade]          // which grades participate
  course_ids    → [Course]         // which courses are tested
  test_ids      → [Test]           // individual tests in this cycle
  isDeleted     Boolean
  deletedAt     Date
  createdBy / updatedBy → User
}
```

### Test (`models/test.js`)

Individual exam session within a cycle:

```
Test {
  store_id       → Store
  test_cycle_id  → TestCycle
  grade_ids      → [Grade]          // grades taking this test
  course_ids     → [Course]         // courses being tested
  student_ids    → [Student]        // specific students enrolled
  period_id      → TeachingPeriod
  date           Date (required)
  start_time     String "HH:mm"
  end_time       String "HH:mm"
  isDeleted      Boolean
  deletedAt      Date
  createdBy / updatedBy → User
}
```

---

## API Endpoints (`routes/tests.js`)

| Method | Path | Description |
|---|---|---|
| GET | `/test-cycles/` | Get all test cycles for current period |
| POST | `/test-cycles/` | Create a test cycle |
| PUT | `/test-cycles/:id` | Edit a test cycle |
| DELETE | `/test-cycles/:id` | Delete a test cycle |
| GET | `/test-cycles/:id/tests` | Get all tests in a cycle |
| POST | `/test-cycles/:cycleId/tests` | Add a test to a cycle |
| PUT | `/test-cycles/:cycleId/tests/:testId` | Edit a test |
| DELETE | `/test-cycles/:cycleId/tests/:testId` | Delete a test |
| GET | `/test-cycles/:id/pdf` | Generate & upload PDF summary |

---

## Test Cycle Creation Flow

1. Validate store + period
2. Call `testHandler.createStoreTestCycle(cycleData, store_id, periodId, user_id)`
3. TestCycle created with `test_ids: []`

---

## Adding Tests to a Cycle

```
POST /test-cycles/:cycleId/tests
Body: {
  grade_ids, course_ids, student_ids,
  date, start_time, end_time
}
```

1. Create `Test` document linked to `test_cycle_id`
2. Push `test._id` into `TestCycle.test_ids`
3. Students can be explicitly listed or derived from `grade_ids` + `course_ids`

---

## Student Resolution for Tests

When `student_ids` is not explicitly provided:
1. Look up all students in the store for the active period
2. Filter by `grade_ids` — students whose `period_grade` matches
3. Filter by `course_ids` — students enrolled in those courses
4. Resulting set = students assigned to this test

---

## PDF Generation for Test Cycles

**Endpoint:** `GET /test-cycles/:id/pdf`

Flow:
1. Fetch the `TestCycle` with all its `Test` documents populated
2. Fetch student data for each test
3. Call `createTestCyclePdf(cycle, tests)` (uses `pdfkit`)
4. Upload PDF to **AWS S3**
5. Return a **pre-signed S3 URL** (60s expiry)

Helper: `helpers/createTestCyclePdf.js`

PDF content typically includes:
- Cycle name/description
- List of tests with dates, times, courses, grades
- Student lists per test

---

## Real-Time Sync

`watchTestCycles(io, userSockets)`:
- MongoDB Change Stream on `TestCycle` collection
- Pushes updates to connected store-users when test cycles are created or modified

---

## Relationship to Grading

Test cycles feed into the grading system:
- Individual test scores → recorded against student/course
- `GradeScenario` defines how test scores combine → `StudentCourseGrade`
- The cycle itself doesn't store scores — scores are in `StudentCourseGrade`

---

## Period Scoping

All test cycles are scoped to a `TeachingPeriod`. When querying:
```javascript
const periodId = await getStoreDefaultPeriod(store_id);
const cycles = await TestCycle.find({
  store_id,
  period_id: periodId,
  isDeleted: false
});
```
