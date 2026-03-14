# ORI Incremental Steps Plan (TDD)

> **Based on:** ArchitecturePack_ORI_v2.md
> **Date:** 2026-03-14
> **Tech Stack:** React 18 + Vite + MUI v6 + Tailwind (FE) | Java Spring (BE) | MySQL (DB)
> **Reference:** Clevai_Tech_Stack_Prefer_PR4260.md — JWT auth, ExcelJS export, Docker deploy

---

## Ordering Strategy

```
Phase A: Foundation (Auth + Config + OCLAG)
Phase B: FORECAST (Plan + Lock + Booking + Schedule)
Phase C: MATCH (Reg3 + Reg4 + Backup)
Phase D: PREPARE (Meeting + Merge + Notify)
Phase E: OPERATE (Monitor + Accept)
Phase F: REPORT (Evaluation + Report + Send)
Phase G: METRICS (Dashboard + Fill Rate + L-level cron)
```

Mỗi step: UI mock trước → API mock → Real validation → Real persistence → Real integration.

---

## PHASE A: FOUNDATION

### Step A1: Project Scaffold + Auth + Layout

**Goal:** SOW chạy được, login/logout hoạt động, sidebar navigation đầy đủ 11 routes.

**Scope:** `src/main.tsx`, `src/App.tsx`, `src/components/Layout.tsx`, `src/services/api.ts`, `src/hooks/useApi.ts`

**Components:**
- `App.tsx` — BrowserRouter + 11 lazy routes
- `Layout.tsx` — AppBar + Drawer (220px) + Outlet
- `api.ts` — Axios instance, baseURL = `https://api-gateway.mikai.tech/api/v1/bp-log`
- JWT interceptor: localStorage `ori_token` → header `Authorization: Bearer <token>`
- 401 interceptor: clear token → redirect `/login`

**Preconditions:** Node 22+, `npm install` success, Vite dev server starts.

**Detailed Logic:**

1. `api.ts`:
```typescript
const api = axios.create({
  baseURL: 'https://api-gateway.mikai.tech/api/v1/bp-log'
});
api.interceptors.request.use(config => {
  const token = localStorage.getItem('ori_token');
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});
api.interceptors.response.use(
  res => res,
  err => {
    if (err.response?.status === 401) {
      localStorage.removeItem('ori_token');
      window.location.href = '/login';
    }
    return Promise.reject(err);
  }
);
```

2. `App.tsx` — 11 routes:
```
/dashboard, /settle-plan, /slot-lock, /booking, /schedule,
/class-matrix, /teacher-fill-rate, /class-monitor,
/merge, /evaluation, /report-management
```

3. `Layout.tsx` — MUI AppBar + Drawer, nav items matching routes.

**Test Cases:**
- [ ] `npm run dev` starts without errors
- [ ] Navigate to all 11 routes → correct page renders (placeholder OK)
- [ ] API interceptor attaches token when `ori_token` exists in localStorage
- [ ] 401 response clears token and redirects to `/login`
- [ ] Sidebar highlights active route

**Artifacts:** `api.ts`, `App.tsx`, `Layout.tsx`, `main.tsx`, `useApi.ts`

**Exit Criteria:** All 11 routes render placeholder. Axios interceptor unit test passes. Build succeeds.

---

### Step A2: ORI Config Constants + PT Map + Time Slot Map

**Goal:** Centralize tất cả constants ORI để các page dùng chung: PT list, max students, TEC rates, time slot mapping WHO↔CASHSTA.

**Scope:** `src/config/ori.ts` (NEW)

**Components:**
- `ori.ts` — export constants

**Preconditions:** Step A1 done.

**Detailed Logic:**

```typescript
// src/config/ori.ts

export const ORI_PT_LIST = [
  { code: 'KMA_DU_8', name: 'Toán DU', maxStudents: 5, tecRate: 90000, lct: 'ORIO-30MI' },
  { code: 'KMA_TT_8', name: 'Toán TT', maxStudents: 5, tecRate: 90000, lct: 'ORIO-30MI' },
  { code: 'KEN_DU_8', name: 'TA DU',   maxStudents: 2, tecRate: 45000, lct: 'ORIO-30MI' },
  { code: 'KEN_TT_8', name: 'TA TT',   maxStudents: 2, tecRate: 45000, lct: 'ORIO-30MI' },
  { code: 'KEN_ES_8', name: 'Early Speak', maxStudents: 2, tecRate: 45000, lct: 'ORIO-25MI' },
  { code: 'KSP_PS_8', name: 'Phil Speak', maxStudents: 2, tecRate: 45000, lct: 'ORIO-25MI' },
  { code: 'KFP_FO_8', name: 'Toán FO', maxStudents: 5, tecRate: 35000, lct: 'ORIO-30MI' },
];

export const PT_MAX_MAP: Record<string, number> = Object.fromEntries(
  ORI_PT_LIST.map(p => [p.code, p.maxStudents])
);

export const ORI_TIME_SLOTS = [
  { who: '0900', cashsta: '0900', label: '09:00' },
  { who: '0930', cashsta: '0930', label: '09:30' },
  { who: '1000', cashsta: '1000', label: '10:00' },
  { who: '1030', cashsta: '1030', label: '10:30' },
  { who: '1400', cashsta: '1400', label: '14:00' },
  { who: '1430', cashsta: '1430', label: '14:30' },
  { who: '1500', cashsta: '1500', label: '15:00' },
  { who: '1530', cashsta: '1530', label: '15:30' },
  { who: '1700', cashsta: '1700', label: '17:00' },
  { who: '1730', cashsta: '1730', label: '17:30' },
  { who: '1800', cashsta: '1800', label: '18:00' },
  { who: '1840', cashsta: '1845', label: '18:40' },  // MISMATCH
  { who: '1920', cashsta: '1930', label: '19:20' },  // MISMATCH
  { who: '2000', cashsta: '2000', label: '20:00' },
  { who: '2040', cashsta: '2040', label: '20:40' },
  { who: '2120', cashsta: '2120', label: '21:20' },
];

export const WHO_TO_CASHSTA: Record<string, string> = Object.fromEntries(
  ORI_TIME_SLOTS.map(s => [s.who, s.cashsta])
);
export const CASHSTA_TO_WHO: Record<string, string> = Object.fromEntries(
  ORI_TIME_SLOTS.map(s => [s.cashsta, s.who])
);

export const DFDL_OPTIONS = [
  { code: 'C1', name: 'Cơ bản' },
  { code: 'C2', name: 'Nâng cao' },
];

export const GRADE_OPTIONS = [
  'G1','G2','G3','G4','G5','G6','G7','G8','G9','G10','G11','G12','K4','K5'
];

export const CUI_STATUS_FLOW = [
  'BOOK_ORI', 'REMINDED', 'CLASS_ACCEPTED', 'EVALUATED', 'REPORT_GEN', 'REPORT_SENT'
];

export const ACCEPTANCE_OPTIONS = ['Done', 'Cancel', 'Fail', 'Pass'];

export const SOURCE_OPTIONS = [
  { code: 'Crossell', name: 'HS cũ thử SP mới' },
  { code: 'D', name: 'HS mới' },
];
```

