# Kế hoạch Kiểm tra & Làm sạch Dữ liệu

**Dataset:** `Physics_Problems_Text_Only.csv`
**Mục đích:** Chuẩn bị dữ liệu train LLM giải bài tập vật lý (Chain-of-Thought)
**Ngày lập:** 2026-05-14

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

## Vấn đề đã phát hiện qua đọc mẫu

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

## Kế hoạch kiểm tra chi tiết

### Phase 1 — Kiểm tra cấu trúc CSV (Structural Validation)

**Mức độ:** CRITICAL | **Có thể tự động hóa:** Hoàn toàn

**1.1 CSV parsing integrity**

- [ ] Parse toàn bộ file bằng thư viện chuẩn (Python `csv` module với `quotechar='"'`)
- [ ] Đếm số cột thực tế mỗi row — phải đúng 5 cột: `id, question, cot, answer, unit`
- [ ] Phát hiện row bị parse sai do `\n` trong cột `cot` không được quote đúng

**1.2 Missing values**

- [ ] Kiểm tra từng cột có giá trị rỗng không (`""` hoặc `NaN`)
- [ ] Cột `unit` có thể là `-` (không đơn vị) — phân biệt với thực sự missing

**1.3 Duplicate ID**

- [ ] Mỗi `id` phải là duy nhất trong toàn file
- [ ] Phát hiện ID bị trùng (cùng ID nhưng khác question/answer)

---

### Phase 2 — Kiểm tra ID và phân nhóm

**Mức độ:** MEDIUM | **Có thể tự động hóa:** Hoàn toàn

**2.1 Phân tích ID prefix**

- [ ] Liệt kê tất cả prefix xuất hiện (`TD`, `LD`, và các prefix khác)
- [ ] Kiểm tra xem mỗi prefix đại diện cho chủ đề/chương khác nhau không
- [ ] Đảm bảo không có ID bị đánh số sai thứ tự (VD: `LD001`... `LD047` có bị thiếu không)

**2.2 Thống kê phân bố theo ID prefix**

- [ ] Số lượng bài theo từng prefix → xác định cân bằng dataset theo chủ đề

---

### Phase 3 — Kiểm tra Answer (Đáp án)

**Mức độ:** CRITICAL | **Có thể tự động hóa:** Phần lớn

**3.1 Phân loại kiểu answer**

Hiện tại có các kiểu answer khác nhau:

| Kiểu                   | Ví dụ                                      | Hành động                             |
| ----------------------- | -------------------------------------------- | ---------------------------------------- |
| Số nguyên             | `45`, `7`, `0`                         | Hợp lệ                                 |
| Số thập phân         | `0.05`, `6.24`                           | Hợp lệ                                 |
| Khoa học chuẩn        | `1.23 × 10^-3`                            | Cần chuẩn hóa format                  |
| Khoa học không chuẩn | `24.45 × 10^-3`                           | Cần chuẩn hóa                         |
| LaTeX/symbolic          | `9\sqrt{3} × 10^-27`, `\sqrt{2} × F₀` | Cần xử lý riêng hoặc loại bỏ      |
| Chữ/định hướng     | `Hướng về phía q₂`                    | Loại bỏ hoặc tách thành task riêng |
| Leading zero            | `06.04`                                    | Sửa thành `6.04`                     |

**3.2 Kiểm tra answer = 0**

- [ ] Liệt kê tất cả records có answer = `0`
- [ ] Xác minh qua CoT: answer 0 phải được giải thích bởi tính đối xứng hoặc triệt tiêu lực

**3.3 Chuẩn hóa scientific notation**

- [ ] Thống nhất format: dùng `e` notation (`1.23e-3`) hoặc `× 10^` — chọn 1 chuẩn
- [ ] Xử lý khoảng cách không nhất quán: `10^-3` vs `10^{-3}` vs `10^-3`

**3.4 Quyết định về symbolic answers**

- [ ] Records có answer symbolic (`\sqrt{2} × F₀`) — không thể dùng để train regression/numeric prediction
- [ ] Lựa chọn: **loại bỏ** hoặc **giữ riêng** để train dưới dạng expression generation

---

### Phase 4 — Kiểm tra Unit (Đơn vị)

**Mức độ:** HIGH | **Có thể tự động hóa:** Phần lớn

**4.1 Chuẩn hóa unit**

| Vấn đề            | Ví dụ          | Chuẩn đề xuất                                       |
| -------------------- | ---------------- | ------------------------------------------------------- |
| Tiếng Việt         | `Độ`         | →`degree` hoặc `°`                               |
| Không có đơn vị | `-`            | Giữ nguyên hoặc đổi thành `dimensionless`       |
| Micro prefix         | `μF`, `μC` | Thống nhất ký tự `μ` (U+03BC) không dùng `u` |

**4.2 Answer-Unit consistency**

