# Kiểm tra lỗi & vấn đề — Nhà Thông Minh v1.0.0 (debug)

- **Ngày kiểm tra:** 2026-10-21
- **Phương pháp:** phân tích tĩnh bản APK — kiểm tra cấu hình manifest (thành phần exported), quét mã DEX tìm token/mật khẩu hardcode, kiểm chứng SSL, mật mã yếu và endpoint không mã hoá.
- **Đối tượng:** `NhaThongMinh-v1.0.0-debug_0_q49u.apk` — xem thêm [ANALYSIS.md](ANALYSIS.md).

## Vấn đề phát hiện

### 1. Bản debug chưa sẵn sàng phát hành — mức cao (đúng tính chất bản debug)
- `debuggable=true`: thiết bị khác có thể gắn debugger/trích xuất dữ liệu khi cắm USB.
- Ký bằng **Android Debug key** công khai: bất kỳ ai cũng tạo được bản giả mạo trùng package để ghi đè/lừa cài đặt.
- => Chỉ dùng trong mạng nhà. Khi phát hành: build release, tắt debuggable, ký bằng keystore riêng và giữ keystore cẩn thận.

### 2. Cho phép traffic không mã hoá (`usesCleartextTraffic=true`) — mức trung bình
- App nói chuyện HTTP/WS thường tới Home Assistant (`192.168.1.10:8123`).
- Trong LAN gia đình: rủi ro thấp, chấp nhận được.
- Nếu thiết bị rời khỏi mạng nhà hoặc mạng nhà có thiết bị lạ, toàn bộ lệnh điều khiển nhà đi dạng rõ. Khuyến nghị: giới hạn cleartext chỉ cho đúng IP của HA bằng Network Security Config thay vì bật toàn cục.

### 3. Lọt thành phần tooling của Compose vào bản build, lại đang exported — mức trung bình
- `androidx.compose.ui.tooling.PreviewActivity` xuất hiện trong manifest với `exported=true` — đây là activity phục vụ xem trước giao diện khi phát triển, không nên có trong app chạy thật; app khác trên cùng máy có thể mở nó.
- Nguyên nhân thường gặp: khai báo thư viện `androidx.compose.ui:ui-tooling` ở `implementation` thay vì `debugImplementation` trong `build.gradle`.

### 4. targetSdk 34 đã cũ so với hiện tại — mức thấp
- Android hiện tại đã ở API 36. App dùng nội bộ thì vẫn chạy bình thường, nhưng nếu định phát hành qua Play Store sẽ bị yêu cầu nâng targetSdk (Play yêu cầu tối thiểu API 35 với app mới từ 8/2025). Nên nâng dần lên 35/36 và kiểm thử lại quyền thông báo + full-screen intent (các bản Android mới siết hai quyền này).

## Đã kiểm tra — không phát hiện lỗi

- **Không hardcode token/mật khẩu Home Assistant** trong APK (không thấy JWT, Bearer token hay mật khẩu dạng rõ trong mã).
- **Không có bằng chứng tắt kiểm chứng SSL** — các class `DisabledHostnameVerifier` tìm thấy là thành phần nội bộ của thư viện OkHttp/Conscrypt, không phải mã của app.
- **Không dùng mật mã yếu** (không thấy DES, AES/ECB, MD5 cho mục đích bảo mật).
- **Endpoint cleartext duy nhất** là IP LAN của Home Assistant — không gọi HTTP ra Internet.
- `ArrivalAlertActivity` và `CrashActivity` đều `exported=false` — app ngoài không thể kích hoạt cảnh báo giả hay mở màn hình crash.
- `BootReceiver` để `exported=true` là bắt buộc để nhận sự kiện `BOOT_COMPLETED`; sự kiện này do hệ thống bảo vệ nên không bị app khác giả mạo.
- `allowBackup=false`, bộ quyền tối thiểu — cấu hình tốt cho app nội bộ.

## Kết luận

Không phát hiện lỗi bảo mật nghiêm trọng trong mã (không lộ token, không tắt SSL, không mật mã yếu). Các vấn đề chính đều thuộc nhóm "bản debug dùng ra ngoài phạm vi phát triển": debuggable + debug key, cleartext toàn cục, và thành phần PreviewActivity lọt vào build. Khắc phục 4 mục ở trên là đủ điều kiện để đóng gói bản release dùng lâu dài.
