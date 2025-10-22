# 第 0 章：專案設置與環境準備

## 📋 本章目標

- 設置 Android 開發環境
- 了解專案結構和配置
- 配置必要的依賴項
- 成功運行應用程式

## 🛠️ 環境需求

### 必需軟體

- **Android Studio** Giraffe | 2022.3.1 或更新版本
- **JDK** 17 或更新版本
- **Android SDK** API Level 24 (Android 7.0) 最低支援
- **Android SDK** API Level 34 目標版本

### 推薦配置

- **記憶體**：至少 8GB RAM（推薦 16GB）
- **硬碟空間**：至少 4GB 可用空間
- **作業系統**：Windows 10+、macOS 10.14+、或 Linux

## 📁 專案結構概覽

```
PomodoroApp/
├── app/
│   ├── build.gradle.kts          # 應用程式級建構檔案
│   ├── proguard-rules.pro        # ProGuard 規則
│   └── src/
│       ├── main/
│       │   ├── java/com/fisher/pomodoroapp/
│       │   │   ├── ui/           # UI 層（Jetpack Compose）
│       │   │   ├── data/         # 資料層（Repository + DataSource）
│       │   │   ├── domain/       # 領域層（Use Cases）
│       │   │   ├── di/           # 依賴注入（Hilt 模組）
│       │   │   ├── service/      # 背景服務
│       │   │   ├── util/         # 工具類
│       │   │   └── doc/          # 教學文檔
│       │   ├── res/              # 資源檔案
│       │   └── AndroidManifest.xml
│       ├── test/                 # 單元測試
│       └── androidTest/          # 儀器測試
├── build.gradle.kts              # 專案級建構檔案
├── gradle.properties             # Gradle 屬性
├── settings.gradle.kts           # Gradle 設定
└── README.md                     # 專案說明文件
```

## ⚙️ 依賴項配置

### 專案級 build.gradle.kts

```kotlin
// Top-level build file where you can add configuration options common to all sub-modules.
plugins {
    id("com.android.application") version "8.1.2" apply false
    id("org.jetbrains.kotlin.android") version "1.9.10" apply false
    id("com.google.dagger.hilt.android") version "2.48" apply false
    id("com.google.devtools.ksp") version "1.9.10-1.0.13" apply false
}
```

### 應用程式級 build.gradle.kts

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("kotlin-kapt")
    id("dagger.hilt.android.plugin")
    id("com.google.devtools.ksp")
}

