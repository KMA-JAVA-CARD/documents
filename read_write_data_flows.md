### Quy trình mã hóa AES và lưu thông tin

#### Kịch Bản 1: Lưu Thông Tin (Mã Hóa - `setInfoSecure`)

1.  **Electron:** Gửi `PIN` ("123456") + `Info` ("Huy|KC...") xuống Java Middleware.
2.  **Java Middleware:**
    * Thêm Padding vào `Info` cho tròn 16 bytes (Ví dụ thêm 5 byte `00` vào cuối).
    * Gửi APDU `[PIN] [Info_Padded]` xuống thẻ.
3.  **Thẻ (Applet):**
    * **Bước 1 (Check PIN):** So khớp PIN đầu vào với PIN đã lưu trong `OwnerPIN`. Nếu sai -> Chặn ngay.
    * **Bước 2 (Tạo Khóa - `genAESKey`):**
        * Lấy PIN ("123456").
        * Cho qua `MessageDigest` (SHA-256) -> Ra chuỗi Hash 32 byte: `8d969e...`
        * Cắt lấy 16 byte đầu -> Nạp vào đối tượng `aesKey`. **Lúc này ta đã có khóa!**
    * **Bước 3 (Mã Hóa):**
        * Khởi động `aesCipher` ở chế độ `ENCRYPT` với `aesKey` vừa tạo.
        * Bỏ `Info_Padded` vào máy.
        * Máy phun ra `Ciphertext` (dữ liệu mã hóa loằng ngoằng).
    * **Bước 4 (Lưu):** Copy `Ciphertext` vào mảng `userData` (EEPROM).
    * **Kết quả:** Trên thẻ bây giờ chỉ chứa dữ liệu đã mã hóa. Nếu ai đó dump bộ nhớ thẻ, họ không đọc được gì cả.

#### Kịch Bản 2: Lấy Thông Tin (Giải Mã - `getInfoSecure`)

1.  **Electron:** Yêu cầu người dùng nhập lại PIN để xem thông tin -> Gửi `PIN` xuống thẻ.
2.  **Thẻ (Applet):**
    * **Bước 1 (Check PIN):** Kiểm tra PIN. Đúng mới đi tiếp.
    * **Bước 2 (Tái Tạo Khóa):**
        * Lại lấy PIN vừa nhập ("123456").
        * Cho qua `MessageDigest` -> Ra chuỗi Hash y hệt lúc trước (`8d969e...`).
        * Nạp vào `aesKey`. -> **Khóa giải mã chính là khóa mã hóa lúc nãy!**
    * **Bước 3 (Giải Mã):**
        * Khởi động `aesCipher` ở chế độ `DECRYPT` với `aesKey`.
        * Lấy `userData` (Ciphertext) từ bộ nhớ ra, bỏ vào máy.
        * Máy phun ra `Plaintext` ("Huy|KC...").
    * **Bước 4 (Gửi):** Trả `Plaintext` về cho Electron.

---