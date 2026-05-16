# Kế hoạch Kiểm tra & Làm sạch Dữ liệu

**Dataset:** `Logic_Based_Educational_Queries.json`
**Mục đích:** Chuẩn bị dữ liệu train LLM dựa trên thuật toán FOL
**Tổng số records:** 411 | **Ngày lập:** 2026-05-14

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
- `answers`: đáp án tương ứng (`"Yes"`, `"No"`, `"A"–"D"`, `"Unknown"`)
- `explanation`: giải thích lý luận cho từng câu hỏi

---

## Thống kê sơ bộ

| Chỉ số           | Giá trị                                                     |
| ---------------- | ----------------------------------------------------------- |
| Tổng records     | 411                                                         |
| Questions/record | 1–2 (avg ≈ 1.97)                                            |
| Premises/record  | 3–36 (avg ≈ 10.80)                                          |
| Phân phối answer | Yes=107, No=307, A=51, B=15, C=7, D=5, Unknown=315, False=1 |

---

## Vấn đề đã phát hiện qua đọc mẫu

| #   | Loại lỗi                           | Ví dụ cụ thể                                                                          |
| --- | ---------------------------------- | ------------------------------------------------------------------------------------- |
| 1   | **Encoding bị hỏng**               | Record 1,`premises-FOL[8]`: `"?forall(x, O(x) -> CR(x))"` — ký tự `∀` bị corrupt      |
| 2   | **Ký hiệu FOL không nhất quán**    | `∀x (...)` vs `ForAll(x, ...)` vs `forall(x, ...)` và `→` vs `->`                     |
| 3   | **Answer ≠ Explanation**           | Record Dr.John: answer Q1=`"Unknown"`, Q2=`"No"` nhưng explanation kết luận ngược lại |
| 4   | **Record gần trùng lặp**           | Record 2 & 3: cùng premises, chỉ đổi từ "strongest" → "correct", cùng answer `"C"`    |
| 5   | **Giá trị answer không nhất quán** | `"False"` xuất hiện 1 lần — không thuộc tập Yes/No/A/B/C/D/Unknown                    |
| 6   | **Tên predicate không nhất quán**  | `WT(x)`, `O(x)` (viết tắt) vs `completed_core_curriculum(x)` (mô tả đầy đủ)           |
| 7   | **idx mapping nghi ngờ**           | Q1 có `idx=[[1]]` nhưng explanation tham chiếu nhiều premises                         |

---

## Kế hoạch kiểm tra chi tiết

### Phase 1 — Kiểm tra cấu trúc (Structural Validation)

**Mức độ:** HIGH | **Có thể tự động hóa:** Hoàn toàn

**1.1 Schema integrity**

- [ ] Mỗi record có đủ 6 keys: `idx`, `premises-FOL`, `premises-NL`, `questions`, `answers`, `explanation`
- [ ] Không có field nào là `null` hoặc mảng rỗng `[]`

**1.2 Alignment giữa các mảng**

- [ ] `len(questions) == len(answers) == len(explanation) == len(idx)` cho từng record
- [ ] `len(premises-FOL) == len(premises-NL)` — hai mảng phải song song 1-1

**1.3 idx boundary check**

- [ ] Mỗi số trong `idx[i]` phải nằm trong `[1, len(premises-NL)]`
- [ ] Không có idx âm hoặc vượt quá số lượng premises

---

### Phase 2 — Kiểm tra chất lượng FOL (FOL Quality)

**Mức độ:** HIGH | **Có thể tự động hóa:** Phần lớn

**2.1 Phát hiện encoding bị hỏng**

- [ ] Tìm ký tự replacement `�` (U+FFFD) hoặc `?` ngay trước `forall`/`exists`
- [ ] Tìm pattern regex `[^\x00-\x7F∀-⋿]` trong chuỗi FOL
- [ ] Log danh sách record index bị ảnh hưởng để sửa tay

**2.2 Kiểm đếm & chuẩn hóa ký hiệu FOL**

Hiện tại dataset dùng nhiều cách viết khác nhau:

| Ký hiệu                | Các biến thể phát hiện được                    |
| ---------------------- | ---------------------------------------------- |
| Universal quantifier   | `∀x (...)`, `ForAll(x, ...)`, `forall(x, ...)` |
| Existential quantifier | `∃x (...)`, `Exists(x, ...)`                   |
| Implication            | `→`, `->`                                      |
| Conjunction            | `∧`, `^`                                       |
| Negation               | `¬`, `NOT`, `~`                                |

- [ ] Thống kê tần suất từng biến thể
- [ ] Chọn 1 chuẩn duy nhất (khuyến nghị: `∀`, `∃`, `→`, `∧`, `¬`)
- [ ] Áp dụng normalization script trên toàn bộ `premises-FOL`

**2.3 Kiểm tra cú pháp FOL cơ bản**

- [ ] Số lượng `(` phải bằng `)` trong mỗi premise
- [ ] Quantifier phải đi kèm variable và body: không chấp nhận `∀x` trống
- [ ] Predicate phải có dạng `name(args)` — không có predicate không có args

---

### Phase 3 — Kiểm tra tính nhất quán logic (Logical Consistency)

**Mức độ:** CRITICAL | **Có thể tự động hóa:** Một phần (cần human review)

**3.1 Answer ↔ Explanation contradiction**

Đây là lỗi nghiêm trọng nhất — khiến model học signal sai.

