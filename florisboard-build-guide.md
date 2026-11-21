# Hướng dẫn build FlorisBoard v0.5.2 (tùy chỉnh popup)

Mục tiêu: tắt long press trên các chữ cái (a, ă, â, u, ư, c...) nhưng vẫn giữ popup của ký tự đặc biệt, rồi build APK để cài.

> Ghi chú: hướng dẫn này tổng hợp từ tài liệu công khai của FlorisBoard và các lỗi build thực tế bạn đã gặp.
> Những chỗ đánh dấu **(chưa xác nhận)** là chưa kiểm tra được trên source v0.5.2.

---

## 1. Chuẩn bị

- **JDK 17 trở lên** (cần cho Gradle và sdkmanager).
- **Android SDK** (cài qua Android Studio hoặc command-line tools).
- **CMake 4.0.2** và **NDK** (cài qua sdkmanager, xem bước 2).
- **Rust toolchain** (cài qua rustup, xem bước 3), vì phần native nằm trong `lib/native/src/main/rust`.
- **unzip**: `sudo apt install unzip` (Debian/Ubuntu) hoặc `sudo pacman -S unzip` (Arch).

---

## 2. Cài Android SDK bằng command-line tools

Đường dẫn dùng trong hướng dẫn này:

| Thứ | Đường dẫn |
| --- | --- |
| File zip | `/home/shiroe/Android/cmdline-tools/commandlinetools-linux-15859902_latest.zip` |
| SDK | `/home/shiroe/Android/Sdk` |
| Repo | `/mnt/Data/hoainh/Documents/florisboard-release-v0.5.2` |

### 2.1. Giải nén đúng cấu trúc

`sdkmanager` yêu cầu thư mục phải tên là `latest`:

```bash
mkdir -p ~/Android/Sdk/cmdline-tools
cd ~/Android/Sdk/cmdline-tools
unzip /home/shiroe/Android/cmdline-tools/commandlinetools-linux-15859902_latest.zip
mv cmdline-tools latest
```

Kiểm tra:

```bash
ls ~/Android/Sdk/cmdline-tools/latest/bin/sdkmanager
```

> Hai thư mục khác nhau, đừng nhầm:
> - `~/Android/cmdline-tools/` chứa file zip.
> - `~/Android/Sdk/cmdline-tools/` là nơi giải nén vào.

### 2.2. Chấp nhận license và cài thành phần cần thiết

```bash
~/Android/Sdk/cmdline-tools/latest/bin/sdkmanager --licenses
~/Android/Sdk/cmdline-tools/latest/bin/sdkmanager "platform-tools" "platforms;android-36"
~/Android/Sdk/cmdline-tools/latest/bin/sdkmanager "cmake;4.0.2"
```

- v0.5.2 nâng `compileSdk` lên 36 (theo release notes), nên chọn `android-36`.
- Log build cho thấy dự án cần **CMake 4.0.2**.
- NDK: log của bạn dùng `ndk/29.0.14206865`. Nếu Gradle báo thiếu NDK, cài bằng `sdkmanager "ndk;29.0.14206865"`.
- Nếu không thấy package, xem danh sách: `sdkmanager --list | grep -i -E "cmake|ndk"`.

---

## 3. Cài Rust toolchain

Lỗi nếu thiếu:

```
CMake Error at CMakeLists.txt:14 (message):
  Rust toolchain not found on this system.  Please install Rust toolchain,
  then try again.
```

