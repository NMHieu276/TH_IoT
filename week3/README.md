# Bài 3 — Điều khiển thiết bị an toàn và Máy trạng thái (FSM)

| Thông tin | Nội dung |
|---|---|
| Sinh viên thực hiện | Nguyễn Minh Hiếu |
| MSSV | 23110168 |
| Bài thực hành | TH03 — Điều khiển bơm DC an toàn với ESP32 |

## Danh mục tệp trong bài

| Tệp tin | Vai trò / Mô tả |
|---|---|
| [`control.ino`](./control.ino) | Mã nguồn chính ESP32: FSM 3 trạng thái (DỪNG/CHẠY/LỖI), ngắt ngoài, timer phần cứng, giao diện Web Server |
| [`fsm_diagram.png`](./fsm_diagram.png) | Sơ đồ chuyển đổi giữa các trạng thái hoạt động |
| [`secrets.h.example`](./secrets.h.example) | Tệp mẫu khai báo SSID và mật khẩu WiFi |

---

## 1. Các thông số cấu hình chính

Em đã lựa chọn và thiết lập các thông số thời gian cùng mức công suất như sau:

| Thông số | Hằng số mã nguồn | Giá trị | Mục đích & Ý nghĩa |
|---|---|---|---|
| Thời gian chạy tối đa | `MAX_RUN_MS` | 20 s | Giới hạn thời gian bơm hoạt động liên tục (phòng tránh tràn hoặc cháy bơm) |
| Thời gian giữ lỗi | `FAULT_HOLD_MS` | 5 s | Khóa hệ thống trong 5 s khi có sự cố, sau đó tự hồi phục về trạng thái DỪNG |
| Thời gian chống dội | `DEBOUNCE_MS` | 50 ms | Lọc nhiễu rung tiếp điểm cơ khí cho nút nhấn |
| Chu kỳ timer phần cứng | `TIMER_PERIOD_US` | 2.0 s (2 000 000 µs) | Định kỳ đọc cảm biến I2C, cập nhật trạng thái và ghi log CSV |
| Các mức công suất PWM | `DUTY_25`, `DUTY_50`, `DUTY_90` | 64, 127, 229 | Tương ứng 25%, 50% và 90% duty cycle trên thang 8-bit (0–255) |

---

## 2. Thiết lập phần cứng và sơ đồ chân (ESP32)

Dựa trên sơ đồ nguyên lý bo mạch Farm House (`farm_house_schematic_rev1.0.1`):

| Chân GPIO | Tín hiệu kết nối | Thiết bị / Linh kiện | Chức năng thực hiện |
|---|---|---|---|
| **GPIO 5** | `MOTOR1_INT1` | Driver công suất A4950ELJTR (IN1) | Điều chế PWM kênh 0 (LEDC), tần số 5 kHz, điều chỉnh tốc độ bơm |
| **GPIO 17** | `MOTOR1_INT2` | Driver công suất A4950ELJTR (IN2) | Cố định mức logic LOW khi bơm vận hành (chiều quay/hãm) |
| **GPIO 36** | `BTN` | Nút bấm vật lý (ngõ vào SENSOR_VP) | Bắt ngắt ngoài `FALLING/CHANGE`. Chân này không hỗ trợ pull-up nội nên cần trở kéo lên 10 kΩ bên ngoài |
| **GPIO 26** | `LED_G` | LED chỉ báo màu xanh | Báo trạng thái hoạt động: Sáng khi CHẠY, Tắt khi DỪNG/LỖI |
| **GPIO 33** / **GPIO 32** | `SDA` / `SCL` | Bus I2C kết nối Arduino Slave | Đọc dữ liệu nhiệt độ, độ ẩm từ slave (địa chỉ `0x08`) |

> *Lưu ý khởi tạo:* Trong hàm `setup()`, GPIO 17 và chân LED được ép về LOW, kênh PWM đưa về 0 trước khi khởi tạo ngắt và timer nhằm bảo đảm an toàn, không kích hoạt tải ngoài ý muốn khi vừa bật nguồn.

---

## 3. Cơ chế ngắt ngoài và chống dội nút nhấn

- **Ngắt ngoài (`btnISR`):** 
  - Hàm phục vụ ngắt được đặt thuộc tính `IRAM_ATTR` để thực thi nhanh trong bộ nhớ RAM.
  - Tuân thủ nguyên tắc phi phong tỏa: chỉ tăng biến đếm `volatile uint32_t btnRawCount`, tuyệt đối không gọi `delay()`, không in `Serial` hay xử lý logic phức tạp trong ISR.
