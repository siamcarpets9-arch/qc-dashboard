# SC-FOS Planning V1: Planning Control, Planning KPI และ Capacity Check

วันที่: 23 กันยายน 2026
Branch ที่แนะนำ: `claude/planning-v1-baseline-kpi-capacity`

งานนี้ต่อยอดจากระบบเดิมตาม `SC-FOS_Planning_Capacity_Development_Brief.docx` และ `SCFOS_Planning_Code_Package` ไม่ได้สร้างระบบใหม่ ไม่ได้ลบ field หรือ API action เดิม ไม่ได้เปลี่ยน localStorage key และไม่มีการ reset Google Sheet

## ไฟล์ที่แก้

| ไฟล์ | สิ่งที่เปลี่ยน |
|---|---|
| `index.html` / `apps-script/index.html` | เพิ่ม Planning Control V1 module ใน `<script>` หลัก และ hook เข้าฟังก์ชันเดิม (ไฟล์ทั้งสองเหมือนกันทุกตัวอักษร) |
| `apps-script/Code.js` | เพิ่ม 7 header ใน `ORDERS_NEW_HEADERS` อ่านค่าใน `extractOrderFromRow_()` และเขียนค่าใน `upsertOrderIntoSheets_()` |
| `scripts/verify-planning-backend.js` | ไฟล์ใหม่ ทดสอบ round-trip ของ Sheet ด้วย mock sheet |

## Column ใหม่ในชีต M/O-S/O ทั้ง 4 แท็บ

ระบบจะต่อ column ท้ายชีตให้อัตโนมัติผ่าน `ensureOrdersHeaders()` ตอนเรียก API ครั้งแรก column เดิมไม่ขยับตำแหน่ง

| Column | ชนิด | ความหมาย |
|---|---|---|
| `OriginalDueDate` | date | วันที่ Commit กับลูกค้าครั้งแรก เขียนทับไม่ได้ |
| `OriginalDueSource` | text | `recorded`: บันทึกตอนเปิด M/O หรือกรอกครั้งแรก, `history`: ได้จาก `dueDateHistory[0].from`, `legacy`: เดาจาก dueDate จึงไม่นับใน Original OTD |
| `BaselinePlanDates` | JSON | `{planning, design, dyeing, weaving, finishing, frozenAt, frozenBy, source, pendingSince}` |
| `ActualFinishDates` | JSON | `{planning, design, dyeing, weaving, finishing}` |
| `PlanRevisionHistory` | JSON | `[{stage, oldDate, newDate, baselineDate, changedAt, changedBy, reason}]` |
| `DelayLog` | JSON | `[{stage, delayType, delayReason, ownerDept, remark, recordedAt, recordedBy, recoveryDate}]` |
| `PlanningCapacityOverride` | JSON | สำรองไว้ก่อน (V1 ยังไม่ใช้) |

การป้องกันข้อมูลหาย: ถ้า payload ที่ส่งมาไม่มี field เหล่านี้ เช่นเครื่องที่ยังเปิดหน้าเว็บเวอร์ชันเก่า ระบบจะคงค่าเดิมในแถวไว้ ไม่เขียนทับเป็นค่าว่าง

`dueDateHistory` ใช้ column เดิม แต่ entry ใหม่จะมี field เพิ่ม: `changedBy, delayType, delayReason, ownerDept, remark, originalDueDate` (ของเดิม `direction, acknowledgedBy, reason, requestedBy` ยังอยู่ครบ)

## Data model

- `originalDueDate` คือ Baseline ของ KPI เขียนทับไม่ได้
- `postponeDispatchDate` (field เดิม) ใช้เป็น Revised Due Date
- `dueDate` (field เดิม) คือกำหนดส่งปัจจุบัน ทุกครั้งที่เลื่อนจะอัปเดตค่านี้ด้วย เพื่อให้ warning, timeline และ board เดิมทำงานเหมือนเดิม
- Current Due = Revised หรือ dueDate
- `shippedDate` (field เดิม) คือ Actual Delivery
- `planDates.planning` คือ Internal Target (ถ้ามี Baseline จะใช้ `baselinePlanDates.planning`)
- `planDates` (field เดิม) คือ Current Plan ส่วน `baselinePlanDates` คือแผนแรกที่ Freeze แล้ว

