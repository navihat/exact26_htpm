# Kế hoạch Kiểm tra & Làm sạch Dữ liệu

**Dataset:** `Physics_Problems_Text_Only.csv`
**Mục đích:** Chuẩn bị dữ liệu train LLM giải bài tập vật lý (Chain-of-Thought)
**Ngày lập:** 2026-05-14
**Ngày kiểm tra:** 2026-05-17
**Tổng records:** 1352

---

## Cấu trúc file

```
id , question , cot , answer , unit
```

| Cột         | Mô tả                                          |
| ------------ | ------------------------------------------------ |
| `id`       | Mã bài toán (VD:`TD401`, `LD001`)         |
| `question` | Đề bài bằng tiếng Anh                       |
| `cot`      | Chain-of-Thought giải từng bước (multi-line) |
| `answer`   | Đáp án cuối cùng                            |
| `unit`     | Đơn vị của đáp án                         |

---

## Vấn đề đã phát hiện qua đọc mẫu (ban đầu)

| #  | Loại lỗi                                      | Ví dụ cụ thể                                                                                             |
| -- | ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| 1  | **ID prefix không nhất quán**          | `TD401`, `TD402` lẫn `LD001`–`LD047` — hai nguồn dữ liệu khác nhau?                           |
| 2  | **Answer format hỗn loạn**              | `45`, `24.45 × 10^-3`, `9\sqrt{3} × 10^-27`, `\sqrt{2} × F₀`, `Hướng về phía q₂`          |
| 3  | **Answer symbolic không tính được**  | `LD041`: answer = `\sqrt{2} × F₀` — là biểu thức đại số, không phải số                       |
| 4  | **Answer có leading zero**               | `LD024`: answer = `06.04` — lỗi định dạng, phải là `6.04`                                       |
| 5  | **Unit không nhất quán**               | `N`, `μF`, `μC`, `J`, `Độ` (tiếng Việt), `-` (không có đơn vị)                        |
| 6  | **CoT không hoàn chỉnh**               | `LD017`: 4 bước đều chỉ "Identify/Calculate" mà không ra kết quả số                              |
| 7  | **CoT thừa meta-comment**                | `LD011`: question có đoạn `"(This is a very direct and clear translation.)"`                          |
| 8  | **Question bị lỗi nối text**           | `LD045`: `"...2 cm away from qCalculate..."` — thiếu dấu cách/xuống dòng                           |
| 9  | **CoT có nhận xét không hợp lệ**    | `LD021`: Step 4 nói _"The problem does not explicitly ask a question"_ nhưng vẫn có answer           |
| 10 | **CoT incomplete nhưng vẫn có answer** | `LD024`: Step 4 nói cần thêm thông tin vị trí, nhưng answer = `06.04`                             |
| 11 | **Answer = 0 cần xác minh**             | `LD036`, `LD039`, `LD042`, `LD043`: answer `0` — hợp lệ (đối xứng điện tích), cần verify |
| 12 | **Answer hướng/định tính**           | `LD047`: answer = `Hướng về phía q₂`, unit = `-` — không phải số                              |
| 13 | **Multi-line trong CSV**                  | Cột `cot` chứa nhiều dòng — dễ bị parse sai nếu dùng csv đơn giản                              |

---

## Kết quả kiểm tra thực tế (2026-05-17)

### Tổng quan nhanh

