# Kế hoạch Kiểm tra & Làm sạch Dữ liệu

**Dataset:** `Logic_Based_Educational_Queries.json`
**Mục đích:** Chuẩn bị dữ liệu train LLM dựa trên thuật toán FOL
**Tổng số records:** 411 | **Ngày lập:** 2026-05-14 | **Ngày kiểm tra lần 2:** 2026-05-16

---

## Cấu trúc mỗi record

```json
{
  "idx":          [[premise_ids cho Q1], [premise_ids cho Q2]],
  "premises-FOL": [...],
  "premises-NL":  [...],
  "questions":    [...],
  "answers":      [...],
  "explanation":  [...]
}
```

- `idx`: danh sách premise index liên quan đến từng câu hỏi (1-based)
- `premises-FOL`: biểu diễn tiền đề bằng First-Order Logic
- `premises-NL`: biểu diễn tiền đề bằng ngôn ngữ tự nhiên (song song 1-1 với FOL)
- `questions`: 1–2 câu hỏi mỗi record
- `answers`: đáp án tương ứng (`"Yes"`, `"No"`, `"A"`–`"D"`, `"Unknown"`)
- `explanation`: giải thích lý luận cho từng câu hỏi

---

## Thống kê cập nhật (sau lần kiểm tra 2026-05-16)

| Chỉ số               | Giá trị                                                         |
| ---------------------- | ----------------------------------------------------------------- |
| Tổng records          | 411                                                               |
| Tổng answer instances | 808                                                               |
| Questions/record       | 1–2 (14 records có 1 Q, 397 records có 2 Q, avg ≈ 1.97)       |
| Premises-NL/record     | 3–36 (avg ≈ 10.80)                                              |
| Premises-FOL/record    | 3–36 (avg ≈ 10.88 — cao hơn do 11 records FOL > NL)           |
| Phân phối answer     | **No=251**, **Yes=165**, Unknown=209, A=105, B=39, C=20, D=19 *(sau fix Phase 3.1)* |
| Records 30+ premises   | Record 36 (34), Record 408 (36), Record 409 (36), Record 410 (36) |

> **Lưu ý:** Nếu sửa 60 lỗi nhãn Phase 3.1 → phân phối thực sẽ là `No=240, Yes=176`

---

## Kết quả kiểm tra chi tiết (2026-05-16)

### Phase 1 — Kiểm tra cấu trúc (Structural Validation)

**1.1 Schema integrity** — ✅ PASS

- [X] Tất cả 411 records có đủ 6 keys
- [X] Không có field nào là `null` hoặc mảng rỗng `[]`

**1.2 Alignment giữa các mảng** — ✅ ĐÃ SỬA HOÀN TOÀN (2026-05-17)

- [X] `len(questions) == len(answers) == len(explanation) == len(idx)` — PASS toàn bộ
- [X] `len(premises-FOL) == len(premises-NL)` — PASS toàn bộ (sau khi sửa)

**Lịch sử sửa lỗi (11 records ban đầu):**