**Test Cases:**
- [ ] `PT_MAX_MAP['KMA_DU_8']` === 5, `PT_MAX_MAP['KEN_DU_8']` === 2
- [ ] `WHO_TO_CASHSTA['1840']` === '1845', `WHO_TO_CASHSTA['1920']` === '1930'
- [ ] `CASHSTA_TO_WHO['1845']` === '1840'
- [ ] All 16 slots present in `ORI_TIME_SLOTS`
- [ ] All 7 PTs present in `ORI_PT_LIST`

**Artifacts:** `src/config/ori.ts`

**Exit Criteria:** Import config anywhere → no error. All mapping unit tests pass.

---

### Step A3: TypeScript Interfaces for ORI Entities

**Goal:** Define TS interfaces cho tất cả entity ORI sử dụng, map 1:1 với DB columns trong ArchitecturePack V2.

**Scope:** `src/types/ori.ts` (NEW)

**Components:**
- `ori.ts` — TypeScript interfaces

**Preconditions:** Step A2 done.

**Detailed Logic:**

```typescript
// src/types/ori.ts

// B1: OriSettlePlanHeader — bp_osph_ori_settle_plan_header
export interface OSPH {
  id: number;
  code: string;               // ORI-{PT}-{YYYYMM}-V{n}
  mypt: string;
  plan_month: string;          // YYYY-MM-DD (1st of month)
  version: number;
  plan_type: 'SALE' | 'VH';
  status: 'DRAFT' | 'ACTIVE' | 'CLOSED';
  total_target: number;
  monthly_budget: number;
  cost_per_session: number;
  forecast_revenue: number;
  expected_conversion: number;
  created_by: string;
  approved_by: string | null;
  approved_at: string | null;
  published: boolean;
  created_at: string;
  updated_at: string;
}

// B2: OriSettlePlanDetail — bp_ospd_ori_settle_plan_detail
export interface OSPD {
  id: number;
  code: string;                // OSPD-{uuid}
  myosph: string;              // FK → OSPH.code
  plan_date: string;           // YYYY-MM-DD
  mywho: string;               // WHO code (0900, 1840...)
  target_sessions: number;
  l42_registered: number;
  l5_jsu: number;
  l8_settled: number;
  teacher_available: number;
  published: boolean;
}

// B3: OriSlotLock — bp_ospl_ori_slot_lock
export interface OSPL {
  id: number;
  plan_date: string;
  ori_time_slot: string;       // CASHSTA code
  mypt: string;
  status: 'LOCKED' | 'UNLOCKED';
  locked_by: string;
  locked_at: string;
}

// B4: OriClagSchedule — bp_oclag_ori_clag_schedule (1 row = 1 student)
export interface OCLAG {
  id: number;
  code: string;                // OCLAG-{uuid}
  myclag: string;              // FK → CLAG.code
  effective_date: string;
  ori_time_slot: string;       // CASHSTA code
  mypt: string;
  myusi: string;               // SĐT học sinh
  student_name: string;
  mygg: string;
  mydfdl: string;              // C1/C2
  teacher_code: string | null;
  teacher_name: string | null;
  backup_code: string | null;
  backup_name: string | null;
  meet_link: string | null;
  meet_code: string | null;
  student_number: number;
  maxtotalstudents: number;
  status: string;              // BOOKED, REMINDED, CLASS_ACCEPTED, EVALUATED, REPORT_GEN, REPORT_SENT, CANCELLED
  acceptance: string | null;   // Done, Cancel, Fail, Pass
  sale_code: string | null;
  booking_time: string | null;
  source: string | null;       // Crossell, D
  cancel_reason: string | null;
  note: string | null;
  zalo_link: string | null;
  ctv_so: string | null;
  published: boolean;
  created_by: string | null;
  created_at: string;
  updated_at: string;
}

// C1: ClassGroup — bp_clag_classgroup (NO published column)
export interface CLAG {
  id: number;
  code: string;
  name: string;
  mypt: string;
  mygg: string;
  mydfdl: string;
  mywho: string;
  mywso: string;
  clagtype: string;            // DYN for ORI
  maxtotalstudents: number;
  student_number: number;
  active: boolean;
}

// C4: ContentUserUlcInstance — bp_cui_content_user_ulc_instance
export interface CUI {
  code: string;
  myusi: string;               // SĐT (student) or teacher_code (teacher)
  myulc: string;               // FK → ULC.code
  mybps: string;               // BOOK_ORI, REMINDED, CLASS_ACCEPTED, EVALUATED, REPORT_GEN, REPORT_SENT
  mycti: string | null;        // FK → CTI.code (report)
  published: boolean;
}

// C5: UserDuty — bp_usid_usiduty
export interface USID {
  code: string;
  myusi: string;               // teacher code
  myclag: string | null;       // FK → CLAG.code (Reg4 only)
  mypt: string;
  mycashsta: string;           // CASHSTA code
  register_type: number;       // 1=Reg1, 3=Reg3, 4=Reg4
  position: 'MAIN' | 'BACKUP' | null;
  isapproved: boolean;
  effective_date: string;
  allocated_myusi: string | null;
  submitted_cancel_at: string | null;
  approved_cancel_at: string | null;
}

// C6: CuiEvent — bp_cuie_cuievent
export interface CUIE {
  code: string;
  mycui: string;               // FK → CUI.code
  mylcet: string;              // Event type: RKN, RSK, RAT, RCM, RRC, RSC, GRP, SRP...
  value1: string | null;
  value2: string | null;
  description: string | null;
  created_at: string;
}

// Availability response
export interface SlotAvailability {
  slot: string;                // CASHSTA code
  slot_label: string;          // "09:00"
  target: number;
  registered: number;
  available: number;
  locked: boolean;
}

// Booking request
export interface BookingRequest {
  student_name: string;
  myusi: string;               // SĐT
  mypt: string;
  mygg: string;
  mydfdl: string;
  effective_date: string;
  ori_time_slot: string;       // CASHSTA code
  source: string;              // Crossell / D
  note?: string;
  zalo_link?: string;
}

// Booking response
export interface BookingResponse {
  oclag_code: string;
  clag_code: string;
  cui_code: string;
}

// Evaluation criteria
export interface EvaluationSubmit {
  cui_code: string;
  knowledge: [number, number]; // [basic, application] each 1-5
  skill: [number, number];     // [problem_solving, presentation]
  attitude: [number, number];  // [learning, focus]
  comment: string;
  recommendation: string;      // C1/C2/C3
}

// Teacher commitment (from vw_teacher_commitment)
export interface TeacherCommitment {
  teaching_date: string;
  myusi: string;
  fullname: string;
  phone: string;
  position: string;
  clag_code: string;
  myust: string;
  mywho: string;
  mypt: string;
  mygg: string;
  mydfdl: string;
  confirm_status: string;
  next_step: string;
  active: boolean;
}

// Merge candidate
export interface MergeCandidate {
  source_clag: string;
  target_clag: string;
  mypt: string;
  mygg: string;
  mydfdl: string;
  slot: string;
  source_students: number;
  target_students: number;
  target_max: number;
}

// Dashboard metrics
export interface DashboardCell {
  slot: string;
  target: number;
  l42_registered: number;
  l5_jsu: number;
  l8_settled: number;
  teacher_available: number;
}
```

