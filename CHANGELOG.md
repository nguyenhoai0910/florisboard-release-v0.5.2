## Changelog: f44de2c..87f1f4f

### Commit
- Save current work before reapplying gitignore (87f1f4f8, 2026-11-21)
- update-VZB4S-20261121-194142-2632 (c6f5981e, 2026-11-21)
- update-oTWrS-20261121-192111-7149 (7fb232fe, 2026-11-21)
- update-pxP6i-20261121-191619-4914 (31adb15c, 2026-11-21)

### Tong quan
```
 .gitignore                                         | 10 ++++
 app/build.gradle.kts                               | 19 ++++++
 .../popupMappings/en.json                          | 67 ----------------------
 .../popupMappings/vi-VN.json                       | 27 ---------
 gradle/libs.versions.toml                          |  2 +-
 5 files changed, 30 insertions(+), 95 deletions(-)
```

### Danh sach file
- sua: .gitignore
- sua: app/build.gradle.kts
- sua: app/src/main/assets/ime/keyboard/org.florisboard.localization/popupMappings/en.json
- sua: app/src/main/assets/ime/keyboard/org.florisboard.localization/popupMappings/vi-VN.json
- sua: gradle/libs.versions.toml

### Chi tiet thay doi

#### `.gitignore`
```diff
diff --git a/.gitignore b/.gitignore
index 9923a24d..1cba89cd 100644
--- a/.gitignore
+++ b/.gitignore
@@ -62,2 +62,12 @@ target/
 **/*.rs.bk
 *.pdb
+
+# Release signing (never commit)
+keystore.properties
+*.keystore
+*.jks
+
+# Local temp files
+*.bak
+*.bak2
+versions-init.gradle
```

#### `app/build.gradle.kts`
```diff
diff --git a/app/build.gradle.kts b/app/build.gradle.kts
index 34c4cbd4..bf24e764 100644
--- a/app/build.gradle.kts
+++ b/app/build.gradle.kts
@@ -16,4 +16,6 @@
 
 import org.jetbrains.kotlin.gradle.dsl.JvmTarget
+import java.io.FileInputStream
+import java.util.Properties
 
 plugins {
@@ -26,4 +28,9 @@ plugins {
 }
 
+val keystoreProps = Properties().apply {
+    val f = rootProject.file("keystore.properties")
+    if (f.exists()) load(FileInputStream(f))
+}
+
 val projectMinSdk: String by project
 val projectTargetSdk: String by project
@@ -56,4 +63,15 @@ android {
     ndkVersion = tools.versions.ndk.get()
 
+    signingConfigs {
+        if (keystoreProps.containsKey("storeFile")) {
+            create("release") {
+                storeFile = file(keystoreProps.getProperty("storeFile"))
+                storePassword = keystoreProps.getProperty("storePassword")
+                keyAlias = keystoreProps.getProperty("keyAlias")
+                keyPassword = keystoreProps.getProperty("keyPassword")
+            }
+        }
+    }
+
     compileOptions {
         sourceCompatibility = JavaVersion.VERSION_11
@@ -135,4 +153,5 @@ android {
 
         named("release") {
+            signingConfigs.findByName("release")?.let { signingConfig = it }
             versionNameSuffix = projectVersionNameSuffix
 
```

