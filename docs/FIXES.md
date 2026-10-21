# Hướng dẫn sửa 4 lỗi — Nhà Thông Minh

Các lỗi dưới đây nằm ở **cấu hình build**, chỉ sửa được trong project nguồn (Android Studio/Gradle) rồi build lại — không vá trực tiếp trên file APK được (xem lý do ở cuối).

## Lỗi 1 + 3: debuggable, debug key, và PreviewActivity lọt vào build

Cả ba cùng nằm trong `build.gradle.kts` của module app.

**a) Đưa thư viện tooling của Compose về đúng `debugImplementation`** (đây là nguyên nhân PreviewActivity xuất hiện trong app và bị exported):

```kotlin
dependencies {
    // Xoá dòng implementation(...) của ui-tooling nếu đang có
    debugImplementation("androidx.compose.ui:ui-tooling")
    debugImplementation("androidx.compose.ui:ui-tooling-preview")
}
```

**b) Chặn cứng trong `AndroidManifest.xml`** (phòng khi thư viện khác kéo tooling vào):

```xml
<activity
    android:name="androidx.compose.ui.tooling.PreviewActivity"
    tools:node="remove" />
```

(nhớ khai báo `xmlns:tools="http://schemas.android.com/tools"` ở thẻ `<manifest>` nếu chưa có.)

**c) Tạo keystore riêng cho bản release** — chạy 1 lần trên máy của bạn, **giữ file này và mật khẩu cẩn thận, không commit lên Git**:

```bash
keytool -genkeypair -v -keystore nhathongminh-release.jks -alias nhathongminh -keyalg RSA -keysize 2048 -validity 10000
```

Tạo file `keystore.properties` (cũng **không commit**, thêm vào `.gitignore`):

```properties
storeFile=../nhathongminh-release.jks
storePassword=<mật khẩu của bạn>
keyAlias=nhathongminh
keyPassword=<mật khẩu của bạn>
```

**d) Cấu hình bản release trong `build.gradle.kts`:**

```kotlin
// Đọc keystore.properties ở đầu file
val keystoreProperties = java.util.Properties().apply {
    val f = rootProject.file("keystore.properties")
    if (f.exists()) load(f.inputStream())
}

android {
    signingConfigs {
        create("release") {
            storeFile = file(keystoreProperties["storeFile"] as String)
            storePassword = keystoreProperties["storePassword"] as String
            keyAlias = keystoreProperties["keyAlias"] as String
            keyPassword = keystoreProperties["keyPassword"] as String
        }
    }
    buildTypes {
        release {
            isDebuggable = false
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
            signingConfig = signingConfigs.getByName("release")
        }
    }
}
```

Sau khi build release: bản mới **không cài đè** lên bản debug đang chạy (khác chữ ký) — gỡ bản debug trước khi cài bản release.

## Lỗi 2: cleartext toàn cục → chỉ cho phép đúng IP của Home Assistant

Tạo file `app/src/main/res/xml/network_security_config.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config cleartextTrafficPermitted="false" />
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="false">192.168.1.10</domain>
    </domain-config>
</network-security-config>
```

Trong `AndroidManifest.xml`, ở thẻ `<application>`:

```xml
<application
    android:networkSecurityConfig="@xml/network_security_config"
    android:usesCleartextTraffic="false"
    ... >
```

Kết quả: app chỉ còn nói HTTP/WS không mã hoá với đúng `192.168.1.10` (Home Assistant), mọi địa chỉ khác bắt buộc HTTPS. Nếu sau này đổi IP HA, sửa IP trong file này.

## Lỗi 4: nâng targetSdk 34 → 35

Trong `build.gradle.kts`:

```kotlin
android {
    compileSdk = 35
    defaultConfig {
        targetSdk = 35
    }
}
```

Lưu ý khi nâng:
- Android 14+ siết quyền `USE_FULL_SCREEN_INTENT`: app cài ngoài Play (sideload) vẫn cấp được qua phần cài đặt quyền của hệ thống — kiểm tra lại tính năng cảnh báo toàn màn hình sau khi nâng.
- Quyền `POST_NOTIFICATIONS` đã khai báo sẵn nên không vướng.
- Build xong nên kiểm thử lại: kết nối WebSocket tới HA, cảnh báo khi người về, tự chạy sau reboot.

## Vì sao không vá trực tiếp trên file APK?

Về kỹ thuật có thể giải nén APK, sửa manifest rồi đóng gói lại, nhưng bản vá sẽ: (1) phải ký bằng một key lạ — không cài đè được lên bản hiện tại và lệch hẳn khỏi project nguồn; (2) không sửa được lỗi nằm trong cấu hình Gradle (nguồn gốc của PreviewActivity và debug flag); (3) mỗi lần build bản mới từ nguồn, lỗi lại quay lại. Sửa ở project nguồn là cách duy nhất dứt điểm.
