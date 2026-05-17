# Fix Log — Phase 3.1: Label Inversion (Answer "No" → "Yes")

**File áp dụng:** `Logic_Based_Educational_Queries.json`
**Ngày sửa:** 2026-05-17
**Người thực hiện:** Claude (tự động + xác minh thủ công trước khi sửa)

---

## Tóm tắt

| Hạng mục                         | Số lượng |
| --------------------------------- | -------- |
| Records cần kiểm tra (từ plan)    | 60       |
| Records đã sửa (No → Yes)        | **49**   |
| Records đã là Yes (sửa trước đó) | 10       |
| Records bỏ qua (chỉ có 1 câu hỏi)| 1        |

**Trường bị thay đổi:** `answers[1]` (Q-slot thứ 2, index 1, 0-based)
**Thay đổi:** `"No"` → `"Yes"`

---

## Mô tả lỗi

### Pattern lỗi hệ thống

Tất cả các records bị ảnh hưởng có cấu trúc giống nhau:

- **Q-slot 1** hỏi dạng: `"is the following statement true?"` / `"is the statement true?"`
- **`answers[1]`** = `"No"` ← **SAI**
- **`explanation[1]`** mô tả chuỗi suy luận dẫn đến kết luận statement là **đúng**

Ví dụ điển hình:

| Record | Câu hỏi (tóm tắt)                                          | answer (cũ) | Explanation kết luận                         |
| ------ | ----------------------------------------------------------- | ----------- | --------------------------------------------- |
| 43     | "There exists at least one student on the honor roll who is eligible for a scholarship." | No → **Yes** | "...at least one honor roll student must be eligible for a scholarship." |
| 82     | "If a student submits assignments on time, then they receive extra credit." | No → **Yes** | "This is exactly what is stated in premise 6..." |
| 275    | `Exists(x, AttendsTutoring(x))`                            | No → **Yes** | "The statement is true because there is at least one student who attends tutoring sessions, as explicitly stated." |
| 312    | `Exists(x, CompletingAssignments(x))`                      | No → **Yes** | "The statement is true because it is explicitly stated that there exists a student completing assignments." |

**Nguyên nhân khả năng:** Bug trong pipeline sinh dữ liệu — khi generate câu hỏi Q-slot 1 dạng `"is this statement true?"`, nhãn bị invert hàng loạt thay vì được gán đúng từ explanation.

---

## Danh sách 49 records đã sửa (No → Yes)