| Phase | Hạng mục                 | Kết quả                                           | Mức độ |
| ----- | -------------------------- | --------------------------------------------------- | --------- |
| 1.1   | CSV parsing integrity      | ✅ PASS — 1352 rows, 0 lỗi parse, đúng 5 cột   | CLEAR     |
| 1.2   | Missing values             | ⚠️ 14 records có `unit` rỗng                  | HIGH      |
| 1.3   | Duplicate ID               | ✅ PASS — 0 duplicate                              | CLEAR     |
| 2.1   | ID prefix & gaps           | ⚠️ 8 prefix, nhiều gap trong numbering           | MEDIUM    |
| 3.1   | Answer type classification | ❌ 164 OTHER, 7 SYMBOLIC, 1 LEADING_ZERO            | CRITICAL  |
| 3.2   | Answer = 0                 | ⚠️ 18 records (nhiều hơn dự kiến)             | MEDIUM    |
| 4.1   | Unit normalization         | ⚠️ 2 chuẩn μ,`-` vs `—`, 14 empty          | HIGH      |
| 5.1   | CoT structure              | ⚠️ 2 CoT thiếu Step 1, 1 CoT chỉ có 2 bước   | HIGH      |
| 5.2   | CoT completeness           | ⚠️ 2 records CoT kết thúc không hoàn chỉnh   | HIGH      |
| 5.3   | CoT-Answer alignment       | ⚠️ 53 records answer không xuất hiện cuối CoT | MEDIUM    |
| 5.4   | Meta-comment in question   | ⚠️ 1 record (LD011)                               | LOW       |
| 5.5   | Text concat in question    | ⚠️ 1 record (LD045) + CH101-105 false alarm       | MEDIUM    |
| 6.1   | Topic distribution         | ⚠️ Thiếu Optics (4) và Mechanics (1)            | LOW       |

---

### Phase 1 — Kiểm tra cấu trúc CSV ✅

**1.1 CSV parsing integrity** — PASS

- Parse bằng `csv.reader` với `quotechar='"'`: 1352 rows, 0 lỗi
- Tất cả rows đúng 5 cột. Multi-line CoT được quote đúng cách.

**1.2 Missing values** — CÓ VẤN ĐỀ

- `id`, `question`, `cot`, `answer`: không có giá trị rỗng
- `unit`: **14 records rỗng** (không phải `-`, mà thực sự là chuỗi rỗng)

```
TD013, DT047, TD369, TD373, TD377, TD380, TD386
DDT327, DDT337, DDT343, CH371, CH372, CH373, CH374
```

Lưu ý: DT047 có answer dạng symbolic (`1/2 . (1/ \sqrt{E_A} + 1/ \sqrt{E_B})`), TD369/377/380/386 có answer dạng text conceptual.

**1.3 Duplicate IDs** — PASS

- 0 ID trùng lặp trong toàn bộ 1352 records.

---

### Phase 2 — ID Prefix Analysis ✅

**2.1 Phân tích prefix**

| Prefix | Số records | Range ID | Số gap | Ghi chú                              |
| ------ | ----------- | -------- | ------- | ------------------------------------- |
| CH     | 290         | 1–380   | 90 gap  | Thiếu nhiều khoảng ở giữa        |
| CHLT   | 20          | 1–20    | 0 gap   | Đầy đủ                            |
| DDT    | 130         | 131–400 | 140 gap | Numbering bắt đầu từ 131, kỳ lạ |
| DT     | 68          | 1–100   | 32 gap  |                                       |
| LD     | 397         | 1–400   | 3 gap   | Thiếu LD093, LD348, LD350            |
| NL     | 190         | 1–400   | 210 gap | Rất nhiều gap                       |
| TD     | 177         | 1–402   | 225 gap | Rất nhiều gap                       |
| THCB   | 80          | 1–135   | 55 gap  |                                       |

**Nhận xét:** Các gap là intentional — dataset lấy mẫu không liên tục từ ngân hàng bài toán lớn hơn. DDT bắt đầu từ 131 là điểm kỳ lạ cần xác nhận.

---

### Phase 3 — Answer Analysis ❌

**3.1 Phân loại kiểu answer (thực tế)**

| Kiểu               | Số records | Ví dụ                                      | Trạng thái          |
| ------------------- | ----------- | -------------------------------------------- | --------------------- |
| DECIMAL             | 731         | `0.045`, `6.76`                          | ✅ Hợp lệ           |
| INTEGER             | 241         | `100`, `7`, `0`                        | ✅ Hợp lệ           |
| SCIENTIFIC (× 10^) | 189         | `24.45 × 10^-3`, `1.23 × 10^-3`        | ⚠️ Cần chuẩn hóa |
| OTHER               | 164         | Xem bên dưới                              | ❌ Nhiều vấn đề   |
| SCIENTIFIC_INT_BASE | 14          | `45 × 10^-3`, `4 × 10^-6`              | ⚠️ Cần chuẩn hóa |
| SYMBOLIC_SQRT       | 7           | `9\sqrt{3} × 10^-27`, `\sqrt{2} × F₀` | ❌ Tách riêng       |
| LEADING_ZERO        | 1           | `LD024: 06.04`                             | ❌ Sửa →`6.04`    |

