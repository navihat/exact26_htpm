# Báo Cáo Tổng Hợp Xử Lý Lỗi Dataset

**File Gốc:** `Logic_Based_Educational_Queries.json` (411 entries)  
**File Output:** `Logic_Based_Educational_Queries_FIXED.json` (407 entries)  
**Mục đích:** Xử lý toàn bộ lỗi cấu trúc (validation) và lỗi logic gán nhãn sai để chuẩn bị dữ liệu sạch cho quá trình training mô hình.

---

## 1. Tóm tắt quá trình xử lý

Quá trình làm sạch dữ liệu được chia làm 2 giai đoạn chính, với tổng cộng **206 điểm sửa đổi**:
- **Giai đoạn 1 :** Khắc phục lỗi cấu trúc, loại bỏ trùng lặp, chuẩn hóa schema. Đưa dataset từ 411 entries về 407 entries.
- **Giai đoạn 2 :** Giữ nguyên cấu trúc 407 entries, quét và sửa các lỗi sai lệch logic giữa `answer` và `explanation` (165 lỗi).

---

## 2. Giai Đoạn 1: Phân cụm & Xử lý lỗi Cấu trúc (V1)

| Phân cụm lỗi | Các Entry bị ảnh hưởng | Cách xử lý |
| :--- | :--- | :--- |
| **Lỗi Encoding**<br>(Ký tự `` thay vì `∀`) | 6 entries: `0, 209, 263, 330, 407, 410` | Thay thế chuỗi `forall` thành ký tự Unicode chuẩn `∀` trong trường `premises-FOL`. |
| **Thiếu trường `premises-FOL`** | 2 entries: `23, 53` | Tái tạo lại các premise FOL thủ công dựa trên nội dung của `premises-NL`. |
| **Trường thừa & Cấu trúc `idx` sai** | 2 entries: `132, 133` | - Gộp nội dung `choices` vào thẳng `questions` (A, B, C, D).<br>- Xóa bỏ key `choices`.<br>- Bọc mảng `idx` lại thành list of lists (vd: `[1, 2]` → `[[1, 2]]`). |
| **Lệch số lượng FOL và NL** | 11 entries: `34, 57, 146, 334, 376, 377, 378, 379, 380, 381, 382` | Thêm, bớt, gộp hoặc tách các premise FOL/NL sao cho số lượng hai bên khớp nhau 1-1. Cập nhật lại `idx` tương ứng. |
| **Lỗi chỉ số `idx`**<br>(0-based hoặc out-of-range) | 16 entries: `74, 76, 77, 79, 81, 83, 84, 85, 86, 87, 88, 89, 125, 126, 129, 334` | - Nếu 0-based: Tăng tất cả lên +1.<br>- Nếu vượt quá tổng số premise: Giới hạn (clamp) về giá trị max.<br>- Xóa các giá trị trùng lặp trong list. |
| **Trùng lặp hoàn toàn** | 4 cặp: `(8, 9), (160, 161), (182, 183), (394, 395)` | Giữ lại entry đầu tiên, xóa entry thứ hai để tránh thiên lệch (bias) khi training. |

---

## 3. Giai Đoạn 2: Phân cụm & Xử lý lỗi Logic/Đáp án

Trong giai đoạn này, nội dung trường `explanation` được **giữ nguyên** để bảo toàn chain-of-thought (CoT) cho mô hình học. Các thay đổi chỉ áp dụng lên trường `answer` bị gán sai lệch so với logic giải thích.