## ฟังก์ชันเดิมที่ hook

| ฟังก์ชัน | การเปลี่ยนแปลง |
|---|---|
| `normalizeOrder()` | เรียก `applyPlanningControlDefaults(order, raw)` ก่อน return **แก้ bug เดิม:** `dueDateHistory` ถูกตัดเหลือแค่ `{from,to,changedAt}` ทำให้ `direction/acknowledgedBy` หายหลัง sync ตอนนี้เก็บทุก field แล้ว |
| `submitNewOrder()` | บันทึก `originalDueDate` เป็นกำหนดส่งที่กรอกตอนเปิด M/O และเตรียม field ใหม่ทั้งหมด |
| `updateOrderField()` | การกรอกกำหนดส่งครั้งแรกจะกลายเป็น Original Due ถ้าแก้ dueDate ตรง ๆ Original Due ยังอยู่ และบันทึกเป็น revision ระบบจะ Freeze Baseline เมื่อติ๊ก "ลงแผนงานแล้ว" และ stamp Actual ของงานวางแผนเมื่อ checklist ครบ 100% |
| `persistOrderChange()` | เรียก `planningBeforePersist()` เพื่อเก็บ plan revision ทุกเส้นทางที่บันทึก, Freeze Baseline ที่ค้างอยู่ และ stamp Actual ของงานดีไซน์จาก Data Logs |
| `recomputeAllSchedules()` | ระบุเหตุผล "Auto re-schedule (EDD / capacity)" ให้ M/O อื่นที่ถูกเลื่อนแผนตามไปด้วย |
| `transferOrderStage()` | stamp Actual ของแผนกต้นทาง ถ้าเสร็จช้ากว่า Baseline จะเปิด Delay modal |
| `transferPlanningDirectToWeaving()` | stamp Actual ของวางแผนและย้อม (กรณีใช้ surplus ครบ) |
| `markOrderReadyToShip()` | stamp Actual ของแผนกตกแต่ง (= Factory Ready) |
| `markOrderShipped()` | ไม่แก้ `shippedDate` ยังคือ Actual Delivery |
| `openDueDateRescheduleModal()` / `submitDueDateReschedule()` | ใช้ modal "เลื่อนเข้า/เลื่อนออก" เดิม แต่บังคับกรอก Delay Type / Delay Reason / Owner Dept / ผู้แก้ไข และแสดง Original Due ทุก flow เรียกผ่าน `applyPlanningDueDateRevision()` |
| `renderOrderDetailModalBody()` | เพิ่มกล่อง DELIVERY CONTROL, Plan vs Actual และ Delay Log ไว้ด้านบน ช่อง "กำหนดเสร็จ" จะเป็น read-only เมื่อมีค่าแล้ว และมีปุ่ม "เลื่อน" แทน |
| `switchView()` | เพิ่มแท็บ `planning` |
| `buildOverviewRowHtml()` | เพิ่ม badge แบบกระชับ: Orig / +วัน / ORIG LATE / AT RISK-OVERDUE รายแผนก |
| `renderTimelineSection()` / `computeTimelineRange()` | แสดงธงโปร่งเป็น Original Due ข้างธงทึบที่เป็น Current Due tooltip ของแต่ละ bar แสดง Baseline / Current / Actual / Variance |
| Analytics "ส่งตรงเวลา" | คำนวณด้วย helper ชุดเดียวกัน (Revised OTD) note ใต้การ์ดแสดง Original OTD และไม่นับข้อมูลย้อนหลัง (summaryOnly) |