**Phân tích nhóm OTHER (164 records):**

| Sub-type                        | Ví dụ                                                                | Record IDs                                          |
| ------------------------------- | ---------------------------------------------------------------------- | --------------------------------------------------- |
| Text conceptual                 | `Do not change`, `the voltage is halfed`, `decreases by 4 times` | TD369, TD373, TD377, TD380, TD386, THCB071, THCB073 |
| Expression symbolic             | `E1 = (3/4)E2`, `8E`, `1/4 \pi`                                  | LD077, DT058, DT060                                 |
| Multi-value (semicolon)         | `0; 0`, `36;12`, `I_D₁ = 1.0; I_D₂ = 1.0; I_total = 2.0`       | TD374, TD376, THCB066–070, THCB072                 |
| Sci-notation Unicode (⁴, ⁻³) | `4.0 × 10⁴`, `1.00 × 10⁷`                                      | DT098, LD338, LD360, LD383, LD390–396              |
| Sci-notation với × (Unicode)  | `0.230 × 10⁻³`, `43.30 × 10⁻³`                               | LD294, LD295                                        |
| Directional text (VN)           | `Hướng về phía q₂`                                              | LD047                                               |
| Answer có label prefix         | `Rtd = 20.0`                                                         | THCB078                                             |

**3.2 Answer = 0** — 18 records

```
LD036, LD039, LD042, LD043, LD059, LD079, LD084, LD095, LD096,
DT001, DT019, DT075, DT096, DT099, NL312 (+ thêm 3 chưa list đủ)
```

Cần verify từng record qua CoT: các bài điện tích đối xứng thường hợp lệ.

**3.3 Scientific notation** — Phát hiện thêm vấn đề mới

- Dùng `×` (U+00D7) + `10^-3` (ASCII caret): 189 records
- Dùng `×` + `10⁻³` (Unicode superscript): ~10 records (LD294, LD295, DT098, LD338+)
- Cần chọn 1 chuẩn thống nhất — đề xuất: `× 10^-3` (ASCII caret)

**3.4 Symbolic answers** — 7 records

```
LD005: 9\sqrt{3} × 10^-27
LD041: \sqrt{2} × F₀
LD085: 2 × sqrt(2) × k × q / a^2
DT047: 1/2 . (1/ \sqrt{E_A} + 1/ \sqrt{E_B})
```

→ Tách sang `Physics_Problems_symbolic.csv`

---

### Phase 4 — Unit Analysis ⚠️

**4.1 Kết quả phân tích unit (56 unique values)**

| Vấn đề                            | Chi tiết                                                                           | Số records |
| ------------------------------------ | ----------------------------------------------------------------------------------- | ----------- |
| Unit rỗng (empty string)            | TD013, DT047, TD369, TD373, TD377, TD380, TD386, DDT327, DDT337, DDT343, CH371–374 | 14          |
| Unit `-` (hyphen)                  | Không có đơn vị                                                                | 84          |
| Unit `—` (em-dash)                | Phải chuẩn hóa về `-`                                                         | 42          |
| `μF` (U+03BC) vs `µF` (U+00B5) | Hai ký tự khác nhau trông giống nhau                                           | 41 vs 14    |
| `μC` (U+03BC)                     | Đã nhất quán (0 record dùng µC U+00B5)                                        | 8 vs 0      |

**Hành động:**

- Thống nhất `-` và `—` về một chuẩn (đề xuất: `-`)
- Thống nhất `µ` (U+00B5) → `μ` (U+03BC) trong tất cả unit
- 14 records empty unit: điều tra từng record (một số là bài conceptual không cần unit)

---

### Phase 5 — CoT Analysis ⚠️

**5.1 Cấu trúc CoT**

- **LD353**: CoT bắt đầu ở Step 2, thiếu Step 1 (bị cắt mất đầu)
- **NL396**: CoT bắt đầu bằng `"CoT: \nIdentify..."` thay vì `"Step 1:"` — format không đúng
- **THCB078**: Chỉ có 2 bước (Step 1 + Step 2), rất ngắn (92 chars) — có thể chấp nhận nếu bài đơn giản

