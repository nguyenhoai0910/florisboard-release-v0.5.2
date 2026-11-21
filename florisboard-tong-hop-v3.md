# FlorisBoard: build release, nâng AGP 9 và cài APK (bản tổng hợp v3)

Bản này thay cho v2, bổ sung: khối ký bản release đã bị mất hai lần và cách khôi phục, thông báo lint baseline, danh sách kiểm tra trước khi build.

## 1. Cấu hình đã build được

| Thành phần | Phiên bản |
|---|---|
| Gradle | 9.3.1 |
| Android Gradle Plugin (AGP) | 9.1.1 |
| Kotlin | 2.4.21 |
| KSP | 2.3.x (không còn theo số Kotlin; `2.3.14` hoặc `2.3.12`) |
| compileSdk | 37 (gói SDK `platforms;android-37.0`) |
| JetPref | 0.4.0 (không dùng bản SNAPSHOT) |
| AboutLibraries | 15.3.0 |
| JDK chạy Gradle | 17 |

AGP và Gradle phải khớp: AGP 9.1 cần Gradle tối thiểu 9.3.1, 9.2 cần 9.4.1, 9.3 cần 9.5.0, 9.4 cần 9.6.0. Thư viện mới (Compose 1.12, core 1.19, JetPref 0.4.0) đòi AGP từ 9.1.0 và compileSdk 37; API 37.0 cần AGP từ 9.1.1.

## 2. Các thay đổi bắt buộc cho AGP 9 (kiểm tra cả 6 mục)

1. **Gradle wrapper:** sửa thẳng `gradle/wrapper/gradle-wrapper.properties`:
   ```bash
   sed -i 's#gradle-[0-9.]*-\(bin\|all\)\.zip#gradle-9.3.1-bin.zip#' gradle/wrapper/gradle-wrapper.properties
   sed -i '/distributionSha256Sum/d' gradle/wrapper/gradle-wrapper.properties
   ./gradlew --version && ./gradlew wrapper
   ```
2. **`gradle/libs.versions.toml`:** `android-gradle-plugin = "9.1.1"`, `kotlin = "2.4.21"`, `ksp = "2.3.14"`, `patrickgold-jetpref = "0.4.0"`, `mikepenz-aboutlibraries = "15.3.0"`.
3. **`gradle.properties`:** `projectCompileSdk=37`; cài gói: `sdkmanager "platforms;android-37.0"` (tên gói có `.0`).
4. **Xoá plugin `kotlin.android`** ở 8 file (`app`, `benchmark`, gốc, `lib/android`, `lib/color`, `lib/compose`, `lib/native`, `lib/snygg`):
   ```bash
   sed -i.bak '/alias(libs.plugins.kotlin.android)/d' app/build.gradle.kts benchmark/build.gradle.kts build.gradle.kts \
     lib/android/build.gradle.kts lib/color/build.gradle.kts lib/compose/build.gradle.kts \
     lib/native/build.gradle.kts lib/snygg/build.gradle.kts
   ```
   Giữ `kotlin-plugin-compose`, `kotlin-serialization`, `ksp`.
5. **`resValues = true`** trong `buildFeatures` của `app/build.gradle.kts`.
6. **Task `generateJsonSchema`** trong `lib/snygg/build.gradle.kts` phải viết lại (xem mục 4).

## 3. Khối ký bản release trong `app/build.gradle.kts` (đã mất hai lần)

Nếu thiếu khối này, kết quả chỉ là `app-release-unsigned.apk` và **không cài được**. Bốn chỗ cần có:

**a) Import ở đầu file:**
```kotlin
import java.io.FileInputStream
import java.util.Properties
```

**b) Ngay dưới khối `plugins { ... }`:**
```kotlin
val keystoreProps = Properties().apply {
    val f = rootProject.file("keystore.properties")
    if (f.exists()) load(FileInputStream(f))
}
```

**c) Trong khối `android { ... }`, dưới `ndkVersion`:**
```kotlin
    signingConfigs {
        if (keystoreProps.containsKey("storeFile")) {
            create("release") {
                storeFile = file(keystoreProps.getProperty("storeFile"))
                storePassword = keystoreProps.getProperty("storePassword")
                keyAlias = keystoreProps.getProperty("keyAlias")
                keyPassword = keystoreProps.getProperty("keyPassword")
            }
        }
    }
```

**d) Dòng đầu trong `named("release") { ... }`:**
```kotlin
        signingConfigs.findByName("release")?.let { signingConfig = it }
```

Không đặt khối `android { }` riêng phía trên `plugins`, vì khi đó plugin Android chưa được áp dụng.