Helper หลักที่ทุกหน้าใช้ร่วมกัน: `getOriginalDueDate`, `getRevisedDueDate`, `getCurrentDueDate`, `getOriginalOTD`, `getRevisedOTD`, `getScheduleResult`, `getDelayDays`, `getExtensionDays`, `computePlanningKpis`, `computeDailyCapacitySnapshot`, `computeWeeklyCapacity`

## Migration logic สำหรับ Legacy

1. **Original Due** ใช้ค่าตามลำดับนี้:
   1. `OriginalDueDate` ที่บันทึกไว้
   2. `dueDateHistory[0].from` (source = `history`)
   3. `dueDate` ปัจจุบัน (source = `legacy`) ระบบจะแสดง "Legacy / Baseline unavailable" และไม่นับใน Original OTD ยกเว้นผู้ใช้ติ๊ก "รวม Legacy" ในหน้า Planning
2. **Revised Due** ใช้ `postponeDispatchDate` ถ้าไม่มีจะใช้ `dueDateHistory[last].to`
3. **Baseline:** M/O เก่าที่ติ๊ก "ลงแผนงานแล้ว" ไว้ก่อน V1 จะถูก Freeze จากแผนปัจจุบันตอนบันทึกครั้งแรก และติดป้าย "Legacy snapshot" ส่วน M/O ใหม่จะ Freeze ตอนติ๊กครั้งแรก (source = `confirmed`) ถ้าติ๊กตอนที่ยังไม่มีวันที่แผน ระบบจะรอ แล้ว Freeze ทันทีที่มีวันที่แผน
4. ค่าที่ migrate แล้วจะถูกเขียนลงชีตตอนที่ M/O นั้นถูกบันทึกครั้งถัดไป ไม่มีการเขียนทับข้อมูลเก่าทั้งชีต
5. Actual Finish ของงานที่เสร็จไปก่อน V1 ระบบไม่เดาให้ ยกเว้นงานดีไซน์ ซึ่งดึงวันที่จริงจาก Data Logs / ใบโอนขยายลาย

## Capacity V1

- **ดีไซน์ / ทอ / ตกแต่ง:** Required Hours ได้จาก `computeProductionSuggestions().baseHours` (engine เดิม) หารด้วย `planDurations` เพื่อได้ load ต่อวัน แล้วเทียบกับ `staffCount × (regularHoursPerDay + otHours)`
- **ย้อม:** ใช้ค่าจาก `computeDyeingSuggestion()` เดิม คิดเป็น กก./วัน (`kgToDye / duration` เทียบ `staffCount × kgPerDay`) ถ้ายังไม่ตั้งค่า หรือมี M/O ที่ยังไม่มีแผนสั่งย้อม (กก.) จะขึ้น **"Capacity data incomplete"**
- ช่วงวันทำงานของแต่ละแผนก = `[planDate − duration, planDate)` ซึ่งตรงกับ `scheduleStageEDD()`
- **Status:** ≤80 SAFE, ≤90 WATCH, ≤100 CRITICAL, >100 OVERBOOKED ตัดสินจากวันที่ใช้กำลังผลิตสูงสุดของสัปดาห์
- เลือกสัปดาห์ได้: This Week / Next Week / +2 / +3

## ผลการทดสอบ

- ตรวจ Inline JS syntax ผ่าน (0 error)
- `node --check apps-script/Code.js` ผ่าน
- `scripts/audit-html.js`: ไม่มี duplicate ID และไม่มี handler หาย มี ID แบบ dynamic เพิ่ม 2 ตัวคือ `planningFilterStart`/`planningFilterEnd`
- `scripts/verify-order-guards.js` และ `scripts/verify-board-detail-guards.js` ผ่าน
- `scripts/verify-planning-backend.js` ผ่าน 10/10
- E2E ใน Chromium ด้วย `index.html` จริง ผ่าน 37/37 ครอบคลุม Legacy load, New M/O, Due revision (บังคับกรอกเหตุผล), History metadata, Original/Revised OTD (ตัวอย่างจาก Brief 10 ต.ค. → 18 ต.ค. ส่งจริง 17 ต.ค.), Baseline freeze ครั้งเดียว, แก้แผนแล้ว Baseline ไม่เปลี่ยน, Plan revision history (รวมกรณีหลาย M/O ถูกบันทึกพร้อมกัน), Actual stamp ครบทุกแผนก, LATE แล้วเปิด Delay modal, Schedule Adherence, Capacity >100% ขึ้น OVERBOOKED, filter M/O-S/O และตลาด, Overview / Timeline badge, refresh แล้วข้อมูลไม่หาย และทุกแท็บเดิมเปิดได้โดยไม่มี JS error