android {
    namespace = "com.fisher.pomodoroapp"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.fisher.pomodoroapp"
        minSdk = 24
        targetSdk = 34
        versionCode = 1
        versionName = "1.0"

        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
        vectorDrawables {
            useSupportLibrary = true
        }
    }

    buildTypes {
        release {
            isMinifyEnabled = false
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
    
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    
    kotlinOptions {
        jvmTarget = "17"
    }
    
    buildFeatures {
        compose = true
    }
    
    composeOptions {
        kotlinCompilerExtensionVersion = "1.5.4"
    }
    
    packaging {
        resources {
            excludes += "/META-INF/{AL2.0,LGPL2.1}"
        }
    }
}

dependencies {
    // Jetpack Compose BOM
    implementation(platform("androidx.compose:compose-bom:2023.10.01"))
    
    // Core Android
    implementation("androidx.core:core-ktx:1.12.0")
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.7.0")
    implementation("androidx.activity:activity-compose:1.8.1")
    
    // Jetpack Compose
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.ui:ui-graphics")
    implementation("androidx.compose.ui:ui-tooling-preview")
    implementation("androidx.compose.material3:material3")
    
    // Navigation
    implementation("androidx.navigation:navigation-compose:2.7.5")
    
    // ViewModel
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.7.0")
    
    // Hilt
    implementation("com.google.dagger:hilt-android:2.48")
    implementation("androidx.hilt:hilt-navigation-compose:1.1.0")
    kapt("com.google.dagger:hilt-compiler:2.48")
    
    // Room
    implementation("androidx.room:room-runtime:2.6.0")
    implementation("androidx.room:room-ktx:2.6.0")
    ksp("androidx.room:room-compiler:2.6.0")
    
    // Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")
    
    // Testing
    testImplementation("junit:junit:4.13.2")
    testImplementation("io.mockk:mockk:1.13.8")
    testImplementation("app.cash.turbine:turbine:1.0.0")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
    
    androidTestImplementation("androidx.test.ext:junit:1.1.5")
    androidTestImplementation("androidx.test.espresso:espresso-core:3.5.1")
    androidTestImplementation(platform("androidx.compose:compose-bom:2023.10.01"))
    androidTestImplementation("androidx.compose.ui:ui-test-junit4")
    
    // Debug
    debugImplementation("androidx.compose.ui:ui-tooling")
    debugImplementation("androidx.compose.ui:ui-test-manifest")
}

// Allow references to generated code
kapt {
    correctErrorTypes = true
}
```

## 🔧 重要配置檔案

### AndroidManifest.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <!-- 權限 -->
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.WAKE_LOCK" />
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
    
    <!-- 前台服務類型 -->
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />

    <application
        android:name=".MyApp"
        android:allowBackup="true"
        android:dataExtractionRules="@xml/data_extraction_rules"
        android:fullBackupContent="@xml/backup_rules"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.PomodoroApp"
        tools:targetApi="31">
        
        <!-- 主要 Activity -->
        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:theme="@style/Theme.PomodoroApp">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
        
        <!-- 計時器前台服務 -->
        <service
            android:name=".service.TimerService"
            android:foregroundServiceType="specialUse"
            android:exported="false">
            <property
                android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE"
                android:value="計時器" />
        </service>
        
    </application>
</manifest>
```

### gradle.properties

```properties
# 項目配置
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
org.gradle.daemon=true
org.gradle.parallel=true
org.gradle.caching=true

# Kotlin
kotlin.code.style=official

# Android
android.useAndroidX=true
android.enableJetifier=true

# 編譯優化
android.nonTransitiveRClass=true
android.defaults.buildfeatures.buildconfig=true
android.defaults.buildfeatures.aidl=false
android.defaults.buildfeatures.renderscript=false
android.defaults.buildfeatures.resvalues=false
android.defaults.buildfeatures.shaders=false

# Compose 編譯器優化
android.experimental.enableScreenshotTest=true
```

## 🚀 首次執行步驟

### 1. 克隆專案

```bash
git clone <repository-url>
cd PomodoroApp
```

### 2. 開啟 Android Studio

1. 啟動 Android Studio
2. 選擇「Open an Existing Project」
3. 導航到專案資料夾並選擇

### 3. 同步專案

1. Android Studio 會自動提示同步 Gradle
2. 點擊「Sync Now」
3. 等待同步完成（可能需要幾分鐘）

### 4. 設置模擬器

1. 開啟 AVD Manager
2. 建立新的虛擬裝置
3. 推薦規格：
   - **Device**: Pixel 6
   - **API Level**: 34 (Android 14)
   - **Target**: Google APIs

### 5. 執行應用程式

1. 確保模擬器正在運行
2. 點擊「Run」按鈕或按 `Shift + F10`
3. 等待應用程式安裝和啟動

## ✅ 驗證安裝

成功安裝後，你應該能看到：

- 📱 番茄鐘應用程式主畫面
- ⏱️ 25:00 的計時器顯示
- 🎨 Material Design 3 風格的 UI
- 🔄 底部導航列（首頁、任務、歷史）

## 🔍 常見問題排解

### Gradle 同步失敗

**解決方案：**
1. 檢查網路連線
2. 清除 Gradle 快取：`./gradlew clean`
3. 重新同步專案

### 編譯錯誤

**解決方案：**
1. 確認 JDK 版本為 17
2. 檢查 Android SDK 是否正確安裝
3. 確認所有依賴項版本相容

### 模擬器無法啟動

**解決方案：**
1. 檢查 BIOS 中的虛擬化設定
2. 確認有足夠的記憶體
3. 嘗試使用不同的模擬器映像

## 📚 延伸閱讀

- [Android 開發者官方文件](https://developer.android.com/)
- [Jetpack Compose 基礎教學](https://developer.android.com/jetpack/compose/tutorial)
- [Kotlin 語言參考](https://kotlinlang.org/docs/)

## 🎯 下一章預告

在下一章中，我們將深入探討**架構設計概覽**，學習：

- Clean Architecture 原則
- MVVM 設計模式
- 專案層級劃分
- 資料流設計

準備好了嗎？讓我們開始建構強大的番茄鐘應用程式！🚀