`keystore.properties` (thư mục gốc, không commit):
```properties
storeFile=/home/ban/florisboard-release.jks
storePassword=MAT_KHAU_STORE
keyAlias=florisboard
keyPassword=MAT_KHAU_KEY
```

Tạo keystore (backup file `.jks` và mật khẩu, mất là không update cùng chữ ký được):
```bash
keytool -genkeypair -v -keystore ~/florisboard-release.jks \
  -keyalg RSA -keysize 2048 -validity 10000 -alias florisboard
```

Nếu thiếu `keystore.properties`, điều kiện `if` bỏ qua việc ký mà không báo lỗi, nên luôn kiểm tra APK sau khi build (mục 7).

## 4. Task `generateJsonSchema` (lib/snygg)

Chỉ dùng để sinh `schemas/stylesheet.schema.json`, không ảnh hưởng APK. AGP 9 đã gỡ `libraryVariants` nên phải viết lại (bản thử, chưa kiểm chứng đầy đủ):

```kotlin
tasks.register<JavaExec>("generateJsonSchema") {
    dependsOn("compileDebugKotlin", "compileDebugJavaWithJavac")
    mainClass.set("org.florisboard.lib.snygg.SnyggJsonSchemaGenerator")
    classpath = files(providers.provider {
        val ownClasses = files(
            tasks.getByName("compileDebugKotlin").outputs.files,
            tasks.getByName("compileDebugJavaWithJavac").outputs.files,
        )
        val dependencyJars = configurations.getByName("debugRuntimeClasspath").incoming.artifactView {
            attributes {
                attribute(
                    org.gradle.api.attributes.Attribute.of("artifactType", String::class.java),
                    "android-classes-jar",
                )
            }
        }.files
        ownClasses + dependencyJars
    })
    args = listOf("schemas/stylesheet.schema.json")
    workingDir = projectDir
    standardOutput = System.out
}
```

Thử chạy: `./gradlew :lib:snygg:generateJsonSchema`. Nếu lỗi `ClassNotFoundException` thì classpath còn thiếu; không cần sinh lại schema thì bỏ qua.

## 5. Bảng lỗi đã gặp

| Lỗi | Nguyên nhân | Cách sửa |
|---|---|---|
| `'org.jetbrains.kotlin.android' plugin is no longer required... since AGP 9.0` | AGP 9 tích hợp sẵn Kotlin | Xoá plugin ở mọi module |
| `Build Type debug contains custom resource values, but the feature is disabled` | AGP 9 tắt mặc định `resValue` | `resValues = true` |
| `Unresolved reference 'libraryVariants'` | AGP 9 gỡ variant API cũ | Viết lại task (mục 4) |
| `Plugin ksp version '2.4.0' was not found` | Số KSP không theo số Kotlin | `ksp = "2.3.14"` hoặc `2.3.12` |
| `Could not find ...jetpref...-SNAPSHOT` | Snapshot bị xoá khỏi kho Maven | Dùng bản release `0.4.0` |
| `Module was compiled with an incompatible version of Kotlin... expected 2.2.0` | Thư viện build bằng Kotlin 2.4, compiler cũ | Nâng Kotlin lên 2.4.x |
| `requires Android Gradle plugin 9.1.0 or higher` / `compile against version 37` | Thư viện mới cần AGP và compileSdk mới | AGP ≥ 9.1.1, compileSdk 37 |
| `Failed to find package 'platforms;android-37'` | Từ API 37 tên gói có `.0` | `sdkmanager "platforms;android-37.0"` |
| `NoClassDefFoundError: ...ProjectTypeBinding` | Gradle quá cũ so với AGP | Nâng Gradle (sửa thẳng `distributionUrl`) |
| `N errors/warnings were listed in the baseline file (app/lint.xml) but not found` | Mục cũ trong baseline đã hết | Chỉ là thông báo, bỏ qua được |

**Lint baseline:** muốn dọn thì `./gradlew :app:updateLintBaseline` rồi `git diff --stat app/lint.xml`. Nếu diff thêm nhiều mục lạ thì `git checkout app/lint.xml`.

**Cảnh báo bỏ qua được:** `by project` / `srcDirs` deprecated, `Configuration.setVisible`, CMake deprecation, `No Android AppExtension found` (AboutLibraries; chỉ cần xử lý nếu danh sách giấy phép trong màn About bị trống).

## 6. Bảo vệ khóa ký

`.gitignore` ban đầu chỉ có `*.jks`. Thêm:
```gitignore
keystore.properties
*.keystore
*.jks
*.bak
*.bak2
versions-init.gradle
```
Kiểm tra: `git ls-files | grep -E "keystore|\.jks|\.bak"` (không được in gì) và `git check-ignore -v keystore.properties`. Nếu `keystore.properties` hoặc `.jks` từng bị commit rồi push công khai thì coi như lộ mật khẩu, nên tạo keystore mới.