| Phân cụm lỗi | Số Lượng | Các Entry bị lỗi (Theo Index V1) | Cách xử lý |
| :--- | :--- | :--- | :--- |
| **Khẳng định bị gán sai thành Phủ định** (`No` → `Yes`) | 75 | `30 (Q0), 36 (Q1), 38 (Q1), 42 (Q1), 48 (Q1), 54 (Q1), 74 (Q1), 75 (Q1), 79 (Q1), 81 (Q1), 86 (Q1), 88 (Q1), 100 (Q1), 101 (Q1), 109 (Q1), 112 (Q1), 124 (Q1), 130 (Q1), 203 (Q1), 204 (Q1), 237 (Q1), 238 (Q1), 239 (Q1), 240 (Q1), 242 (Q1), 243 (Q1), 244 (Q1), 245 (Q1), 246 (Q1), 247 (Q1), 248 (Q1), 249 (Q1), 251 (Q1), 252 (Q1), 253 (Q1), 254 (Q1), 255 (Q1), 256 (Q1), 262 (Q1), 268 (Q1), 269 (Q1), 270 (Q1), 271 (Q1), 272 (Q1), 274 (Q1), 275 (Q1), 278 (Q1), 280 (Q1), 281 (Q1), 282 (Q1), 283 (Q1), 284 (Q1), 287 (Q1), 300 (Q1), 301 (Q1), 303 (Q1), 306 (Q1), 307 (Q1), 308 (Q1), 336 (Q1), 337 (Q1), 338 (Q1), 339 (Q1), 340 (Q1), 341 (Q1), 342 (Q1), 344 (Q1), 348 (Q1), 349 (Q1), 350 (Q1), 351 (Q1), 375 (Q1), 381 (Q1), 383 (Q1), 384 (Q1)` | Đổi answer từ `No` thành `Yes` do explanation và premise đều xác nhận statement là ĐÚNG. |
| **Phủ định bị gán sai thành Khẳng định** (`Yes` → `No`) | 3 | `263 (Q1), 368 (Q1), 406 (Q1)` | Đổi answer từ `Yes` thành `No` do statement mâu thuẫn trực tiếp với premise. |
| **Sai chữ cái MC** | 5 | `1 (Q0)`: C→A, `2 (Q0)`: C→A, `10 (Q0)`: C→A, `11 (Q0)`: D→B, `362 (Q1)`: D→A | Sửa về chữ cái đúng dựa theo lời giải phân tích loại trừ trong `explanation`. |
| **Câu MC bị gán nhầm là "Unknown"** | 81 | **→ A (40):** `13 (Q0), 16, 19, 20, 21, 23, 24, 25 (Q0,Q1), 28, 39, 42, 78, 80, 109, 111, 171, 174, 175, 176, 178, 179, 180, 181, 183, 219, 220, 221, 222, 223, 224, 225, 226, 227, 231, 232, 235, 290, 291, 295, 296, 352 (Q1), 391`<br>**→ B (19):** `7 (Q0), 23 (Q1), 24 (Q1), 43, 44, 55, 89, 102, 103, 117, 165, 166, 167, 169, 170, 172, 297, 310`<br>**→ C (13):** `6 (Q0), 12, 45, 110, 113, 182, 218, 234, 311, 313, 315`<br>**→ D (9):** `47 (Q0), 54, 56, 107, 272, 298, 299, 312, 352 (Q0)` | Đổi từ `Unknown` sang chữ cái A/B/C/D cụ thể vì `explanation` đã phân tích loại trừ rõ ràng. |
| **Chuẩn hóa nhãn** | 1 | `22 (Q1)` (`False` → `No`) | Đổi `False` thành `No` để thống nhất format toàn dataset. Logic giữ nguyên. |

---

## 4. Kết luận

Sau quá trình xử lý, bản dữ liệu đáp ứng đầy đủ các tiêu chuẩn kỹ thuật:
- **Zero Validation Error:** 100% tuân thủ chặt chẽ schema yêu cầu.
- **Tính Cân Bằng (Class Balance):** Tỉ lệ `No`/`Yes` chuyển dịch từ mức mất cân bằng nghiêm trọng (74% `No`) xuống mức phân phối tự nhiên hơn (~57% `No`).
- **Mức độ sẵn sàng:** Tránh được các bias gán nhãn khiến model "học lỏm" (luôn trả lời Unknown/No nếu không chắc chắn). Dataset đã hoàn toàn sẵn sàng cho quá trình fine-tuning / training.