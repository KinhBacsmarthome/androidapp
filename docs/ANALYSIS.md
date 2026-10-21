# Báo cáo phân tích APK — Nhà Thông Minh v1.0.0 (debug)

- **File phân tích:** `NhaThongMinh-v1.0.0-debug_0_q49u.apk`
- **Ngày phân tích:** 2026-10-21
- **Phương pháp:** phân tích tĩnh (giải nén APK, đọc AndroidManifest, quét chuỗi trong các file DEX) — không chạy app, không giải mã ngược mã nguồn.

## 1. Nhận dạng

| Mục | Giá trị |
|---|---|
| Tên app | Nhà Thông Minh |
| Package | `com.nhathongminh.dashboard.debug` |
| Phiên bản | 1.0.0 (versionCode 1) |
| Main Activity | `com.nhathongminh.dashboard.MainActivity` |
| Kích thước APK | 21,248,601 byte (~21 MB) |
| Tổng dung lượng giải nén | ~65 MB, 943 file |
| Số file DEX | 10 (multidex) |
| Thư viện native (.so) | Không có — app thuần Kotlin/JVM |

Hậu tố `.debug` trong package và tên file cho thấy đây là biến thể build debug.

## 2. Nền tảng & công nghệ

- **minSdk 26** (Android 8.0), **targetSdk 34** (Android 14).
- Build bằng **Android Gradle Plugin 8.5.2**.
- **Kotlin** + **kotlinx-coroutines 1.8.1**, AndroidX.
- Giao diện **Jetpack Compose** (có `androidx.compose.ui.tooling.PreviewActivity`, gói `ui/theme`).
- Kiến trúc theo gói (clean architecture):
  - `com.nhathongminh.dashboard.data` — tầng dữ liệu
  - `com.nhathongminh.dashboard.domain` — tầng nghiệp vụ
  - `com.nhathongminh.dashboard.homeassistant` — tích hợp Home Assistant
  - `com.nhathongminh.dashboard.dashboard` — màn hình dashboard
  - `com.nhathongminh.dashboard.alert` — cảnh báo
  - `com.nhathongminh.dashboard.media` — đa phương tiện
  - `com.nhathongminh.dashboard.settings` — cài đặt
  - `com.nhathongminh.dashboard.ui.theme` — theme Compose

## 3. Chữ ký số

- Ký bằng **Android Debug key**: chứng thư tự ký `CN=Android Debug, O=Android, C=US`, serial `0x1`.
- Chỉ dùng chữ ký APK Signature Scheme v2 (không có chữ ký v1/JAR).
- Kết luận: bản build phục vụ phát triển/kiểm thử nội bộ, đúng chuẩn debug key mặc định của Android Studio.

## 4. Quyền (permissions) — 7 quyền

| Quyền | Mục đích |
|---|---|
| `INTERNET` | Kết nối tới Home Assistant |
| `ACCESS_NETWORK_STATE` | Kiểm tra trạng thái mạng |
| `POST_NOTIFICATIONS` | Hiển thị thông báo |
| `RECEIVE_BOOT_COMPLETED` | Tự chạy lại sau khi thiết bị khởi động lại |
| `USE_FULL_SCREEN_INTENT` | Hiện cảnh báo toàn màn hình (kiểu báo thức) |
| `WAKE_LOCK` | Đánh thức/bật màn hình khi có cảnh báo |
| `com.nhathongminh.dashboard.debug.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION` | Quyền nội bộ do AndroidX tự sinh |

Nhận xét: bộ quyền **tối thiểu và đúng mục đích** — không xin vị trí, camera, micro, danh bạ hay bộ nhớ.

## 5. Thành phần (components)

**Activities (5):**
- `com.nhathongminh.dashboard.MainActivity` — màn hình chính.
- `com.nhathongminh.dashboard.alert.ArrivalAlertActivity` — cảnh báo khi có người về nhà, hiện toàn màn hình.
- `com.nhathongminh.dashboard.CrashActivity` — màn hình xử lý khi app gặp lỗi (crash handler tự viết).
- `androidx.activity.ComponentActivity`, `androidx.compose.ui.tooling.PreviewActivity` — thành phần thư viện Compose.

**Receivers (2):**
- `com.nhathongminh.dashboard.BootReceiver` — nhận sự kiện khởi động máy.
- `androidx.profileinstaller.ProfileInstallReceiver` — thành phần AndroidX.

**Services:** không có.

**Providers (1):** `androidx.startup.InitializationProvider` (AndroidX Startup).

Cấu hình application: `debuggable=true`, `allowBackup=false`, `usesCleartextTraffic=true`.

## 6. Mạng & tích hợp Home Assistant

- Dùng **OkHttp** + **WebSocket** — đúng mô hình Home Assistant API: WebSocket để nhận trạng thái thiết bị theo thời gian thực, REST cho lệnh điều khiển.
- Tìm thấy endpoint mặc định trong mã: **`192.168.1.10:8123`** — địa chỉ Home Assistant trong mạng LAN (8123 là cổng mặc định của HA).
- `usesCleartextTraffic=true`: chấp nhận HTTP/WS không mã hoá — bình thường với kết nối LAN tới HA, nhưng không an toàn nếu kết nối ra ngoài Internet.

## 7. Nhận xét bảo mật

1. **Không phát hành bản này**: `debuggable=true` + debug key công khai nghĩa là bất kỳ ai cũng ký được bản giả mạo trùng package. Chỉ dùng trong nhà.
2. **Traffic LAN không mã hoá**: chấp nhận được cho tablet treo tường trong mạng nhà; nếu sau này truy cập HA từ xa, nên chuyển sang HTTPS/WSS qua reverse proxy hoặc VPN.
3. **Điểm tốt**: `allowBackup=false`, bộ quyền tối thiểu, không nhúng thư viện quảng cáo/theo dõi nào được nhận diện.

## 8. Kết luận

"Nhà Thông Minh" là app dashboard Home Assistant viết bằng Kotlin + Jetpack Compose, thiết kế cho thiết bị Android cố định trong nhà (kiểu tablet treo tường): tự khởi động cùng máy, giữ khả năng đánh thức màn hình và phát cảnh báo toàn màn hình khi có người về. Bản APK được phân tích là **bản debug v1.0.0** — phù hợp để chạy thử trong mạng LAN, chưa phù hợp để phát hành.