**Sau khi build qua, commit ngay** để không mất các sửa đổi lần nữa: `git add -A && git commit -m "AGP 9 migration"`.

## 7. Danh sách kiểm tra trước khi build release

```bash
cd /mnt/Data/hoainh/Documents/florisboard-release-v0.5.2
grep distributionUrl gradle/wrapper/gradle-wrapper.properties        # gradle-9.3.1
grep -n "^android-gradle-plugin\|^kotlin =\|^ksp\|jetpref" gradle/libs.versions.toml
grep -n "projectCompileSdk" gradle.properties                         # 37
grep -rn "kotlin.android" --include="*.kts" .                         # không in gì
grep -n "resValues\|signingConfigs\|keystoreProps" app/build.gradle.kts
grep -rn "libraryVariants" --include="*.kts" .                        # không in gì
cat keystore.properties | sed 's/Password=.*/Password=***/'           # đủ 4 khóa
```

Build và kiểm tra APK đã ký:
```bash
./gradlew assembleRelease > build.log 2>&1; grep -n "^e:" build.log | head -20; tail -n 15 build.log
ls app/build/outputs/apk/release/        # phải là app-release.apk, không phải ...-unsigned.apk
apksigner verify --print-certs app/build/outputs/apk/release/app-release.apk
```

`apksigner` nằm ở `~/Android/Sdk/build-tools/<phiên bản>/` nếu chưa có trong PATH.

## 8. Cài APK lên máy Samsung

Nếu cài bằng giao diện bị kẹt ở "Installing":

1. Reboot máy; gỡ bản FlorisBoard cùng package nhưng khác chữ ký.
2. Bật *Cài đặt ứng dụng không xác định* cho app đang mở APK; tắt Auto Blocker; tắt DNS/VPN chặn quảng cáo.
3. Xoá cache và buộc dừng: Package installer, Google Play Store, Google Play services, Android Developer Verifier; rồi reboot.
4. Cài bằng ADB (cho thấy lỗi thật):
```bash
adb devices
adb install -r -d app/build/outputs/apk/release/app-release.apk
```
Nếu đứng im, tắt xác minh khi cài qua ADB rồi thử lại:
```bash
adb shell settings put global verifier_verify_adb_installs 0
adb shell settings put global package_verifier_enable 0
adb install -r -d app-release.apk
```
Bật lại bằng cách đặt hai giá trị về `1`. Xem nó kẹt ở đâu: `adb logcat | grep -i -E "PackageInstaller|PackageManager|verif|dexopt"`.

**Kết nối ADB:** bật Tùy chọn nhà phát triển (bấm 7 lần *Số hiệu bản dựng*) và *Gỡ lỗi USB*, cắm cáp có truyền dữ liệu, bấm *Cho phép*, rồi `adb devices` phải hiện `device`. Không dây (Android 11+): `adb pair IP:CỔNG` rồi `adb connect IP:CỔNG`.

## 9. Ghi chú nền

- Bản debug nặng ~40MB do không có R8/ProGuard, native library chưa tối ưu và gồm nhiều ABI; bản release thu nhỏ đáng kể. Có thể giới hạn ABI (ví dụ chỉ `arm64-v8a`).
- Cài từ F-Droid nhanh hơn vì được hệ thống coi là nguồn quen thuộc; APK tự ký bị trình quét bảo mật kiểm tra lâu hơn.
- Quy tắc xác minh nhà phát triển của Google có hiệu lực từ 30/09/2026 ở Brazil, Indonesia, Singapore, Thái Lan, dự kiến toàn cầu năm 2027. Việt Nam chưa nằm trong đợt đầu; ADB và "advanced flow" vẫn cài được app chưa xác minh.
- Kiểm tra thư viện còn cũ không: dùng `versions-init.gradle` với `gradle-versions-plugin` rồi `./gradlew dependencyUpdates -I versions-init.gradle`; nâng theo nhóm (Gradle → AGP → Kotlin + KSP → Compose BOM → còn lại) và build thử sau mỗi bước.

## 10. Việc cần làm tiếp

- Kiểm tra lại khối ký bản release (mục 3) còn nguyên, build, rồi `apksigner verify`.
- Cài APK bằng `adb install -r -d ...` và ghi lại kết quả (`Success`, mã `INSTALL_FAILED_...`, hoặc đứng im).
- Commit các thay đổi; backup file `.jks` và mật khẩu ở nơi an toàn.
- Tuỳ chọn: chạy `./gradlew :lib:snygg:generateJsonSchema` và `:app:updateLintBaseline`.
