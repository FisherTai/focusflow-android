# 第 1 章：架構設計概覽

## 📋 本章目標

- 理解 Clean Architecture 設計原則
- 掌握 MVVM 設計模式在 Android 中的應用
- 了解專案層級劃分和職責分離
- 設計高效的資料流架構

## 🏗️ Clean Architecture 概述

Clean Architecture 由 Robert C. Martin 提出，是一種軟體架構設計原則，強調：

### 核心原則

1. **獨立性**：各層級相互獨立，可單獨測試
2. **依賴規則**：依賴關係只能指向內層
3. **可測試性**：業務邏輯可在沒有 UI、資料庫的情況下測試
4. **框架獨立性**：不依賴特定框架

### 架構分層

```
🌐 外層 (Framework & Drivers)
├── 📱 UI Layer (Presentation)
├── 💾 Data Layer 
│
🏢 中層 (Interface Adapters)
├── 🔄 Repository Interfaces
├── 📋 Use Cases Interfaces
│
🎯 內層 (Enterprise Business Rules)
└── 💡 Domain Layer (Business Logic)
```

## 📐 MVVM 設計模式

在 Android 開發中，我們採用 MVVM (Model-View-ViewModel) 模式：

### 架構組成

```
┌─────────────────────────────────────────────┐
│                   View                      │
│              (Jetpack Compose)              │
│  ┌─────────────────────────────────────┐   │
│  │           UI Components             │   │
│  │    ┌─────────┐  ┌─────────────┐   │   │
│  │    │ Screen  │  │ Components  │   │   │
│  │    └─────────┘  └─────────────┘   │   │
│  └─────────────────────────────────────┘   │
└─────────────────┬───────────────────────────┘
                  │ State & Events
                  ▼
┌─────────────────────────────────────────────┐
│                ViewModel                    │
│         (Business Logic)                    │
│  ┌─────────────────────────────────────┐   │
│  │           Use Cases             │   │
│  │    ┌─────────┐  ┌─────────────┐   │   │
│  │    │ Timer   │  │  Tasks      │   │   │
│  │    │UseCase  │  │ UseCase     │   │   │
│  │    └─────────┘  └─────────────┘   │   │
│  └─────────────────────────────────────┘   │
└─────────────────┬───────────────────────────┘
                  │ Data Requests
                  ▼
┌─────────────────────────────────────────────┐
│                 Model                       │
│            (Data Layer)                     │
│  ┌─────────────────────────────────────┐   │
│  │           Repository                │   │
│  │    ┌─────────┐  ┌─────────────┐   │   │
│  │    │Database │  │   Remote    │   │   │
│  │    │DataSource│  │ DataSource  │   │   │
│  │    └─────────┘  └─────────────┘   │   │
│  └─────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

## 🏢 專案層級劃分

### 1. UI Layer (表現層)

**職責：**
- 處理用戶互動
- 顯示資料和狀態
- 導航管理

**組成：**
```kotlin
ui/
├── home/               # 首頁計時器
│   ├── HomeScreen.kt
│   ├── HomeViewModel.kt
│   ├── HomeUiState.kt
│   └── TimerState.kt
├── tasks/              # 任務管理
│   ├── TaskListScreen.kt
│   ├── TaskListViewModel.kt
│   └── TaskUIData.kt
├── history/            # 歷史記錄
│   ├── HistoryScreen.kt
│   └── HistoryViewModel.kt
├── components/         # 共用元件
│   ├── TopBar.kt
│   ├── BottomBar.kt
│   └── FullScreenHint.kt
└── theme/              # 主題設計
    ├── Color.kt
    ├── Theme.kt
    └── ThemeExtensions.kt
```

### 2. Domain Layer (領域層)

**職責：**
- 封裝業務邏輯
- 定義使用案例
- 不依賴任何框架

**組成：**
```kotlin
domain/
├── model/              # 領域模型
│   ├── Task.kt
│   ├── PomodoroSession.kt
│   └── TimerMode.kt
├── repository/         # Repository 介面
│   ├── TaskRepository.kt
│   └── SessionRepository.kt
└── usecase/            # 使用案例
    ├── timer/
    │   ├── StartTimerUseCase.kt
    │   ├── PauseTimerUseCase.kt
    │   └── ResetTimerUseCase.kt
    ├── task/
    │   ├── AddTaskUseCase.kt
    │   ├── UpdateTaskUseCase.kt
    │   └── DeleteTaskUseCase.kt
    └── session/
        ├── SaveSessionUseCase.kt
        └── GetSessionHistoryUseCase.kt
```

### 3. Data Layer (資料層)

**職責：**
- 資料存取和管理
- 實作 Repository 介面
- 處理不同資料來源

**組成：**
```kotlin
data/
├── repository/         # Repository 實作
│   ├── TaskRepositoryImpl.kt
│   └── SessionRepositoryImpl.kt
├── sources/            # 資料來源
│   ├── database/       # 本地資料庫
│   │   ├── AppDatabase.kt
│   │   ├── entities/
│   │   │   ├── TaskEntity.kt
│   │   │   └── SessionEntity.kt
│   │   └── dao/
│   │       ├── TaskDao.kt
│   │       └── SessionDao.kt
│   └── remote/         # 遠端資料（預留）
│       └── api/
└── mapper/             # 資料轉換
    ├── TaskMapper.kt
    └── SessionMapper.kt