**Test Cases:**
- [ ] All interfaces importable without TS errors
- [ ] `npm run build` succeeds (tsc compile check)
- [ ] OCLAG interface has all 31 columns from ArchitecturePack V2 §1.2
- [ ] OSPH interface matches §1.3
- [ ] USID interface covers reg_type 1/3/4

**Artifacts:** `src/types/ori.ts`

**Exit Criteria:** `tsc -b` passes. All page components can import types.

---

## PHASE B: FORECAST

### Step B1: Settle Plan — List + Create Header (UI + API)

**Goal:** SettlePlanPage hiển thị danh sách plan headers với filter, tạo plan mới.

**Scope:** `src/pages/SettlePlanPage.tsx`

**Components:**
- Filter bar: PT dropdown (7 PT), Month picker, Plan Type (SALE/VH)
- Plan list table: code, PT, month, version, status, target, created_by
- Create dialog: mypt, plan_month, plan_type, total_target, cost_per_session
- KPI summary cards: Total Target, Budget, Sessions

**Preconditions:** Steps A1-A3 done.

**Detailed Logic:**

1. **Load plans:**
```
GET /ori/settle-plan/headers?mypt={pt}&planMonth={month}&status={status}
→ Response: List<OSPH>
→ Render MUI DataGrid: code, mypt, plan_month, version, plan_type, status, total_target
→ Status Chip: DRAFT=grey, ACTIVE=green, CLOSED=red
```

2. **Create plan:**
```
POST /ori/settle-plan/headers
Body: { mypt, plan_month, plan_type, total_target, monthly_budget, cost_per_session, expected_conversion }
→ Toast "Plan created"
→ Refresh list
```

3. **Plan actions (row buttons):**
- [Generate Grid] → `POST /ori/settle-plan/headers/{code}/generate`
- [Activate] → `POST /ori/settle-plan/headers/{code}/activate`
- [New Version] → `POST /ori/settle-plan/headers/{code}/new-version`

**Tables read:** `bp_osph`
**Tables write:** `bp_osph`, `bp_ospd` (on generate)

**Test Cases:**
- [ ] Filter by PT → only matching plans shown
- [ ] Create plan with valid data → plan appears in list with status DRAFT
- [ ] Create plan without required field → form validation error
- [ ] Generate grid → 200 response, toast success
- [ ] Activate → status changes to ACTIVE, previous active → CLOSED
- [ ] New Version → new row with version+1

**Artifacts:** `SettlePlanPage.tsx`

**Exit Criteria:** CRUD plan header works. Plan list renders. Status transitions work.

---

### Step B2: Settle Plan — Detail Grid + Excel Import/Export

**Goal:** Plan detail grid: ngày × 16 slot với target_sessions editable. Excel import/export.

**Scope:** `src/pages/SettlePlanPage.tsx` (extend)

**Components:**
- Detail grid: rows = dates (1-31), columns = 16 slots (WHO codes). Cell = target_sessions
- Editable cells (click to edit, blur to save)
- L-level display: `{l42}/{target}` with color coding
- Excel toolbar: [Download Template] [Import] [Export]

**Preconditions:** Step B1 done.