**5.2 CoT completeness**

- **LD021**: CoT có câu `"The problem does not explicitly ask a question to be solved"` — mâu thuẫn với việc vẫn có answer
- **LD024**: CoT có câu `"the specific positions of q1 and q2 relative to q3 are required"` nhưng answer = `06.04` (leading zero) — CoT tự mâu thuẫn

**5.3 CoT-Answer alignment**

Phương pháp: tìm giá trị `answer` trong 300 ký tự cuối của CoT.

- **53 records** không tìm thấy answer value trong đoạn cuối CoT
- Cần review thủ công: nhiều trường hợp answer đúng nhưng được viết dưới dạng khác (VD: answer=`0.045` nhưng CoT viết `45 × 10^-3`)
- Priority review: LD007, LD008, LD009, LD010, LD012 (diff >40%)

**5.4 Meta-comment trong question**

- **LD011**: question kết thúc bằng `"(This is a very direct and clear translation.)"` → xóa phần này

**5.5 Text concat errors**

- **LD045**: `"...2 cm away from qCalculate the magnitude..."` — `q` và `Calculate` bị dính nhau, thiếu dấu chấm/xuống dòng
- **CH101–CH105**: pattern `Hz` theo sau bởi chữ hoa (VD: `100Hz with`) — là cách viết bình thường, không phải lỗi

---

### Phase 6 — Distribution Analysis ⚠️

**6.1 Phân bố theo chủ đề vật lý (overlap — một bài có thể thuộc nhiều chủ đề)**

| Chủ đề                  | Số records (approx)      |
| -------------------------- | ------------------------- |
| Current / Resistance       | 546                       |
| Capacitor                  | 463                       |
| Coulomb's Law / E. Force   | 454                       |
| Electric Field             | 443                       |
| Energy / Power             | 392                       |
| Electric Potential/Voltage | 387                       |
| Oscillation / Wave         | 351                       |
| Magnetic Field             | 343                       |
| Optics                     | 4 (**rất ít**)    |
| Mechanics                  | 1 (**thiếu hụt**) |

**Nhận xét:** Dataset tập trung nặng vào **điện học (electrostatics + circuits + EM)**. Optics và Mechanics gần như vắng mặt — cần xác nhận đây là intentional.

**6.3 Phân bố answer = 0**

- 18 records (1.3% tổng) → tỉ lệ thấp, không gây bias đáng kể

---

## Vấn đề mới phát hiện (chưa có trong kế hoạch ban đầu)

| #  | Loại lỗi mới                                    | Ví dụ                                                                | Records ảnh hưởng                      |
| -- | -------------------------------------------------- | ---------------------------------------------------------------------- | ----------------------------------------- |
| 14 | **Answer multi-value (semicolon)**           | `0; 0`, `36;12`, `I_D₁=1.0; I_D₂=1.0`                          | TD374, TD376, THCB066–073                |
| 15 | **Answer conceptual/qualitative (EN)**       | `Do not change`, `decreases by 4 times`, `the voltage is halfed` | TD369, TD380, TD386, THCB071, THCB073     |
| 16 | **Answer dùng Unicode superscript**         | `4.0 × 10⁴`, `1.00 × 10⁷`, `0.230 × 10⁻³`                 | ~12 records (LD294, LD295, DT098, LD338+) |
| 17 | **Unit em-dash `—` vs hyphen `-`**      | 42 records dùng `—`, 84 dùng `-`                                | 126 records                               |
| 18 | **Answer với label prefix**                 | `Rtd = 20.0`                                                         | THCB078                                   |
| 19 | **CoT bắt đầu từ Step 2 (bị truncate)** | LD353: CoT bắt đầu bằng `"Step 2:"`                              | LD353                                     |
| 20 | **CoT có prefix `CoT:` không chuẩn**    | NL396:`"CoT: \nIdentify..."`                                         | NL396                                     |
| 21 | **DDT numbering bắt đầu từ 131**         | DDT131–DDT400 (không có DDT001–DDT130)                             | Toàn bộ DDT (130)                       |
| 22 | **Answer percentage**                        | `50%`                                                                | TD373                                     |
| 23 | **Empty unit cho numeric answer**            | DDT327: ans=`0.60`, unit=`` (rỗng)                                  | DDT327, DDT337, DDT343, CH371–374        |

