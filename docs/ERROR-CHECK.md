# Kiểm tra lỗi & vấn đề — Nhà Thông Minh v1.0.0 (debug)

- **Ngày kiểm tra:** 2026-10-21 (đã đối chiếu lại với mã nguồn tại `/workspace/hassnotifyfullscreen` cùng ngày)
- **Phương pháp:** phân tích tĩnh bản APK + đọc trực tiếp project nguồn — kiểm tra cấu hình manifest, build Gradle, quét mã DEX tìm token/mật khẩu hardcode, kiểm chứng SSL, mật mã yếu và endpoint không mã hoá.
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

### 3. Bản release chưa có cấu hình ký (signingConfig) — mức trung bình
- Trong `app/build.gradle`, `buildTypes.release` chỉ có `minifyEnabled` và `proguardFiles` — **không có `signingConfig`**.
- Hệ quả: `./gradlew assembleRelease` sẽ cho ra APK **chưa ký**, không cài được lên thiết bị. Đây là việc cần bổ sung trước khi nghĩ tới phát hành (cách làm ở [FIXES.md](FIXES.md)).

### 4. targetSdk 34 đã cũ so với hiện tại — mức thấp
- Android hiện tại đã ở API 36. App dùng nội bộ thì vẫn chạy bình thường, nhưng nếu định phát hành qua Play Store sẽ bị yêu cầu nâng targetSdk (Play yêu cầu tối thiểu API 35 với app mới từ 8/2025). Nên nâng dần lên 35 và kiểm thử lại quyền thông báo + full-screen intent (các bản Android mới siết hai quyền này).

## Đính chính so với bản kiểm tra đầu (chỉ dựa trên APK)

**PreviewActivity không phải lỗi.** Bản kiểm tra đầu (trước khi có mã nguồn) từng liệt kê `androidx.compose.ui.tooling.PreviewActivity` (exported=true trong bản debug) là lỗi "lọt thành phần dev". Đối chiếu mã nguồn cho thấy thư viện `ui-tooling` **đã được khai báo đúng ở `debugImplementation`** — activity này chỉ xuất hiện trong bản debug theo đúng thiết kế của Compose tooling, và **tự biến mất ở bản release**. Không cần sửa gì. (Nó vẫn xuất hiện trong mọi bản debug bạn build — đó là bình thường.)

## Đã kiểm tra — không phát hiện lỗi

- **Không hardcode token/mật khẩu Home Assistant** trong APK (không thấy JWT, Bearer token hay mật khẩu dạng rõ trong mã). Token HA được lưu bằng `androidx.security:security-crypto` (mã hoá) — cách làm đúng.
- **Không có bằng chứng tắt kiểm chứng SSL** — các class `DisabledHostnameVerifier` tìm thấy là thành phần nội bộ của thư viện OkHttp/Conscrypt, không phải mã của app.
- **Không dùng mật mã yếu** (không thấy DES, AES/ECB, MD5 cho mục đích bảo mật).
- **Endpoint cleartext duy nhất** là IP LAN của Home Assistant — không gọi HTTP ra Internet.
- `ArrivalAlertActivity` và `CrashActivity` đều `exported=false` — app ngoài không thể kích hoạt cảnh báo giả hay mở màn hình crash.
- `BootReceiver` để `exported=true` là bắt buộc để nhận sự kiện `BOOT_COMPLETED`; sự kiện này do hệ thống bảo vệ nên không bị app khác giả mạo.
- `allowBackup=false`, bộ quyền tối thiểu — cấu hình tốt cho app nội bộ.

## Kết luận

Không phát hiện lỗi bảo mật nghiêm trọng trong mã (không lộ token, không tắt SSL, không mật mã yếu). Các việc cần làm để đóng gói bản release: thêm signingConfig + keystore riêng, giới hạn cleartext đúng IP Home Assistant, nâng targetSdk lên 35 — hướng dẫn chi tiết ở [FIXES.md](FIXES.md), file đã sửa sẵn ở thư mục [fixes/](../fixes/).
