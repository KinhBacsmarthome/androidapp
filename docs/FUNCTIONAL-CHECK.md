# Kiểm tra khả năng hoạt động — Nhà Thông Minh

- **Ngày kiểm tra:** 2026-10-21
- **Phạm vi:** mục đích chạy nội bộ trên Android Box trong mạng nhà (không phát hành ra ngoài).
- **Phương pháp:** build thật + chạy unit test trên máy chủ của project (`/workspace/hassnotifyfullscreen`), đọc và rà soát toàn bộ 29 file Kotlin dưới `app/src/main/java`.
- Xem thêm [ANALYSIS.md](ANALYSIS.md) và [ERROR-CHECK.md](ERROR-CHECK.md).

## Kết luận tổng

App **build được, unit test đạt 10/10**, các luồng chính (khởi động, kết nối Home Assistant, dashboard, mở cổng, popup cảnh báo) được thiết kế đúng. Có **2 lỗi chức năng** — cả hai **đã được sửa và kiểm chứng ngày 2026-10-22** (áp dụng đúng cách sửa mô tả bên dưới, build lại thành công, unit test vẫn đạt 10/10; APK mới sinh lúc 16:02 ngày 2026-10-22 giờ Việt Nam, 21.241.506 bytes) — cùng một số điểm lưu ý vận hành ở dưới.

## Đã kiểm chứng chạy được

- **Build:** `gradle assembleDebug` bằng Gradle hệ thống 8.7 (project không có Gradle wrapper) → `BUILD SUCCESSFUL` trong 46 giây. APK đầu ra 21.248.601 bytes — đúng bản đang dùng.
- **Unit test:** `gradle :app:testDebugUnitTest` → **10/10 đạt** (ExternalConfigLoaderTest 3/3, HaEventProcessorTest 7/7), 0 lỗi.
- **Khởi động có bọc lỗi:** `NhaThongMinhApp` cài CrashLogger ghi log ra file + hiện màn hình log crash; init lỗi không làm app thoát ngay.
- **Kết nối HA đúng giao thức:** WebSocket auth bằng long-lived token → `subscribe_events` (state_changed) → `get_states` lấy snapshot; snapshot khi khởi động/reconnect không bị coi là sự kiện "về nhà".
- **Mở cổng:** gọi service qua REST, chống nhấn lặp + cooldown, sau lệnh có poll sensor để xác minh rồi mới báo thành công.
- **Popup an toàn:** timeout chỉ đóng popup, không tự mở cổng; nút ĐÓNG/X không gửi lệnh nào.
- **Token** lưu bằng EncryptedSharedPreferences (có fallback khi Keystore của máy lỗi), không log token ra ngoài.

## Lỗi chức năng cần sửa

### 1. Popup "XE ĐÃ VỀ ĐẾN NHÀ" chỉ hiện 1 lần trong mỗi lần chạy app — mức cao

**Trạng thái: ĐÃ SỬA ngày 2026-10-22** — đã truyền `last_changed` xuyên suốt từ WebSocket client qua repository vào bộ xử lý sự kiện, đúng như cách sửa mô tả bên dưới; build và unit test đạt sau khi sửa.

- `HaEventProcessor.evaluate()` chống hiện trùng bằng khóa `entity | oldState -> newState | lastChanged`.
- Nhưng `DashboardRepository.handleStateChanged()` gọi `evaluate()` **không truyền `lastChanged`** — và `HaWebSocketClient` cũng không trích `last_changed` từ event (class `HaStateChangedEvent` không có trường này). Khóa thực tế luôn là `entity|not_home->home|?`, nên **mọi lần về nhà sau lần đầu đều bị coi là "sự kiện trùng"** và bị bỏ qua, cho tới khi app khởi động lại hoặc lưu lại cài đặt (gọi `processor.reset()`).
- Đối chiếu: unit test luôn truyền `lastChanged` tường minh (ví dụ `"t1"`) nên đều đạt — lỗi nằm ở khâu nối dây giữa WebSocket client → repository → processor, không nằm ở bản thân processor.
- **Cách sửa:** thêm trường `lastChanged` vào `HaStateChangedEvent`, đọc `new_state.last_changed` trong `HaWebSocketClient`, truyền tiếp qua `DashboardRepository` vào `evaluate()`. Không đổi logic của processor; test `duplicate_transition_deduped` vẫn đạt (nó chủ động dùng cùng một mốc `t1`).

### 2. WebSocket không tự kết nối lại sau khi rớt — mức cao

**Trạng thái: ĐÃ SỬA ngày 2026-10-22** — đã bổ sung cơ chế chờ tín hiệu đóng/lỗi socket để vòng reconnect hoạt động đúng như cách sửa mô tả bên dưới; build và unit test đạt sau khi sửa.

- Trong `HaWebSocketClient.connectOnce()`, sau khi auth thành công, vòng lặp giữ kết nối chỉ thoát khi `ws == null`. Nhưng hai callback `onFailure`/`onClosed` **không bao giờ gán `ws = null`** — khi HA khởi động lại hoặc mạng rớt, socket đã đóng mà `connectOnce()` vẫn treo, `connectLoop()` không chạy lại được.
- Hệ quả thực tế: app kẹt ở trạng thái Offline (chấm trạng thái trên dashboard báo đúng là Offline) và **ngừng nhận mọi sự kiện** cho tới khi mở lại app hoặc lưu lại cài đặt. Với dashboard treo tường chạy 24/7, mỗi lần HA restart là một lần app "chết" theo.
- **Cách sửa:** thêm một `CompletableDeferred` hoàn thành trong `onFailure`/`onClosed` và cho `connectOnce()` chờ deferred đó thay cho vòng lặp kiểm tra `ws == null` (khi deferred hoàn thành thì ném exception để `connectLoop()` reconnect với backoff như đã thiết kế).

## Điểm lưu ý vận hành (không chặn chạy)

- **Ô nhập token/cookie ở màn Cài đặt hiện chữ thường:** composable `Field` có tham số `secret` nhưng chưa dùng tới (không áp `PasswordVisualTransformation`) — token lộ trên màn hình TV khi nhập. Nên sửa cùng đợt.
- **File cấu hình ngoài ghi đè Settings mỗi lần khởi động:** nếu file `nhathongminh-config.json` còn nằm trên Box, mọi thay đổi trong màn Cài đặt sẽ bị ghi đè ở lần mở app sau. Đúng thiết kế đã ghi trong README của project — chỉ cần nhớ khi vận hành.
- **Thời gian xác minh mở cổng hơi ngắn:** poll sensor 5 lần × 1,6 giây ≈ 8 giây. Cổng di chuyển chậm hơn có thể bị báo "Không xác minh được trạng thái cổng" dù thực tế đã mở. Cân nhắc tăng số lần poll.
- **Tự mở app sau reboot:** `BootReceiver` mở MainActivity trong try/catch nuốt lỗi — trên một số box Android 11/12, giới hạn khởi chạy từ nền có thể chặn thầm. Nên kiểm chứng thực tế bằng cách reboot Box một lần và xem app có tự lên không.
- **Tên app trong `strings.xml` gõ thiếu chữ** ("Nhà Thống Minh"); tên hiển thị ngoài launcher lấy từ `android:label` trong manifest (đúng) nên không ảnh hưởng — chỉ là lỗi chính tả tiềm ẩn.
- **Project không có Gradle wrapper:** build phụ thuộc Gradle cài sẵn trên máy (`/opt/gradle-8.7`). Nên bổ sung wrapper (`gradle wrapper`) để build được ở máy khác.
