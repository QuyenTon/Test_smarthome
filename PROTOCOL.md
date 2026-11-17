# Giao thức Smart Room (backend ⇄ ESP32)

Bản v1. Mọi bên (firmware, backend, dashboard) làm theo đúng file này.

## Kết nối

- ESP32 là **TCP server**, cổng **3333**, IP tĩnh **192.168.50.2** (mask 255.255.255.0).
- Laptop đặt IP tĩnh **192.168.50.1/24**, nối thẳng cáp RJ45.
- Một lúc chỉ phục vụ **1 client** (backend). Client thứ hai phải chờ client đầu ngắt.
- Mỗi tin nhắn là **một dòng ASCII kết thúc bằng `\n`** (chấp nhận cả `\r\n`). Tối đa 63 ký tự/dòng.
- Tên lệnh không phân biệt hoa thường. Phần sau dấu `:` giữ nguyên.

## Backend → ESP32 (lệnh)

| Lệnh | Tác dụng | Phản hồi |
|---|---|---|
| `L1_ON` / `L1_OFF` | Bật / tắt đèn 1 | `OK` + `STATE:...` |
| `L2_ON` / `L2_OFF` | Bật / tắt đèn 2 | `OK` + `STATE:...` |
| `FAN_ON` / `FAN_OFF` | Bật / tắt quạt | `OK` + `STATE:...` |
| `MSG:<nội dung>` | Hiện dòng chữ lên LCD (≤ 32 ký tự, ASCII) | `OK` |
| `SET_GAS_TH:<0..4095>` | Đặt ngưỡng báo khí gas (giá trị ADC thô) | `OK` + `STATE:...` |
| `GET_STATUS` | Hỏi trạng thái hiện tại | `STATE:...` |
| `PING` | Kiểm tra kết nối | `PONG` |

Lệnh sai: `ERR:UNKNOWN_CMD`, `ERR:EMPTY`, `ERR:TOO_LONG`, `ERR:BAD_VALUE`.

## ESP32 → Backend

| Dòng | Khi nào gửi |
|---|---|
| `DATA:T=28.5,H=65.0,GAS=312` | Định kỳ mỗi **2 giây** (T: °C, H: %, GAS: ADC thô 0–4095) |
| `STATE:L1=1,L2=0,FAN=0,TH=2000,ALM=0` | Ngay khi client kết nối, khi trạng thái đổi, khi có `GET_STATUS` |
| `ALARM:GAS=1,VAL=2150` | Khi khí gas **vượt** ngưỡng (`GAS=1`) |
| `ALARM:GAS=0,VAL=1700` | Khi khí gas **xuống lại** dưới ngưỡng trừ 10 % (`GAS=0`) |

## Quy tắc báo động

- ESP32 tự so sánh với ngưỡng, **không phụ thuộc dashboard**. Khi chuyển trạng thái, ESP32 phải
  - gửi dòng `ALARM`,
  - hiện cảnh báo lên LCD,
  - (tuỳ chọn) bật buzzer.
- Ngưỡng mặc định 2000 là **giá trị tạm**, cần chỉnh theo thực tế sau khi đo MQ-2 trên board.
- Dưới ngưỡng có độ trễ 10 % để tránh nhấp nháy.

## Ví dụ phiên làm việc

```
ESP32 → STATE:L1=0,L2=0,FAN=0,TH=2000,ALM=0
ESP32 → DATA:T=28.4,H=64.9,GAS=640
PC    → L1_ON
ESP32 → OK
ESP32 → STATE:L1=1,L2=0,FAN=0,TH=2000,ALM=0
PC    → MSG:Xin chao
ESP32 → OK
ESP32 → DATA:T=28.5,H=65.0,GAS=2105
ESP32 → ALARM:GAS=1,VAL=2105
```

Thử bằng tay (Windows có thể dùng PuTTY, chọn *Raw*, IP 192.168.50.2, cổng 3333): gõ `PING` rồi Enter, phải nhận `PONG`.
