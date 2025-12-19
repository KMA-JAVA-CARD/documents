### 💳 Quy Trình Thanh Toán & Tích/Đổi Điểm (Loyalty Transaction)

Quy trình này đảm bảo tính toàn vẹn dữ liệu (Data Integrity) và chống chối bỏ (Non-repudiation) bằng cách kết hợp xác thực PIN, Chữ ký số RSA và cơ chế đồng bộ dữ liệu lai (Hybrid Sync).

#### Tổng Quan Luồng Xử Lý

Hệ thống sử dụng cơ chế **3-Bước (3-Step Flow)**:

1. **Sync-First:** Đồng bộ số dư chuẩn từ Server về Thẻ.
2. **Secure Transaction:** Thực hiện giao dịch có ký số (RSA Signature).
3. **Finalize:** Cập nhật số dư mới nhất vào Thẻ.

---

#### Kịch Bản 1: Thanh Toán & Tích Điểm (Pay & Earn)

*Người dùng thanh toán bằng Tiền mặt/Thẻ tín dụng và được cộng điểm thưởng vào thẻ thành viên.*

**1. Khởi Tạo & Xác Thực (Pre-Transaction):**

* **Electron:** Kết nối đầu đọc, yêu cầu người dùng nhập PIN.
* **Middleware -> Thẻ:** Gửi lệnh `VERIFY_PIN`.
* *Thẻ:* Kiểm tra PIN và thực hiện verify bằng ký số. Nếu tất cả đúng -> Mở khóa thẻ (PIN Authenticated & RSA challenge passed).


* **Electron -> Backend:** Gọi API lấy thông tin Member & Số điểm (Database Balance).
* **Electron -> Middleware:**
* So sánh điểm trên thẻ (Card Balance) và điểm từ Server (DB Balance).
* Nếu lệch: Gọi lệnh `UPDATE_POINTS` để ghi đè số điểm chuẩn từ Server vào Thẻ (Sync-First).



**2. Chuẩn Bị Giao Dịch (Preparation):**

* **Electron:** Tính toán số tiền (`amount`) và điểm thưởng dự kiến.
* **Electron:** Tạo chuỗi dữ liệu gốc (Raw Data) để ký:
```
Format: "EARN|{AMOUNT}|{TIMESTAMP}"
Ví dụ:  "EARN|500000|1702891234567"

```


* **Electron:** Chuyển chuỗi Raw Data sang Hex.

**3. Ký Số Giao Dịch (Signing - Non-repudiation):**

* **Electron -> Middleware:** Gửi yêu cầu `signChallenge(dataHex)`.
* **Middleware -> Thẻ:** Gửi APDU `INS_SIGN_CHALLENGE` (0x33).
* **Thẻ (Applet):**
* Kiểm tra trạng thái PIN (phải `isValidated`).
* Dùng **Private Key** (RSA 1024) ký lên dữ liệu Hash của giao dịch.
* Trả về: **Chữ ký số (Signature Hex)**.



**4. Xử Lý Giao Dịch (Processing):**

* **Electron -> Backend:** Gọi API `POST /transaction`.
* *Body:* `{ type: 'EARN', amount: 500000, signature: '...', rawData: '...' }`


* **Backend (NestJS):**
* Lấy Public Key của thẻ trong DB.
* **Verify Signature:** Kiểm tra xem `signature` có đúng là được ký từ `rawData` bởi thẻ này không. (Chống giả mạo request).
* Tính toán điểm thưởng: `500,000 / 10,000 = 50 điểm`.
* Cập nhật DB: `Balance = Balance + 50`.
* Lưu lịch sử giao dịch kèm chữ ký làm bằng chứng.
* Trả về: `{ success: true, newBalance: 150 }`.



**5. Đồng Bộ Cuối (Finalization):**

* **Electron:** Nhận `newBalance` (150).
* **Electron -> Middleware:** Gọi `updatePoints(150)`.
* **Middleware -> Thẻ:** Gửi APDU `INS_UPDATE_POINTS` (0x41) chứa số điểm mới (High Byte/Low Byte).
* **Thẻ:** Ghi số dư mới vào bộ nhớ EEPROM.
* **Electron:** Hiển thị thông báo "Thành công".

---

#### Kịch Bản 2: Thanh Toán Bằng Điểm (Redeem)

*Người dùng sử dụng điểm tích lũy để trừ vào hóa đơn.*

**Quy trình tương tự kịch bản EARN, chỉ khác ở Bước 2 và Bước 4:**

**2. Chuẩn Bị Giao Dịch:**

* **Electron:** Kiểm tra số dư hiện tại có đủ không.
* **Electron:** Tạo chuỗi dữ liệu ký:
```
Format: "REDEEM|{POINTS_TO_USE}|{TIMESTAMP}"
Ví dụ:  "REDEEM|50|1702891234567"

```



**4. Xử Lý Giao Dịch (Backend):**

* **Backend:**
* Verify Signature.
* Kiểm tra số dư trong DB: `CurrentBalance >= 50`?
* Nếu đủ:
* Cập nhật DB: `Balance = Balance - 50`.
* Trả về: `{ success: true, newBalance: 100 }`.


* Nếu không đủ: Trả về Lỗi (kể cả khi thẻ hiển thị đủ điểm nhưng DB bảo không -> Từ chối, vì DB là "Source of Truth").



---

#### 🛡️ Các Cơ Chế Bảo Mật Áp Dụng

| Cơ chế | Mục đích | Tại bước |
| --- | --- | --- |
| **PIN Verification** | Xác thực chủ sở hữu thẻ, mở khóa thẻ. | Trước khi giao dịch |
| **RSA Signature** | **Chống chối bỏ (Non-repudiation):** Chứng minh giao dịch được thực hiện bởi thẻ thật, không phải request giả mạo từ Postman/Hacker. | Bước 1, 3 & 4 |
| **Data Integrity Hash** | Đảm bảo dữ liệu (Số tiền, Loại giao dịch) không bị sửa đổi trên đường truyền. | Bước 3 (Ký trên Hash) |
| **Hybrid Sync** | **Chống Hack điểm Offline:** Luôn lấy số dư Server ghi đè lên thẻ trước và sau giao dịch. Thẻ chỉ đóng vai trò hiển thị nhanh (Cache). | Bước 1 & 5 |
| **Timestamping** | **Chống tấn công phát lại (Replay Attack):** Hacker không thể bắt gói tin cũ (có chữ ký cũ) để gửi lại lần 2, vì Timestamp đã cũ hoặc Server đã lưu vết signature này. | Bước 2 |
| | | |