#### `app/src/main/assets/ime/keyboard/org.florisboard.localization/popupMappings/en.json`
```diff
diff --git a/app/src/main/assets/ime/keyboard/org.florisboard.localization/popupMappings/en.json b/app/src/main/assets/ime/keyboard/org.florisboard.localization/popupMappings/en.json
index 266db3c2..6242a61d 100644
--- a/app/src/main/assets/ime/keyboard/org.florisboard.localization/popupMappings/en.json
+++ b/app/src/main/assets/ime/keyboard/org.florisboard.localization/popupMappings/en.json
@@ -1,71 +1,4 @@
 {
   "all": {
-    "a": {
-      "relevant": [
-        { "$": "auto_text_key", "code":  230, "label": "æ" },
-        { "$": "auto_text_key", "code":  227, "label": "ã" },
-        { "$": "auto_text_key", "code":  229, "label": "å" },
-        { "$": "auto_text_key", "code":  257, "label": "ā" },
-        { "$": "auto_text_key", "code":  224, "label": "à" },
-        { "$": "auto_text_key", "code":  225, "label": "á" },
-        { "$": "auto_text_key", "code":  226, "label": "â" },
-        { "$": "auto_text_key", "code":  228, "label": "ä" }
-      ]
-    },
-    "c": {
-      "relevant": [
-        { "$": "auto_text_key", "code":  231, "label": "ç" }
-      ]
-    },
-    "e": {
-      "relevant": [
-        { "$": "auto_text_key", "code":  275, "label": "ē" },
-        { "$": "auto_text_key", "code":  234, "label": "ê" },
-        { "$": "auto_text_key", "code":  233, "label": "é" },
-        { "$": "auto_text_key", "code":  232, "label": "è" },
-        { "$": "auto_text_key", "code":  235, "label": "ë" }
-      ]
-    },
-    "i": {
-      "relevant": [
-        { "$": "auto_text_key", "code":  236, "label": "ì" },
-        { "$": "auto_text_key", "code":  239, "label": "ï" },
-        { "$": "auto_text_key", "code":  237, "label": "í" },
-        { "$": "auto_text_key", "code":  238, "label": "î" },
-        { "$": "auto_text_key", "code":  299, "label": "ī" }
-      ]
-    },
-    "n": {
-      "relevant": [
-        { "$": "auto_text_key", "code":  241, "label": "ñ" },
-        { "$": "auto_text_key", "code":  324, "label": "ń" }
-      ]
-    },
-    "o": {
-      "relevant": [
-        { "$": "auto_text_key", "code":  245, "label": "õ" },
-        { "$": "auto_text_key", "code":  333, "label": "ō" },
-        { "$": "auto_text_key", "code":  339, "label": "œ" },
-        { "$": "auto_text_key", "code":  248, "label": "ø" },
-        { "$": "auto_text_key", "code":  242, "label": "ò" },
-        { "$": "auto_text_key", "code":  246, "label": "ö" },
-        { "$": "auto_text_key", "code":  243, "label": "ó" },
-        { "$": "auto_text_key", "code":  244, "label": "ô" }
-      ]
-    },
-    "s": {
-      "relevant": [
-        { "$": "auto_text_key", "code":  223, "label": "ß" }
-      ]
-    },
-    "u": {
-      "relevant": [
-        { "$": "auto_text_key", "code":  250, "label": "ú" },
-        { "$": "auto_text_key", "code":  363, "label": "ū" },
-        { "$": "auto_text_key", "code":  252, "label": "ü" },
-        { "$": "auto_text_key", "code":  251, "label": "û" },
-        { "$": "auto_text_key", "code":  249, "label": "ù" }
-      ]
-    },
     "~right": {
       "main": { "code":   44, "label": "," },
```

#### `app/src/main/assets/ime/keyboard/org.florisboard.localization/popupMappings/vi-VN.json`
```diff
diff --git a/app/src/main/assets/ime/keyboard/org.florisboard.localization/popupMappings/vi-VN.json b/app/src/main/assets/ime/keyboard/org.florisboard.localization/popupMappings/vi-VN.json
index 8d57c20e..6242a61d 100644
--- a/app/src/main/assets/ime/keyboard/org.florisboard.localization/popupMappings/vi-VN.json
+++ b/app/src/main/assets/ime/keyboard/org.florisboard.localization/popupMappings/vi-VN.json
@@ -1,31 +1,4 @@
 {
   "all": {
-    "a": {
-      "relevant": [
-        { "$": "auto_text_key", "code": 226, "label": "â" },
-        { "$": "auto_text_key", "code": 259, "label": "ă" }
-      ]
-    },
-    "e": {
-      "relevant": [
-        { "$": "auto_text_key", "code": 234, "label": "ê" }
-      ]
-    },
-    "o": {
-      "relevant": [
-        { "$": "auto_text_key", "code": 244, "label": "ô" },
-        { "$": "auto_text_key", "code": 417, "label": "ơ" }
-      ]
-    },
-    "u": {
-      "relevant": [
-        { "$": "auto_text_key", "code": 432, "label": "ư" }
-      ]
-    },
-    "d": {
-      "relevant": [
-        { "$":  "auto_text_key", "code": 273, "label": "đ" }
-      ]
-    },
     "~right": {
       "main": { "code":   44, "label": "," },
```

#### `gradle/libs.versions.toml`
```diff
diff --git a/gradle/libs.versions.toml b/gradle/libs.versions.toml
index 71225aea..dab5ccf8 100644
--- a/gradle/libs.versions.toml
+++ b/gradle/libs.versions.toml
@@ -22,5 +22,5 @@ material-kolor = "4.0.5"
 mikepenz-aboutlibraries = "13.1.0"
 patrickgold-compose-tooltip = "0.2.0-rc02"
-patrickgold-jetpref = "20251119T222500Z-SNAPSHOT"
+patrickgold-jetpref = "0.3.0-rc01"
 
 # Testing
```