| Record | FOL (trước) | NL | Lệch   | Hành động                                                              | Trạng thái  |
| ------ | ----------- | -- | ------- | ----------------------------------------------------------------------- | ----------- |
| 34     | 27          | 27 | NL +2  | Đã sửa từ trước (NL[25–26] đã xóa)                                 | ✅ Đã sửa  |
| 57     | 19          | 19 | FOL +2 | Đã sửa từ trước (FOL EcoFriendly, Safe conditions đã xóa)          | ✅ Đã sửa  |
| 146    | 10          | 10 | FOL +1 | Đã sửa từ trước                                                       | ✅ Đã sửa  |
| 334    | 15          | 15 | FOL +2 | Đã sửa từ trước                                                       | ✅ Đã sửa  |
| 376    | 9           | 9  | FOL +9 | Đã sửa từ trước                                                       | ✅ Đã sửa  |
| 377    | 22          | 13 | FOL +9 | **Xóa FOL[13..21]** (9 entries: AllowedToTakeExam, Midterm, LabReport…) | ✅ Đã sửa |
| 378    | 17          | 13 | FOL +4 | **Xóa FOL[13..16]** (4 entries: GPA(Lan), WorksRegularHours, Lab, Temp)  | ✅ Đã sửa |
| 379    | 16          | 13 | FOL +3 | **Xóa FOL[13..15]** (3 entries: WorkHours(Minh), MustDeclareMajor, Semester) | ✅ Đã sửa |
| 380    | 19          | 13 | FOL +6 | **Xóa FOL[13..18]** (6 entries: Priority, Status(Hà), Internship, ApprovedByAdvisor…) | ✅ Đã sửa |
| 381    | 17          | 12 | FOL +5 | **Xóa FOL[12..16]** (5 entries: Ranking/Workshop, MaxScore, PartTimeJob, Deadline…) | ✅ Đã sửa |
| 382    | 14          | 13 | FOL +1 | **Xóa FOL[13]** (`TotalCredits(RegularProgram) = 132`)               | ✅ Đã sửa  |

> **Ghi chú:** Toàn bộ entries bị xóa là phần FOL không có NL tương ứng và **không được tham chiếu bởi bất kỳ idx nào** (đã kiểm tra conflict trước khi xóa). Không có idx nào bị ảnh hưởng.

**1.3 idx boundary check** — ✅ PASS (không có idx ngoài biên)

Tuy nhiên **11 slots có idx rỗng `[]`** (không tham chiếu premise nào):

| Record | Q-slot bị rỗng          |
| ------ | ------------------------- |
| 48     | slot 1                    |
| 54     | slot 1                    |
| 57     | slot 1                    |
| 72     | slot 1                    |
| 79     | slot 0                    |
| 81     | slot 0 và slot 1 (cả 2) |
| 89     | slot 1                    |
| 125    | slot 1                    |
| 129    | slot 1                    |
| 334    | slot 1                    |

---

### Phase 2 — Kiểm tra chất lượng FOL (FOL Quality)

**2.1 Phát hiện encoding bị hỏng** — ✅ PASS (hoàn toàn sạch)

- [X] Không tìm thấy ký tự U+FFFD (`�`)
- [X] Không tìm thấy `?` ngay trước `forall`/`exists`

> **So với lần kiểm tra trước:** Lỗi encoding `"?forall(x,..."` ở Record 1 đã được sửa.

**2.2 Thống kê ký hiệu FOL** — ⚠️ CÒN INCONSISTENT (nhỏ)

| Ký hiệu / Biến thể   | Số lần xuất hiện |
| ------------------------ | -------------------- |
| `→` (Unicode)         | 4,082                |
| `¬` (Unicode)         | 2,329                |
| `∀` (Unicode)         | 2,252                |
| `ForAll(` (functional) | 1,881                |
| `∧` (Unicode)         | 583                  |
| `Exists(` (functional) | 413                  |
| `∃` (Unicode)         | 408                  |
| `~` (tilde, ASCII)     | **7**          |
| `->` (ASCII arrow)     | **6**          |

- [X] Đã thống kê tần suất
- [ ] Chuẩn hóa `->` → `→` (6 chỗ cần sửa)
- [ ] Chuẩn hóa `~` → `¬` (7 chỗ cần sửa)
- [ ] Quyết định chuẩn: `∀`/`∃` hay `ForAll(`/`Exists(` (cả hai phổ biến, cần chọn 1)

**2.3 Kiểm tra cú pháp FOL cơ bản** — ✅ PASS

- [X] Không có premise nào lệch số dấu ngoặc
- [X] Không có quantifier trống body

---

### Phase 3 — Kiểm tra tính nhất quán logic (Logical Consistency)

**3.1 Answer ↔ Explanation contradiction** — ✅ ĐÃ SỬA (2026-05-17) → xem `FIX_PHASE_3_1_LABEL_INVERSION.md`

