# Architecture Pack V2 — Clevai ORI System

> **Version:** 2.0
> **Date:** 2026-03-14
> **Status:** DRAFT — Pending review
> **Scope:** ORI (Orientation / Trial Class) full lifecycle across SOW + TEW + SCS + bp-log-service

---

## Table of Contents

1. [Entity Relationship Description](#1-entity-relationship-description)
2. [Process Description](#2-process-description)
3. [UI Wireframe](#3-ui-wireframe)
4. [User Flows](#4-user-flows)
5. [Complex Logic](#5-complex-logic)
6. [Feature & Layer](#6-feature-layer)
7. [Design Suggestions & Risks](#7-design-suggestions--risks)

---

# 1. Entity Relationship Description

## 1.1 Entity Overview

ORI sử dụng **4 bảng ORI-specific** (OSPH, OSPD, OSPL, OCLAG) + **tái sử dụng 13 bảng platform** hiện có. Không tạo entity mới.

### Layer A — Reference / Master (dùng nguyên, không sửa)

| # | Entity | Table | Vai trò trong ORI | Ghi chú |
|---|--------|-------|-------------------|---------|
| A1 | ProductType | `bp_pt_producttype` | 7 PT cho ORI: KMA_DU_8, KMA_TT_8, KEN_DU_8, KEN_TT_8, KEN_ES_8, KSP_PS_8, KFP_FO_8 | code = varchar, dùng làm FK |
| A2 | GradeGroup | `bp_gg_gradegroup` | Khối lớp: G1-G12, K4, K5 | mycashsta = default CASHSTA theo khối (không dùng cho ORI) |
| A3 | DifficultyGrade | `bp_dfdl_difficultygrade` | ORI dùng: C1, C2 | C1=Cơ bản, C2=Nâng cao. HS C1 và C2 KHÔNG học chung lớp |
| A4 | WeeklyHourOption | `bp_who_weeklyhouroption` | Ca học cho Plan (OSPD.mywho) | 16 slot: 0900→2120. Format HHMM |
| A5 | CalShiftStart | `bp_cashsta_calshiftstart` | Ca học cho Teacher + OCLAG | 50+ slot. ORI dùng: 0900,0930,...2120. USID.mycashsta + OCLAG.ori_time_slot trỏ vào đây |
| A6 | UserItem | `bp_usi_useritem` | User (GV, staff). ORI: GV = myusi trong USID | HS trial KHÔNG tạo USI — dùng SĐT làm identifier trên OCLAG |
| A7 | LearningComponentType | `bp_lct_learningcomponenttype` | ORI types: ORIO-25MI (SH), ORIO-30MI (SH), ORI-25MI (SS), ORI-30MI (SS) | ULC ORI dùng ORIO-30MI hoặc ORIO-25MI |

### Layer B — ORI Config (4 bảng ORI-specific)

| # | Entity | Table | Vai trò | PK pattern |
|---|--------|-------|---------|------------|
| B1 | OriSettlePlanHeader | `bp_osph_ori_settle_plan_header` | Header plan tháng: PT, version, plan_type, budget | `ORI-{PT}-{YYYYMM}-V{n}` |
| B2 | OriSettlePlanDetail | `bp_ospd_ori_settle_plan_detail` | Chi tiết plan: ngày × slot → target + L-levels | `OSPD-{uuid}` |
| B3 | OriSlotLock | `bp_ospl_ori_slot_lock` | Lock/unlock slot booking | Composite: (planDate, oriTimeSlot, mypt) |
| B4 | **OriClagSchedule** | `bp_oclag_ori_clag_schedule` | **Bảng trung tâm ORI: 1 row = 1 học sinh trial** | `OCLAG-{uuid}` |

### Layer C — Transaction (tái sử dụng platform, không sửa schema)

| # | Entity | Table | Vai trò trong ORI | Ghi chú |
|---|--------|-------|-------------------|---------|
| C1 | ClassGroup | `bp_clag_classgroup` | Lớp ORI: clagtype=DYN | **Không có cột published**. Không có cột date — date nằm ở OCLAG |
| C2 | UniqueLC | `bp_ulc_uniquelearningcomponent` | Buổi học ORI: mylct=ORIO-30MI | 1 CLAG = 1 ULC (1:1). **Không có cột myusi** |
| C3 | ClagUlc | `bp_clag_ulc` | Junction CLAG ↔ ULC | ORI: luôn 1:1 |
| C4 | ContentUserUlcInstance | `bp_cui_content_user_ulc_instance` | Instance HS/GV trong buổi học | myusi = SĐT (HS) hoặc teacher_code (GV). mybps track status |
| C5 | UserDuty | `bp_usid_usiduty` | Đăng ký/phân công GV | reg_type: 1=Reg1, 3=Reg3, 4=Reg4. mycashsta = ca học |
| C6 | CuiEvent | `bp_cuie_cuievent` | Event log per CUI | mylcet = event type (RKN, RSK, RAT, RCM, RRC, RSC, GRP, SRP...) |
| C7 | VcrMeeting | `bp_vcr_meeting` | Google Meet room | 1 per CLAG confirmed |
| C8 | UsiVcrMeeting | `bp_usi_vcr_meeting` | Join link per participant | 1 per student + 1 per teacher |
| C9 | ContentItem | `bp_cti_contentitem` | Report PDF/content | myctt = 'ORI-REPORT' |
| C10 | PodEgt | `bp_pod_egt` | Evaluation score | egt = grade (A/B/C/D). **Không có POD cho HS trial** → cần xem lại |
| C11 | BppProcess | `bp_bpp_process` | Workflow tracker | BPPOrientation |
| C12 | BpsStep | `bp_bps_step` | Workflow step | Track từng phase |
| C13 | BpeEvent | `bp_bpe_event` | Workflow event | Track từng action |

### Views (tái sử dụng)

| View | Source | Vai trò trong ORI |
|------|--------|-------------------|
| `vw_teacher_commitment` | USID Reg4 + USI | TEW lịch dạy: GV thấy ORI commitment sau Reg4 |
| `vw_teacher_commitment_grouping` | Grouped version | TEW lịch dạy grouped by date+slot |

---

## 1.2 OCLAG — Bảng trung tâm ORI (Chi tiết)

`bp_oclag_ori_clag_schedule` — **1 row = 1 học sinh trial trong 1 lớp**

| # | Cột | Type | Nullable | Default | Source | Mô tả |
|---|-----|------|----------|---------|--------|-------|
| 1 | `id` | int | NO | auto | System | PK |
| 2 | `code` | varchar(100) | NO | | System | `OCLAG-{uuid}` |
| 3 | `myclag` | varchar(255) | NO | | Booking | FK → CLAG.code. Nhiều OCLAG có thể cùng myclag (nhiều HS cùng lớp) |
| 4 | `effective_date` | date | NO | | Booking | Ngày học thử |
| 5 | `ori_time_slot` | varchar(20) | NO | | Booking | Ca học — **CASHSTA code** (0900, 1430, 1845...) |
| 6 | `mypt` | varchar(128) | NO | | Booking | PT: KMA_DU_8, KEN_DU_8... |
| 7 | `myusi` | varchar(20) | YES | | Booking | **SĐT học sinh** — identifier thay USI. CUI.myusi trỏ vào giá trị này |
| 8 | `student_name` | varchar(100) | YES | | Booking | Tên HS |
| 9 | `mygg` | varchar(10) | YES | | Booking | Khối: G3-G12 |
| 10 | `mydfdl` | varchar(10) | YES | C2 | Booking | Trình độ: C1/C2 |
| 11 | `teacher_code` | varchar(50) | YES | | Reg4 sync | GV chính (USID.myusi khi reg_type=4, position=MAIN) |
| 12 | `teacher_name` | varchar(100) | YES | | Reg4 sync | Tên GV chính |
| 13 | `backup_code` | varchar(50) | YES | | Backup sync | GV backup |
| 14 | `backup_name` | varchar(100) | YES | | Backup sync | Tên GV backup |
| 15 | `meet_link` | varchar(255) | YES | | Meeting sync | Google Meet link |
| 16 | `meet_code` | varchar(50) | YES | | Meeting sync | VCR code |
| 17 | `student_number` | int | YES | 1 | Booking | STT HS trong CLAG |
| 18 | `maxtotalstudents` | int | YES | 5 | PT config | Max theo PT (Math=5, English=2) |
| 19 | `status` | varchar(30) | YES | BOOKED | Flow | BOOKED → REMINDED → CLASS_ACCEPTED → EVALUATED → REPORT_GEN → REPORT_SENT |
| 20 | `acceptance` | varchar(20) | YES | | SO accept | Nghiệm thu: Done/Cancel/Fail/Pass |
| 21 | `sale_code` | varchar(50) | YES | | Booking | Mã SALE đã book |
| 22 | `booking_time` | datetime | YES | | Booking | Thời điểm book |
| 23 | `source` | varchar(50) | YES | | Booking | `Crossell` (HS mới) / `D` (HS cũ thử SP mới) |
| 24 | `cancel_reason` | varchar(255) | YES | | Cancel | Lý do hủy |
| 25 | `note` | varchar(255) | YES | | Booking | Ghi chú |
| 26 | `zalo_link` | varchar(255) | YES | | Prepare | Link nhóm Zalo |
| 27 | `ctv_so` | varchar(50) | YES | | Booking | CTV/SO phụ trách |
| 28 | `published` | bit(1) | YES | 1 | System | Soft delete |
| 29 | `created_by` | varchar(100) | YES | | System | Người tạo |
| 30 | `created_at` | datetime | NO | | System | |
| 31 | `updated_at` | datetime | NO | | System | |

**Đề xuất thay đổi so với DB hiện tại:**
- Đổi tên `student_phone` → `myusi` (đồng nhất naming convention)
- Bỏ `report_status` (nghiệm thu dùng `acceptance`, report tracking dùng `status`)

**OCLAG Sync Rules:**

| Event | Cột OCLAG cần update | Source |
|-------|---------------------|--------|
| Booking | Tạo row mới: tất cả cột booking | Form booking SOW |
| Reg4 auto-match | `teacher_code`, `teacher_name` | USID Reg4 MAIN |
| Backup assign | `backup_code`, `backup_name` | USID Reg4 BACKUP |
| Meeting create | `meet_link`, `meet_code` | VCR |
| SO accept | `acceptance` | ClassMonitor SOW |
| Status change | `status` | Theo flow CUI.mybps |
| Cancel | `status`=CANCELLED, `cancel_reason` | Cancel action |
| Merge | `myclag` (target), `student_number` (recalc) | Merge logic |

---

## 1.3 OSPH — Settle Plan Header

`bp_osph_ori_settle_plan_header`

| Cột | Type | Mô tả | Example |
|-----|------|-------|---------|
| `code` | varchar | PK | `ORI-KMA_DU_8-202606-V1` |
| `mypt` | varchar | PT code | `KMA_DU_8` |
| `plan_month` | date | Tháng plan (1st of month) | `2026-06-01` |
| `version` | int | Version number | 1, 2, 3... |
| `plan_type` | varchar | `SALE` / `VH` | VH = vận hành (dùng cho booking availability) |
| `status` | varchar | `DRAFT` → `ACTIVE` → `CLOSED` | |
| `total_target` | int | Tổng session target tháng | 485 |
| `monthly_budget` | decimal | Ngân sách tháng (VND) | 150000000 |
| `cost_per_session` | decimal | Chi phí/session theo PT | KMA=90000, KEN=45000, KFP=35000 |
| `forecast_revenue` | decimal | Doanh thu dự kiến | |
| `expected_conversion` | decimal | Tỷ lệ chuyển đổi target | 0.25 |
| `created_by` | varchar | SALE tạo | |
| `approved_by` | varchar | TM approve | |
| `approved_at` | datetime | | |
| `published` | bit(1) | Soft delete | |

**Ràng buộc:**
- 1 PT × 1 tháng có thể có nhiều version, chỉ 1 ACTIVE
- Khi activate version mới → version cũ tự CLOSED
- VH plan là source of truth cho booking availability
- SALE plan dùng cho tracking doanh số

---

## 1.4 OSPD — Settle Plan Detail

`bp_ospd_ori_settle_plan_detail`

| Cột | Type | Mô tả |
|-----|------|-------|
| `code` | varchar | PK: `OSPD-{uuid}` |
| `myosph` | varchar | FK → OSPH.code |
| `plan_date` | date | Ngày cụ thể |
| `mywho` | varchar | Ca — **WHO code** (0900, 0930...). Backend map sang CASHSTA khi tính L-level |
| `target_sessions` | int | Target số session |
| `l42_registered` | int | L4.2: Đã book (cron update từ OCLAG count) |
| `l5_jsu` | int | L5: Đã join (acceptance=Done) |
| `l8_settled` | int | L8: Đã settle (status=REPORT_SENT) |
| `teacher_available` | int | Số GV Reg3 sẵn sàng slot này |
| `published` | bit(1) | |

**WHO ↔ CASHSTA mapping** (backend config):
```
1840 (WHO/OSPD) ↔ 1845 (CASHSTA/OCLAG)
1920 (WHO/OSPD) ↔ 1930 (CASHSTA/OCLAG)
// Tất cả slot khác: WHO = CASHSTA (match tự nhiên)
```

---

## 1.5 CLAG — Class Group

`bp_clag_classgroup` (platform table, tái sử dụng)

| Cột quan trọng | Ý nghĩa trong ORI |
|----------------|-------------------|
| `code` | `CLAG-ORI-{uuid}` |
| `mypt` | PT code |
| `mygg` | Khối |
| `mydfdl` | Trình độ (C1/C2) — **bắt buộc match khi xếp HS** |
| `mywho` | Ca — **WHO code** |
| `mywso` | Day-of-week — NOT NULL (convention Clevai) |
| `clagtype` | `DYN` (dynamic) cho ORI |
| `maxtotalstudents` | Math=5, English=2 |
| `student_number` | Số HS hiện tại |
| `active` | binary(1) |

**Không có:** `published`, `effective_date`. Date context = OCLAG.effective_date.

**Max students by PT:**

| PT | Code | Max |
|----|------|-----|
| Toán DU | KMA_DU_8 | 5 |
| Toán TT | KMA_TT_8 | 5 |
| Tiếng Anh DU | KEN_DU_8 | 2 |
| Tiếng Anh TT | KEN_TT_8 | 2 |
| Early Speak | KEN_ES_8 | 2 |
| Phil Speak | KSP_PS_8 | 2 |
| Toán FO | KFP_FO_8 | 5 |

---

## 1.6 ULC — Unique Learning Component

`bp_ulc_uniquelearningcomponent` (platform table, tái sử dụng)

| Cột quan trọng | Ý nghĩa trong ORI |
|----------------|-------------------|
| `code` | `ULC-ORI-{uuid}` |
| `mylct` | `ORIO-30MI` hoặc `ORIO-25MI` |
| `mypt` | PT code |
| `mygg` | Khối |
| `mydfdl` | Trình độ |
| `mycap` | Capacity string (Clevai convention) |
| `published` | bit(1) |

**Không có:** `myusi`, `myclag`. Link tới CLAG qua `bp_clag_ulc` junction.

**ORI: 1 CLAG = 1 ULC** (1:1 qua CLAG_ULC). ULC tạo 1 lần khi tạo CLAG mới, không tạo thêm.

---

## 1.7 CUI — Content User ULC Instance

`bp_cui_content_user_ulc_instance` (platform table, tái sử dụng)

| Cột | Ý nghĩa trong ORI |
|-----|-------------------|
| `code` | `CUI-ORI-{uuid}` |
| `myusi` | **HS:** SĐT (= OCLAG.myusi). **GV:** teacher_code (= USID.myusi) |
| `myulc` | FK → ULC.code |
| `mybps` | Status: `BOOK_ORI` → `DONE` / `PASS` / `FAIL` |
| `mycti` | FK → CTI.code (report) |
| `published` | bit(1) |

**CUI trong ORI có 2 loại:**
1. **CUI cho HS** — tạo khi booking. myusi = SĐT từ OCLAG.myusi
2. **CUI cho GV** — tạo khi schedule (sau Reg4 confirm). myusi = teacher_code

**TEW lịch dạy** query CUI JOIN ULC để hiện schedule → cần CUI cho GV để ORI hiện trên TEW.

---

## 1.8 USID — User Duty (Teacher Registration/Assignment)

`bp_usid_usiduty` (platform table, tái sử dụng)

| register_type | Tên | Actor | Ý nghĩa | Key fields |
|--------------|------|-------|---------|------------|
| **1** | Reg1 | Admin | GV đăng ký dạy PT+ca nào (1 lần) | myusi, mypt, mycashsta, mygg, mydfdl |
| **3** | Reg3 | GV (TEW) | GV đăng ký rảnh ngày+ca cụ thể | myusi, mypt, mycashsta, effective_date |
| **4** | Reg4 | Auto/TM | Phân công GV vào lớp | myusi, myclag, mypt, mycashsta, position (MAIN/BACKUP), isapproved, effective_date |

**USID ORI fields:**

| Cột | Type | Mô tả |
|-----|------|-------|
| `code` | varchar(256) | PK. Reg4: `USID-R4-{uuid}` |
| `myusi` | varchar(128) | Teacher code: `te22`, `phuclive`... |
| `myclag` | varchar(128) | FK → CLAG.code (Reg4 only) |
| `mypt` | varchar(128) | PT code |
| `mycashsta` | varchar(128) | Ca — **CASHSTA code** |
| `register_type` | int | 1/3/4 |
| `position` | varchar(45) | MAIN / BACKUP (Reg4 only) |
| `isapproved` | bit(1) | GV đã confirm? (Reg4/Reg48) |
| `effective_date` | date | Ngày áp dụng |
| `allocated_myusi` | varchar | Ai assign: 'SYSTEM' (auto) hoặc TM code (manual) |

---

## 1.9 Entity Relationship Diagram (Text)

```
OSPH ──1:N──> OSPD (plan header → daily×slot details)
                     ↕ L-level sync (backend map WHO↔CASHSTA)
OCLAG (1 row = 1 HS) ──N:1──> CLAG ──1:1──> CLAG_ULC ──1:1──> ULC
  │                              │
  │ myusi (SĐT)                  │ myclag
  ↓                              ↓
CUI (1 per HS + 1 per GV) ──N:1──> ULC
  │
  ├──> CUIE (event log: RKN, RSK, RAT, RCM, RRC, RSC, GRP, SRP)
  └──> CTI (report content)

USID (Reg1/3/4) ──> teacher assignment
  │ myclag (Reg4)
  ↓
CLAG ←── sync teacher_code/backup_code to OCLAG

VCR (meeting) ──1:N──> USI_VCR (join links per participant)
  │ sync meet_link/meet_code to OCLAG

OSPL (slot lock) ── check trước booking

vw_teacher_commitment ← VIEW từ USID Reg4 → TEW hiện lịch
```

---

## 1.10 Bảng tham chiếu nhanh

### PT Config

| PT Code | Tên | Max HS | TEC Rate (VND) | LCT |
|---------|-----|--------|----------------|-----|
| KMA_DU_8 | Toán DU | 5 | 90,000 | ORIO-30MI |
| KMA_TT_8 | Toán TT | 5 | 90,000 | ORIO-30MI |
| KEN_DU_8 | TA DU | 2 | 45,000 | ORIO-30MI |
| KEN_TT_8 | TA TT | 2 | 45,000 | ORIO-30MI |
| KEN_ES_8 | Early Speak | 2 | 45,000 | ORIO-25MI |
| KSP_PS_8 | Phil Speak | 2 | 45,000 | ORIO-25MI |
| KFP_FO_8 | Toán FO | 5 | 35,000 | ORIO-30MI |

### 16 ORI Time Slots (OSPD dùng WHO, OCLAG/USID dùng CASHSTA)

| # | WHO (OSPD) | CASHSTA (OCLAG/USID) | Giờ bắt đầu |
|---|-----------|---------------------|-------------|
| 1 | 0900 | 0900 | 09:00 |
| 2 | 0930 | 0930 | 09:30 |
| 3 | 1000 | 1000 | 10:00 |
| 4 | 1030 | 1030 | 10:30 |
| 5 | 1400 | 1400 | 14:00 |
| 6 | 1430 | 1430 | 14:30 |
| 7 | 1500 | 1500 | 15:00 |
| 8 | 1530 | 1530 | 15:30 |
| 9 | 1700 | 1700 | 17:00 |
| 10 | 1730 | 1730 | 17:30 |
| 11 | 1800 | 1800 | 18:00 |
| 12 | 1840 | **1845** | 18:40/18:45 |
| 13 | 1920 | **1930** | 19:20/19:30 |
| 14 | 2000 | 2000 | 20:00 |
| 15 | 2040 | 2040 | 20:40 |
| 16 | 2120 | 2120 | 21:20 |

### CUI Status Flow (mybps)

```
BOOK_ORI → REMINDED → CLASS_ACCEPTED → EVALUATED → REPORT_GEN → REPORT_SENT
                                                                      ↓
                                            DONE / PASS / FAIL (final states)
```

### OCLAG Status Flow

```
BOOKED → REMINDED → CLASS_ACCEPTED → EVALUATED → REPORT_GEN → REPORT_SENT
   ↓
CANCELLED (at any point, with cancel_reason)
```

---

# 2. Process Description

## 2.0 Process Overview — 5 Phases, 17 Steps

```
Phase 1: FORECAST  (S1-S3)  — Plan + Book
Phase 2: MATCH     (S4-S7)  — Teacher Registration + Assignment
Phase 3: PREPARE   (S8-S11) — Meeting + Merge + Notify
Phase 4: OPERATE   (S12-S14)— Monitor + Accept
Phase 5: REPORT    (S15-S17)— Evaluate + Report + Send
```

## 2.1 Phase 1: FORECAST

### S1-S2: Settle Plan Creation

| Step | Action | Actor | System | FE | BE Endpoint | DB Write |
|------|--------|-------|--------|-----|------------|----------|
| S1.1 | Tạo plan header | SALE | SOW | SettlePlanPage | `POST /ori/settle-plan/headers` | INSERT `bp_osph` |
| S1.2 | Generate grid ngày×slot | SALE | SOW | SettlePlanPage | `POST /ori/settle-plan/headers/{code}/generate` | INSERT 16×N rows `bp_ospd` |
| S1.3 | Fill target (manual hoặc Excel import) | SALE | SOW | SettlePlanPage | `PATCH /ori/settle-plan/details-batch` hoặc `POST .../import` | UPDATE `bp_ospd.target_sessions` |
| S1.4 | Activate plan | SALE/TM | SOW | SettlePlanPage | `POST /ori/settle-plan/headers/{code}/activate` | UPDATE `bp_osph.status`=ACTIVE, version cũ=CLOSED |
| S1.5 | New version | SALE | SOW | SettlePlanPage | `POST /ori/settle-plan/headers/{code}/new-version` | INSERT new `bp_osph` + clone `bp_ospd` |

**Volume:** ~248 OSPD rows/version/PT (31 days × 8 slots avg). ~8 versions/month.

### S3: Student Booking

| Step | Action | Actor | FE | BE Endpoint | DB Write | Logic |
|------|--------|-------|-----|------------|----------|-------|
| S3.1 | Check availability | SALE | BookingPage | `GET /ori/booking/availability?date&mypt` | READ `bp_ospd` (VH plan), `bp_ospl` (locks), COUNT `bp_oclag` | Return: [{slot, target, registered, available, locked}] |
| S3.2 | Submit booking | SALE | BookingPage | `POST /ori/booking` | See below | CL-01 algorithm |

**S3.2 Booking write chain:**

```
1. CHECK bp_ospl: slot not locked
2. CHECK bp_ospd: VH plan capacity available
3. FIND bp_oclag: existing CLAG match (mypt+mygg+mydfdl+slot+date, student_number < max)
   → If not found: INSERT bp_clag_classgroup (CLAG-ORI-{uuid})
                    INSERT bp_ulc (ULC-ORI-{uuid})
                    INSERT bp_clag_ulc
4. INSERT bp_oclag (1 row for this student)
5. INSERT bp_cui (myusi = SĐT from OCLAG.myusi, myulc = ULC.code, mybps = BOOK_ORI)
6. UPDATE bp_clag.student_number++
7. UPDATE bp_ospd.l42_registered++ (map CASHSTA→WHO)
```

**Source field logic:**
- `Crossell` = HS mới (chưa từng học Clevai)
- `D` = HS cũ thử sản phẩm mới

---

## 2.2 Phase 2: MATCH

### S4: Reg3 — Teacher Availability Registration

| Step | Action | Actor | FE | BE Endpoint | DB Write |
|------|--------|-------|-----|------------|----------|
| S4.1 | GV chọn PT + ngày + ca | OTE | TEW CalendarRegistration | `POST /ori/reg3/register` | INSERT `bp_usid` (register_type=3, mycashsta, effective_date) |
| S4.2 | GV hủy đăng ký | OTE | TEW CalendarRegistration | `DELETE /ori/reg3/{code}` | UPDATE `bp_usid.submitted_cancel_at` |

**API path:** `/bp-log/ori/reg3/*` (riêng cho ORI, không dùng `/lms/teaching-schedule-*`)

**Validation:**
- Check Reg1 tồn tại cho teacher+PT+cashsta (GV phải đăng ký Reg1 trước)
- Check không duplicate (cùng teacher+date+slot)

### S5: Auto Reg4 — System Teacher Matching

| Step | Action | Actor | Trigger | DB Write | Logic |
|------|--------|-------|---------|----------|-------|
| S5.1 | Tìm CLAG chưa có GV | CRON | Every 30min 09-21h | READ `bp_oclag` LEFT JOIN `bp_usid` (reg4) | CL-02 |
| S5.2 | Match Reg3 → Reg4 | CRON | | INSERT `bp_usid` (reg_type=4, position=MAIN, isapproved=0) | Priority scoring |
| S5.3 | Sync teacher to OCLAG | CRON | | UPDATE `bp_oclag.teacher_code`, `teacher_name` | Sync rule |

**Manual trigger:** `POST /ori/reg4/auto-match?date&mypt` (từ SCS)

### S6: Reg4 Confirm (Reg48)

| Step | Action | Actor | FE | Mechanism | DB Write |
|------|--------|-------|-----|-----------|----------|
| S6.1 | GV thấy assignment | OTE | TEW | `GET /teaching/teacher-get-commitments` → reads `vw_teacher_commitment` | READ only |
| S6.2 | GV confirm | OTE | TEW | `POST /teaching/teacher-confirm` | UPDATE `bp_usid.isapproved=1, approved_at=NOW()` |
| S6.3 | GV reject | OTE | TEW | `POST /teaching/teacher-confirm` (reject) | UPDATE `bp_usid.approved_cancel_at=NOW()` → trigger re-match |

**Sau confirm → Schedule action:** Tạo CUI cho GV (myusi=teacher_code, myulc=ULC.code). Để TEW lịch dạy hiện ORI class.

### S7: Backup Teacher Assignment

| Step | Action | Actor | FE | BE Endpoint | DB Write |
|------|--------|-------|-----|------------|----------|
| S7.1 | TM assign backup | TM | SCS OriReg4Page | `POST /bp-log/usid` (position=BACKUP) | INSERT `bp_usid` (reg_type=4, position=BACKUP, isapproved=1) |
| S7.2 | Sync to OCLAG | System | | | UPDATE `bp_oclag.backup_code`, `backup_name` |

---

## 2.3 Phase 3: PREPARE

### S8: Meeting Creation

| Step | Action | Trigger | DB Write |
|------|--------|---------|----------|
| S8.1 | Create VCR per confirmed CLAG | CRON daily 16:00 | INSERT `bp_vcr_meeting` |
| S8.2 | Create USI_VCR per participant | | INSERT `bp_usi_vcr_meeting` (1 per student + 1 per teacher) |
| S8.3 | Sync to OCLAG | | UPDATE `bp_oclag.meet_link`, `meet_code` |

**Condition:** CLAG có teacher confirmed (USID.isapproved=1)

### S9-S11: Notify + Send Link

| Step | Action | Actor | DB Write |
|------|--------|-------|----------|
| S9 | Save Zalo link | SALE | UPDATE `bp_oclag.zalo_link` |
| S11 | Send meet link | System/SALE | INSERT `bp_cuie` (mylcet=BF-NT-SLK) + UPDATE `bp_oclag.status`=REMINDED |

### S10: Merge Underfull Classes

| Step | Action | Trigger | Logic | DB Write |
|------|--------|---------|-------|----------|
| S10.1 | Find merge candidates | CRON daily 15:00 hoặc manual từ FE | CL-03: Group by (mypt+mygg+mydfdl+slot), filter từ FE | |
| S10.2 | Execute merge | | Move CUI từ source → target CLAG | UPDATE `bp_cui.myulc`, UPDATE `bp_clag.student_number`, UPDATE `bp_oclag.myclag` |
| S10.3 | Soft-delete source | | | UPDATE source `bp_oclag.published=0`, source `bp_clag.student_number=0` |

**Merge từ FE:** SOW MergePage gửi filter (date, mypt, mygg, mydfdl, slot) → BE tìm candidates theo filter đó → execute.

---

## 2.4 Phase 4: OPERATE

### S12: Call Remind

| Step | Action | Actor | FE | DB Write |
|------|--------|-------|-----|----------|
| S12.1 | SO gọi HS trước 20 phút | SO/SALE | ClassMonitorPage | INSERT `bp_cuie` (BF-NT-CRS) |
| S12.2 | Update status | | | UPDATE `bp_oclag.status`=REMINDED, `bp_cui.mybps`=REMINDED |

### S13: Class Monitor

| Step | Action | Actor | FE | BE Endpoint |
|------|--------|-------|-----|------------|
| S13.1 | View live classes | SO | ClassMonitorPage | `GET /ori/monitor/classes?date&mypt&time_slot` |
| S13.2 | View students | SO | ClassMonitorPage (detail panel) | Response includes OCLAG flat data |
| S13.3 | CallTE escalation | SO/TM | ClassMonitorPage ([CallTE]) | Manual → CL-04 |

### S14: Accept (Nghiệm thu)

| Step | Action | Actor | FE | BE Endpoint | DB Write |
|------|--------|-------|-----|------------|----------|
| S14.1 | SO nghiệm thu per HS | SO | ClassMonitorPage | `PATCH /ori/monitor/classes/{clagCode}/accept` | UPDATE `bp_oclag.acceptance` per student |
| S14.2 | Update CUI | | | | UPDATE `bp_cui.mybps`=CLASS_ACCEPTED |
| S14.3 | Update L5/L8 | | | | UPDATE `bp_ospd.l5_jsu++` (if acceptance=Done) |

**Nghiệm thu values:** Done (OK), Cancel (hủy), Fail (fail chất lượng), Pass (TM approve)

---

## 2.5 Phase 5: REPORT

### S15: Teacher Evaluation

| Step | Action | Actor | FE | BE Endpoint | DB Write |
|------|--------|-------|-----|------------|----------|
| S15.1 | GV chấm điểm 6 criteria | OTE | EvaluationPage (SOW) | `POST /ori/evaluation` | INSERT 6 `bp_cuie` events |

**6 Evaluation Criteria (3 nhóm × 2 sub):**

| Nhóm | Weight | Sub-criteria 1 | Sub-criteria 2 |
|------|--------|---------------|---------------|
| Knowledge | 40% | Kiến thức cơ bản | Khả năng vận dụng |
| Skill | 30% | Kỹ năng giải bài | Kỹ năng trình bày |
| Attitude | 30% | Thái độ học tập | Mức độ tập trung |

**Score:** Mỗi sub-criteria 1-5 → trung bình nhóm → weighted average = final score.

### S16: Report Generation

| Step | Action | Actor | FE | BE Endpoint | DB Write |
|------|--------|-------|-----|------------|----------|
| S16.1 | TM trigger batch gen | TM | ReportManagementPage | `POST /ori/report/generate` | INSERT `bp_cti` + INSERT `bp_cuie` (GRP) |
| S16.2 | Update status | | | | UPDATE `bp_oclag.status`=REPORT_GEN, `bp_cui.mybps`=REPORT_GEN |

### S17: Send Report

| Step | Action | Actor | FE | BE Endpoint | DB Write |
|------|--------|-------|-----|------------|----------|
| S17.1 | SALE gửi report cho PH | SALE | ReportManagementPage | `POST /ori/report/send` | INSERT `bp_cuie` (SRP) |
| S17.2 | Update status | | | | UPDATE `bp_oclag.status`=REPORT_SENT, `bp_cui.mybps`=REPORT_SENT |

---

## 2.6 Cron Jobs (5 jobs, chạy trên bp-log-service)

| Job | Schedule (UTC) | Local (GMT+7) | Action | Tables affected |
|-----|---------------|---------------|--------|----------------|
| Auto Reg4 Match | `0 0/30 2-14 * * *` | Every 30 min 09-21h | CL-02: Match Reg3→Reg4 | INSERT `bp_usid`, UPDATE `bp_oclag` |
| Auto Merge | `0 0 8 * * *` | Daily 15:00 | CL-03: Merge underfull | UPDATE `bp_cui`, `bp_clag`, `bp_oclag` |
| Meeting Creation | `0 0 9 * * *` | Daily 16:00 | Create VCR | INSERT `bp_vcr`, `bp_usi_vcr`, UPDATE `bp_oclag` |
| L-Level Update | fixedDelay 900s | Every 15 min | Recompute OSPD metrics | UPDATE `bp_ospd` (l42, l5, l8) |
| Settle Plan Alert | `0 0 2 * * *` | Daily 09:00 | Warn if attainment <80% | READ only |

---

# 3. UI Wireframe

## 3.0 Screen Inventory — 3 Systems, 12 Screens

| # | Screen | System | Phase | Actor | Path |
|---|--------|--------|-------|-------|------|
| 1 | Settle Plan Dashboard | SOW | FORECAST | SALE, TM | `/settle-plan` |
| 2 | Slot Lock Management | SOW | FORECAST | TM | `/slot-lock` |
| 3 | Booking Form | SOW | FORECAST | SALE | `/booking` |
| 4 | Schedule (Class Matrix) | SOW | FORECAST | SALE, TM, SO | `/schedule` |
| 5 | Reg3 Teacher Registration | TEW | MATCH | OTE | `/calendar-registration` |
| 6 | Reg4 Management | SCS | MATCH | TM | `/ori-matching` |
| 7 | Teacher Fill Rate | SOW | MATCH | TM | `/teacher-fill-rate` |
| 8 | Class Monitor | SOW | OPERATE | SO | `/class-monitor` |
| 9 | Merge Tool | SOW | PREPARE | TM | `/merge` |
| 10 | Evaluation Form | SOW | REPORT | OTE | `/evaluation` |
| 11 | Report Management | SOW | REPORT | TM, SALE | `/report-management` |
| 12 | Dashboard (KPI) | SOW | ALL | ALL | `/dashboard` |

---

## 3.1 Settle Plan Dashboard

```
┌─────────────────────────────────────────────────────────┐
│ [KPI Cards]                                             │
│ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│ │ Target   │ │ Booked   │ │ Complete │ │ Budget   │   │
│ │ 485      │ │ 312 (64%)│ │ 245 (79%)│ │ 62%      │   │
│ └──────────┘ └──────────┘ └──────────┘ └──────────┘   │
│                                                         │
│ [Filters] PT:[____] Month:[____] Plan Type:[VH/SALE]   │
│ [New Plan] [Import Excel] [Export] [New Version]        │
│                                                         │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ Date ↓  │ 0900 │ 0930 │ 1000 │ ... │ 2040 │ 2120 ││
│ │─────────┼──────┼──────┼──────┼─────┼──────┼──────││
│ │ 2026-03 │ 3/5  │ 2/3  │ 0/4  │     │ 1/2  │ 0/0  ││
│ │ -15 Sat │ 🟢   │ 🟡   │ 🔴   │     │ 🟡   │ ⚪   ││
│ │─────────┼──────┼──────┼──────┼─────┼──────┼──────││
│ │ 2026-03 │ 4/5  │ 3/3  │ 2/4  │     │ 2/2  │ 1/1  ││
│ │ -16 Sun │ 🟢   │ 🟢   │ 🟡   │     │ 🟢   │ 🟢   ││
│ └─────────────────────────────────────────────────────┘ │
│ Cell format: L4.2/L4.1 (registered/target)              │
│ Color: 🟢 ≥80% │ 🟡 50-80% │ 🔴 <50% │ ⚪ no target  │
│                                                         │
│ [Version History Sidebar]                               │
│  V3 (ACTIVE) - 2026-03-10 - "Adjusted slots"          │
│  V2 (CLOSED) - 2026-03-05                              │
│  V1 (CLOSED) - 2026-03-01                              │
└─────────────────────────────────────────────────────────┘
```

**API calls:**
- `GET /ori/settle-plan/headers?mypt&planMonth&status`
- `GET /ori/settle-plan/headers/{code}/details`
- `GET /ori/settle-plan/template?mypt&planType`
- `POST /ori/settle-plan/headers/{code}/import` (file upload)
- `GET /ori/settle-plan/headers/{code}/export`

---

## 3.2 Booking Form

```
┌─────────────────────────────────────────────────────────┐
│ BOOKING FORM                                            │
│                                                         │
│ Thông tin học sinh:                                     │
│ ┌─────────────────────┐ ┌──────────────────────┐       │
│ │ Tên HS: [________] │ │ SĐT: [___________]  │       │
│ └─────────────────────┘ └──────────────────────┘       │
│ ┌─────────────────────┐ ┌──────────────────────┐       │
│ │ PT: [KMA_DU_8  ▼]  │ │ Khối: [G6 ▼]        │       │
│ └─────────────────────┘ └──────────────────────┘       │
│ ┌─────────────────────┐ ┌──────────────────────┐       │
│ │ DFDL: ○C1 ●C2      │ │ Source: ○Crossell ○D │       │
│ └─────────────────────┘ └──────────────────────┘       │
│ ┌─────────────────────┐                                 │
│ │ Ngày: [2026-03-15]  │ Zalo: [__________]             │
│ └─────────────────────┘ Note: [__________]             │
│                                                         │
│ [CHECK AVAILABILITY]                                    │
│                                                         │
│ ┌───────────────────────────────────────────────────┐   │
│ │ 7-Day Availability Grid                           │   │
│ │ Slot  │ T2/15 │ T3/16 │ T4/17 │ T5/18 │ ...    │   │
│ │───────┼───────┼───────┼───────┼───────┼────────│   │
│ │ 09:00 │ 2/5🟢 │ 🔒    │ 3/5🟡 │ 5/5🔴 │        │   │
│ │ 09:30 │ 0/5🟢 │ 1/5🟢 │ 0/5🟢 │ 2/5🟢 │        │   │
│ │ ...   │       │       │       │       │        │   │
│ └───────────────────────────────────────────────────┘   │
│                                                         │
│ Click cell → [BOOK] button appears                      │
└─────────────────────────────────────────────────────────┘
```

**API calls:**
- `GET /ori/booking/availability?effectiveDate&mypt`
- `POST /ori/booking` (submit)

---

## 3.3 Reg3 — Teacher Registration (TEW)

```
┌─────────────────────────────────────────────────────────┐
│ ĐĂNG KÝ LỊCH RẢNH ORI          GV: Nguyễn Văn A (OTE) │
│                                                         │
│ PT: [KMA_DU_8 ▼]   Tuần: W11 (09/03 - 15/03)         │
│                                                         │
│ ┌───────────────────────────────────────────────────┐   │
│ │ Slot  │ T2/09 │ T3/10 │ T4/11 │ T5/12 │ T6/13  │   │
│ │───────┼───────┼───────┼───────┼───────┼────────│   │
│ │ 09:00 │  ☐   │  ☑   │  ☐   │  ☑   │  ☐    │   │
│ │ 09:30 │  ☐   │  ☐   │  ☑   │  ☐   │  ☑    │   │
│ │ 14:00 │  ☑   │  ☑   │  ☑   │  ☑   │  ☑    │   │
│ │ ...   │       │       │       │       │        │   │
│ └───────────────────────────────────────────────────┘   │
│ Đã chọn: 8 ca                                          │
│                                                         │
│ [HỦY] [ĐĂNG KÝ]                                        │
└─────────────────────────────────────────────────────────┘
```

**API calls (ORI-specific):**
- `POST /ori/reg3/register` → INSERT USID (reg_type=3)
- `GET /ori/reg3/my-registrations?from_date&to_date` → List USID
- `DELETE /ori/reg3/{code}` → Cancel

---

## 3.4 Reg4 Management (SCS)

```
┌─────────────────────────────────────────────────────────┐
│ XẾP GV VÀO LỚP ORI                                    │
│ Date: [2026-03-15]  PT: [All ▼]  [TÌM KIẾM]          │
│                                                         │
│ [AUTO MATCH] ← Trigger CL-02                           │
│                                                         │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ Slot │ CLAG          │PT│GG│HS│ GV Chính  │Status│ ││
│ │──────┼───────────────┼──┼──┼──┼──────────┼──────┼─││
│ │ 0900 │ CLAG-ORI-a1b2 │MA│G6│3/5│ te22     │✓ REG48│ ││
│ │ 0900 │ CLAG-ORI-c3d4 │MA│G7│2/5│ —        │✗ Chưa │ ││
│ │ 1400 │ CLAG-ORI-e5f6 │EN│G6│1/2│ phuclive │⏳ REG4│ ││
│ │ 1400 │ CLAG-ORI-g7h8 │MA│G8│4/5│ te11     │✓ REG48│ ││
│ └─────────────────────────────────────────────────────┘ │
│                                                         │
│ Click row → [Xếp Backup] modal                         │
│ ┌──────────────────────────────┐                        │
│ │ GV Reg3 sẵn sàng:           │                        │
│ │ ○ te33 (Rank A, ORI ONLY)   │                        │
│ │ ○ te44 (Rank B)             │                        │
│ │ [ASSIGN BACKUP]              │                        │
│ └──────────────────────────────┘                        │
└─────────────────────────────────────────────────────────┘
```

**API calls:**
- `GET /ori/classes/matrix?effectiveDate&mypt`
- `POST /ori/reg4/auto-match?date_from&date_to&mypt`
- `GET /ori/reg3/available?date&timeSlot&mypt`
- `POST /usid` (backup assign)

---

## 3.5 Class Monitor

```
┌─────────────────────────────────────────────────────────┐
│ CLASS MONITOR                                           │
│ Date: [Today] PT: [All ▼] Slot: [All ▼]               │
│                                                         │
│ [Upcoming: 12] [In Class: 3] [Done: 24] [Issue: 2]    │
│                                                         │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ Slot │ CLAG    │ GV    │ Status│ HS │ Meet │ Action││
│ │──────┼─────────┼───────┼───────┼────┼──────┼──────││
│ │ 0900 │ ORI-a1  │ te22  │ 🟢Live│ 3/5│ Link │[Mon] ││
│ │ 0930 │ ORI-b2  │ te11  │ ⏳Wait│ 2/5│ Link │[Call]││
│ │ 1400 │ ORI-c3  │ —     │ 🔴NoTE│ 1/2│ —    │[Esc] ││
│ └─────────────────────────────────────────────────────┘ │
│                                                         │
│ Click row → Student Detail Panel (right side):          │
│ ┌──────────────────────────────────────┐                │
│ │ HS      │ SĐT        │ Reminded │ NT │               │
│ │─────────┼────────────┼──────────┼────│               │
│ │ Nguyễn A│ 0912345678 │ ✓ 08:40  │[▼] │               │
│ │ Trần B  │ 0987654321 │ ✗        │[▼] │               │
│ │                                      │               │
│ │ NT dropdown: Done / Cancel / Fail    │               │
│ │ [SAVE ACCEPTANCE]                    │               │
│ └──────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────┘
```

**API calls:**
- `GET /ori/monitor/classes?date&mypt&time_slot`
- `PATCH /ori/monitor/classes/{clagCode}/accept`

---

## 3.6 Evaluation Form

```
┌─────────────────────────────────────────────────────────┐
│ ĐÁNH GIÁ HỌC SINH ORI                                  │
│ Lớp: CLAG-ORI-a1b2 │ 2026-03-15 │ 09:00 │ KMA_DU_8   │
│                                                         │
│ Học sinh: Nguyễn Văn A │ G6 │ C2                       │
│                                                         │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ KNOWLEDGE (40%)                                     │ │
│ │   Kiến thức cơ bản:    ①②③④⑤                      │ │
│ │   Khả năng vận dụng:   ①②③④⑤                      │ │
│ │                                                     │ │
│ │ SKILL (30%)                                         │ │
│ │   Kỹ năng giải bài:    ①②③④⑤                      │ │
│ │   Kỹ năng trình bày:   ①②③④⑤                      │ │
│ │                                                     │ │
│ │ ATTITUDE (30%)                                      │ │
│ │   Thái độ học tập:      ①②③④⑤                      │ │
│ │   Mức độ tập trung:     ①②③④⑤                      │ │
│ │                                                     │ │
│ │ Nhận xét: [________________________]                │ │
│ │ Đề xuất:  ○C1  ○C2  ○C3                            │ │
│ │                                                     │ │
│ │ Weighted Score: 3.8 / 5.0                           │ │
│ └─────────────────────────────────────────────────────┘ │
│                                                         │
│ [BỎ QUA] [GỬI ĐÁNH GIÁ]                                │
└─────────────────────────────────────────────────────────┘
```

**API calls:**
- `GET /ori/evaluation/class/{clagCode}` → list students
- `POST /ori/evaluation` → submit 6 criteria + comment + recommendation

---

## 3.7-3.12 Remaining Screens (Summary)

| Screen | Key UI | Key API |
|--------|--------|---------|
| **Slot Lock** | Date picker + PT filter + 16-slot toggle grid | `GET /ori/slot-lock`, `POST /ori/slot-lock/toggle` |
| **Schedule** | Date×Slot matrix with CLAG details | `GET /ori/schedule`, `POST /ori/schedule`, `DELETE /ori/schedule/{code}` |
| **Teacher Fill Rate** | Teacher list + slot detail with progress bars | `GET /ori/metrics/fill-rate?date&mypt` |
| **Merge Tool** | Filter (date, PT, GG, DFDL) → candidates table → [Execute] | `GET /ori/merge/candidates`, `POST /ori/merge/execute` |
| **Report Management** | Missing eval table + ready-to-gen table + [Generate] + [Send] | `GET /ori/report`, `POST /ori/report/generate`, `POST /ori/report/send` |
| **Dashboard KPI** | Date range + PT filter → date×slot heatmap with L-levels | `GET /ori/metrics/dashboard?date_from&date_to&mypt` |

---

# 4. User Flows

## 4.1 UF-01: SALE Books Student (S3)

```
SALE                        SOW                         bp-log-service              DB
 │                           │                           │                           │
 │── Open BookingPage ──────>│                           │                           │
 │                           │── GET /booking/availability ──>│                      │
 │                           │                           │── SELECT bp_ospd (VH)────>│
 │                           │                           │── SELECT bp_ospl (locks)─>│
 │                           │                           │── COUNT bp_oclag ────────>│
 │                           │<── [{slot, target, avail, locked}] ──│               │
 │<── Show 7-day grid ──────│                           │                           │
 │                           │                           │                           │
 │── Fill form + click slot ─>│                          │                           │
 │── Click [BOOK] ──────────>│                           │                           │
 │                           │── POST /booking ─────────>│                           │
 │                           │                           │── CHECK bp_ospl (lock) ──>│
 │                           │                           │── CHECK bp_ospd (cap) ───>│
 │                           │                           │── FIND bp_oclag (CLAG) ──>│
 │                           │                           │   (or CREATE new CLAG+ULC)│
 │                           │                           │── INSERT bp_oclag ───────>│
 │                           │                           │── INSERT bp_cui ─────────>│
 │                           │                           │── UPDATE bp_clag +student>│
 │                           │                           │── UPDATE bp_ospd +l42 ───>│
 │                           │<── {clagCode, cuiCode} ──│                           │
 │<── Toast "Booked!" ──────│                           │                           │
```

---

## 4.2 UF-02: Teacher Reg3 + Reg4 + Confirm (S4-S6)

```
OTE                         TEW                         bp-log-service              DB
 │                           │                           │                           │
 │── Open Reg3 page ────────>│                           │                           │
 │── Check slots ───────────>│                           │                           │
 │── Click [ĐĂNG KÝ] ──────>│                           │                           │
 │                           │── POST /ori/reg3/register >│                          │
 │                           │                           │── INSERT bp_usid (reg3) ─>│
 │<── "Đã đăng ký" ────────│                           │                           │

 ═══ CRON: Auto Reg4 (every 30min) ════════════════════════════════════════════════
                                        bp-log-service                              DB
                                         │── FIND unassigned CLAGs ────────────────>│
                                         │── FIND matching Reg3 teachers ──────────>│
                                         │── INSERT bp_usid (reg4, MAIN) ──────────>│
                                         │── UPDATE bp_oclag (teacher_code) ───────>│

 OTE                         TEW                         bp-log-service              DB
 │── Open lịch dạy ────────>│                           │                           │
 │                           │── GET /teaching/teacher-get-commitments ──>│          │
 │                           │                           │── SELECT vw_teacher_commitment ─>│
 │<── Show ORI commitment ──│                           │                           │
 │── Click [✓ Confirm] ────>│                           │                           │
 │                           │── POST /teaching/teacher-confirm ──>│                │
 │                           │                           │── UPDATE bp_usid.isapproved=1 ─>│
 │                           │                           │── INSERT bp_cui (teacher CUI) ──>│
 │<── "Đã xác nhận" ───────│                           │                           │
```

---

## 4.3 UF-03: SO Monitors + Accepts (S12-S14)

```
SO                          SOW                         bp-log-service              DB
 │                           │                           │                           │
 │── Open ClassMonitor ─────>│                           │                           │
 │                           │── GET /monitor/classes ──>│                           │
 │                           │                           │── SELECT bp_oclag (flat) ─>│
 │<── Show class list ──────│                           │                           │
 │                           │                           │                           │
 │── Click class row ───────>│                           │                           │
 │<── Show student panel ───│                           │                           │
 │                           │                           │                           │
 │── Set NT: Done per HS ──>│                           │                           │
 │── Click [SAVE] ─────────>│                           │                           │
 │                           │── PATCH /monitor/classes/{clag}/accept ──>│          │
 │                           │                           │── UPDATE bp_oclag.acceptance ──>│
 │                           │                           │── UPDATE bp_cui.mybps ────────>│
 │                           │                           │── UPDATE bp_ospd.l5_jsu++ ────>│
 │<── "Đã nghiệm thu" ────│                           │                           │
```

---

## 4.4 UF-04: TM Manages Reg4 + Backup (S5-S7)

```
TM                          SCS                         bp-log-service              DB
 │                           │                           │                           │
 │── Open OriReg4Page ──────>│                           │                           │
 │                           │── GET /ori/classes/matrix ──>│                        │
 │<── Show class grid ──────│                           │                           │
 │                           │                           │                           │
 │── Click [AUTO MATCH] ───>│                           │                           │
 │                           │── POST /ori/reg4/auto-match ──>│                     │
 │                           │                           │── CL-02 algorithm ───────>│
 │<── "Matched 5/8" ────────│                           │                           │
 │                           │                           │                           │
 │── Click row → Backup ───>│                           │                           │
 │                           │── GET /ori/reg3/available ──>│                        │
 │<── Show Reg3 pool ───────│                           │                           │
 │── Select teacher ────────>│                           │                           │
 │── Click [ASSIGN BACKUP] ─>│                          │                           │
 │                           │── POST /usid (BACKUP) ──>│                           │
 │                           │                           │── INSERT bp_usid ────────>│
 │                           │                           │── UPDATE bp_oclag.backup ─>│
 │<── "Backup assigned" ────│                           │                           │
```

---

## 4.5 UF-05: Teacher Evaluates + TM Generates Report (S15-S17)

```
OTE                         SOW                         bp-log-service              DB
 │── Open EvaluationPage ──>│                           │                           │
 │                           │── GET /evaluation/class/{clag} ──>│                  │
 │<── Show student list ────│                           │                           │
 │── Score 6 criteria ─────>│                           │                           │
 │── Click [GỬI ĐÁNH GIÁ] ─>│                          │                           │
 │                           │── POST /evaluation ─────>│                           │
 │                           │                           │── INSERT 6 bp_cuie ──────>│
 │                           │                           │── UPDATE bp_oclag.status ─>│
 │<── "Đã đánh giá" ───────│                           │                           │

TM                          SOW                         bp-log-service              DB
 │── Open ReportMgmt ──────>│                           │                           │
 │── Click [GENERATE] ─────>│                           │                           │
 │                           │── POST /report/generate ─>│                          │
 │                           │                           │── INSERT bp_cti ─────────>│
 │                           │                           │── INSERT bp_cuie (GRP) ──>│
 │                           │                           │── UPDATE bp_oclag.status ─>│
 │<── "Generated 15 reports" │                          │                           │

SALE                        SOW                         bp-log-service              DB
 │── Click [SEND] ─────────>│                           │                           │
 │                           │── POST /report/send ────>│                           │
 │                           │                           │── INSERT bp_cuie (SRP) ──>│
 │                           │                           │── UPDATE bp_oclag.status ─>│
 │<── "Report sent" ────────│                           │                           │
```

---

# 5. Complex Logic

## CL-01: Auto Booking (S3)

**Input:** studentName, myusi (SĐT), mypt, mygg, mydfdl, effectiveDate, oriTimeSlot (CASHSTA), source
**Output:** OCLAG code, CLAG code, CUI code

```
FUNCTION autoBook(request):
  // 1. Validate slot not locked
  lock = SELECT FROM bp_ospl WHERE plan_date=request.date
         AND ori_time_slot=request.slot AND mypt=request.mypt
  IF lock.status = 'LOCKED' → ERROR "Slot đã khóa"

  // 2. Check VH plan capacity
  whoCode = CASHSTA_TO_WHO.getOrDefault(request.slot, request.slot)
  ospd = SELECT FROM bp_ospd d
         JOIN bp_osph h ON d.myosph = h.code
         WHERE h.mypt=request.mypt AND h.plan_type='VH' AND h.status='ACTIVE'
           AND d.plan_date=request.date AND d.mywho=whoCode
  IF ospd IS NULL OR ospd.l42_registered >= ospd.target_sessions
    → ERROR "Slot đã hết chỗ"

  // 3. Find existing CLAG with capacity
  maxStudents = PT_MAX_MAP.get(request.mypt)  // Math=5, English=2
  clag = SELECT DISTINCT c.code FROM bp_clag_classgroup c
         JOIN bp_oclag_ori_clag_schedule o ON c.code = o.myclag
         WHERE c.mypt=request.mypt AND c.mygg=request.mygg
           AND c.mydfdl=request.mydfdl
           AND o.ori_time_slot=request.slot AND o.effective_date=request.date
           AND o.published=1
           AND c.student_number < c.maxtotalstudents
         ORDER BY c.student_number DESC  -- prefer fuller classes
         LIMIT 1

  // 4. Create CLAG if not found
  IF clag IS NULL:
    clag = INSERT bp_clag_classgroup (
      code='CLAG-ORI-{uuid}', mypt, mygg, mydfdl,
      mywho=WHO_code, mywso='7', clagtype='DYN',
      maxtotalstudents=maxStudents, student_number=0
    )
    ulc = INSERT bp_ulc (
      code='ULC-ORI-{uuid}', mylct='ORIO-30MI', mypt, mygg, mydfdl
    )
    INSERT bp_clag_ulc (myclag=clag.code, myulc=ulc.code)

  // 5. Create OCLAG row (1 per student)
  studentNum = COUNT bp_oclag WHERE myclag=clag.code AND published=1
  oclag = INSERT bp_oclag (
    code='OCLAG-{uuid}', myclag=clag.code,
    effective_date, ori_time_slot=request.slot, mypt,
    myusi=request.phone, student_name, mygg, mydfdl,
    student_number=studentNum+1, maxtotalstudents=maxStudents,
    status='BOOKED', booking_time=NOW(), sale_code=currentUser,
    source=request.source, published=1
  )

  // 6. Create CUI
  ulcCode = SELECT myulc FROM bp_clag_ulc WHERE myclag=clag.code
  cui = INSERT bp_cui (
    code='CUI-ORI-{uuid}', myusi=request.phone,
    myulc=ulcCode, mybps='BOOK_ORI', published=1
  )

  // 7. Update counters
  UPDATE bp_clag SET student_number=student_number+1 WHERE code=clag.code
  UPDATE bp_ospd SET l42_registered=l42_registered+1 WHERE code=ospd.code

  RETURN {oclagCode, clagCode, cuiCode}
```

**Edge cases:**
- Duplicate booking (same phone + same PT + same date) → reject
- Booking outside operating hours → reject
- Student already in CLAG → reject

---

## CL-02: Auto Reg4 Match (S5)

**Input:** effectiveDate, optional mypt filter
**Output:** {matched, unmatched, total}

```
FUNCTION autoReg4Match(date, mypt):
  // 1. Find CLAGs without MAIN teacher
  unassigned = SELECT DISTINCT o.myclag, o.ori_time_slot, o.mypt, c.mygg, c.mydfdl
    FROM bp_oclag_ori_clag_schedule o
    JOIN bp_clag_classgroup c ON o.myclag = c.code
    WHERE o.effective_date = date AND o.published = 1
      AND (mypt IS NULL OR o.mypt = mypt)
      AND o.teacher_code IS NULL  -- no teacher assigned yet
    GROUP BY o.myclag

  matched = 0
  FOR EACH clag IN unassigned:
    // 2. Find available Reg3 teacher
    teacher = SELECT u.myusi, u.code
      FROM bp_usid_usiduty u
      WHERE u.register_type = 3
        AND u.mypt = clag.mypt
        AND u.mycashsta = clag.ori_time_slot
        AND u.effective_date = date
        AND u.submitted_cancel_at IS NULL
        AND u.myusi NOT IN (
          -- Exclude already assigned at same date+slot
          SELECT u2.myusi FROM bp_usid_usiduty u2
          WHERE u2.register_type = 4
            AND u2.effective_date = date
            AND u2.mycashsta = clag.ori_time_slot
        )
      ORDER BY priority_score(u.myusi) DESC  -- See priority below
      LIMIT 1

    IF teacher FOUND:
      // 3. Create Reg4
      INSERT bp_usid (
        code='USID-R4-{uuid}', myusi=teacher.myusi,
        myclag=clag.myclag, mypt=clag.mypt,
        mycashsta=clag.ori_time_slot,
        register_type=4, position='MAIN', isapproved=0,
        effective_date=date, allocated_myusi='SYSTEM'
      )
      // 4. Sync to OCLAG
      UPDATE bp_oclag SET teacher_code=teacher.myusi,
        teacher_name=(SELECT fullname FROM bp_usi WHERE code=teacher.myusi)
        WHERE myclag=clag.myclag AND effective_date=date AND published=1
      matched++

  RETURN {matched, unmatched=total-matched, total}
```

**Priority scoring:**
```
score = 0
IF teacher.ori_only = true  → score += 5
IF teacher.rank = 'A'       → score += 3
IF teacher.rank = 'B'       → score += 2
IF teacher.fill_rate > 80%  → score += 2
IF teacher.reject_rate < 10%→ score += 2
IF teacher.classes_today < 3→ score += 1
```

---

## CL-03: Auto Merge (S10)

**Input:** effectiveDate, mypt (từ FE filter)
**Output:** {mergedClasses, movedStudents, deletedClasses}

```
FUNCTION autoMerge(date, mypt, mygg, mydfdl, slot):
  // 1. Find underfull CLAGs matching filter
  candidates = SELECT o.myclag, c.student_number, c.maxtotalstudents
    FROM bp_oclag_ori_clag_schedule o
    JOIN bp_clag_classgroup c ON o.myclag = c.code
    WHERE o.effective_date = date AND o.published = 1
      AND o.mypt = mypt
      AND (mygg IS NULL OR c.mygg = mygg)
      AND (mydfdl IS NULL OR c.mydfdl = mydfdl)
      AND (slot IS NULL OR o.ori_time_slot = slot)
      AND c.student_number > 0
      AND c.student_number < c.maxtotalstudents
    GROUP BY o.myclag
    ORDER BY c.student_number DESC  -- target = fullest first

  // 2. Group by (mypt, mygg, mydfdl, slot)
  groups = GROUP candidates BY (mypt, mygg, mydfdl, ori_time_slot)

  mergedClasses = 0, movedStudents = 0, deletedClasses = 0

  FOR EACH group:
    IF group.size < 2 → SKIP  // nothing to merge

    target = group[0]  // fullest class
    remaining = target.max - target.current

    FOR i = 1 TO group.size-1:
      source = group[i]
      IF source.student_number <= remaining:
        // 3. Move CUIs
        targetUlc = SELECT myulc FROM bp_clag_ulc WHERE myclag=target.myclag
        sourceUlc = SELECT myulc FROM bp_clag_ulc WHERE myclag=source.myclag
        UPDATE bp_cui SET myulc=targetUlc WHERE myulc=sourceUlc

        // 4. Move OCALGs
        UPDATE bp_oclag SET myclag=target.myclag
          WHERE myclag=source.myclag AND effective_date=date AND published=1

        // 5. Update counts
        UPDATE bp_clag SET student_number = student_number + source.student_number
          WHERE code = target.myclag
        UPDATE bp_clag SET student_number = 0 WHERE code = source.myclag

        // 6. Soft-delete source OCLAG (OLD records, now moved)
        // Note: the moved OCALGs now point to target

        remaining -= source.student_number
        movedStudents += source.student_number
        deletedClasses++
      ELSE:
        CONTINUE  // cannot fit

    mergedClasses++

  RETURN {mergedClasses, movedStudents, deletedClasses}
```

**Constraints:**
- Only merge same (mypt + mygg + mydfdl + slot) — C1 và C2 KHÔNG merge
- Cannot exceed maxtotalstudents
- Filter từ FE quyết định scope merge

---

## CL-04: Teacher Backup Escalation (S13)

```
FUNCTION escalate(clagCode):
  // Level 1: Call main teacher (T+5min)
  mainUsid = SELECT FROM bp_usid WHERE myclag=clagCode
    AND register_type=4 AND position='MAIN' AND isapproved=1
  INSERT bp_cuie (mylcet='BF-RD-CATE', mycui=teacher_cui)
  WAIT 3 min
  IF teacher joins → OK

  // Level 2: Activate backup (T+8min)
  backupUsid = SELECT FROM bp_usid WHERE myclag=clagCode
    AND register_type=4 AND position='BACKUP'
  IF backup EXISTS:
    Notify backup teacher
    IF backup joins:
      UPDATE bp_oclag SET teacher_code=backup.myusi, teacher_name=backup.name
        WHERE myclag=clagCode AND effective_date=today
      RETURN OK

  // Level 3: Cancel/Reschedule
  UPDATE bp_oclag SET status='CANCELLED', cancel_reason='No teacher available'
    WHERE myclag=clagCode AND effective_date=today
  Notify parents → offer reschedule
```

---

## CL-05: Report Generation Pipeline (S16)

```
FUNCTION generateReports(mypt, effectiveDate):
  // Find evaluated students
  students = SELECT o.* FROM bp_oclag o
    WHERE o.mypt=mypt AND o.effective_date=effectiveDate
      AND o.status='EVALUATED' AND o.published=1

  FOR EACH student:
    // Collect CUIE scores
    cuiCode = SELECT code FROM bp_cui WHERE myusi=student.myusi AND myulc IN (
      SELECT myulc FROM bp_clag_ulc WHERE myclag=student.myclag
    )
    scores = SELECT mylcet, value1 FROM bp_cuie WHERE mycui=cuiCode
      AND mylcet IN ('EVAL-KNOWLEDGE','EVAL-SKILL','EVAL-ATTITUDE')

    // Generate CTI (report content)
    INSERT bp_cti (code='CTI-ORI-{uuid}', myctt='ORI-REPORT')
    INSERT bp_cuie (mycui=cuiCode, mylcet='AF-FL-GRP')
    UPDATE bp_cui SET mycti='CTI-ORI-{uuid}', mybps='REPORT_GEN' WHERE code=cuiCode
    UPDATE bp_oclag SET status='REPORT_GEN' WHERE code=student.code
```

---

## CL-06: L-Level Computation (Cron every 15min)

```
FUNCTION updateLLevels():
  activePlans = SELECT * FROM bp_osph WHERE status='ACTIVE' AND plan_type='VH'

  FOR EACH plan:
    details = SELECT * FROM bp_ospd WHERE myosph=plan.code

    FOR EACH detail:
      cashstaSlot = WHO_TO_CASHSTA.getOrDefault(detail.mywho, detail.mywho)

      l42 = SELECT COUNT(*) FROM bp_oclag
        WHERE mypt=plan.mypt AND effective_date=detail.plan_date
          AND ori_time_slot=cashstaSlot AND published=1

      l5 = SELECT COUNT(*) FROM bp_oclag
        WHERE mypt=plan.mypt AND effective_date=detail.plan_date
          AND ori_time_slot=cashstaSlot AND published=1
          AND acceptance='Done'

      l8 = SELECT COUNT(*) FROM bp_oclag
        WHERE mypt=plan.mypt AND effective_date=detail.plan_date
          AND ori_time_slot=cashstaSlot AND published=1
          AND status='REPORT_SENT'

      UPDATE bp_ospd SET l42_registered=l42, l5_jsu=l5, l8_settled=l8
        WHERE code=detail.code
```

---

# 6. Feature & Layer

## 6.1 Module Breakdown

| Module | Phase | FE System | BE Service | Key Tables |
|--------|-------|-----------|------------|------------|
| **Settle Plan** | FORECAST | SOW SettlePlanPage | OriSettlePlanService | bp_osph, bp_ospd |
| **Slot Lock** | FORECAST | SOW SlotLockPage | OriSlotLockService (within SettlePlan) | bp_ospl |
| **Booking** | FORECAST | SOW BookingPage | OriBookingService | bp_oclag, bp_clag, bp_ulc, bp_clag_ulc, bp_cui |
| **Schedule** | FORECAST | SOW SchedulePage | OriScheduleService | bp_oclag, bp_clag |
| **Reg3** | MATCH | TEW CalendarRegistration | OriReg3Service (NEW) | bp_usid (reg_type=3) |
| **Reg4** | MATCH | SCS OriReg4Page | OriReg4MatchingService | bp_usid (reg_type=4), bp_oclag |
| **Meeting** | PREPARE | (Cron only) | OriMeetingService | bp_vcr, bp_usi_vcr, bp_oclag |
| **Merge** | PREPARE | SOW MergePage | OriMergeService | bp_oclag, bp_clag, bp_cui |
| **Monitor** | OPERATE | SOW ClassMonitorPage | OriMonitorService | bp_oclag, bp_cui |
| **Evaluation** | REPORT | SOW EvaluationPage | OriEvaluationService | bp_cuie, bp_oclag |
| **Report** | REPORT | SOW ReportMgmtPage | OriReportService | bp_cti, bp_cuie, bp_oclag |
| **Metrics/Dashboard** | ALL | SOW DashboardPage | OriMetricsService | bp_oclag, bp_ospd |

## 6.2 Layer Responsibility

### Frontend Layer

| System | Tech | Responsibility | Source of truth |
|--------|------|---------------|-----------------|
| **SOW** | React 18 + MUI v6 + Vite | 9 pages: Plan, Lock, Book, Schedule, Monitor, Merge, Eval, Report, Dashboard | API response |
| **TEW** | React 17 + Redux | 1 module: Reg3 Calendar Registration + Lịch dạy (commitment view) | API response |
| **SCS** | React 17 + Redux | 1 module: Reg4 Management (auto-match + backup) | API response |

**FE rules:**
- Không có business logic trên FE — chỉ gọi API + render
- Validation chỉ ở mức form (required fields, format) — business validation ở BE
- State: React hooks (SOW), Redux (TEW/SCS)

### Backend Layer

| Service | Responsibility | Key logic |
|---------|---------------|-----------|
| **OriSettlePlanService** | CRUD plan + Excel import/export + grid generation | Version management, slot generation |
| **OriBookingService** | CL-01 auto-booking + availability check | CLAG find/create, OCLAG write, capacity check |
| **OriReg3Service** (NEW) | ORI-specific Reg3 CRUD | Validate Reg1 prerequisite, duplicate check |
| **OriReg4MatchingService** | CL-02 auto-match + manual trigger | Priority scoring, conflict detection |
| **OriMergeService** | CL-03 merge candidates + execute | Group by filter, capacity check, CUI move |
| **OriScheduleService** | CLAG+ULC+OCLAG CRUD | Create/update/soft-delete |
| **OriMonitorService** | Flat OCLAG query + accept | Direct OCLAG read (no JOIN needed) |
| **OriEvaluationService** | 6-criteria evaluation + weighted score | CUIE write, score computation |
| **OriReportService** | Report generation + send | CTI create, CUIE write, status update |
| **OriMetricsService** | Dashboard + fill rate + L-level | OCLAG aggregate, OSPD comparison |
| **OriScheduledTasks** | 5 cron jobs | Auto Reg4, merge, meeting, L-level, alert |

### Database Layer

**Source of truth mapping:**

| Data | Source of truth | Denormalized to |
|------|----------------|-----------------|
| Plan target | `bp_ospd.target_sessions` | — |
| Student booking | `bp_oclag` (1 row = 1 student) | `bp_cui.myusi` mirrors OCLAG.myusi |
| Teacher assignment | `bp_usid` (Reg4) | `bp_oclag.teacher_code/backup_code` |
| Meeting link | `bp_vcr` | `bp_oclag.meet_link/meet_code` |
| Evaluation score | `bp_cuie` (6 events) | `bp_oclag.status`=EVALUATED |
| L-levels | Computed from `bp_oclag` | Cached in `bp_ospd.l42/l5/l8` |
| Teacher schedule | `bp_usid` Reg4 | `vw_teacher_commitment` (VIEW) |

## 6.3 API Contract Summary

### SOW → bp-log-service (`/api/v1/bp-log/ori/*`)

| # | Method | Path | Module | Request | Response |
|---|--------|------|--------|---------|----------|
| 1 | GET | `/settle-plan/headers` | Plan | ?mypt, ?planMonth, ?status | List<OSPH> |
| 2 | POST | `/settle-plan/headers` | Plan | {mypt, planMonth, planType, totalTarget...} | OSPH |
| 3 | GET | `/settle-plan/headers/{code}/details` | Plan | — | List<OSPD> |
| 4 | POST | `/settle-plan/headers/{code}/generate` | Plan | — | "Generated" |
| 5 | PATCH | `/settle-plan/details-batch` | Plan | {updates: [{code, targetSessions}]} | "Updated" |
| 6 | POST | `/settle-plan/headers/{code}/new-version` | Plan | — | OSPH (new) |
| 7 | POST | `/settle-plan/headers/{code}/activate` | Plan | — | "Activated" |
| 8 | GET | `/settle-plan/template` | Plan | ?mypt, ?planType | Excel file |
| 9 | POST | `/settle-plan/headers/{code}/import` | Plan | MultipartFile | {imported, total} |
| 10 | GET | `/settle-plan/headers/{code}/export` | Plan | — | Excel file |
| 11 | GET | `/slot-lock` | Lock | ?dateFrom, ?dateTo, ?mypt | List<OSPL> |
| 12 | POST | `/slot-lock/toggle` | Lock | {planDate, oriTimeSlot, mypt} | "Toggled" |
| 13 | GET | `/booking/availability` | Book | ?effectiveDate, ?mypt | [{slot, target, registered, available, locked}] |
| 14 | POST | `/booking` | Book | {studentName, myusi, mypt, mygg, mydfdl, date, slot, source, note, zalo} | {oclagCode, clagCode, cuiCode} |
| 15 | GET | `/schedule` | Schedule | ?effectiveDate, ?mypt | List<OCLAG+CLAG> |
| 16 | POST | `/schedule` | Schedule | {mypt, mygg, mydfdl, date, slot} | {clagCode, oclagCode} |
| 17 | DELETE | `/schedule/{code}` | Schedule | — | "Deleted" |
| 18 | GET | `/classes/matrix` | Class | ?effectiveDate, ?mypt | List<{slot, clag, teachers...}> |
| 19 | GET | `/monitor/classes` | Monitor | ?date, ?mypt, ?time_slot | List<OCLAG flat> |
| 20 | PATCH | `/monitor/classes/{clagCode}/accept` | Monitor | {acceptance per student} | "Accepted" |
| 21 | POST | `/reg4/auto-match` | Match | ?effectiveDate, ?mypt | {matched, unmatched, total} |
| 22 | GET | `/merge/candidates` | Merge | ?effectiveDate, ?mypt, ?mygg, ?mydfdl, ?slot | List<candidates> |
| 23 | POST | `/merge/execute` | Merge | {effectiveDate, mypt, mygg, mydfdl, slot} | {merged, moved, deleted} |
| 24 | GET | `/evaluation/class/{clagCode}` | Eval | — | List<students> |
| 25 | POST | `/evaluation` | Eval | {cuiCode, knowledge[2], skill[2], attitude[2], comment, recommendation} | {weightedScore} |
| 26 | GET | `/report` | Report | ?mypt, ?status | List<reports> |
| 27 | POST | `/report/generate` | Report | {mypt, effectiveDate} | {generated} |
| 28 | POST | `/report/send` | Report | {cuiCode} | "Sent" |
| 29 | GET | `/metrics/dashboard` | Metrics | ?date_from, ?date_to, ?mypt | {dates, data: {date: {slot: {target, l42, l5, l8}}}} |
| 30 | GET | `/metrics/fill-rate` | Metrics | ?date, ?mypt | [{slot, total, filled, rate}] |

### TEW → bp-log-service (ORI Reg3: NEW)

| # | Method | Path | Request | Response |
|---|--------|------|---------|----------|
| 31 | POST | `/ori/reg3/register` | {mypt, mycashsta, effective_date, weekdays[]} | List<USID created> |
| 32 | GET | `/ori/reg3/my-registrations` | ?from_date, ?to_date | List<USID reg3> |
| 33 | DELETE | `/ori/reg3/{code}` | — | "Cancelled" |

### TEW → bp-log-service (existing endpoints, reuse)

| # | Method | Path | Usage in ORI |
|---|--------|------|-------------|
| 34 | GET | `/teaching/teacher-get-commitments` | GV xem ORI commitment (từ `vw_teacher_commitment`) |
| 35 | POST | `/teaching/teacher-confirm` | GV confirm/reject Reg4 |

### SCS → bp-log-service

| # | Method | Path | Usage |
|---|--------|------|-------|
| 36 | GET | `/ori/classes/matrix` | Reuse #18 |
| 37 | POST | `/ori/reg4/auto-match` | Reuse #21 |
| 38 | GET | `/ori/reg3/available` | Reg3 pool for backup: ?date, ?slot, ?mypt |
| 39 | POST | `/usid` | Existing endpoint: assign backup teacher |

## 6.4 Permission Matrix

| Screen | SALE | OTE | TM | SO | ADMIN |
|--------|------|-----|-----|-----|-------|
| Settle Plan | C/R/U (own PT) | — | R/Approve | R | Full |
| Slot Lock | — | — | C/R/U | — | Full |
| Booking | C/R | — | R | — | Full |
| Schedule | R | — | R/U | R | Full |
| Reg3 (TEW) | — | C/R/U (own) | R (all) | — | Full |
| Reg4 (SCS) | — | R (own, TEW) | R/U/Trigger | — | Full |
| Fill Rate | — | — | R | R | Full |
| Monitor | R (own) | — | R | R/U (accept) | Full |
| Merge | — | — | R/U | — | Full |
| Evaluation | — | C/R/U (own class) | R | — | Full |
| Report Mgmt | R/Send | — | R/Generate | — | Full |
| Dashboard | R | — | R | R | Full |

---

# 7. Design Suggestions & Risks

## 7.1 Gợi ý hoàn thiện

| # | Vấn đề | Gợi ý | Priority |
|---|--------|-------|----------|
| 1 | **POD_EGT không có POD** | HS trial không có POD → không ghi bp_pod_egt. Evaluation score lưu ở bp_cuie. Nếu cần aggregate score, thêm cột `weighted_score` vào OCLAG | Medium |
| 2 | **OCLAG.myusi rename** | Đổi `student_phone` → `myusi` cần ALTER TABLE + update FE (BookingPage, ClassMonitorPage, demoData) | High |
| 3 | **CUI cho GV timing** | Sau teacher-confirm, backend cần tạo CUI (myusi=teacher_code, myulc=ULC.code). Trigger: POST /teaching/teacher-confirm → hook vào ORI service | High |
| 4 | **Reg3 API riêng** | Cần tạo `OriReg3Controller` + `OriReg3Service` với path `/ori/reg3/*`. Validate Reg1 prerequisite | High |
| 5 | **CASHSTA↔WHO mapping** | Config cứng 2 cặp (1840↔1845, 1920↔1930). Cần unit test cho mapping | Medium |
| 6 | **Merge scope** | FE gửi filter → BE merge theo filter đó. Cần validate filter không rỗng (tránh merge toàn bộ) | High |
| 7 | **Audit trail** | Mọi thay đổi OCLAG nên log vào bp_cuie hoặc audit table. Hiện chỉ có created_at/updated_at | Low |
| 8 | **Concurrent booking** | 2 SALE book cùng lúc vào cùng slot → race condition trên student_number. Cần `SELECT FOR UPDATE` hoặc optimistic locking | High |
| 9 | **OCLAG denormalization sync** | teacher_code, meet_link, status phải sync từ source tables. Nếu update USID trực tiếp (không qua ORI API) → OCLAG stale | Medium |
| 10 | **Evaluation weight TBD** | 3 nhóm × 2 sub = 6 criteria. Weight trong mỗi nhóm (2 sub bằng nhau?) chưa confirm | Medium |

## 7.2 Risks

| Risk | Impact | Mitigation |
|------|--------|------------|
| **OCLAG sync stale** | Monitor hiện data cũ | Cron L-level update cũng verify OCLAG sync |
| **CASHSTA data chưa clean** | Slot mismatch giữa plan vs actual | User sẽ clean sau. Backend mapping tạm thời |
| **CUI cho GV chưa implement** | TEW không hiện lịch ORI | Priority cao trong incremental plan |
| **Race condition booking** | Overbooking lớp | DB-level locking on bp_clag.student_number |
| **Merge xóa nhầm** | Mất data HS | Soft delete only (published=0), never hard delete |

---

> **End of Architecture Pack V2**
> Next: Incremental Steps Plan (separate file)
