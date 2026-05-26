# Phần 1 - Phân tích logic

## 1. Tại sao Line Coverage đạt 90% nhưng vẫn tiềm ẩn nhiều bug logic?

**Line Coverage** (độ phủ dòng lệnh) chỉ đơn giản là một phép đếm: code của bạn có bao nhiêu dòng, và test case đã chạy qua được bao nhiêu dòng trong số đó.

Một dự án có thể đạt **90% (thậm chí 100%) Line Coverage** nhưng vẫn nhiều bug vì những lý do sau:

### Bỏ qua các nhánh ẩn (Implicit Branches)

Khi một câu lệnh `if` không có `else`, việc chỉ test kịch bản `true` đã đủ để phủ 100% số dòng code trong khối đó. Tuy nhiên, luồng dữ liệu khi điều kiện là `false` bị bỏ ngỏ hoàn toàn.

### Lỗi do trạng thái dữ liệu (Data State Bugs)

Dòng code thực hiện phép chia:

```java
a / b
```

được chạy qua và được tính là đã cover.

Nhưng test case chỉ truyền:

```java
b = 2
```

chứ không truyền:

```java
b = 0
```

Dòng code được cover 100% nhưng vẫn có thể crash trên staging nếu `b = 0`.

### Đánh giá đoản mạch (Short-circuit Evaluation)

Trong câu lệnh:

```java
if (A || B)
```

chỉ cần `A` đúng, hệ thống đã chạy vào bên trong mà không cần quan tâm đến `B`.

Dòng lệnh `if` được đánh dấu là **đã chạy**, nhưng hành vi khi:

- `A` sai, `B` đúng
- `A` đúng, `B` sai
- Cả hai cùng sai

chưa hề được kiểm chứng.

---

## 2. Phân biệt Line Coverage và Branch Coverage

| Tiêu chí | Line Coverage (Độ phủ dòng) | Branch Coverage (Độ phủ nhánh) |
|-----------|----------------------------|--------------------------------|
| **Định nghĩa** | Tỷ lệ các dòng lệnh (statements) được thực thi ít nhất 1 lần | Tỷ lệ các hướng đi/nhánh logic (true/false) được thực thi |
| **Mục tiêu** | Đảm bảo không có "code chết" (dead code) không bao giờ được gọi tới | Đảm bảo mọi đường đi của luồng kiểm soát (control flow) đều an toàn |
| **Độ khó** | Dễ đạt được chỉ với các test case cơ bản (Happy Path) | Khó hơn, đòi hỏi phải thiết kế test case chi tiết cho cả Happy Path và Unhappy Path (Edge Cases) |

---

## 3. Tầm quan trọng của Branch Coverage

Theo quy tắc nghiệp vụ:

> "Các điều kiện rẽ nhánh phải được kiểm tra cẩn thận cả hai nhánh (true và false)."

Branch Coverage ép buộc lập trình viên và QA phải suy nghĩ theo hướng:

> "Điều gì sẽ xảy ra nếu người dùng KHÔNG làm thế này?"

Bằng cách đo lường Branch Coverage, bạn sẽ ngay lập tức phát hiện ra:

- Những câu lệnh `if` chưa được test trường hợp `false`
- Những vòng lặp `for` chưa được test trường hợp mảng rỗng (`0 phần tử`)
- Những khối `try-catch` mà phần `catch` chưa bao giờ bị trigger

Đây chính là **"lưới lọc" giữ lại các lỗi logic nhỏ thường lọt lên staging.**

---

## 4. Ví dụ minh họa

### Ví dụ 1: Câu lệnh if thiếu nhánh else (Nhánh ẩn)

#### Code:

```java
public double calculateDiscount(boolean isVip, double totalAmount) {
    double discount = 0.0;            // Dòng 1

    if (isVip) {                      // Dòng 2
        discount = 0.2;               // Dòng 3
    }

    return totalAmount * (1 - discount); // Dòng 4
}
```

### Điểm thiếu sót của Line Coverage

Nếu chúng ta viết 1 test case:

```java
isVip = true
```

chương trình sẽ chạy qua:

- Dòng 1
- Dòng 2
- Dòng 3
- Dòng 4

Kết quả:

```text
Line Coverage = 100%
```

Dev có thể tự tin đóng task.

### Cách Branch Coverage phát hiện

Dù Line Coverage là:

```text
100%
```

Branch Coverage chỉ đạt:

```text
50%
```

Hệ thống sẽ báo rằng:

```java
if (isVip)
```

vẫn chưa đi qua nhánh:

```java
false
```

Điều này nhắc QA phải viết thêm test case:

```java
isVip = false
```

để đảm bảo khách hàng bình thường mua hàng vẫn xử lý:

```java
discount = 0.0
```

đúng logic.

---

### Ví dụ 2: Khối điều kiện phức tạp (Short-circuit)

#### Code:

```javascript
function applyCoupon(cart, couponCode) {

    if (cart !== null && cart.getTotal() > 100000) {
        cart.applyDiscount(couponCode);
    }

    return cart;
}
```

### Điểm thiếu sót của Line Coverage

Chỉ cần một test case:

```javascript
cart hợp lệ
total = 200000
```

toàn bộ:

- Dòng 1
- Dòng 2
- Dòng 3
- Dòng 4

đều chuyển xanh.

Kết quả:

```text
Line Coverage = 100%
```

Tuy nhiên nếu trên staging xảy ra:

```javascript
cart = null
```

Line Coverage cao vẫn không bảo vệ được hệ thống khỏi lỗi logic hoặc crash.

### Cách Branch Coverage phát hiện

Điều kiện:

```javascript
(cart !== null && cart.getTotal() > 100000)
```

thực chất chứa nhiều nhánh nhỏ.

Branch Coverage (hoặc sâu hơn là Condition Coverage) sẽ yêu cầu các kịch bản:

**Trường hợp 1:**

```javascript
cart === null
```

Mục tiêu:

- Kiểm tra có ném lỗi hay không
- Có đi xuống `return cart` an toàn hay không

**Trường hợp 2:**

```javascript
cart !== null
total < 100000
```

Mục tiêu:

- Kiểm tra logic chặn mã giảm giá

Nhờ vậy, Branch Coverage ép chúng ta mô phỏng gần như toàn bộ hành vi thực tế của môi trường chạy ứng dụng.

---

### Kết luận

- **Line Coverage** trả lời câu hỏi:

> "Code này đã được chạy chưa?"

- **Branch Coverage** trả lời câu hỏi:

> "Mọi hướng đi của code đã được kiểm tra chưa?"

Một dự án chỉ nhìn vào **Line Coverage cao** dễ tạo cảm giác an toàn giả. Trong thực tế, **Branch Coverage** thường có giá trị hơn để phát hiện lỗi logic và giảm bug trên môi trường staging/production.