| # | Record | Q-slot | Câu hỏi (tóm tắt)                                                   |
|---|--------|--------|----------------------------------------------------------------------|
| 1 | **43** | 1 | "There exists at least one student on the honor roll who is eligible for a scholarship." |
| 2 | **75** | 1 | "If a school has students, then it has teachers." |
| 3 | **76** | 1 | "If everyone is a teacher, then there is at least one person." |
| 4 | **79** | 1 | "If submitting assignments on time implies good grades, then..." |
| 5 | **80** | 1 | "If achieving good grades implies studying regularly, and studying regularly implies..." |
| 6 | **82** | 1 | "If a student submits assignments on time, then they receive extra credit." |
| 7 | **87** | 1 | "Everyone is a student who understands lectures and manages time effectively." |
| 8 | **89** | 1 | "If someone studies, then they are both qualified and hardworking." |
| 9 | **90** | 1 | "If someone is an undergraduate, then they study." |
| 10 | **101** | 1 | "If a student attends all lectures, they will have a high GPA." |
| 11 | **102** | 1 | "If there exists at least one student who has received a full scholarship..." |
| 12 | **103** | 1 | "If there exists at least one student who has been disciplined..." |
| 13 | **104** | 1 | "If a student violates the school policy on academic integrity..." |
| 14 | **114** | 1 | "If a student does not do their homework, then they will not attend class." |
| 15 | **131** | 1 | "If a chemical reaction involves energy change, then it produces heat." |
| 16 | **135** | 1 | "Bob will be helped by his classmates." |
| 17 | **136** | 1 | "Jake passed Calculus 2." |
| 18 | **137** | 1 | "John has to study at the library." |
| 19 | **138** | 1 | "Mary got an A on the course." |
| 20 | **139** | 1 | "Peter Griffin will get a warning letter." |
| 21 | **140** | 1 | "Tu is confident in the math exam." |
| 22 | **141** | 1 | "Tuan can register for an internship course." |
| 23 | **142** | 1 | "Kahne will receive a certificate." |
| 24 | **143** | 1 | "Coke will have an academic warning." |
| 25 | **144** | 1 | "The Titan computer will be replaced." |
| 26 | **145** | 1 | "The AlphaNet model achieves high accuracy and will be deployed." |
| 27 | **146** | 1 | "JavaScript supports event handling to respond to user actions." |
| 28 | **168** | 1 | "There exists at least one person who has submitted a late assignment." |
| 29 | **169** | 1 | "If a person received a warning for previously late assignment, they cannot participate in group work." |
| 30 | **172** | 1 | "If a person applied for the exchange opportunity, they are qualified." |
| 31 | **173** | 1 | "If a person did not participate in organizational activities, they did not receive extra credit." |
| 32 | **174** | 1 | "If a person did not follow the school's policy, they did not pass the subject." |
| 33 | **177** | 1 | "If there exists someone who attended the midterm exam, then they will receive their score." |
| 34 | **178** | 1 | "If there exists someone who lends money to a friend, they are responsible." |
| 35 | **181** | 1 | "If a person is allowed to borrow books, then they have a library card." |
| 36 | **182** | 1 | "If all students are participating in the budget discussion, then all students are engaged." |
| 37 | **206** | 1 | "Is it true that some students study online?" |
| 38 | **221** | 1 | "All students have skills." |
| 39 | **222** | 1 | "If there exists a student who adheres to HCMUT's code of conduct and studies effectively..." |
| 40 | **223** | 1 | "If a student is enrolled in a course and receives advice from a counselor..." |
| 41 | **224** | 1 | "If a student is enrolled in a course, then they use online resources." |
| 42 | **225** | 1 | "If a student is not studying, then they are not enrolled in a course." |
| 43 | **228** | 1 | "If a student is preparing for an exam and collaborates with peers..." |
| 44 | **229** | 1 | "If a student is preparing for an exam, then they are revising." |
| 45 | **230** | 1 | "If there exists at least one student who is attending a tutorial session..." |
| 46 | **235** | 1 | "If a student is revising, then they are achieving good grades." |
| 47 | **238** | 1 | "If a high school student is not proficient in mathematics, then they are not enrolled." |
| 48 | **275** | 1 | `Exists(x, AttendsTutoring(x))` |
| 49 | **312** | 1 | `Exists(x, CompletingAssignments(x))` |

---

## Records đã là "Yes" (sửa từ trước — không cần can thiệp)

| Record | Trạng thái |
| ------ | ---------- |
| 110    | Đã là Yes  |
| 113    | Đã là Yes  |
| 167    | Đã là Yes  |
| 171    | Đã là Yes  |
| 176    | Đã là Yes  |
| 180    | Đã là Yes  |
| 184    | Đã là Yes  |
| 227    | Đã là Yes  |
| 234    | Đã là Yes  |
| 237    | Đã là Yes  |

---

## Records bỏ qua

| Record | Lý do                               |
| ------ | ------------------------------------ |
| 134    | Chỉ có 1 câu hỏi (Q-slot 1 không tồn tại) — không áp dụng pattern này |

---

## Tác động lên phân phối nhãn

**Trước khi sửa (từ kế hoạch ban đầu):**

```
No=300, Unknown=209, Yes=116, A=105, B=39, C=20, D=19
```

**Sau khi sửa Phase 3.1 (49 records No→Yes):**

```
No=251, Unknown=209, Yes=165, A=105, B=39, C=20, D=19
```

Tỉ lệ Yes:No trước: ~1:2.6 → Sau: ~1:1.52 (cân bằng hơn đáng kể)

---

## Xác minh sau khi sửa

Chạy script sau để xác nhận không còn record nào bị lỗi pattern:

```python
import json

with open('Logic_Based_Educational_Queries.json', encoding='utf-8') as f:
    data = json.load(f)

fixed = [43,75,76,79,80,82,87,89,90,101,102,103,104,114,131,135,136,137,
         138,139,140,141,142,143,144,145,146,168,169,172,173,174,177,178,
         181,182,206,221,222,223,224,225,228,229,230,235,238,275,312]

for idx_r in fixed:
    r = data[idx_r - 1]
    assert r['answers'][1] == 'Yes', f'Record {idx_r} still has wrong answer!'

print('All 49 records verified: answers[1] == "Yes" ✓')
```