**1 lỗi nhãn rõ ràng:**

| Record       | Q-slot | Answer   | Kết luận trong explanation                                                  |
| ------------ | ------ | -------- | ----------------------------------------------------------------------------- |
| **43** | 1      | `"No"` | _"...confirming a positive feedback loop, so the answer is **Yes**."_ |

**60 lỗi nhãn mang tính hệ thống (systematic label inversion):**

Tất cả 60 records này đều có pattern: Q-slot 1 hỏi _"is the following statement true?"_, answer=`"No"`, nhưng explanation kết luận _"making the statement true."_

Đây là lỗi invert nhãn hàng loạt — answer nên là `"Yes"` cho tất cả 60 records này.

**Danh sách 60 records:**
75, 76, 79, 80, 82, 87, 89, 90, 101, 102, 103, 104, 110, 113, 114, 131, 134, 135, 136, 137, 138, 139, 140, 141, 142, 143, 144, 145, 146, 167, 168, 169, 171, 172, 173, 174, 176, 177, 178, 180, 181, 182, 184, 206, 221, 222, 223, 224, 225, 227, 228, 229, 230, 234, 235, 237, 238, 275, 312

**1 trường hợp biên (không phải lỗi):**

| Record | Q-slot | Answer    | Ghi chú                                                                        |
| ------ | ------ | --------- | ------------------------------------------------------------------------------- |
| 328    | 1      | `"Yes"` | Explanation kết luận "consequent always false" — vacuous truth, logic đúng |

**3.2 Answer=`"False"` anomaly** — ✅ PASS (không còn xuất hiện)

**3.3 idx mapping** — xem 1.3 ở trên (11 slots rỗng)

---

### Phase 4 — Phát hiện trùng lặp (Deduplication)

**4.1 Exact duplicate** — ❌ FAIL (4 cặp trùng hoàn toàn)

| Cặp | Records                         | Hành động    |
| ---- | ------------------------------- | --------------- |
| 1    | **8** và **9**     | Xóa record 9   |
| 2    | **24** và **25**   | Xóa record 25  |
| 3    | **160** và **161** | Xóa record 161 |
| 4    | **394** và **395** | Xóa record 395 |

**4.2 Near-duplicate (premises-FOL giống 100% + Jaccard question > 0.85)**

| Records                       | Jaccard | Answer      | Hành động đề xuất                                                   |
| ----------------------------- | ------- | ----------- | ------------------------------------------------------------------------- |
| **1** & **2**     | 0.902   | Cùng `C` | Review — chỉ đổi "strongest" → "correct"; xem xét gộp hoặc xóa 1 |
| **182** & **183** | 1.000   | Cùng       | Exact duplicate (xóa 183)                                                |
| **204** & **205** | 0.912   | Cùng       | Options MCQ khác nhẹ; human review                                      |
| **330** & **331** | 0.932   | Cùng       | Variable P vs U trong options; human review                               |

> Records 165/166, 294/295, 314/315 — cùng FOL nhưng Jaccard < 0.45, câu hỏi thực sự khác → **giữ lại**.

---

### Phase 5 — Phân tích phân phối (Distribution Analysis)

**5.1 Label balance** — ⚠️ MẤT CÂN BẰNG

```
Yes/No: No=300 (37.1%), Yes=116 (14.4%)  → tỉ lệ ~1:2.6
MCQ:    A=105 (13.0%), B=39 (4.8%), C=20 (2.5%), D=19 (2.4%)  → A chiếm ~66%
Unknown: 209 (25.9%)
```

Sau khi sửa 60+1 lỗi nhãn: `No=239, Yes=177` → tỉ lệ cân bằng hơn (~1:1.35)

- [ ] Quyết định chiến lược xử lý imbalance
- [ ] Quyết định giữ/loại answer=`"Unknown"`

**5.2 Premise length** — ⚠️ 4 records có 30+ premises

