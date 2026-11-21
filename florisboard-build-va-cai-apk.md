# FlorisBoard: build release và xử lý lỗi cài APK bị kẹt "Installing"

## 1. Vì sao APK debug nặng (~40MB) còn bản tải về chỉ ~14.76MB

- Bản debug không có R8/ProGuard (không minify, không xoá code thừa).
- Không shrink resources.
- Thư viện native (Rust/C++) chưa tối ưu, còn debug symbols.
- Gồm nhiều ABI (`arm64-v8a`, `armeabi-v7a`, `x86`, `x86_64`), trong khi bản tải về thường chỉ một ABI.

**Cách giảm:**
- Build release: `./gradlew assembleRelease`
- Giới hạn ABI (ví dụ chỉ `arm64-v8a`) bằng `abiFilters` hoặc `splits { abi { ... } }`
- Bật `isMinifyEnabled = true` và `isShrinkResources = true` cho release
- Dùng Android Studio: **Build → Analyze APK** để xem phần nào chiếm nhiều nhất

## 2. Tạo key và ký bản release

**Tạo keystore:**

```bash
keytool -genkeypair -v -keystore ~/florisboard-release.jks \
  -keyalg RSA -keysize 2048 -validity 10000 -alias florisboard
```

Backup file `.jks` và mật khẩu. Mất là không update cùng chữ ký được nữa.

**`keystore.properties`** (thư mục gốc, không commit lên git):

```properties
storeFile=/home/ban/florisboard-release.jks
storePassword=MAT_KHAU_STORE
keyAlias=florisboard
keyPassword=MAT_KHAU_KEY
```

**`app/build.gradle.kts`:**

```kotlin
import java.util.Properties
import java.io.FileInputStream

val keystoreProps = Properties().apply {
    val f = rootProject.file("keystore.properties")
    if (f.exists()) load(FileInputStream(f))
}

android {
    signingConfigs {
        create("release") {
            storeFile = file(keystoreProps["storeFile"] as String)
            storePassword = keystoreProps["storePassword"] as String
            keyAlias = keystoreProps["keyAlias"] as String
            keyPassword = keystoreProps["keyPassword"] as String
        }
    }
    buildTypes {
        release {
            signingConfig = signingConfigs.getByName("release")
        }
    }
}
```

**Build:** `./gradlew assembleRelease` → APK ở `app/build/outputs/apk/release/`

Kiểm tra chữ ký: `apksigner verify --print-certs app-release.apk`

## 3. Vì sao cài từ F-Droid nhanh còn APK tự build thì kẹt

- F-Droid cài qua PackageInstaller và được hệ thống coi là nguồn quen thuộc.
- APK cài tay, chữ ký lạ sẽ bị trình quét bảo mật (Play Protect hoặc trình quét của hãng) kiểm tra lâu hơn, có thể cần mạng.
- Tắt màn hình không làm tiến trình chạy tiếp, thậm chí còn bị hạn chế nền.
- APK universal ~40MB nhiều ABI cũng tốn thời gian dex2oat.

## 4. Quy định xác minh nhà phát triển của Google (sideloading)

- Quy tắc mới có hiệu lực từ **30/09/2026** ở Brazil, Indonesia, Singapore, Thái Lan.
- Dự kiến mở rộng toàn cầu trên máy Android được chứng nhận vào **2027**.
- **Việt Nam chưa nằm trong đợt đầu**, nên việc APK bị kẹt nhiều khả năng không do quy định này.
- Người dùng nâng cao vẫn có thể cài app chưa xác minh ("advanced flow"). Cài bằng ADB vẫn là cách vượt qua các chặn sideload.

## 5. Checklist xử lý khi APK kẹt ở "Installing" (máy Samsung)

1. **Khởi động lại máy** hẳn rồi cài lại.
2. **Gỡ bản FlorisBoard đang có** (F-Droid hoặc cùng package) nếu khác chữ ký.
3. **Quyền cài app ngoài:** Cài đặt → Ứng dụng → ⋮ → Quyền truy cập đặc biệt → *Cài đặt ứng dụng không xác định* → bật cho app đang mở file APK.
4. **Tắt Auto Blocker** (đã làm) và các trình quét: *Xác minh ứng dụng qua USB*, *Play Protect* nếu có.
5. **Tắt DNS/VPN chặn quảng cáo** (Private DNS kiểu AdGuard/NextDNS) vì có thể chặn máy chủ xác minh.
6. **Xoá cache/dữ liệu và buộc dừng:** Package installer, Google Play Store, Google Play services, Android Developer Verifier (bật *Hiện ứng dụng hệ thống* để thấy), rồi reboot.
7. **Dung lượng trống:** ít nhất 1–2 GB.
8. **Thử trình cài khác:** file manager mặc định hoặc SAI (Split APKs Installer).
9. **Tắt mạng/chế độ máy bay** khi cài: đôi khi nhanh hơn, đôi khi chờ timeout, không chắc chắn.
10. **Cài bằng ADB** (cách chắc nhất, cho thấy lỗi thật).

## 6. Kết nối ADB và cài app

**Chuẩn bị trên máy:**
1. Cài đặt → Giới thiệu điện thoại → Thông tin phần mềm → bấm 7 lần *Số hiệu bản dựng*.
2. Cài đặt → Tùy chọn nhà phát triển → bật **Gỡ lỗi USB**.
3. Để Auto Blocker tắt (nó chặn kết nối USB dữ liệu).

**Cách 1: Cáp USB**
1. Dùng cáp có truyền dữ liệu, chọn chế độ USB là *Truyền tệp*.
2. Bấm **Cho phép** (tick *Luôn cho phép*) trên hộp thoại gỡ lỗi USB.
3. Chạy `adb devices`:
   - `device` → đã kết nối
   - `unauthorized` → chưa bấm Cho phép trên máy
   - `no permissions` (Linux) → `adb kill-server` rồi `sudo adb start-server`

**Cách 2: Không dây (Android 11+, cùng Wi-Fi)**
```bash
adb pair IP:CỔNG_GHÉP_NỐI     # nhập mã 6 số hiển thị trên máy
adb connect IP:CỔNG           # cổng ở màn hình Gỡ lỗi không dây (khác cổng ghép nối)
adb devices
```

**Cài app:**
```bash
adb install -r -d app/build/outputs/apk/release/app-release.apk
```

Nếu đứng im, tắt xác minh khi cài qua ADB rồi thử lại:
```bash
adb shell settings put global verifier_verify_adb_installs 0
adb shell settings put global package_verifier_enable 0
adb install -r -d app-release.apk
```
Bật lại bằng cách đặt hai giá trị về `1`.

**Xem nó kẹt ở đâu:**
```bash
adb logcat | grep -i -E "PackageInstaller|PackageManager|verif|dexopt"
```

**Chưa có `adb`:** cài *Android SDK Platform-Tools* (Linux: `sudo apt install adb`; macOS: `brew install android-platform-tools`; Windows: tải từ developer.android.com).

## 7. Việc cần làm tiếp

- Chạy `adb devices` rồi `adb install -r -d ...` và ghi lại kết quả (`Success`, mã `INSTALL_FAILED_...`, hoặc đứng im).
- Nếu lỗi, dán mã lỗi hoặc vài dòng cuối của `logcat` để xác định chính xác nguyên nhân.
