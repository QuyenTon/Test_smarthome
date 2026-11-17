# Smart Room qua Ethernet (WT32-ETH01)

Gồm 3 phần, làm độc lập được nhờ giao thức chung trong **PROTOCOL.md**:

```
smart-room/
├─ PROTOCOL.md            giao thức lệnh (gửi cho Quỳnh làm theo)
├─ backend/               chạy trên laptop
│  ├─ server.py           cầu nối TCP (ESP32) <-> WebSocket + lưu SQLite
│  ├─ mock_esp32.py       ESP32 giả lập, để thử khi chưa có board
│  ├─ static/index.html   dashboard (1 file, không cần internet)
│  └─ requirements.txt
└─ firmware_eth/          project ESP-IDF cho WT32-ETH01
   ├─ main/               main.c, eth_net.c (Ethernet), tcp_server.c, proto.c (giao thức), app_stub.c (giả lập ngoại vi)
   └─ host_test/          thử bộ phân tích giao thức ngay trên máy tính
```

## A. Chạy thử dashboard, không cần board (5 phút)

Cần Python 3.10 trở lên.

```
cd backend
pip install -r requirements.txt
python mock_esp32.py                              # cửa sổ 1: ESP32 giả lập, cổng 3333
python server.py --esp 127.0.0.1:3333             # cửa sổ 2: backend
```

Mở http://localhost:8000. Phải thấy số đo chạy, đồ thị cuộn, bật/tắt đèn quạt được.
Khí gas giả lập dao động chu kỳ 90 giây nên sẽ tự báo động. Để ép báo động, gõ `gas 2500` vào cửa sổ 1 (`gas auto` để thả).

## B. Nạp firmware Ethernet cho WT32-ETH01

1. Cài **ESP-IDF 5.4.x** (code viết cho dòng 5.x, chưa thử với 6.0; nếu Quỳnh đã dùng bản khác thì dùng theo bản của Quỳnh).
   Nếu dùng VS Code (như Quỳnh): cài extension "ESP-IDF" của Espressif, bấm F1, chạy
   `ESP-IDF: Configure ESP-IDF Extension`, chọn bản 5.4.x trong wizard. Mở thư mục `firmware_eth`, chạy
   `ESP-IDF: Set Espressif Device Target` (esp32), `ESP-IDF: Select Port to Use` (cổng COM của CP2102),
   rồi build / nạp / monitor bằng nút ở thanh trạng thái hoặc phím Ctrl+E D.
   Không dùng VS Code: cài Espressif Installation Manager (`winget install Espressif.EIM`) rồi `eim install -i v5.4.2`;
   mở terminal ESP-IDF và kiểm tra `idf.py --version`.
2. Nối CP2102 với module:

   | CP2102 | WT32-ETH01 |
   |---|---|
   | TXD | RX0 |
   | RXD | TX0 |
   | GND | GND |
   | 5V | 5V |

3. Nối **IO0 xuống GND**, bấm reset (chạm chân EN xuống GND một lần, hoặc cấp nguồn lại) để vào chế độ nạp.
4. Trong thư mục `firmware_eth`:
   ```
   idf.py set-target esp32
   idf.py -p COM5 flash monitor        (đổi COM5 theo máy)
   ```
5. Nạp xong: **bỏ dây IO0 khỏi GND**, reset lại. (IO0 còn nhận xung 50 MHz của LAN8720 nên không được để nối GND khi chạy.)
6. Log đúng sẽ có: `Day mang da cam (link up)`, `IP tinh: 192.168.50.2`, `TCP server dang cho o cong 3333`.

Nếu module tự reset liên tục khi cắm cáp mạng, nguồn 5V của CP2102 không đủ dòng: dùng nguồn 5V ngoài (1A) cho module.

## C. Nối laptop với module qua RJ45

1. Đặt IP tĩnh cho cổng LAN laptop: `192.168.50.1`, mask `255.255.255.0`, **bỏ trống gateway**.
   Windows (CMD quyền Admin, tên "Ethernet" đổi theo máy):
   ```
   netsh interface ip set address name="Ethernet" static 192.168.50.1 255.255.255.0
   ```
2. `ping 192.168.50.2` phải có phản hồi.
3. Thử giao thức bằng tay: PuTTY, chọn *Raw*, IP 192.168.50.2, cổng 3333. Gõ `PING` + Enter, nhận `PONG`. Gõ `L1_ON`, nhận `OK` và `STATE:...`.
4. Chạy dashboard thật:
   ```
   python server.py --esp 192.168.50.2:3333
   ```
   Mở http://localhost:8000. Module trơn chưa có cảm biến nên số đo là giá trị giả lập của `app_stub.c`.

## D. Ghép với board và firmware 

Lớp mạng chỉ gọi 5 hàm trong `main/app_hooks.h`. Khi có board và code ngoại vi đã sửa:

1. Bỏ `app_stub.c` khỏi `main/CMakeLists.txt`.
2. Viết 5 hàm đó gọi sang driver thật (đèn, quạt, LCD, DHT11, MQ-2 đọc ADC).
3. Trong `app_main` của Quỳnh, gọi `eth_net_start()` rồi `tcp_server_start()`.
4. **Tránh các chân của Ethernet:** GPIO0, 16, 18, 19, 21, 22, 23, 25, 26, 27 (pin map theo sơ đồ board: đèn 2 và 15, quạt 14, buzzer 17, DHT 4, I2C 32/33, MQ-2 39/36).
5. Đừng dùng UART0 cho lệnh nữa, mọi lệnh đi qua TCP.
6. Chỉnh `PROTO_GAS_TH_DEFAULT` trong `proto.h` (hiện 2000 là giá trị tạm) sau khi đo MQ-2 trên board.

## Đã kiểm tra gì, chưa kiểm tra gì

- Đã chạy thử: bộ phân tích giao thức `proto.c` (host_test, tất cả đạt), backend + mock ESP32 + WebSocket đầu cuối, logic dashboard trong jsdom (không có lỗi JS).
- **Chưa biên dịch bằng ESP-IDF và chưa chạy trên phần cứng.** `tcp_server.c` và `app_stub.c` mới kiểm tra cú pháp bằng shim; `eth_net.c` chưa kiểm tra. Nếu `idf.py build` báo lỗi tên trường (nhất là trong `eth_net.c`, tên trường cấu hình EMAC đổi giữa các bản IDF 5.x), gửi nguyên log lỗi để sửa.
- Giao diện dashboard chưa xem trên trình duyệt thật.
