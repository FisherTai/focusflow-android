# FocusFlow - Modern Android Architecture Sample

*用於展示Android 架構和 Jetpack Compose UI的專案*

## 主要功能

- **番茄鐘計時器** - 25分鐘專注時間與5分鐘休息時間
- **任務管理** - 新增、編輯、刪除和完成任務
- **歷史記錄** - 查看過往的番茄鐘會話記錄
- **背景通知** - 前台服務支援，計時器在背景運行
- **現代化 UI** - 深、淺色模式

## 架構設計

基本遵循 **Modern Android App Architecture** ：

```
📦 app
├── 🎨 ui/                    # UI Layer
│   ├── home/                 # 首頁計時器
│   ├── tasks/                # 任務管理
│   ├── history/              # 歷史記錄
│   ├── components/           
│   └── theme/                
├── 💾 data/                  # Data Layer
│   ├── repository/           # Repository
│   └── sources/              
│       ├── database/         # Room DB
│       └── remote/           # 遠端資料(若後續更新)
├── 🏢 domain/                # Domain Layer
├── 🔌 di/                    # Dependency Injection
├── 🔔 service/               # 背景服務
└── 🛠️ util/                  
```

## 技術棧

### 核心技術

- **Kotlin** - 主要開發語言
- **Jetpack Compose** - 聲明式 UI 框架
- **Material Design 3** - 設計系統

### 架構元件

- **MVVM** - 架構
- **Hilt** - 依賴注入
- **Room** - 本地資料庫
- **Navigation Compose** - 導航
- **Coroutines & Flow** - 非同步處理
- **mockk, turbine, kotlinxCoroutinesTest** - Data、ViewModel測試組件

### 其他函式庫
- **compose-swipebox** - 滑動手勢

## 主要畫面

### 首頁 (Home)

- 顯示倒計時
- 開始/暫停/重設按鈕
- 專注/休息狀態切換

### 任務清單 (Tasks)

- 新增任務
- 滑動刪除/編輯
- 標記進行中任務

### 歷史記錄 (History)

- 顯示紀錄
- 日期篩選

## 學習重點

本專案主要用於練習以下技術：

### 架構設計

- Modern Android App Architecture
- MVVM + Repository Pattern
- Dependency Injection

### Jetpack Compose

- Jetpack Compose UI 開發
- Navigation
- Custom Components

### Android 元件

- Foreground Service
- Notification
- Room 資料庫

### 測試框架

- **MockK** - 用於模擬測試依賴物件
- **Turbine** - 用於測試 StateFlow 和 SharedFlow
- **Kotlinx Coroutines Test** - 協程測試，用於測試 suspend 函數和協程邏輯