**Detailed Logic:**

1. **Load detail:**
```
GET /ori/settle-plan/headers/{code}/details
→ Response: List<OSPD>
→ Pivot: group by plan_date → row, each mywho → column
→ Cell display: "{l42_registered}/{target_sessions}" + color (🟢≥80%, 🟡50-80%, 🔴<50%)
```

2. **Edit cell:**
```
Click cell → MUI TextField inline edit
Blur/Enter → PATCH /ori/settle-plan/details-batch
Body: { updates: [{ code: ospd.code, target_sessions: newValue }] }
→ Optimistic update UI
```

3. **Excel operations:**
```
Download template: GET /ori/settle-plan/template?mypt={pt}&planType={type}
→ Browser download .xlsx

Export: GET /ori/settle-plan/headers/{code}/export
→ Browser download .xlsx

Import: POST /ori/settle-plan/headers/{code}/import
→ FormData with file
→ Response: { imported: 248, total: 248 }
→ Refresh grid
```

**Tables read:** `bp_ospd`
**Tables write:** `bp_ospd` (batch update)

**Test Cases:**
- [ ] Grid renders dates × 16 slots correctly
- [ ] Edit cell → PATCH called → cell updated
- [ ] Color coding: 0/5 = red, 3/5 = yellow, 5/5 = green
- [ ] Download template → .xlsx file
- [ ] Import Excel → grid populated
- [ ] Export → .xlsx with current data

**Artifacts:** `SettlePlanPage.tsx` (extended with detail grid)

**Exit Criteria:** Full OSPD grid editable. Excel round-trip works.

---

### Step B3: Slot Lock Management

**Goal:** SlotLockPage: toggle lock/unlock per date × slot × PT.

**Scope:** `src/pages/SlotLockPage.tsx`

**Components:**
- Filters: PT dropdown, date range picker
- Grid: rows = dates, columns = 16 slots
- Cell: toggle button (🔒 locked / 🔓 unlocked)
- Bulk operations: Lock/Unlock selected

**Preconditions:** Steps A1-A3 done.

**Detailed Logic:**

1. **Load locks:**
```
GET /ori/slot-lock?dateFrom={from}&dateTo={to}&mypt={pt}
→ Response: List<OSPL>
→ Pivot: date × slot → lock status
→ Unlisted cells = UNLOCKED (default)
```

2. **Toggle lock:**
```
Click cell →
POST /ori/slot-lock/toggle
Body: { plan_date, ori_time_slot (CASHSTA code), mypt }
→ Toast "Slot locked/unlocked"
→ Toggle UI
```

**Tables read:** `bp_ospl`
**Tables write:** `bp_ospl`

**Test Cases:**
- [ ] Grid renders with correct lock states
- [ ] Click unlocked cell → becomes locked → API called
- [ ] Click locked cell → becomes unlocked → API called
- [ ] Filter by PT → only that PT's locks shown
- [ ] Locked slot shows 🔒 icon in booking page (integration later)

**Artifacts:** `SlotLockPage.tsx`

**Exit Criteria:** Lock/unlock works. Grid renders correctly for any date range.

---

### Step B4: Booking — Availability Check + 7-Day Grid

**Goal:** BookingPage phần 1: form nhập thông tin HS + check availability hiển thị 7-day grid.

**Scope:** `src/pages/BookingPage.tsx`

**Components:**
- Student info form: student_name, myusi (SĐT), mypt, mygg, mydfdl, source, note, zalo_link
- Date picker for starting date
- [CHECK AVAILABILITY] button
- 7-day × 16-slot grid: cell = `{registered}/{target}` with color + lock icon

**Preconditions:** Steps A1-A3, B3 done.

**Detailed Logic:**

1. **Form validation (FE only):**
```
Required: student_name, myusi (SĐT format: 10 digits), mypt, mygg, mydfdl, date
Zod schema:
  myusi: z.string().regex(/^0\d{9}$/)
  mypt: z.enum(ORI_PT_LIST.map(p => p.code))
  mygg: z.enum(GRADE_OPTIONS)
  mydfdl: z.enum(['C1', 'C2'])
  source: z.enum(['Crossell', 'D'])
```

2. **Check availability:**
```
GET /ori/booking/availability?effectiveDate={date}&mypt={pt}
→ Response: [{slot, target, registered, available, locked}]
→ Render 7 columns (date, date+1, ..., date+6) × 16 slots
→ Cell color: 🟢 available > 0 + unlocked, 🔒 locked, 🔴 full
→ Click available cell → highlight + show [BOOK] button
```

**Tables read:** `bp_ospd` (VH plan), `bp_ospl` (locks), `bp_oclag` (count)

**Test Cases:**
- [ ] Form validates SĐT format (10 digits starting with 0)
- [ ] Required fields show error when empty
- [ ] Availability grid shows 7 days × 16 slots
- [ ] Locked slots show lock icon, not clickable
- [ ] Full slots show red, not clickable
- [ ] Available slots clickable → highlight

**Artifacts:** `BookingPage.tsx`

**Exit Criteria:** Availability check renders correctly. Form validation works.

---

### Step B5: Booking — Submit + CL-01 Integration

**Goal:** BookingPage phần 2: submit booking gọi POST /ori/booking → backend chạy CL-01.

**Scope:** `src/pages/BookingPage.tsx` (extend)

**Components:**
- [BOOK] button (appears after clicking available slot)
- Confirmation dialog: summary of student + slot + date
- Success toast with OCLAG code + CLAG code

**Preconditions:** Step B4 done.

**Detailed Logic:**

1. **Submit booking:**
```
User clicks [BOOK] → Confirm dialog →
POST /ori/booking
Body: BookingRequest {
  student_name, myusi, mypt, mygg, mydfdl,
  effective_date, ori_time_slot (CASHSTA code),
  source, note, zalo_link
}
→ Response: BookingResponse { oclag_code, clag_code, cui_code }
→ Toast "Đã book! OCLAG: {code}"
→ Refresh availability grid
```

