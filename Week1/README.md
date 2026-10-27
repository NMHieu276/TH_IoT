# Bài 1 — Mô hình hóa hệ thống nhà kính IoT

| Thông tin | Nội dung |
|---|---|
| Sinh viên thực hiện | Nguyễn Minh Hiếu |
| MSSV | 23110168 |
| Hình thức | Làm cá nhân |

## 1. Thông số áp dụng cho bài làm

Em giữ nguyên khung của mẫu và chỉ thay các giá trị cho phù hợp với kit của mình:

| Thành phần | Giá trị gợi ý trong mẫu | Giá trị em sử dụng |
|---|---|---|
| Vi điều khiển / tên node | ESP32, node01 | ESP32, `node23110168` |
| Cảm biến | DHT11 | DHT11 |
| Thiết bị đầu ra để thử | LED trên breadboard | LED đỏ và LED xanh có sẵn trên bo Farm House |
| Máy chạy broker + dịch vụ | PC của nhóm | Laptop/PC cá nhân |
| Topic gửi số đo | nhom01/node01/telemetry | `23110168/node23110168/telemetry` |
| Topic nhận lệnh | nhom01/node01/cmd | `23110168/node23110168/cmd` |
| Topic phản hồi | nhom01/node01/ack | `23110168/node23110168/ack` |

## 2. Danh sách sơ đồ

| STT | Loại sơ đồ | Ảnh | Mã nguồn PlantUML |
|---|---|---|---|
| 1 | Use Case – chức năng hệ thống | [01_chuc_nang.png](./01_chuc_nang.png) | [01-chuc-nang.puml](./01-chuc-nang.puml) |
| 2 | Sequence – luồng gửi số đo | [02_luong_so_do.png](./02_luong_so_do.png) | [02-luong-so-do.puml](./02-luong-so-do.puml) |
| 3 | Sequence – luồng gửi lệnh | [03_luong_lenh.png](./03_luong_lenh.png) | [03-luong-lenh.puml](./03-luong-lenh.puml) |
| 4 | Activity – xử lý một lệnh | [04_hoat_dong.png](./04_hoat_dong.png) | [04-hoat-dong.puml](./04-hoat-dong.puml) |
| 5 | Deployment – triển khai trên kit | [05_trien_khai.png](./05_trien_khai.png) | [05-trien-khai.puml](./05-trien-khai.puml) |

## 3. Sơ đồ chân ESP32 trên bo nhà kính

Tra cứu từ schematic (trang 4 – khối ESP32):

| Thiết bị | Tín hiệu | GPIO | Linh kiện đi kèm |
|---|---|---|---|
| Cảm biến nhiệt độ – độ ẩm | `DATA_RH` | 23 | Trở kéo lên `R27` |
| Bus I²C (màn hình / cảm biến) | `SDA`, `SCL` | 21, 22 | Điện trở trên đường bus |
| LED đỏ | `LED_R` | 25 | Transistor `MMBT3904` – `Q18` |
| LED xanh | `LED_G` | 26 | Transistor `MMBT3904` – `Q12` |
| Động cơ 1 | `MOTOR1_INT1`, `MOTOR1_INT2` | 32, 33 | Driver `44950LITR` – `U2` |
| Động cơ 2 | `MOTOR2_INT1`, `MOTOR2_INT2` | 18, 19 | Driver `44950LITR` – `U3` |
| Dải LED 3 | `ARRAY_LED3` | 14 | Đệm `N531` – `U11` |
| Dải LED 4 | `ARRAY_LED4` | 13 | Đệm `N531` – `U12` |

## 4. Trả lời các câu hỏi

**a) Dữ liệu cảm biến được truyền qua những đâu?**
ESP32 đọc DHT11, đóng gói thành bản tin rồi publish lên MQTT Broker. Broker chuyển bản tin tới `ApplicationService`; dịch vụ kiểm tra hợp lệ, lưu vào `MeasurementStore`. Khi người dùng mở `WebUI`, giao diện hỏi dịch vụ để lấy bản ghi mới nhất.

**b) Một lệnh điều khiển đi theo đường nào?**
Người dùng bấm trên `WebUI` → `ApplicationService` kiểm tra quyền và tạo lệnh ở trạng thái pending → publish qua Broker → ESP32 nhận, kiểm tra và đặt mức GPIO cho LED. Sau đó ESP32 gửi `ack` theo chiều ngược lại để dịch vụ cập nhật kết quả và hiển thị cho người dùng.

**c) Broker có trực tiếp đọc cảm biến không?**
Không. Việc đọc cảm biến hoàn toàn do ESP32 đảm nhận; Broker chỉ nhận và phân phối bản tin MQTT giữa các bên đã đăng ký topic.

**d) Nhận được `ack` thì có thể kết luận điều gì?**
`ack` chỉ cho biết node đã nhận lệnh hợp lệ và đã ghi mức logic ra chân GPIO. Nó không phải bằng chứng vật lý rằng LED/tải thật sự hoạt động, vì chưa có phép đo độc lập nào ở phía tải.

**e) So với mẫu, em đã thay đổi những gì?**
- Thay mã nhóm `nhom01` bằng MSSV `23110168`, tên node `node01` thành `node23110168`.
- Cập nhật lại toàn bộ topic telemetry / cmd / ack theo tên mới.
- Xác định broker và dịch vụ ứng dụng chạy trên máy cá nhân thay vì PC của nhóm.

**f) Đường nối trong sơ đồ Use Case có biểu diễn luồng dữ liệu không?**
Không. Đường nối actor – use case chỉ thể hiện người dùng tham gia vào chức năng nào, không nói gì về thứ tự xử lý hay chiều truyền dữ liệu.

**g) Sơ đồ Activity khác Sequence ở điểm nào?**
Activity tập trung vào *quy trình*: các bước, điều kiện rẽ nhánh và lane nào chịu trách nhiệm. Sequence tập trung vào *tương tác*: đối tượng nào gửi bản tin/lời gọi cho đối tượng nào và theo thứ tự thời gian ra sao.

## 5. Những phần sẽ kiểm chứng ở các bài tiếp theo

- **Phía kit/PC:** cài và chạy Mosquitto, viết dịch vụ ứng dụng bằng Python, nạp firmware cho ESP32.
- **Phía nhà kính thật:** đấu nối phần cứng, cấp nguồn cho tải công suất và đo kiểm tín hiệu cảm biến/điều khiển trên mạch thực tế.
