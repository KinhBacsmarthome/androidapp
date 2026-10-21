# File đã sửa sẵn — cách áp dụng

Các file trong thư mục này là bản đã sửa của project tại `/workspace/hassnotifyfullscreen`, tương ứng với các lỗi trong [docs/ERROR-CHECK.md](../docs/ERROR-CHECK.md) và hướng dẫn ở [docs/FIXES.md](../docs/FIXES.md).

## Copy đè vào project

| File trong repo này | Đích trong project |
|---|---|
| `fixes/app/build.gradle` | `app/build.gradle` |
| `fixes/app/src/main/AndroidManifest.xml` | `app/src/main/AndroidManifest.xml` |
| `fixes/app/src/main/res/xml/network_security_config.xml` | `app/src/main/res/xml/network_security_config.xml` (tạo thư mục `xml` nếu chưa có) |
| `fixes/gradle.properties` | `gradle.properties` ở gốc project |

## Việc chỉ bạn làm được (trên máy build)

1. **Tạo keystore + `keystore.properties`** theo mục 1 của [docs/FIXES.md](../docs/FIXES.md). Thêm `keystore.properties` và file `.jks` vào `.gitignore`.
2. **Cài Android SDK Platform 35** nếu máy chưa có: `sdkmanager "platforms;android-35"`.
3. Build bản release:

```bash
./gradlew assembleRelease
```

APK ra tại `app/build/outputs/apk/release/NhaThongMinh-v1.0.0-release.apk`.

## Lưu ý

- Bản release có applicationId `com.nhathongminh.dashboard` (không có hậu tố `.debug`) nên cài **song song** được với bản debug đang chạy — dữ liệu/cài đặt không tự chuyển sang, cần đăng nhập/cấu hình lại Home Assistant trong bản release.
- Sau khi build release, kiểm thử: kết nối WebSocket tới HA, cảnh báo toàn màn hình khi người về, tự chạy sau reboot, mở link TikTok/YouTube trong mục media.
- Nếu đổi địa chỉ IP của Home Assistant, sửa IP trong `network_security_config.xml` (app sẽ chặn cleartext tới địa chỉ khác).