---

## Kế hoạch kiểm tra chi tiết (cập nhật)

### Phase 1 — Kiểm tra cấu trúc CSV ✅ HOÀN THÀNH

**1.1 CSV parsing integrity**

- [X] Parse toàn bộ file bằng thư viện chuẩn (Python `csv` module với `quotechar='"'`)
- [X] Đếm số cột thực tế mỗi row — **PASS: tất cả đúng 5 cột**
- [X] Phát hiện row bị parse sai — **PASS: 0 lỗi**

**1.2 Missing values**

- [X] Kiểm tra từng cột có giá trị rỗng — **14 records `unit` rỗng**
- [X] Phân biệt unit `-` (no-unit) với thực sự missing — **Đã phân biệt**

**1.3 Duplicate ID**

- [X] Mỗi `id` phải là duy nhất — **PASS: 0 duplicate**

---

### Phase 2 — Kiểm tra ID và phân nhóm ✅ HOÀN THÀNH

**2.1 Phân tích ID prefix**

- [X] Liệt kê tất cả prefix: **8 prefix** (CH, CHLT, DDT, DT, LD, NL, TD, THCB)
- [X] Kiểm tra gap numbering — **Nhiều gap, là intentional sampling**
- [X] DDT bắt đầu từ 131 — **Cần xác nhận nguồn gốc**

**2.2 Thống kê phân bố**

- [X] LD: 397 records (nhiều nhất), CHLT: 20 (ít nhất)

---

### Phase 3 — Kiểm tra Answer ⚠️ CẦN XỬ LÝ

**3.1 Phân loại kiểu answer** ✅ Đã phân loại

- [X] DECIMAL: 731 — hợp lệ
- [X] INTEGER: 241 — hợp lệ
- [X] SCIENTIFIC (× 10^): 203 — cần chuẩn hóa format
- [X] SYMBOLIC_SQRT: 7 — tách sang tập riêng
- [X] LEADING_ZERO: 1 (LD024) — sửa → `6.04`
- [ ] **TODO**: Xử lý 164 records OTHER (xem bảng phân loại sub-type bên trên)

**3.2 Kiểm tra answer = 0**

- [X] Tìm được 18 records
- [ ] **TODO**: Verify từng record qua CoT (đặc biệt: DT001, DT019, DT075, DT096, DT099)

**3.3 Chuẩn hóa scientific notation**

- [ ] **TODO**: Thống nhất `× 10^-3` (ASCII caret) — replace cả Unicode superscript ⁻³ và `× 10^{-3}`

**3.4 Quyết định về symbolic answers**

- [ ] **TODO**: Tách 7 records có `sqrt`/`frac` sang `Physics_Problems_symbolic.csv`

---

### Phase 4 — Kiểm tra Unit ⚠️ CẦN XỬ LÝ

**4.1 Chuẩn hóa unit**

- [X] Phát hiện inconsistency μ (U+03BC) vs µ (U+00B5)
- [X] Phát hiện em-dash `—` (42 records) vs hyphen `-` (84 records)
- [ ] **TODO**: Chuẩn hóa `µ` → `μ` (U+03BC) trong toàn bộ unit
- [ ] **TODO**: Chuẩn hóa `—` → `-` trong cột unit
- [ ] **TODO**: Điều tra 14 records empty unit — gán unit phù hợp hoặc dùng `-`

**4.2 Answer-Unit consistency**

- [ ] **TODO**: 6 records numeric answer (DDT327, DDT337, DDT343, CH371–374) có unit rỗng — gán unit

---

### Phase 5 — Kiểm tra CoT ⚠️ CẦN XỬ LÝ

**5.1 Cấu trúc CoT**

- [X] Phát hiện 2 CoT không có "Step 1": LD353, NL396
- [ ] **TODO LD353**: CoT bị cắt mất Step 1 — viết lại hoặc loại bỏ
- [ ] **TODO NL396**: Sửa format — xóa prefix `"CoT: \n"`, đánh lại Step 1