## ข้อควรรู้และข้อจำกัด

1. **ลำดับ deploy:** ต้อง deploy Apps Script (`Code.js`) เข้า deployment ID เดิมก่อน แล้วจึง merge frontend ไป GitHub Pages ถ้าทำกลับลำดับ backend เก่าจะไม่รู้จัก field ใหม่ และข้อมูลใหม่จะหายหลัง reload
2. `shippedDate` เดิม stamp ด้วยเวลา UTC ถ้าบันทึกส่งมอบก่อน 07:00 วันที่จะเป็นเมื่อวาน ครั้งนี้ไม่ได้แก้ตามคำสั่ง "Keep markOrderShipped() unchanged" ส่วนวันที่ของ Planning ใช้เวลาท้องถิ่น
3. ถ้ากรอก "Postpone Dispatch Date" ในฟอร์มเปิด M/O ระบบจะนับเป็น Revised Due ตั้งแต่วันแรก
4. Warning เดิมยังใช้ `dueDate` (= Current Due) ตาม Brief ข้อ 14 ส่วน KPI ใช้ Original Due
5. Supabase dual-write (`syncOrderToSupabase_`) ยังไม่ได้ส่ง field ใหม่ เพราะตารางปลายทางยังไม่มี column นี้
6. Capacity ของแผนกทอคิดเป็น man-hour ตาม Brief ไม่ได้คิดจำนวนจอ (จำนวนจอยังใช้ใน scheduler เดิม)

## อัปเดต 24 กันยายน 2026: ติ๊กโอนงานระหว่างแผนก

ใช้ในกรณีที่พนักงานยังไม่พร้อมกรอกรายละเอียด แต่งานส่งต่อไปแผนกถัดไปแล้ว

- ถ้ากด "โอนงานไป..." ตอนที่แผนกยังกรอกงานไม่ครบ 100% (หรือแผนกทอยังไม่ได้ยืนยันรับผ้าปั๊มลาย/ไหมครบ) ระบบจะเปิดหน้าต่าง **ติ๊กโอนงาน** แทนการปฏิเสธเฉย ๆ ผู้ใช้ต้องติ๊กยืนยันและใส่ชื่อผู้โอน หมายเหตุไม่บังคับ
- ใช้ได้กับการโอน วางแผน → ย้อม, ย้อม → ทอ, ทอ → ตกแต่ง และวางแผน → ทอ (ข้ามแผนกย้อม เมื่อใช้ไหม surplus ครบ)
- ระบบเก็บ `order[แผนก].quickTransfer = {by, at, toStage, percentAtTransfer, note, detailsPending}` ใน JSON ของแผนกนั้นใน column เดิม **ไม่มี column ใหม่ และไม่ต้องแก้ Apps Script**
- ในหน้ารายละเอียด M/O และบนการ์ดของแผนกที่รับงาน จะมีป้าย **"ติ๊กโอนแล้ว • รอกรอกรายละเอียด"** จนกว่าแผนกเดิมจะกรอกครบ 100%
- ปุ่มยืนยันพร้อมส่งของแผนกตกแต่งยังต้องมี QC PASS เหมือนเดิม
- Planning V1: วันที่ติ๊กโอนถูกบันทึกเป็น Actual Finish ของแผนกต้นทาง
- ผลทดสอบ E2E: 42/42 ผ่าน (มีเคสติ๊กโอนเพิ่ม 5 เคส)
