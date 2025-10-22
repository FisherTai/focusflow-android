# 番茄鐘應用程式教學系列

歡迎來到番茄鐘應用程式的完整教學系列！這個教學將帶你從零開始，深入了解如何使用現代 Android 開發技術建構一個功能完整的番茄鐘應用程式。

## 📚 教學概述

本教學系列採用循序漸進的方式，從基礎概念到進階實作，讓你全面掌握現代 Android 應用程式開發。

### 🎯 學習目標

- 學會使用 **Jetpack Compose** 建構現代化 UI
- 掌握 **Clean Architecture** 架構設計原則
- 實作 **MVVM** 設計模式
- 使用 **Hilt** 進行依賴注入
- 學會 **Room** 資料庫操作
- 掌握 **Background Service** 和通知系統
- 學會撰寫單元測試和 UI 測試

### 🛠️ 技術棧

- **Kotlin** - 主要開發語言
- **Jetpack Compose** - 聲明式 UI 框架
- **Material Design 3** - 設計系統
- **Hilt** - 依賴注入框架
- **Room** - 本地資料庫
- **Navigation Compose** - 導航管理
- **Coroutines & Flow** - 非同步處理
- **MockK, Turbine** - 測試框架

## 📖 教學章節

### [第 0 章：專案設置與環境準備](./00-setup.md)
- 開發環境設置
- 專案結構說明
- 依賴項配置
- 第一次執行

### [第 1 章：架構設計概覽](./01-architecture.md)
- Clean Architecture 原則
- MVVM 設計模式
- 專案層級劃分
- 資料流設計

### [第 2 章：UI 層實作](./02-ui-layer.md)
- Jetpack Compose 基礎
- Material Design 3 應用
- 導航系統設計
- 主要畫面實作
  - 計時器畫面
  - 任務管理畫面
  - 歷史記錄畫面

### [第 3 章：資料層實作](./03-data-layer.md)
- Room 資料庫設計
- Repository 模式實作
- 資料來源管理
- 資料轉換和快取

### [第 4 章：領域層設計](./04-domain-layer.md)
- 業務邏輯封裝
- Use Case 實作
- 領域模型設計
- 資料驗證

### [第 5 章：依賴注入](./05-dependency-injection.md)
- Hilt 框架介紹
- 模組設計
- 作用域管理
- 測試配置

### [第 6 章：背景服務](./06-background-service.md)
- Foreground Service 實作
- 通知系統設計
- 計時器背景運行
- 生命週期管理

### [第 7 章：測試實作](./07-testing.md)
- 單元測試設計
- ViewModel 測試
- Repository 測試
- UI 測試實作

### [第 8 章：進階主題](./08-advanced.md)
- 效能優化
- 記憶體管理
- 深色模式實作
- 多語言支援

## 🚀 開始學習

建議按照章節順序進行學習，每個章節都包含：

- 📝 概念講解
- 💻 實作範例
- 🎯 練習題目
- 🔍 延伸閱讀

## 📁 專案結構

```
app/src/main/java/com/fisher/pomodoroapp/
├── 📦 ui/                    # UI 層
│   ├── home/                 # 首頁計時器
│   ├── tasks/                # 任務管理
│   ├── history/              # 歷史記錄
│   ├── components/           # 共用元件
│   └── theme/                # 主題設計
├── 💾 data/                  # 資料層
│   ├── repository/           # Repository 實作
│   └── sources/              # 資料來源
│       ├── database/         # Room 資料庫
│       └── remote/           # 遠端資料（預留）
├── 🏢 domain/                # 領域層
├── 🔌 di/                    # 依賴注入
├── 🔔 service/               # 背景服務
├── 🛠️ util/                  # 工具類
└── 📚 doc/                   # 教學文檔（本系列）
```

## 💡 學習建議

1. **循序漸進**：建議按照章節順序學習
2. **動手實作**：每個概念都要親自寫程式碼
3. **理解原理**：不只是複製程式碼，要理解背後的設計原理
4. **多做練習**：完成每章的練習題目
5. **持續學習**：關注 Android 最新技術發展

## 🤝 貢獻與回饋

如果你發現教學中的錯誤或有改進建議，歡迎：

- 提出 Issue
- 發送 Pull Request
- 分享學習心得

祝你學習愉快！🎉