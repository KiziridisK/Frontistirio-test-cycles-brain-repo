# Frontistirio — Test Cycles Brain

> **Full-stack reference** for test cycle and individual test management — creation, upsert logic, student resolution, PDF export, and the student home page integration. Covers backend and Angular frontend.

---

## Backend: MongoDB Models

### `models/testCycle.js`
```javascript
{
  description: String,
  store_id: ObjectId → Store,
  period_id: ObjectId → TeachingPeriod,
  grade_ids: [ObjectId → Grade],       // which grades participate
  course_ids: [ObjectId → Course],     // which courses are tested
  test_ids: [ObjectId → Test],         // individual tests in this cycle
  isDeleted: Boolean, deletedAt: Date,
  createdBy/updatedBy: ObjectId → User,
  timestamps: true
}
```

### `models/test.js`
```javascript
{
  store_id: ObjectId → Store,
  test_cycle_id: ObjectId → TestCycle,
  grade_ids: [ObjectId → Grade],
  course_ids: [ObjectId → Course],
  student_ids: [ObjectId → Student],    // explicitly set students for this test
  period_id: ObjectId → TeachingPeriod,
  date: Date (required),
  start_time: String 'HH:mm' (required),
  end_time: String 'HH:mm' (required),
  isDeleted: Boolean, deletedAt: Date,
  createdBy/updatedBy: ObjectId → User,
  timestamps: true
}
```

---

## Backend: Routes (`routes/tests.js`)

| Method | Path | Role | Controller |
|---|---|---|---|
| GET | `/test-cycles/get-store-test-cycles` | isStoreUser | `getStoreTestCycles` |
| GET | `/test-cycles/get-store-test-cycle-tests` | isStoreUser | `getStoreTestCycleTests` |
| POST | `/test-cycles/upsert-store-test-cycles` | isStoreUser | `upsertStoreTestCycles` |
| POST | `/test-cycles/upsert-store-test-cycle-tests` | isStoreUser | `upsertStoreTestCycleTests` |
| POST | `/test-cycles/delete-test-cycle-test` | isStoreUser | `deleteStoreTestCycleTest` |
| POST | `/test-cycles/delete-store-test-cycle` | isStoreUser | `deleteStoreTestCycle` |
| POST | `/test-cycles/export-store-test-cycle-report` | isStoreUser | `exportTestCycleReport` |

Also on students route:
| GET | `/students/get-store-student-test-cycle-tests` | **isStudent** | `getStoreStudentTestCycleTests` |

---

## Backend: Controller Logic

### `upsertStoreTestCycles` (Create or Edit TestCycle)
```javascript
// body: { description, grade_ids[], course_ids[], existing? }
// Validation: description required, grade_ids and course_ids must be non-empty arrays

if (existing_cycle) {
  // UPDATE existing cycle
  test_cycle = await testHandler.updateStoreTestCycle(
    existing_cycle, store_id, periodId, description, grade_ids, course_ids, user_id, session
  );
} else {
  // CREATE new cycle
  test_cycle = await testHandler.createStoreTestCycle(...);
}
```

### `upsertStoreTestCycleTests` (Add/Edit Tests in a Cycle)
The frontend sends an array of tests — some may have `_id` (existing, to update) and some may not (new, to create):

```javascript
const existing_tests = tests.filter(t => t?._id);  // have _id → update
const new_tests = tests.filter(t => !t?._id);       // no _id → create

// Update existing tests:
const updated_tests = await testHandler.updateStoreTestCycleTests(
  existing_tests, store_id, periodId, user_id, session
);

// Create new tests:
const newly_created_tests = await testHandler.createStoreTestCycleTests(
  new_tests, store_id, periodId, user_id, session
);

// Add all test IDs to the cycle (using $addToSet to prevent duplicates):
await TestCycle.findByIdAndUpdate(test_cycle_id, {
  $addToSet: { test_ids: { $each: [...testIds] } }
}, { session });
```

### `deleteStoreTestCycleTest`
Soft-deletes the Test document and removes its `_id` from `TestCycle.test_ids`.

### `deleteStoreTestCycle`
Soft-deletes the TestCycle and all its associated Tests.

### `getStoreTestCycles(storeId, periodId)` (handler)
Returns test cycles filtered by `store_id` + `period_id`, with `isDeleted: false`.

### `getStoreTestCycleTests(test_cycle_id)` (handler)
Returns all non-deleted Test documents where `test_cycle_id` matches, with populated `grade_ids` and `course_ids`.

---

## Backend: PDF Export (`exportTestCycleReport`)

1. Fetch the TestCycle document (populated with test_ids)
2. Fetch all Test documents for the cycle
3. For each test: fetch its grades, courses, students
4. Call `createTestCyclePdf(cycle, tests)` → pdfkit buffer (DejaVu fonts)
5. Upload to S3 (`logeion-educational-material` bucket, path: `test-cycle-reports/{group_id}/{store_id}/`)
6. Return pre-signed URL (60s)

---

## Backend: Student Test Cycle Integration

`getStoreStudentTestCycleTests(req, res)` (on students route, role: **isStudent**):
- Uses `student_id` from query params
- Finds all TestCycles for the store's current period
- Filters Tests where `student_ids` includes this student
- Returns upcoming tests (sorted by date)