| Record | Số premises NL |
| ------ | --------------- |
| 36     | 34              |
| 408    | 36              |
| 409    | 36              |
| 410    | 36              |

- [ ] Kiểm tra token length với tokenizer model đích

**5.3 Topic diversity** — [ ] Chưa thực hiện

---

## Tổng hợp lỗi theo mức độ ưu tiên

| Độ ưu tiên | Phase | Loại lỗi                                                                     | Số records                              | Records bị ảnh hưởng                                                                                                                        |
| -------------- | ----- | ------------------------------------------------------------------------------ | ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| ✅ DONE        | 3.1   | Lỗi nhãn hệ thống — **đã sửa 2026-05-17** (49 No→Yes, 10 đã Yes, 1 skip) | ~~**60**~~ → **0**                | Xem `FIX_PHASE_3_1_LABEL_INVERSION.md`                                                                                                         |
| ✅ DONE        | 1.2   | FOL/NL premise count lệch — **đã sửa 2026-05-17**                          | ~~**11**~~ → **0**                | ~~34,57,146,334,376–382~~                                                                                                                      |
| 🟠 HIGH        | 4.1   | Exact duplicate records                                                        | **4 cặp (8 records)**             | (8,9),(24,25),(160,161),(394,395)                                                                                                               |
| 🟠 HIGH        | 4.2   | Near-duplicate (cần human review)                                             | **1 cặp thêm + 2 cần xem xét** | (182,183),(204,205),(330,331)                                                                                                                   |
| 🟡 MEDIUM      | 1.3   | idx rỗng `[]` không tham chiếu premise                                    | **10 records / 11 slots**          | 48,54,57,72,79,81,89,125,129,334                                                                                                                |
| 🟡 MEDIUM      | 2.2   | FOL notation không nhất quán (`->`, `~`)                                | **~13 premises**                   | rải rác                                                                                                                                       |
| 🟡 MEDIUM      | 2.2   | Hai phong cách quantifier song song (`∀` vs `ForAll(`)                   | dataset-wide                             | —                                                                                                                                              |
| 🟢 LOW         | 5.1   | Label imbalance (No:Yes ≈ 2.6:1, A chiếm 66% MCQ)                            | —                                       | dataset-wide                                                                                                                                    |
| 🟢 LOW         | 5.2   | 4 records có 30+ premises — kiểm tra token limit                            | **4**                              | 36,408,409,410                                                                                                                                  |

---

## Hành động cần làm (ưu tiên từ cao xuống thấp)

### Bước 1 — Sửa lỗi nhãn (CRITICAL) ✅ HOÀN THÀNH 2026-05-17

- [X] **Record 43, Q-slot 1:** Đổi answer từ `"No"` → `"Yes"`
- [X] **49 records hệ thống:** Đổi answer từ `"No"` → `"Yes"` tại Q-slot 1
  - 10 records (110,113,167,171,176,180,184,227,234,237) đã là Yes từ trước
  - 1 record (134) bỏ qua vì chỉ có 1 câu hỏi
- Xem chi tiết: `FIX_PHASE_3_1_LABEL_INVERSION.md`

### Bước 2 — Xóa duplicate (HIGH)

- [ ] Xóa records: **9, 25, 161, 183, 395** (giữ bản đầu tiên)
- [ ] Human review cặp (204,205) và (330,331) để quyết định

### Bước 3 — Align FOL/NL (HIGH) ✅ HOÀN THÀNH 2026-05-17

- [X] Records **377–382**: Đã xóa FOL thừa (tổng 29 entries) — không có idx conflict
- [X] Records **34, 57, 146, 334, 376**: Đã sửa từ các lần trước

### Bước 4 — Sửa notation nhỏ (MEDIUM)

- [ ] Thay `->` → `→` (6 chỗ)
- [ ] Thay `~` → `¬` (7 chỗ)
- [ ] Quyết định chuẩn `∀` hay `ForAll(` rồi normalize toàn dataset