```

## 🔄 資料流設計

### 單向資料流 (Unidirectional Data Flow)

```
User Action → ViewModel → Use Case → Repository → DataSource
     ↑                                                ↓
UI State ← ViewModel ← Use Case ← Repository ← DataSource
```

### 詳細流程

1. **用戶操作** → 觸發 UI 事件
2. **ViewModel** → 接收事件，調用對應 Use Case
3. **Use Case** → 執行業務邏輯，調用 Repository
4. **Repository** → 協調不同資料來源
5. **DataSource** → 執行實際資料操作
6. **回傳結果** → 沿著相反路徑回傳到 UI

### 狀態管理

```kotlin
// UI State 範例
data class HomeUiState(
    val timerState: TimerState = TimerState.IDLE,
    val timeRemaining: Long = 25 * 60 * 1000L, // 25分鐘
    val currentMode: TimerMode = TimerMode.FOCUS,
    val currentTask: Task? = null,
    val isLoading: Boolean = false,
    val error: String? = null
)

// Timer State
enum class TimerState {
    IDLE,       // 閒置
    RUNNING,    // 運行中
    PAUSED,     // 暫停
    COMPLETED   // 完成
}

// Timer Mode
enum class TimerMode {
    FOCUS,      // 專注時間 (25分鐘)
    BREAK       // 休息時間 (5分鐘)
}
```

## 🔌 依賴注入架構

使用 **Hilt** 管理依賴關係：

```kotlin
// Application 模組
@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    
    @Provides
    @Singleton
    fun provideAppDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "pomodoro_database"
        ).build()
    }
}

// Repository 模組
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    
    @Binds
    abstract fun bindTaskRepository(
        taskRepositoryImpl: TaskRepositoryImpl
    ): TaskRepository
}
```

## 🧪 測試架構

### 測試金字塔

```
        🔺 E2E Tests (少量)
       🔺🔺 Integration Tests (適量)
      🔺🔺🔺 Unit Tests (大量)
```

### 測試分層

1. **Unit Tests**
   - ViewModel 測試
   - Use Case 測試
   - Repository 測試

2. **Integration Tests**
   - 資料庫測試
   - API 測試

3. **UI Tests**
   - 畫面互動測試
   - 導航測試

## 💡 設計原則實作

### 1. 依賴反轉原則 (DIP)

```kotlin
// ❌ 錯誤：直接依賴具體實作
class HomeViewModel {
    private val database = AppDatabase.getInstance()
}

// ✅ 正確：依賴抽象介面
class HomeViewModel @Inject constructor(
    private val taskRepository: TaskRepository
) {
    // 實作
}
```

### 2. 單一職責原則 (SRP)

```kotlin
// ❌ 錯誤：職責混雜
class TaskManager {
    fun addTask() { /* 新增任務 */ }
    fun saveToDatabase() { /* 存儲資料 */ }
    fun sendNotification() { /* 發送通知 */ }
}

// ✅ 正確：職責分離
class AddTaskUseCase { /* 只負責新增任務邏輯 */ }
class TaskRepository { /* 只負責資料存取 */ }
class NotificationManager { /* 只負責通知管理 */ }
```

### 3. 開放封閉原則 (OCP)

```kotlin
// 可擴展的 Repository 設計
interface TaskRepository {
    suspend fun getTasks(): Flow<List<Task>>
}

class LocalTaskRepository : TaskRepository { /* 本地實作 */ }
class RemoteTaskRepository : TaskRepository { /* 遠端實作 */ }
class CacheTaskRepository : TaskRepository { /* 快取實作 */ }
```

## 📊 架構優勢

### 1. 可維護性
- 各層職責清晰
- 修改影響範圍有限
- 程式碼組織有序

### 2. 可測試性
- 依賴可注入和模擬
- 業務邏輯獨立
- 測試覆蓋率高

### 3. 可擴展性
- 新功能易於添加
- 不同實作可替換
- 支援團隊協作

### 4. 可重用性
- 業務邏輯可重用
- 元件模組化
- 跨平台潛力

## 🎯 實作重點

### ViewModel 設計模式

```kotlin
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val startTimerUseCase: StartTimerUseCase,
    private val pauseTimerUseCase: PauseTimerUseCase,
    private val resetTimerUseCase: ResetTimerUseCase
) : ViewModel() {
    
    private val _uiState = MutableStateFlow(HomeUiState())
    val uiState: StateFlow<HomeUiState> = _uiState.asStateFlow()
    
    fun startTimer() {
        viewModelScope.launch {
            startTimerUseCase()
                .onSuccess { 
                    _uiState.update { it.copy(timerState = TimerState.RUNNING) }
                }
                .onFailure { error ->
                    _uiState.update { it.copy(error = error.message) }
                }
        }
    }
}
```

## 📚 延伸閱讀

- [Clean Architecture by Robert Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Android Architecture Guidelines](https://developer.android.com/topic/architecture)
- [MVVM Pattern in Android](https://developer.android.com/topic/libraries/architecture/viewmodel)

## 🎯 下一章預告

在下一章中，我們將深入探討 **UI 層實作**，學習：

- Jetpack Compose 進階技巧
- Material Design 3 應用
- 導航系統設計
- 複雜 UI 狀態管理

準備好建構美觀且功能強大的用戶介面了嗎？🎨