# Sửa các lỗi đã phát hiện — Nhà Thông Minh

Đã đối chiếu với mã nguồn thật tại `/workspace/hassnotifyfullscreen` (Gradle bản Groovy DSL). Các file **đã sửa sẵn** nằm trong thư mục [fixes/](../fixes/) của repo này — chỉ cần copy đè vào project (cách áp dụng ở [fixes/README.md](../fixes/README.md)).

## 1. Thêm cấu hình ký bản release (lỗi 1 + 3)

Hiện tại `buildTypes.release` chưa có `signingConfig` nên APK release build ra sẽ không có chữ ký, không cài được.

**Bước 1 — tạo keystore** (chạy 1 lần trên máy của bạn; giữ file + mật khẩu cẩn thận, **không commit lên Git**):

```bash
keytool -genkeypair -v -keystore nhathongminh-release.jks -alias nhathongminh -keyalg RSA -keysize 2048 -validity 10000
```

**Bước 2 — tạo file `keystore.properties`** ở thư mục gốc project (thêm vào `.gitignore`, **không commit**):

```properties
storeFile=../nhathongminh-release.jks
storePassword=<mật khẩu của bạn>
keyAlias=nhathongminh
keyPassword=<mật khẩu của bạn>
```

(đường dẫn `storeFile` tính tương đối từ thư mục `app/` — chỉnh lại nếu bạn để keystore chỗ khác.)

**Bước 3 — `app/build.gradle`**: đọc `keystore.properties` và gắn signing config (đã có sẵn trong [fixes/app/build.gradle](../fixes/app/build.gradle)):

```groovy
def keystorePropertiesFile = rootProject.file("keystore.properties")
def keystoreProperties = new Properties()
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
}

android {
    signingConfigs {
        release {
            if (keystorePropertiesFile.exists()) {
                storeFile file(keystoreProperties['storeFile'])
                storePassword keystoreProperties['storePassword']
                keyAlias keystoreProperties['keyAlias']
                keyPassword keystoreProperties['keyPassword']
            }
        }
    }
    buildTypes {
        release {
            signingConfig signingConfigs.release
            // ...
        }
    }
}
```

Ghi chú:
- Bản debug có `applicationIdSuffix = ".debug"` nên bản release (không hậu tố) sẽ cài **song song** như một app riêng — không cần gỡ bản debug, nhưng dữ liệu/cài đặt cũng không tự chuyển sang.
- Bản release tự động có `debuggable=false`.

## 2. Giới hạn cleartext chỉ cho đúng IP Home Assistant (lỗi 2)

Tạo file mới `app/src/main/res/xml/network_security_config.xml` (đã có sẵn trong thư mục fixes):

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config cleartextTrafficPermitted="false" />
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="false">192.168.1.10</domain>
    </domain-config>
</network-security-config>
```

Trong `AndroidManifest.xml`, thẻ `<application>` đổi thành:

```xml
<application
    android:networkSecurityConfig="@xml/network_security_config"
    android:usesCleartextTraffic="false"
    ... >
```

Nếu sau này đổi địa chỉ Home Assistant, nhớ sửa IP trong file XML này — app sẽ chặn cleartext tới địa chỉ khác.

## 3. Nâng compileSdk / targetSdk 34 → 35 (lỗi 4)

Trong `app/build.gradle`:

```groovy
android {
    compileSdk = 35
    defaultConfig {
        targetSdk = 35
        // ...
    }
}
```

Thêm vào `gradle.properties` để tắt cảnh báo "AGP 8.5.2 chưa chính thức hỗ trợ compileSdk 35" (build vẫn chạy):

```properties
android.suppressUnsupportedCompileSdk=35
```

Điều kiện: máy build phải có **Android SDK Platform 35** (cài qua SDK Manager hoặc `sdkmanager "platforms;android-35"`).

Kiểm thử lại sau khi nâng: kết nối WebSocket tới HA, thông báo, cảnh báo toàn màn hình (`USE_FULL_SCREEN_INTENT` bị Android 14+ siết — app sideload cấp quyền này trong phần cài đặt hệ thống), tự chạy sau reboot.

## 4. PreviewActivity — không cần sửa

`androidx.compose.ui:ui-tooling` đã ở đúng `debugImplementation` trong project. `PreviewActivity` chỉ có mặt trong bản debug (đúng thiết kế Compose tooling) và tự biến mất ở bản release. Mục này từng bị báo nhầm ở bước kiểm tra chỉ dựa trên APK — đã đính chính trong [ERROR-CHECK.md](ERROR-CHECK.md).

## Tuỳ chọn nâng cao (không bắt buộc): bật minify cho bản release

Đổi trong `buildTypes.release`: `minifyEnabled = true` và thêm `shrinkResources = true`. Các thư viện đang dùng (Compose, OkHttp, coroutines, security-crypto) đều có sẵn consumer ProGuard rules nên thường không cần thêm rule thủ công — nhưng **phải test kỹ** bản release sau khi bật (đặc biệt là WebSocket và các luồng đọc JSON) trước khi dùng chính thức.