**5.2 CoT completeness check**

- [X] LD021: CoT tự mâu thuẫn ("does not explicitly ask")
- [X] LD024: CoT tự mâu thuẫn ("specific positions required" nhưng có answer)
- [ ] **TODO**: Human review LD021 và LD024 — viết lại CoT hoặc loại bỏ

**5.3 CoT-Answer alignment**

- [X] 53 records answer không xuất hiện trong 300 chars cuối CoT
- [ ] **TODO**: Review thủ công 53 records — phân biệt "answer đúng nhưng format khác" vs "answer sai"

**5.4 Meta-comment trong question**

- [X] LD011: `"(This is a very direct and clear translation.)"`
- [ ] **TODO LD011**: Xóa phần meta-comment khỏi question

**5.5 Lỗi nối văn bản trong question**

- [X] LD045: `"qCalculate"` — thiếu dấu chấm/xuống dòng
- [ ] **TODO LD045**: Sửa thành `"q₁. Calculate..."` hoặc tương đương

---

### Phase 6 — Phân tích phân phối ✅ HOÀN THÀNH

**6.1 Phân bố chủ đề**

- [X] Dataset tập trung vào điện học — Optics (4) và Mechanics (1) gần như vắng mặt
- [ ] **TODO**: Xác nhận với team xem thiếu Optics/Mechanics là intentional

**6.3 Phân bố answer = 0**

- [X] 18 records (1.3%) — không gây bias đáng kể

---

## Thứ tự ưu tiên thực hiện (cập nhật)

```
[CRITICAL] Phase 3.1  — Xử lý 164 records OTHER (multi-value, conceptual, unicode sci-notation)
[CRITICAL] Phase 3.4  — Tách 7 SYMBOLIC records sang tập riêng
[CRITICAL] Phase 5.2  — Sửa/loại LD021, LD024 (CoT tự mâu thuẫn)
[CRITICAL] Phase 5.1  — Sửa LD353 (CoT bị truncate), NL396 (format sai)
[HIGH]     Phase 5.3  — Review 53 records CoT-Answer misalignment
[HIGH]     Phase 4.1  — Chuẩn hóa unit: µ→μ, —→-, gán unit cho 14 empty
[HIGH]     Phase 3.3  — Chuẩn hóa scientific notation (Unicode superscript → ASCII)
[HIGH]     Phase 5.5  — Sửa LD045 (text concat)
[MEDIUM]   Phase 3.2  — Verify 18 records answer=0 qua CoT
[MEDIUM]   Phase 3.1  — Fix LD024 leading zero (06.04 → 6.04)
[MEDIUM]   Phase 5.4  — Xóa meta-comment LD011
[LOW]      Phase 2.1  — Xác nhận DDT numbering từ 131 và nguồn gốc
[LOW]      Phase 6.1  — Xác nhận thiếu Optics/Mechanics là intentional
```

---

## Output sau khi làm sạch

- `report_phase_X.csv` — danh sách record ID bị lỗi và loại lỗi
- `Physics_Problems_cleaned.csv` — dataset sau khi xử lý
- `Physics_Problems_symbolic.csv` — tập riêng cho các bài có answer symbolic
- `cleaning_log.md` — số records loại bỏ, sửa đổi, lý do

---

## Ghi chú quan trọng

> Cột `cot` chứa nhiều dòng (`\n` embedded) — **bắt buộc** phải parse bằng thư viện csv chuẩn với `quotechar='"'`, không dùng `split(',')` đơn giản.

> Answer symbolic (`\sqrt{2} × F₀`) chiếm 7 records — tách sang tập riêng nếu mục tiêu là train model sinh số.

> **164 records OTHER** là vấn đề lớn nhất cần xử lý: gồm text conceptual (EN), multi-value, percentage, expression. Cần quyết định: loại bỏ hay tách sang classification task.

> Dataset tập trung vào **điện học (electrostatics + circuits + EM)** — không có Optics và Mechanics. Cần xác nhận intentional trước khi kết luận dataset đủ cho "physics" tổng quát.

> **53 records** có potential CoT-Answer misalignment cần human review — đây là rủi ro chất lượng lớn vì model sẽ học CoT không nhất quán với answer.