- [ ] **Với câu hỏi Yes/No:** Scan explanation tìm keyword xác nhận
  - Nếu answer=`"Yes"`: explanation phải chứa pattern khẳng định ở câu cuối
  - Nếu answer=`"No"`: explanation phải chứa pattern phủ định
  - Flag nếu mâu thuẫn để human review
- [ ] **Với câu hỏi MCQ (A/B/C/D):** Kiểm tra option letter được nhắc đến cuối explanation có khớp answer
- [ ] **Với answer=`"Unknown"`:** Kiểm tra explanation không kết luận dứt khoát một option cụ thể

**3.2 Answer ↔ idx consistency**

- [ ] Nếu answer=`"No"` hoặc `"Unknown"` nhưng `idx[i]` có đủ premises — kiểm tra logic
- [ ] Nếu `idx[i] == []` (rỗng) — record thiếu thông tin liên kết

**3.3 Human review list**

- [ ] Tất cả record có answer=`"Unknown"` mà explanation có vẻ dứt khoát → cần người kiểm tra
- [ ] Record Dr.John (offset ~line 420) là ví dụ cần sửa ngay

---

### Phase 4 — Phát hiện trùng lặp (Deduplication)

**Mức độ:** HIGH | **Có thể tự động hóa:** Hoàn toàn

**4.1 Exact duplicate**

- [ ] Hash `premises-NL` + `questions` + `answers` → tìm collision
- [ ] Nếu trùng hoàn toàn: giữ 1 record, xóa phần còn lại

**4.2 Near-duplicate (premises giống, question khác nhẹ)**

- [ ] So sánh `premises-FOL` giống nhau 100%
- [ ] Tính Jaccard similarity trên token của `questions` (ngưỡng > 0.85)
- [ ] Phân loại:
  - **Cùng answer:** Xem xét merge hoặc loại bỏ 1
  - **Khác answer:** Giữ cả 2, kiểm tra xem câu hỏi có thực sự khác nghĩa không

---

### Phase 5 — Phân tích phân phối (Distribution Analysis)

**Mức độ:** MEDIUM | **Có thể tự động hóa:** Hoàn toàn

**5.1 Label balance**

Hiện tại phân phối **mất cân bằng nghiêm trọng:**

```
Yes/No questions: Yes=107 (~26%), No=307 (~74%)  → tỉ lệ 1:3
MCQ questions:    A=51, B=15, C=7, D=5           → A chiếm ~66%
Unknown:          315 records                    → cần xem xét có nên giữ không
```

- [ ] Quyết định chiến lược xử lý imbalance: oversampling / undersampling / weighted loss
- [ ] Quyết định xem answer=`"Unknown"` có nên là một class riêng hay loại bỏ khỏi training

**5.2 Premise length distribution**

- [ ] Vẽ histogram số lượng premises/record
- [ ] Với records có 30+ premises: kiểm tra xem token length có vượt context window model không
- [ ] Xác định max token length và số records bị ảnh hưởng

**5.3 Topic diversity**

- [ ] Phân tích tần suất các chủ đề qua predicate names (Python, scholarship, faculty, transport...)
- [ ] Kiểm tra xem dataset có bị dominated bởi một vài chủ đề không

---

### Phase 6 — Tiêu chí xử lý record

| Tình trạng                                     | Hành động                            |
| ---------------------------------------------- | ------------------------------------ |
| Answer mâu thuẫn hoàn toàn với explanation     | Sửa tay hoặc loại bỏ                 |
| FOL encoding bị corrupt không phục hồi được    | Loại bỏ                              |
| `premises-FOL` và `premises-NL` khác độ dài    | Loại bỏ hoặc align lại tay           |
| `idx` tham chiếu premise không tồn tại         | Sửa idx hoặc loại bỏ                 |
| Answer=`"False"` (dị thường)                   | Đổi thành `"No"` nếu context phù hợp |
| Near-duplicate cùng answer                     | Loại bỏ bản trùng                    |
| Near-duplicate khác answer                     | Flag để human review                 |
| Answer=`"Unknown"` nhưng explanation dứt khoát | Human review + sửa answer            |

---

## Thứ tự ưu tiên thực hiện

```
[CRITICAL] Phase 3.1 — Answer/Explanation contradiction
[CRITICAL] Phase 1.1 — Schema integrity & array alignment
[HIGH]     Phase 2.1 — Encoding corruption detection
[HIGH]     Phase 4   — Deduplication
[MEDIUM]   Phase 2.2 — FOL notation normalization
[MEDIUM]   Phase 5.1 — Label imbalance analysis
[LOW]      Phase 1.3 — idx boundary check
[LOW]      Phase 5.2 — Token length distribution
```

---

## Output sau khi làm sạch

Mỗi bước nên tạo ra:

- `report_phase_X.json` — danh sách record index bị lỗi và loại lỗi
- `Logic_Based_Educational_Queries_cleaned.json` — dataset sau khi xử lý
- `cleaning_log.md` — ghi lại số records bị loại bỏ, sửa đổi, lý do

---

## Ghi chú quan trọng

> Việc chuẩn hóa ký hiệu FOL (Phase 2.2) phải được thực hiện **trước khi tokenize** vì model sẽ học `→` và `->` như các token khác nhau, ảnh hưởng trực tiếp đến khả năng suy luận logic.

> Answer=`"Unknown"` chiếm **~39% tổng answers** — cần quyết định rõ ràng về cách xử lý class này trước khi bắt đầu training.