2. **Backend CL-01 (reference — not FE responsibility):**
```
Check lock → Check capacity → Find/create CLAG → INSERT OCLAG → INSERT CUI → Update counters
```

3. **Error handling:**
```
409 "Slot đã khóa" → Toast error
409 "Hết chỗ" → Toast error + refresh grid
409 "Duplicate booking" → Toast "HS đã book ngày này"
```

**Tables write (by backend):** `bp_oclag`, `bp_clag`, `bp_ulc`, `bp_clag_ulc`, `bp_cui`, `bp_ospd`

**Test Cases:**
- [ ] Click [BOOK] → confirmation dialog shows correct info
- [ ] Confirm → API called → success toast with codes
- [ ] Grid refreshes after booking (available count -1)
- [ ] Locked slot error → meaningful toast
- [ ] Full slot error → meaningful toast
- [ ] Duplicate booking → error toast

**Artifacts:** `BookingPage.tsx` (complete)

**Exit Criteria:** End-to-end booking works. OCLAG row created in DB.

---

### Step B6: Schedule Page — Class Grid + CRUD

**Goal:** SchedulePage hiển thị date × slot grid các lớp ORI (từ OCLAG), có thể tạo/xóa schedule.

**Scope:** `src/pages/SchedulePage.tsx`

**Components:**
- Filters: date picker, PT dropdown
- Grid: rows = slots (16), columns = date / CLAG code / PT / GG / DFDL / students / teacher / status
- Expandable row → student list (from OCLAG same myclag)
- [Create Schedule] → manual CLAG creation
- [Delete] → soft delete OCLAG

**Preconditions:** Steps A1-A3 done.

**Detailed Logic:**

1. **Load schedule:**
```
GET /ori/schedule?effectiveDate={date}&mypt={pt}
→ Response: List<OCLAG+CLAG joined>
→ Group by myclag → show 1 row per CLAG
→ Expand → show students (individual OCLAG rows)
```

2. **Create manual schedule:**
```
POST /ori/schedule
Body: { mypt, mygg, mydfdl, effective_date, ori_time_slot }
→ Creates empty CLAG + ULC + OCLAG placeholder
→ Refresh grid
```

3. **Delete schedule:**
```
DELETE /ori/schedule/{oclagCode}
→ Soft delete (published=0)
→ Refresh grid
```

**Tables read:** `bp_oclag`, `bp_clag`
**Tables write:** `bp_oclag`, `bp_clag`, `bp_ulc`, `bp_clag_ulc`

**Test Cases:**
- [ ] Grid shows CLAGs grouped by slot
- [ ] Expand CLAG → shows individual students
- [ ] Filter by PT → correct CLAGs shown
- [ ] Create schedule → new CLAG appears
- [ ] Delete → OCLAG soft-deleted, disappears from grid

**Artifacts:** `SchedulePage.tsx`

**Exit Criteria:** Schedule grid renders. CRUD operations work.

---

## PHASE C: MATCH

### Step C1: Class Matrix Page

**Goal:** ClassMatrixPage: slot × CLAG grid hiển thị teacher assignment status, capacity.

**Scope:** `src/pages/ClassMatrixPage.tsx`

**Components:**
- Filters: date picker, PT dropdown
- Matrix: rows = slots (16), each row has CLAG cards
- CLAG card: code, PT, GG, DFDL, `{student_number}/{max}`, teacher name, status chip
- Status chips: ✗ No Teacher (red), ⏳ Reg4 Pending (yellow), ✓ Confirmed (green)

**Preconditions:** Steps A1-A3 done.

**Detailed Logic:**

1. **Load matrix:**
```
GET /ori/classes/matrix?effectiveDate={date}&mypt={pt}
→ Response: List<{ slot, clag_code, mypt, mygg, mydfdl, student_number, maxtotalstudents,
                    teacher_code, teacher_name, isapproved, backup_code, backup_name }>
→ Group by slot → render cards per slot row
```

2. **Teacher status logic (FE display only):**
```
IF teacher_code IS NULL → "Chưa xếp GV" (red chip)
ELSE IF isapproved = false → "Chờ xác nhận" (yellow chip)
ELSE → "Đã xác nhận: {teacher_name}" (green chip)
```

**Tables read:** `bp_oclag` (grouped by myclag), `bp_usid` (Reg4 status)

**Test Cases:**
- [ ] Matrix shows all CLAGs for selected date
- [ ] Cards display correct student count / max
- [ ] Teacher status chips render correctly
- [ ] Filter by PT narrows results
- [ ] Empty slots show "Không có lớp"

**Artifacts:** `ClassMatrixPage.tsx`

**Exit Criteria:** Matrix renders. All 3 teacher status states display correctly.

---

### Step C2: Teacher Fill Rate Page

**Goal:** TeacherFillRatePage: fill rate per slot — bao nhiêu CLAG có GV / tổng CLAG.

**Scope:** `src/pages/TeacherFillRatePage.tsx`

**Components:**
- Filters: date picker, PT dropdown
- Table: slot | total CLAGs | filled (has teacher) | fill rate (%) | progress bar
- Summary: overall fill rate

**Preconditions:** Steps A1-A3 done.

**Detailed Logic:**

1. **Load fill rate:**
```
GET /ori/metrics/fill-rate?date={date}&mypt={pt}
→ Response: List<{ slot, total, filled, rate }>
→ Render MUI Table with LinearProgress bars
→ Color: ≥80% green, 50-80% yellow, <50% red
```

**Tables read:** `bp_oclag`, `bp_usid` (aggregated)

**Test Cases:**
- [ ] Table shows all 16 slots
- [ ] Progress bars reflect fill rate
- [ ] Color coding: red <50%, yellow 50-80%, green ≥80%
- [ ] Summary row shows overall rate
- [ ] Filter by PT works