This is what populates the home page dashboard for a logged-in student.

---

## Backend: Handler Exports

```javascript
exports.createStoreTestCycle(cycleData, store_id, periodId, user_id, session)
exports.updateStoreTestCycle(cycleId, store_id, periodId, description, grade_ids, course_ids, user_id, session)
exports.getStoreTestCycles(storeId, periodId)
exports.getStoreTestCycle(testCycleId)             // single cycle
exports.getStoreTestCycleTests(testCycleId)        // tests for a cycle
exports.createStoreTestCycleTests(tests[], store_id, periodId, user_id, session)
exports.updateStoreTestCycleTests(tests[], store_id, periodId, user_id, session)
exports.deleteStoreTestCycleTest(testId, testCycleId, session)
exports.watchTestCycles(io, userSockets)            // Change Stream
```

---

## Frontend: TypeScript Models

```typescript
interface TestCycle {
  _id?: string; description: string;
  store_id?: string; period_id?: string;
  grade_ids: string[]; course_ids: string[];
  test_ids: string[];
  isDeleted?: boolean;
}

interface Test {
  _id?: string; test_cycle_id: string;
  grade_ids: string[]; course_ids: string[];
  student_ids: string[];
  period_id?: string;
  date: Date | string;
  start_time: string;   // 'HH:mm'
  end_time: string;     // 'HH:mm'
  isDeleted?: boolean;
}
```

---

## Frontend: NgRx State

```typescript
// Slice: test_cycles
interface TestCycleState { test_cycles: TestCycle[] }

// Actions:
setTestCycles({ test_cycles })              // bootstrap
addTestCycle({ test_cycle })                // socket: testCycleAdded
updatePeriodFields({ id, changes })         // socket: testCycleUpdated
deleteTestCycle({ id })                     // socket: testCycleDeleted

// Selector:
selectTestCycles → Observable<TestCycle[]>
```

**Note:** Individual `Test` documents are **NOT in NgRx** — fetched on demand in `TestCycleDetailsComponent`. Only the `TestCycle` objects (with `test_ids` array) are in state.

---

## Frontend: Service (`services/test-cycle.service.ts`)

```typescript
getStoreTestCycles(student_ids, course_id)
  → GET /test-cycles/get-store-test-cycles  (params unused server-side)
  → returns { test_cycles: TestCycle[], success }

getStoreTestCycleTests(test_cycle_id)
  → GET /test-cycles/get-store-test-cycle-tests?test_cycle_id=xxx
  → returns { tests: Test[], success }

upsertStoreTestCycles(description, course_ids, grade_ids, existing?)
  → POST /test-cycles/upsert-store-test-cycles
  body: { description, course_ids, grade_ids, existing: existingCycleId | undefined }
  → returns { success, test_cycle: TestCycle, message }

createTestsForCycle(test_cycle_id, tests[])
  → POST /test-cycles/upsert-store-test-cycle-tests
  body: { test_cycle_id, tests }
  → returns { success, test_cycle }

removerTestFromCycle(test_cycle_id, test_id)
  → POST /test-cycles/delete-test-cycle-test
  body: { test_cycle_id, test_id }

deleteTestCycle(test_cycle_id, permanently)
  → POST /test-cycles/delete-store-test-cycle
  body: { test_cycle_id, permanently }

exportTestCycleReport(test_cycle_id)
  → POST /test-cycles/export-store-test-cycle-report
  body: { test_cycle_id }
  → returns { success, downloadUrl, filename }
```

---

## Frontend: Components

### `TestCyclesComponent` (`/test-cycles`, role: store-user)
Lists test cycles from NgRx `selectTestCycles`. Shows `TestCycleItemComponent` cards with cycle description, grade/course count.

### `AddTestCycleComponent`
Form: description + select grades (from NgRx `selectAllGrades`) + select courses (from NgRx `selectAllCourses`). Calls `upsertStoreTestCycles()`. Same component handles both create and edit (edit passes `existing` cycle ID).

### `TestCycleDetailsComponent` (`/test-cycles/:id`) — 17KB TS + 8KB HTML
Most complex test component:
- On init: calls `getStoreTestCycleTests(cycleId)` to load individual tests
- Add test form: date picker, start/end time, select courses and grades
- Tests are sent as an array — new ones have no `_id`, existing ones have `_id`
- Calls `createTestsForCycle(cycleId, tests)` which handles both create and update
- Delete individual test: `removerTestFromCycle(cycleId, testId)`
- Export PDF: `exportTestCycleReport(cycleId)` → `window.open(res.downloadUrl)`

### `TestCycleItemComponent`
Card view: shows cycle description, associated grades and courses, test count.

---

## Student Home Page Integration (`home.page.ts`)

When role is `student`:
```typescript
// After finding student from NgRx:
this.studentsService.getStoreStudentTestCycleTests(student._id)
  .subscribe(resData => {
    if (resData.success) this.upcomingTestCycles = resData.tests;
  });
```
Displays upcoming tests in a schedule format on the dashboard.

---

## Real-Time Sync

`watchTestCycles(io, userSockets)` — Change Stream on `TestCycle` collection:
- Emits `testCycleAdded`, `testCycleUpdated`, `testCycleDeleted`
- Individual `Test` changes are NOT watched — the details page re-fetches on load