Cài bằng rustup (không dùng gói Rust của distro, vì cần thêm target Android):

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env
cargo --version
```

Thêm các target Android (ABI `arm64-v8a` tương ứng `aarch64-linux-android`; Gradle thường cấu hình cả các ABI khác nên thêm đủ bốn):

```bash
rustup target add aarch64-linux-android armv7-linux-androideabi i686-linux-android x86_64-linux-android
```

Danh sách target chính xác mà `CMakeLists.txt` yêu cầu, và việc có cần thêm công cụ như `cargo-ndk` hay không: **(chưa xác nhận)**.

---

## 4. Chỉ cho Gradle biết SDK nằm ở đâu

Lỗi nếu thiếu:

```
SDK location not found. Define a valid SDK location with an ANDROID_HOME
environment variable or by setting the sdk.dir path in local.properties
```

Cách sửa: tạo file `local.properties` ở thư mục gốc repo.

```bash
echo "sdk.dir=/home/shiroe/Android/Sdk" > /mnt/Data/hoainh/Documents/florisboard-release-v0.5.2/local.properties
```

Hoặc dùng biến môi trường (chỉ có hiệu lực trong terminal đang mở):

```bash
export ANDROID_HOME=/home/shiroe/Android/Sdk
```

---

## 5. Sửa popup chữ cái

Popup mapping trong các bản gần đây nằm trong extension:

- Các mapping được khai báo trong `extension.json` của `org.florisboard.localization`.
- Mỗi ngôn ngữ có một file JSON mapping riêng.
- Khối dấu câu dùng chung nằm ở mục `~right`.

Các bước:

1. Mở `extension.json` của `org.florisboard.localization`, tìm popup mapping mà subtype tiếng Việt đang dùng.
2. Mở file JSON tương ứng.
3. Xóa (hoặc làm rỗng) các mục của **chữ cái** (a, ă, â, u, ư, c...).
4. **Giữ nguyên** `~right`, hàng số và các phím ký hiệu để còn popup ký tự đặc biệt.
5. Lưu file.

Đường dẫn chính xác của file trong v0.5.2: **(chưa xác nhận)**.

---

## 6. Build

Chạy trong terminal đã `source ~/.cargo/env`, nếu không `cargo` sẽ không nằm trong PATH:

```bash
cd /mnt/Data/hoainh/Documents/florisboard-release-v0.5.2
git submodule update --init --recursive
./gradlew --stop
./gradlew assembleDebug
```

`./gradlew --stop` tắt Gradle daemon cũ, vì daemon nhớ môi trường lúc chưa có Rust.

APK thường nằm trong `app/build/outputs/apk/debug/`.

> Dòng cảnh báo `WRONG_INVOCATION_KIND` của Kotlin compiler chỉ là warning, không làm build hỏng. Hãy tìm dòng có chữ `error` hoặc `FAILED`.

---

## 7. Cài lên điện thoại

- Bản debug được ký bằng khóa debug, nên **không cài đè** được lên bản FlorisBoard cài từ F-Droid hoặc Play Store. Gỡ bản cũ trước, hoặc kiểm tra xem bản debug có dùng package name khác không **(chưa xác nhận)**.
- Cài APK, rồi vào Settings của Android để bật FlorisBoard làm bàn phím.
- Kiểm tra subtype tiếng Việt đã dùng đúng popup mapping vừa sửa chưa.

---

## 8. Xử lý lỗi thường gặp

| Triệu chứng | Hướng xử lý |
| --- | --- |
| `SDK location not found` | Làm bước 4 (tạo `local.properties` hoặc đặt `ANDROID_HOME`). |
| `CMake '4.0.2' was not found in SDK` | `sdkmanager "cmake;4.0.2"` (bước 2.2). |
| `Rust toolchain not found` | Cài Rust bằng rustup và thêm target Android (bước 3), rồi `./gradlew --stop`. |
| `sdkmanager: command not found` hoặc không chạy | Kiểm tra thư mục phải là `cmdline-tools/latest/bin/`, và đã cài JDK. |
| `unzip: command not found` | Cài `unzip` (bước 1). |
| Thiếu license | Chạy `sdkmanager --licenses` và chấp nhận hết. |
| Thiếu NDK | `sdkmanager "ndk;<phiên-bản-trong-log>"`. |
| Đã cài Rust nhưng Gradle vẫn báo thiếu | Mở terminal mới (hoặc `source ~/.cargo/env`), rồi `./gradlew --stop` và build lại. |
| Build xong nhưng popup chữ cái vẫn hiện | Kiểm tra lại JSON đã sửa, và subtype đang dùng có trỏ đúng mapping đó không. |
| Editor vẽ khung nét đứt quanh chữ có dấu | Không phải lỗi file: tắt `Unicode Highlight` hoặc Trust workspace trong VS Code. |

Nếu gặp lỗi khác, copy vài dòng lỗi đầu tiên có chữ `error` của Gradle (không cần cả stack trace) để xem lại.



keytool -genkeypair -v -keystore ~/florisboard-release.jks \
  -keyalg RSA -keysize 2048 -validity 10000 -alias florisboard

storeFile=/mnt/Data/hoainh/Documents/florisboard-release-v0.5.2/florisboard-release.jks
storePassword=florisboard-release-a093dd56-2784-4136-b7db-b60c336e893d
keyAlias=florisboard
keyPassword=florisboard-release-a093dd56-2784-4136-b7db-b60c336e893d