# Nhà Thông Minh — Android Dashboard

App Android làm bảng điều khiển nhà thông minh, kết nối **Home Assistant** để hiển thị và điều khiển thiết bị trong nhà theo thời gian thực.

Repo này lưu hồ sơ phân tích từ bản APK `NhaThongMinh-v1.0.0-debug` (xem báo cáo đầy đủ tại [docs/ANALYSIS.md](docs/ANALYSIS.md)).

## Thông tin nhanh

| Mục | Giá trị |
|---|---|
| Tên app | Nhà Thông Minh |
| Package | `com.nhathongminh.dashboard.debug` |
| Phiên bản | 1.0.0 (versionCode 1) |
| Android tối thiểu | Android 8.0 (API 26) |
| Android đích | Android 14 (API 34) |
| Ngôn ngữ / UI | Kotlin + Jetpack Compose |
| Kết nối | Home Assistant qua WebSocket + REST (OkHttp) |
| Kích thước APK | ~21 MB (bản debug) |

## Tính năng nhận diện được từ bản build

- Dashboard Home Assistant cập nhật trạng thái thiết bị theo thời gian thực qua WebSocket.
- Cảnh báo toàn màn hình khi có người về nhà (`ArrivalAlertActivity`, quyền full-screen intent + wake lock — phù hợp cho tablet treo tường).
- Tự khởi động lại cùng thiết bị sau khi reboot (`BootReceiver`).
- Thông báo đẩy trên thiết bị (quyền `POST_NOTIFICATIONS`).

## Lưu ý quan trọng

Đây là **bản debug**: ký bằng Android Debug key, `debuggable=true`, cho phép traffic HTTP không mã hoá trong mạng LAN. Chỉ nên dùng trong mạng nhà — **không phát hành** bản này ra ngoài. Muốn phát hành cần build bản release ký bằng keystore riêng.

Endpoint Home Assistant mặc định tìm thấy trong bản build: `192.168.1.10:8123` (IP nội bộ LAN).