**Artifacts:** `TeacherFillRatePage.tsx`

**Exit Criteria:** Fill rate page renders correctly with progress bars.

---

### Step C3: Manual Reg4 Trigger (SOW → SCS via shared API)

**Goal:** Từ ClassMatrixPage, thêm button [AUTO MATCH] trigger auto Reg4 matching.

**Scope:** `src/pages/ClassMatrixPage.tsx` (extend)

**Components:**
- [AUTO MATCH] button on ClassMatrixPage toolbar
- Confirmation dialog: "Chạy auto-match cho ngày {date}, PT {pt}?"
- Result toast: "Matched {n}/{total} classes"

**Preconditions:** Step C1 done.

**Detailed Logic:**

1. **Trigger auto-match:**
```
POST /ori/reg4/auto-match?effectiveDate={date}&mypt={pt}
→ Response: { matched, unmatched, total }
→ Toast "Matched {matched}/{total}"
→ Refresh matrix (teacher_code now populated)
```

**Tables write (by backend CL-02):** `bp_usid` (INSERT Reg4), `bp_oclag` (UPDATE teacher_code)

**Test Cases:**
- [ ] Click [AUTO MATCH] → confirm dialog
- [ ] Confirm → API called → result toast shows counts
- [ ] Matrix refreshes → previously unassigned CLAGs now show teacher
- [ ] Matching respects: same PT, same date, same slot, no conflict
- [ ] No Reg3 available → "Matched 0/{total}"

**Artifacts:** `ClassMatrixPage.tsx` (extended)

**Exit Criteria:** Auto-match trigger works from SOW. Matrix updates.

---

## PHASE D: PREPARE

### Step D1: Merge Page — Find Candidates + Execute

**Goal:** MergePage: filter → find merge candidates → select → execute merge.

**Scope:** `src/pages/MergePage.tsx`

**Components:**
- Filters: date, mypt, mygg, mydfdl, slot (all optional except date + mypt)
- [FIND CANDIDATES] button
- Candidates table: source CLAG | target CLAG | students to move | capacity remaining
- [EXECUTE MERGE] button per candidate pair

**Preconditions:** Steps A1-A3 done.

**Detailed Logic:**

1. **Find candidates:**
```
GET /ori/merge/candidates?effectiveDate={date}&mypt={pt}&mygg={gg}&mydfdl={dfdl}&slot={slot}
→ Response: List<MergeCandidate>
→ Render table: source_clag, target_clag, source_students, target_students, target_max
→ Show "Can merge: {source_students} HS → {target_clag} ({target_students}/{target_max})"
```

2. **Execute merge:**
```
POST /ori/merge/execute
Body: { effectiveDate, mypt, mygg, mydfdl, slot }  // merge all candidates in this group
→ Response: { mergedClasses, movedStudents, deletedClasses }
→ Toast "Merged: {movedStudents} HS, {deletedClasses} classes deleted"
→ Refresh candidates (should be empty or reduced)
```

**Backend CL-03 (reference):** Move CUIs from source ULC → target ULC, move OCALGs myclag → target, update counts, soft-delete source.

**Tables write (by backend):** `bp_cui`, `bp_clag`, `bp_oclag`

**Test Cases:**
- [ ] Filter required: date + mypt minimum
- [ ] Candidates table shows mergeable pairs
- [ ] Only same (mypt + mygg + mydfdl + slot) pairs shown
- [ ] C1 and C2 never shown as merge candidates together
- [ ] Execute merge → counts updated → toast
- [ ] After merge, refreshed grid shows fewer CLAGs

**Artifacts:** `MergePage.tsx`

**Exit Criteria:** Merge flow works end-to-end. Students moved correctly.

---

## PHASE E: OPERATE

### Step E1: Class Monitor — View + Student Detail

**Goal:** ClassMonitorPage: live class list cho SO, click row → student detail panel.

**Scope:** `src/pages/ClassMonitorPage.tsx`

**Components:**
- Filters: date (default today), PT, slot
- Summary chips: [Upcoming: N] [In Class: N] [Done: N] [Issue: N]
- Class table: slot, CLAG code, teacher, status (live/waiting/no-teacher), student count, meet link
- Click row → right panel: student list (name, SĐT, reminded status, acceptance dropdown)

**Preconditions:** Steps A1-A3 done.

**Detailed Logic:**

1. **Load classes:**
```
GET /ori/monitor/classes?date={date}&mypt={pt}&time_slot={slot}
→ Response: List<OCLAG flat> (all OCLAG rows for date, grouped by myclag on FE)
→ Group by myclag → 1 row per CLAG
→ Status logic:
  - 🟢 Live: now is within slot time AND teacher_code not null
  - ⏳ Waiting: slot time coming up AND teacher confirmed
  - 🔴 No Teacher: teacher_code is null
  - ✅ Done: all students have acceptance
```

2. **Student detail (click row):**
```
Same data from response (OCLAG rows same myclag)
→ Right panel: DataGrid of students
→ Columns: student_name, myusi (SĐT), status, acceptance
→ [Open Meet] button → window.open(meet_link)
```

**Tables read:** `bp_oclag`

**Test Cases:**
- [ ] Default loads today's classes
- [ ] Summary chips count correctly
- [ ] Click CLAG row → student panel opens
- [ ] Meet link button opens new tab
- [ ] Filter by slot → only that slot's classes
- [ ] Status icons render correctly per logic

**Artifacts:** `ClassMonitorPage.tsx`

**Exit Criteria:** Monitor renders live class data. Student detail panel works.

---

### Step E2: Class Monitor — Accept (Nghiệm thu)

**Goal:** SO nghiệm thu per student: Done/Cancel/Fail/Pass dropdown + save.

**Scope:** `src/pages/ClassMonitorPage.tsx` (extend)

**Components:**
- Student row: acceptance dropdown (Done, Cancel, Fail, Pass)
- [SAVE ACCEPTANCE] button per CLAG
- Confirmation dialog before save