- [ ] Nếu answer là chữ (`Hướng về phía q₂`) nhưng unit = `-` — kiểm tra có phù hợp không
- [ ] Nếu answer là `0` với unit `N` — hợp lệ
- [ ] Nếu unit = `N` nhưng answer là symbolic expression — không nhất quán

---

### Phase 5 — Kiểm tra CoT (Chain-of-Thought)

**Mức độ:** HIGH | **Có thể tự động hóa:** Một phần

**5.1 Cấu trúc CoT**

- [ ] Mỗi CoT phải có ít nhất Step 1 đến Step 4
- [ ] Bước cuối cùng phải đưa ra kết quả số — không được kết thúc bằng câu hỏi hay nhận xét
- [ ] Phát hiện CoT kết thúc bằng dấu `.` chứ không phải bằng giá trị số

**5.2 CoT completeness check**

- [ ] Scan CoT tìm pattern `"does not explicitly ask"`, `"additional information is required"`, `"the specific positions ... are required"` — dấu hiệu CoT không hoàn chỉnh
- [ ] Kết quả cuối của CoT phải khớp với cột `answer`

**5.3 CoT-Answer alignment**

- [ ] Trích xuất số cuối cùng trong CoT và so sánh với cột `answer`
- [ ] Flag nếu số trong CoT ≠ answer (sai sót khi nhập liệu)

**5.4 Meta-comment trong question**

- [ ] Tìm pattern `"(This is a..."`, `"(Note:"`, `"(Translation..."` trong cột `question`
- [ ] Xóa các phần meta-comment không liên quan đến bài toán

**5.5 Lỗi nối văn bản trong question**

- [ ] Tìm pattern chữ liền với chữ hoa không có khoảng cách: `"...qCalculate"`, `"...μCFind"`, v.v.
- [ ] Sửa lại bằng cách thêm dấu cách hoặc dấu chấm câu phù hợp

---

### Phase 6 — Phân tích phân phối (Distribution Analysis)

**Mức độ:** MEDIUM | **Có thể tự động hóa:** Hoàn toàn

**6.1 Phân bố chủ đề vật lý**

- [ ] Phân loại bài theo chủ đề dựa trên keyword trong question: Coulomb's Law, resultant force, capacitor, electric field...
- [ ] Kiểm tra xem dataset có bị dominated bởi 1 chủ đề không

**6.2 Phân bố độ khó**

- [ ] Đo proxy độ khó: số bước CoT, số điện tích trong bài, có liên quan hình học không
- [ ] Đảm bảo không quá thiếu bài ở mức khó

**6.3 Phân bố answer = 0**

- [ ] Đếm tỉ lệ answer = `0` — nếu quá nhiều sẽ khiến model bias về `0`

---

### Phase 7 — Tiêu chí xử lý record

| Tình trạng                                               | Hành động                                              |
| ---------------------------------------------------------- | --------------------------------------------------------- |
| Answer là symbolic expression (`\sqrt{2} × F₀`)       | Tách sang tập riêng hoặc loại khỏi numeric training |
| Answer là chữ định hướng (`Hướng về phía q₂`) | Loại bỏ hoặc tách sang classification task            |
| CoT kết thúc không hoàn chỉnh                         | Sửa tay hoặc loại bỏ                                  |
| Question có lỗi nối text (`qCalculate`)               | Sửa text                                                 |
| Meta-comment trong question                                | Xóa comment                                              |
| Answer có leading zero (`06.04`)                        | Sửa thành `6.04`                                      |
| Answer không khớp với CoT                               | Human review + sửa                                       |
| Unit dùng tiếng Việt (`Độ`)                         | Chuẩn hóa thành tiếng Anh                             |
| ID trùng lặp                                             | Điều tra nguồn gốc, giữ bản chính xác hơn        |

---

## Thứ tự ưu tiên thực hiện

```
[CRITICAL] Phase 1.1 — CSV parsing integrity (multi-line CoT)
[CRITICAL] Phase 3.1 — Phân loại và chuẩn hóa answer format
[CRITICAL] Phase 5.2 — CoT completeness (CoT không ra kết quả)
[HIGH]     Phase 5.3 — CoT-Answer alignment
[HIGH]     Phase 4   — Chuẩn hóa unit
[HIGH]     Phase 5.5 — Lỗi nối văn bản trong question
[MEDIUM]   Phase 2   — Phân tích ID prefix
[MEDIUM]   Phase 5.4 — Xóa meta-comment
[MEDIUM]   Phase 6   — Distribution analysis
[LOW]      Phase 1.3 — Duplicate ID check
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

> Answer symbolic (`\sqrt{2} × F₀`) chiếm một phần nhỏ nhưng cần quyết định rõ: nếu mục tiêu là train model sinh ra số, các record này phải được tách ra. Nếu mục tiêu là sinh biểu thức đại số, chúng rất có giá trị.

> Dataset hiện tập trung chủ yếu vào **tĩnh điện học (electrostatics)** — cần xác nhận xem đây là intentional hay còn thiếu các chủ đề vật lý khác.