- **Xử lý chống dội bằng phần mềm (Software Debounce):**
  - Thực hiện kiểm tra trong vòng lặp `loop()`. Khi phát hiện biến đếm thay đổi, hệ thống lưu mốc thời gian và chờ sau khi tín hiệu ổn định tối thiểu 50 ms mới xác nhận 1 lần nhấn hợp lệ.
  - Giải pháp này giúp tiết kiệm linh kiện phần cứng (tụ chống dội) mà vẫn đảm bảo độ tin cậy.

---

## 4. Timer phần cứng và điều khiển công suất PWM

- **Timer phần cứng (Hardware Timer 0):**
  - Cấu hình đếm với prescaler 80 (đưa xung nhịp 80 MHz về 1 MHz, tương đương mỗi tick = 1 µs).
  - Đặt ngưỡng alarm ở 2 000 000 µs (2 giây) và kích hoạt chế độ tự động nạp lại (auto-reload).
  - ISR ngắt timer (`onTimer`) chỉ bật cờ `timerFlag = true`. Việc truy vấn cảm biến I2C và in dòng trạng thái CSV qua Serial Monitor được xử lý ngoài `loop()` khi kiểm tra thấy cờ này bật.
- **Điều khiển PWM (LEDC):**
  - Sử dụng module LEDC phần cứng của ESP32 với tần số 5 kHz, độ phân giải 8-bit (0–255), giúp bơm vận hành êm và phản hồi tức thời.

---

## 5. Máy trạng thái (FSM) và kịch bản thử nghiệm

Hệ thống hoạt động theo mô hình máy trạng thái 3 trạng thái an toàn:

`DỪNG` ⇄ `CHẠY` → `LỖI` (sau khi hết 20s hoặc cảm biến lỗi) → tự động về `DỪNG` (sau 5s)

- **Trạng thái DỪNG (`STATE_STOP`):** Bơm tắt hoàn toàn, LED tắt. Đây là trạng thái an toàn mặc định lúc khởi động.
- **Trạng thái CHẠY (`STATE_RUN`):** Bơm kích hoạt theo duty cycle đã chọn, LED sáng. Bộ đếm thời gian liên tục so sánh với `MAX_RUN_MS`.
- **Trạng thái LỖI (`STATE_ERROR`):** Bơm lập tức bị ngắt, bỏ qua mọi lệnh yêu cầu CHẠY từ cả nút nhấn và Web. Sau khoảng giữ lỗi `FAULT_HOLD_MS = 5 s`, hệ thống tự động đưa về trạng thái DỪNG an toàn.

### Các ca kiểm thử thực tế

1. **Ca 1 — Vận hành bình thường:**
   - *Thao tác:* Nhấn nút hoặc gửi lệnh `start` từ Web khi hệ thống đang ở trạng thái DỪNG.
   - *Kỳ vọng & Kết quả:* Trạng thái chuyển sang CHẠY, bơm hoạt động đúng duty cycle đã đặt, LED xanh sáng. Nhấn lại nút hoặc gửi `stop` thì chuyển ngay về DỪNG.
2. **Ca 2 — Tự ngắt quá thời gian (Timeout Protection):**
   - *Thao tác:* Kích hoạt bơm CHẠY nhưng không phát lệnh tắt.
   - *Kỳ vọng & Kết quả:* Sau đúng 20 giây (`MAX_RUN_MS`), hệ thống phát hiện quá thời gian, tự ngắt bơm và chuyển sang trạng thái LỖI (`TIMEOUT`). Trong 5 giây giữ lỗi, lệnh bật bị từ chối; sau 5 giây tự quay về DỪNG.
3. **Ca 3 — Bảo vệ khi cảm biến gặp sự cố:**
   - *Thao tác:* Trong khi bơm đang CHẠY, ngắt đường truyền cảm biến (rút dây I2C hoặc slave phản hồi lỗi).
   - *Kỳ vọng & Kết quả:* Dữ liệu cảm biến không hợp lệ (`SENSOR_INVALID`), máy trạng thái lập tức dừng bơm và chuyển sang LỖI, sau đó hồi phục về DỪNG.

---

## 6. Các phương thức điều khiển hệ thống

1. **Nút bấm vật lý (GPIO 36):** Nhấn để đảo trạng thái giữa DỪNG và CHẠY.
2. **Giao diện Web (`http://<IP_ESP32>/`):**
   - `POST /api/cmd?action=start|stop` : Bật / tắt bơm.
   - `POST /api/duty?pct=25|50|90` : Đặt công suất bơm.
   - `GET /api/status` : Trả về dữ liệu JSON gồm trạng thái, duty cycle, thời gian chạy, nhiệt độ và độ ẩm.
3. **Lệnh qua cổng Serial (115200 bps):**
   - Phím `1`, `2`, `3`: Đổi công suất tương ứng 25%, 50%, 90%.
   - Phím `i`: Quét nhanh bus I2C tìm thiết bị.
   - Phím `s`: In bản tóm tắt trạng thái hiện thời của vi điều khiển.