**Preconditions:** Step E1 done.

**Detailed Logic:**

1. **Set acceptance:**
```
Student panel → acceptance dropdown per row →
Click [SAVE ACCEPTANCE] →
PATCH /ori/monitor/classes/{clagCode}/accept
Body: { acceptances: [{ oclag_code, acceptance }] }
→ Backend updates: bp_oclag.acceptance, bp_cui.mybps, bp_ospd.l5_jsu
→ Toast "Đã nghiệm thu"
→ Refresh class list
```

**Tables write (by backend):** `bp_oclag` (acceptance), `bp_cui` (mybps=CLASS_ACCEPTED), `bp_ospd` (l5_jsu++)

**Test Cases:**
- [ ] Dropdown shows: Done, Cancel, Fail, Pass
- [ ] Save without selecting → error
- [ ] Save with selection → API called → toast success
- [ ] L5 count increments for "Done" acceptance
- [ ] Multiple students can be accepted in one save

**Artifacts:** `ClassMonitorPage.tsx` (complete)

**Exit Criteria:** Acceptance flow works. Backend updates OCLAG + CUI + OSPD.

---

## PHASE F: REPORT

### Step F1: Evaluation Page — 6 Criteria Scoring

**Goal:** EvaluationPage: teacher chấm điểm 6 criteria cho từng HS trong lớp.

**Scope:** `src/pages/EvaluationPage.tsx`

**Components:**
- Class selector: CLAG code (from URL param or dropdown)
- Student list: students in this CLAG
- Scoring form per student:
  - Knowledge (40%): Kiến thức cơ bản [1-5], Khả năng vận dụng [1-5]
  - Skill (30%): Kỹ năng giải bài [1-5], Kỹ năng trình bày [1-5]
  - Attitude (30%): Thái độ học tập [1-5], Mức độ tập trung [1-5]
  - Comment textarea
  - Recommendation: C1/C2/C3 radio
- Weighted score display (auto-calculated on FE)
- [GỬI ĐÁNH GIÁ] button

**Preconditions:** Steps A1-A3 done.

**Detailed Logic:**

1. **Load students:**
```
GET /ori/evaluation/class/{clagCode}
→ Response: List<{ student_name, myusi, mygg, mydfdl, cui_code, already_evaluated }>
→ Render list, grey out already evaluated
```

2. **FE score computation (display only, backend recomputes):**
```typescript
const knowledgeAvg = (knowledge[0] + knowledge[1]) / 2;
const skillAvg = (skill[0] + skill[1]) / 2;
const attitudeAvg = (attitude[0] + attitude[1]) / 2;
const weightedScore = knowledgeAvg * 0.4 + skillAvg * 0.3 + attitudeAvg * 0.3;
```

3. **Submit evaluation:**
```
POST /ori/evaluation
Body: EvaluationSubmit {
  cui_code, knowledge: [4, 3], skill: [3, 4], attitude: [5, 4],
  comment: "HS tiến bộ tốt", recommendation: "C2"
}
→ Backend: INSERT 6 bp_cuie (RKN, RSK, RAT per sub-criteria)
→ Backend: UPDATE bp_oclag.status = 'EVALUATED'
→ Response: { weightedScore: 3.8 }
→ Toast "Đã đánh giá: {weightedScore}/5.0"
→ Grey out student in list
```

**Tables write (by backend):** `bp_cuie` (6 events), `bp_oclag` (status), `bp_cui` (mybps)

**Test Cases:**
- [ ] Student list loads correctly
- [ ] All 6 criteria render with 1-5 rating buttons
- [ ] Weighted score auto-calculates as user clicks
- [ ] Submit with all criteria filled → success
- [ ] Submit with missing criteria → validation error
- [ ] Already evaluated student greyed out
- [ ] Weighted formula: 40/30/30 correct

**Artifacts:** `EvaluationPage.tsx`

**Exit Criteria:** Full evaluation flow works. 6 CUIE events created per student.

---

### Step F2: Report Management — Generate + Send

**Goal:** ReportManagementPage: TM generates reports, SALE sends to parents.

**Scope:** `src/pages/ReportManagementPage.tsx`

**Components:**
- Filters: PT, date range, status (EVALUATED / REPORT_GEN / REPORT_SENT)
- Tab 1: "Chưa đánh giá" — students with status < EVALUATED
- Tab 2: "Sẵn sàng tạo report" — status = EVALUATED
- Tab 3: "Sẵn sàng gửi" — status = REPORT_GEN
- Tab 4: "Đã gửi" — status = REPORT_SENT
- [GENERATE REPORTS] button (Tab 2) → batch generate
- [SEND REPORT] button (Tab 3) → per student or batch

**Preconditions:** Step F1 done.

**Detailed Logic:**

1. **Load reports:**
```
GET /ori/report?mypt={pt}&status={status}&dateFrom={from}&dateTo={to}
→ Response: List<{ oclag data + cui data + evaluation status }>
→ Separate into 4 tabs by status
```

2. **Generate reports:**
```
POST /ori/report/generate
Body: { mypt, effectiveDate }
→ Backend CL-05: Creates CTI + CUIE(GRP) per evaluated student
→ Response: { generated: 15 }
→ Toast "Đã tạo {n} reports"
→ Students move from Tab 2 → Tab 3
```

3. **Send reports:**
```
POST /ori/report/send
Body: { cui_code }  (or batch: { cui_codes: [...] })
→ Backend: INSERT CUIE(SRP) + UPDATE status=REPORT_SENT
→ Response: { sent: 1 }
→ Toast "Đã gửi report"
→ Student moves from Tab 3 → Tab 4
```

**Tables write (by backend):** `bp_cti`, `bp_cuie`, `bp_oclag`, `bp_cui`