### Bước 5 — Xử lý idx rỗng (MEDIUM)

- [ ] Kiểm tra 11 slots idx rỗng — bổ sung premise references hoặc ghi nhận là intentional

---

## Phase 6 — Tiêu chí xử lý record (cập nhật)

| Tình trạng                                            | Hành động                            |
| ------------------------------------------------------- | --------------------------------------- |
| Answer mâu thuẫn rõ với explanation                 | Sửa answer theo explanation            |
| FOL/NL length mismatch — NL thiếu (nhóm 376–382)    | Bổ sung NL hoặc cắt FOL thừa        |
| FOL/NL length mismatch — trường hợp khác (34,57…) | Kiểm tra tay, align hoặc loại bỏ    |
| Exact duplicate                                         | Giữ 1, xóa bản còn lại             |
| Near-duplicate cùng answer                             | Human review, xem xét xóa             |
| idx rỗng `[]`                                        | Bổ sung references hoặc ghi chú      |
| `->` hoặc `~` trong FOL                            | Script normalize toàn bộ              |
| Records 30+ premises                                    | Kiểm tra token limit với model đích |

---

## Index tra cứu: Record → Số dòng trong file JSON

> Mỗi record bắt đầu bằng `  {` (2 space). Dùng `Ctrl+G` trong VS Code để nhảy thẳng đến dòng.
> Script tra nhanh: `python -c "f=open('Logic_Based_Educational_Queries.json',encoding='utf-8'); lines=[i+1 for i,l in enumerate(f) if l.startswith('  {')]; print(lines[382-1])"`

**Records có lỗi (từ kế hoạch làm sạch):**

