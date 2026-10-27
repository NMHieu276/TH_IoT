# Bài 2 — Đọc dữ liệu cảm biến và kiểm tra lỗi

**Sinh viên:** Nguyễn Minh Hiếu — **MSSV:** 23110168

## Nội dung nộp

| File | Mô tả |
|---|---|
| [`sensor(delay).ino`](./sensor(delay).ino) | Chương trình đọc cảm biến, chờ bằng `delay()` |
| [`sensor(millis).ino`](./sensor(millis).ino) | Chương trình đọc cảm biến không chặn bằng `millis()`, kèm kiểm tra ADC và quét I2C |
| [`data.csv`](./data.csv) | Dữ liệu thu được từ Serial Monitor |
| [`bieudo.png`](./bieudo.png) | Biểu đồ dữ liệu gốc và dữ liệu sau lọc |
| [`so_do_ket_noi.png`](./so_do_ket_noi.png) | Sơ đồ nối dây trên mô phỏng |

## 1. Thiết lập thí nghiệm

- **Cách thực hiện:** chọn phương án A (đo trong thời gian ngắn), chạy mô phỏng trên Wokwi.
- **Cảm biến:** Wokwi không có DHT11 nên em dùng DHT22 để thay thế, chân DATA nối vào **GPIO 14**.
- **Phần mở rộng:**
  - Biến trở nối **GPIO 34** để đọc thử ADC (giá trị 0–4095, quy đổi ra điện áp 0–3.3 V).
  - LCD1602 giao tiếp I2C qua **GPIO 21 (SDA)** và **GPIO 22 (SCL)**, đồng thời dùng để quét địa chỉ trên bus I2C.
- **Chu kỳ lấy mẫu:** 2 giây/lần. DHT cần khoảng thời gian tối thiểu giữa hai lần đọc theo datasheet, đọc nhanh hơn dễ nhận giá trị lỗi.
- **Định dạng log:** `timestamp,raw_value,converted_value,unit,valid`

## 2. Kết nối chân

| Linh kiện | Tín hiệu | GPIO ESP32 |
| :--- | :--- | :--- |
| DHT22 (thay cho DHT11) | DATA | 14 |
| LCD1602 I2C | SDA / SCL | 21 / 22 |
| Biến trở | Ngõ vào ADC | 34 |

## 3. Phát hiện và báo lỗi cảm biến

- Trong lúc mô phỏng đang chạy, Wokwi không cho sửa mạch, vì vậy em dừng mô phỏng, tháo dây DATA của cảm biến rồi chạy lại.
- Khi đó `dht.readTemperature()` trả về `nan`. Chương trình kiểm tra bằng `isnan()`, gán cột `valid = 0` và hiện `Loi cam bien!` trên LCD.
- Dòng log ghi nhận được:

  ```
  2585,nan,nan,C,0
  ```

## 4. Lọc nhiễu dữ liệu

- **Phương pháp:** trung bình trượt (Moving Average), cửa sổ gồm 5 mẫu gần nhất.
- **Nhận xét về độ mượt:** khi em thay đổi nhiệt độ liên tục trên cảm biến, dữ liệu gốc dao động rất mạnh (từ 21.4 °C nhảy lên 45.8 °C) và biểu đồ có nhiều đỉnh nhọn. Sau khi lọc, các bước nhảy được san phẳng, đường cong liền mạch hơn.
- **Nhận xét về độ trễ:** vì mỗi giá trị lọc là trung bình của 5 mẫu (mỗi mẫu cách nhau 2 s), đường đã lọc phản ứng chậm hơn tín hiệu thật khoảng 4–6 giây mỗi khi nhiệt độ thay đổi đột ngột. Cửa sổ càng lớn thì càng mượt nhưng càng trễ.

## 5. So sánh `delay()` và `millis()`

| Tiêu chí | Bản A – `delay(2000)` | Bản B – `millis()` |
|---|---|---|
| Cơ chế chờ | Chặn: CPU đứng yên trong 2 s | Không chặn: chỉ so sánh thời gian rồi tiếp tục vòng `loop()` |
| Phản hồi khi gõ phím trên Serial Monitor | Phải đợi hết khoảng `delay` mới in ra thông báo | In ra thông báo gần như ngay lập tức |
| Chu kỳ đọc cảm biến | 2 s (cộng thêm thời gian xử lý) | Đúng 2 s nhờ so sánh với `previousMillis` |

**Kết luận:** bản dùng `millis()` phù hợp hơn cho hệ thống IoT vì vẫn đọc cảm biến đúng chu kỳ mà không làm treo các tác vụ khác như nhận lệnh hay xử lý giao tiếp.