**Test Cases:**
- [ ] 4 tabs render with correct counts
- [ ] Generate → only EVALUATED students processed
- [ ] Send → only REPORT_GEN students processed
- [ ] Tab counts update after actions
- [ ] Batch generate works for multiple students
- [ ] Filter by PT narrows results

**Artifacts:** `ReportManagementPage.tsx`

**Exit Criteria:** Full report flow: Generate → Send. Status transitions correct.

---

## PHASE G: METRICS

### Step G1: Dashboard KPI Page

**Goal:** DashboardPage: KPI overview + date×slot heatmap với L-levels.

**Scope:** `src/pages/DashboardPage.tsx`

**Components:**
- Filters: date range (default: current month), PT dropdown
- KPI cards: Total Target, L4.2 Registered (%), L5 JSU (%), L8 Settled (%), Budget Used (%)
- Heatmap grid: rows = dates, columns = 16 slots
  - Cell: `{l42}/{target}` with gradient color
  - Hover → tooltip: "L4.2: {n}, L5: {n}, L8: {n}"
- Trend chart (optional): daily L4.2 line chart

**Preconditions:** Steps A1-A3 done.

**Detailed Logic:**

1. **Load dashboard:**
```
GET /ori/metrics/dashboard?date_from={from}&date_to={to}&mypt={pt}
→ Response: {
    summary: { totalTarget, totalL42, totalL5, totalL8, budgetUsed },
    details: { "2026-03-15": { "0900": { target, l42, l5, l8 }, ... }, ... }
  }
→ Render KPI cards from summary
→ Render heatmap from details
```

2. **Heatmap color logic:**
```
rate = l42 / target
Color: rate ≥ 0.8 → green, 0.5-0.8 → yellow, < 0.5 → red, target=0 → grey
```

3. **KPI card logic:**
```
Card 1: Total Target = SUM(target_sessions)
Card 2: L4.2 = SUM(l42) / SUM(target) * 100%
Card 3: L5 = SUM(l5) / SUM(l42) * 100%
Card 4: L8 = SUM(l8) / SUM(l5) * 100%
```

**Tables read:** `bp_ospd` (aggregated)

**Test Cases:**
- [ ] KPI cards show correct percentages
- [ ] Heatmap renders dates × 16 slots
- [ ] Color coding correct: green ≥80%, yellow 50-80%, red <50%
- [ ] Hover tooltip shows L4.2, L5, L8
- [ ] Filter by PT → only that PT's data
- [ ] Date range change → data refreshes

**Artifacts:** `DashboardPage.tsx`

**Exit Criteria:** Dashboard renders with real data. KPI accurate. Heatmap readable.

---

## Summary: Step Order & Dependencies

```
A1 (Scaffold) → A2 (Config) → A3 (Types)
                                    ↓
    ┌───────────────────────────────┴────────────────────────────────┐
    ↓                               ↓                                ↓
B1 (Plan List)                  B3 (Slot Lock)                  B6 (Schedule)
    ↓                               ↓
B2 (Plan Grid)                  B4 (Booking Avail)              C1 (Class Matrix)
                                    ↓                                ↓
                                B5 (Booking Submit)             C2 (Fill Rate)
                                                                     ↓
                                                                C3 (Auto Match)

D1 (Merge) ← independent after A3

E1 (Monitor View) → E2 (Monitor Accept)

F1 (Evaluation) → F2 (Report Mgmt)

G1 (Dashboard) ← independent after A3
```

**Critical path:** A1 → A2 → A3 → B4 → B5 (booking is the core flow, must work first)

**Parallel tracks after A3:**
- Track 1: B1 → B2 (Settle Plan)
- Track 2: B3 → B4 → B5 (Slot Lock + Booking)
- Track 3: B6 → C1 → C2 → C3 (Schedule + Matrix + Match)
- Track 4: E1 → E2 (Monitor)
- Track 5: F1 → F2 (Evaluation + Report)
- Track 6: D1 (Merge — standalone)
- Track 7: G1 (Dashboard — standalone)

---

## Checklist tổng hợp

| Step | Name | Est. | Tables | API Endpoints |
|------|------|------|--------|--------------|
| A1 | Scaffold + Auth + Layout | 4h | — | — |
| A2 | ORI Config Constants | 1h | — | — |
| A3 | TypeScript Interfaces | 2h | — | — |
| B1 | Settle Plan List + Create | 4h | bp_osph, bp_ospd | #1-7 |
| B2 | Settle Plan Grid + Excel | 4h | bp_ospd | #3,5,8-10 |
| B3 | Slot Lock | 3h | bp_ospl | #11-12 |
| B4 | Booking Availability | 3h | bp_ospd, bp_ospl, bp_oclag | #13 |
| B5 | Booking Submit | 3h | bp_oclag, bp_clag, bp_ulc, bp_cui | #14 |
| B6 | Schedule Page | 4h | bp_oclag, bp_clag | #15-17 |
| C1 | Class Matrix | 3h | bp_oclag, bp_usid | #18 |
| C2 | Teacher Fill Rate | 2h | bp_oclag, bp_usid | #30 |
| C3 | Auto Match Trigger | 1h | bp_usid, bp_oclag | #21 |
| D1 | Merge | 3h | bp_oclag, bp_clag, bp_cui | #22-23 |
| E1 | Monitor View | 3h | bp_oclag | #19 |
| E2 | Monitor Accept | 2h | bp_oclag, bp_cui, bp_ospd | #20 |
| F1 | Evaluation | 4h | bp_cuie, bp_oclag, bp_cui | #24-25 |
| F2 | Report Management | 3h | bp_cti, bp_cuie, bp_oclag | #26-28 |
| G1 | Dashboard KPI | 4h | bp_ospd, bp_oclag | #29 |
| **Total** | | **~53h** | | **30 endpoints** |

---

> **End of Incremental Steps Plan**
> Reference: ArchitecturePack_ORI_v2.md for entity details, process flows, and complex logic pseudocode.