| Record | Dòng  | Loại lỗi                        | Record | Dòng  | Loại lỗi                        |
| ------ | ----- | -------------------------------- | ------ | ----- | -------------------------------- |
| 8      | 402   | Exact duplicate (giữ lại)       | 167    | 8684  | Systematic label inversion       |
| 9      | 452   | Exact duplicate (xóa)           | 168    | 8752  | Systematic label inversion       |
| 24     | 1118  | Exact duplicate (giữ lại)       | 169    | 8795  | Systematic label inversion       |
| 25     | 1186  | Exact duplicate (xóa)           | 171    | 8879  | Systematic label inversion       |
| 43     | 2306  | Label error rõ ràng (No→Yes)    | 172    | 8921  | Systematic label inversion       |
| 48     | 2569  | idx rỗng slot 1                 | 173    | 8965  | Systematic label inversion       |
| 54     | 2881  | idx rỗng slot 1                 | 174    | 9009  | Systematic label inversion       |
| 57     | 3044  | idx rỗng slot 1                 | 176    | 9097  | Systematic label inversion       |
| 72     | 3833  | idx rỗng slot 1                 | 177    | 9141  | Systematic label inversion       |
| 75     | 4008  | Systematic label inversion       | 178    | 9185  | Systematic label inversion       |
| 76     | 4058  | Systematic label inversion       | 180    | 9273  | Systematic label inversion       |
| 79     | 4200  | idx rỗng slot 0 + label inv.    | 181    | 9317  | Systematic label inversion       |
| 80     | 4237  | Systematic label inversion       | 182    | 9361  | Systematic label inv. + near-dup |
| 81     | 4274  | idx rỗng slot 0 & 1             | 183    | 9407  | Near-duplicate (xóa)             |
| 82     | 4313  | Systematic label inversion       | 184    | 9453  | Systematic label inversion       |
| 87     | 4497  | Systematic label inversion       | 206    | 10295 | Systematic label inversion       |
| 89     | 4569  | idx rỗng slot 1 + label inv.    | 221    | 11156 | Systematic label inversion       |
| 90     | 4608  | Systematic label inversion       | 222    | 11215 | Systematic label inversion       |
| 101    | 5217  | Systematic label inversion       | 223    | 11259 | Systematic label inversion       |
| 102    | 5281  | Systematic label inversion       | 224    | 11306 | Systematic label inversion       |
| 103    | 5337  | Systematic label inversion       | 225    | 11349 | Systematic label inversion       |
| 104    | 5387  | Systematic label inversion       | 227    | 11439 | Systematic label inversion       |
| 110    | 5661  | Systematic label inversion       | 228    | 11486 | Systematic label inversion       |
| 113    | 5788  | Systematic label inversion       | 229    | 11530 | Systematic label inversion       |
| 114    | 5830  | Systematic label inversion       | 230    | 11574 | Systematic label inversion       |
| 125    | 6286  | idx rỗng slot 1                 | 234    | 11764 | Systematic label inversion       |
| 129    | 6530  | idx rỗng slot 1                 | 235    | 11798 | Systematic label inversion       |
| 131    | 6645  | Systematic label inversion       | 237    | 11892 | Systematic label inversion       |
| 134    | 6796  | Systematic label inversion       | 238    | 11938 | Systematic label inversion       |
| 135    | 6827  | Systematic label inversion       | 275    | 13686 | Systematic label inversion       |
| 136    | 6867  | Systematic label inversion       | 312    | 15503 | Systematic label inversion       |
| 137    | 6901  | Systematic label inversion       | 334    | 16625 | idx rỗng slot 1                 |
| 138    | 6941  | Systematic label inversion       | 394    | 19734 | Exact duplicate (giữ lại)       |
| 139    | 6978  | Systematic label inversion       | 395    | 19781 | Exact duplicate (xóa)           |
| 140    | 7021  | Systematic label inversion       | 204    | —     | Near-dup (human review)          |
| 141    | 7062  | Systematic label inversion       | 205    | —     | Near-dup (human review)          |
| 142    | 7100  | Systematic label inversion       | 330    | —     | Near-dup (human review)          |
| 143    | 7144  | Systematic label inversion       | 331    | —     | Near-dup (human review)          |
| 144    | 7182  | Systematic label inversion       | 377    | 18879 | FOL/NL mismatch ✅ đã sửa       |
| 145    | 7225  | Systematic label inversion       | 378    | 18944 | FOL/NL mismatch ✅ đã sửa       |
| 146    | 7268  | Systematic label inversion       | 379    | 19004 | FOL/NL mismatch ✅ đã sửa       |
| 160    | 8197  | Exact duplicate (giữ lại)       | 380    | 19062 | FOL/NL mismatch ✅ đã sửa       |
| 161    | 8265  | Exact duplicate (xóa)           | 381    | 19124 | FOL/NL mismatch ✅ đã sửa       |
|        |       |                                  | 382    | 19181 | FOL/NL mismatch ✅ đã sửa       |

---

## Output sau khi làm sạch

- `report_phase_X.json` — danh sách record index bị lỗi và loại lỗi
- `Logic_Based_Educational_Queries_cleaned.json` — dataset sau khi xử lý
- `cleaning_log.md` — ghi lại số records bị loại bỏ, sửa đổi, lý do

---

## Ghi chú quan trọng

> ~~**[NEW]** Lỗi nghiêm trọng nhất là 60 records bị invert nhãn hàng loạt~~ → **✅ ĐÃ SỬA 2026-05-17**: 49 records đã đổi `"No"` → `"Yes"` tại Q-slot 1. Xem `FIX_PHASE_3_1_LABEL_INVERSION.md`.

> **[NEW]** Nhóm records 376–382 có FOL đầy đủ nhưng NL bị truncate — nhiều khả năng do giới hạn output khi tạo dữ liệu. Ưu tiên bổ sung NL thay vì xóa FOL.

> Việc chuẩn hóa ký hiệu FOL (Phase 2.2) phải được thực hiện **trước khi tokenize** vì model sẽ học `→` và `->` như các token khác nhau.

> Answer=`"Unknown"` chiếm **~25.9% tổng answers** — cần quyết định rõ ràng về cách xử lý class này trước khi training.
