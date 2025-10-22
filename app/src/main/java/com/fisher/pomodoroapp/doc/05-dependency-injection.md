# 第 5 章：依賴注入

## 📋 本章目標

- 深入理解依賴注入原理和優勢
- 掌握 Hilt 框架的進階使用
- 設計模組化的依賴注入架構
- 實作測試友好的依賴管理

## 🔌 依賴注入概述

### 什麼是依賴注入？

依賴注入（Dependency Injection, DI）是一種設計模式，通過外部容器來提供物件所需的依賴，而不是在物件內部創建。

### 優勢

1. **解耦合**：降低類別間的耦合度
2. **可測試性**：容易進行單元測試
3. **可配置性**：靈活配置不同實作
4. **可維護性**：便於修改和擴展

### 依賴注入類型

```kotlin
// ❌ 沒有依賴注入 - 緊耦合
class TaskViewModel {
    private val repository = TaskRepositoryImpl() // 直接創建依賴
}

// ✅ 構造器注入
class TaskViewModel @Inject constructor(
    private val repository: TaskRepository // 注入介面
)

// ✅ 屬性注入
class TaskViewModel {
    @Inject
    lateinit var repository: TaskRepository
}

// ✅ 方法注入
class TaskViewModel {
    private lateinit var repository: TaskRepository
    
    @Inject
    fun setRepository(repository: TaskRepository) {
        this.repository = repository
    }
}
```

## 🏗️ Hilt 框架深入

### Application 設置

```kotlin
// MyApp.kt
@HiltAndroidApp
class MyApp : Application() {
    
    override fun onCreate() {
        super.onCreate()
        
        // 可以在這裡進行全域初始化
        if (BuildConfig.DEBUG) {
            Timber.plant(Timber.DebugTree())
        }
    }
}
```

### Activity 和 Fragment 注入

```kotlin
// MainActivity.kt
@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    
    @Inject
    lateinit var batteryOptimizationHelper: BatteryOptimizationHelper
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // 檢查電池優化設定
        if (!batteryOptimizationHelper.isIgnoringBatteryOptimizations()) {
            batteryOptimizationHelper.requestIgnoreBatteryOptimizations(this)
        }
        
        setContent {
            PomodoroAppTheme {
                PomodoroApp()
            }
        }
    }
}

// 如果使用 Fragment
@AndroidEntryPoint
class SettingsFragment : Fragment() {
    
    @Inject
    lateinit var configurationRepository: ConfigurationRepository
    
    // Fragment 實作...
}
```

## 📦 模組設計

### 資料庫模組

```kotlin
// di/DatabaseModule.kt
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    
    @Provides
    @Singleton
    fun provideAppDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            AppDatabase.DATABASE_NAME
        )
        .addMigrations(MIGRATION_1_2, MIGRATION_2_3) // 資料庫遷移
        .fallbackToDestructiveMigration() // 僅開發階段使用
        .build()
    }
    
    @Provides
    fun provideTaskDao(database: AppDatabase): TaskDao = database.taskDao()
    
    @Provides
    fun provideSessionDao(database: AppDatabase): SessionDao = database.sessionDao()
    
    @Provides
    fun provideConfigurationDao(database: AppDatabase): ConfigurationDao = database.configurationDao()
    
    // 資料庫遷移定義
    companion object {
        private val MIGRATION_1_2 = object : Migration(1, 2) {
            override fun migrate(database: SupportSQLiteDatabase) {
                database.execSQL("ALTER TABLE tasks ADD COLUMN priority INTEGER NOT NULL DEFAULT 2")
            }
        }
        
        private val MIGRATION_2_3 = object : Migration(2, 3) {
            override fun migrate(database: SupportSQLiteDatabase) {
                database.execSQL("""
                    CREATE TABLE IF NOT EXISTS sessions_backup (
                        id INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL,
                        task_id INTEGER,
                        session_type TEXT NOT NULL,
                        start_time INTEGER NOT NULL,
                        end_time INTEGER,
                        duration_millis INTEGER NOT NULL,
                        is_completed INTEGER NOT NULL,
                        notes TEXT NOT NULL,
                        interruptions INTEGER NOT NULL DEFAULT 0
                    )
                """.trimIndent())
                
                database.execSQL("""
                    INSERT INTO sessions_backup SELECT 
                        id, task_id, session_type, start_time, end_time, 
                        duration_millis, is_completed, notes, 0 as interruptions 
                    FROM sessions
                """.trimIndent())
                
                database.execSQL("DROP TABLE sessions")
                database.execSQL("ALTER TABLE sessions_backup RENAME TO sessions")
            }
        }
    }
}
```

### Repository 模組

```kotlin
// di/RepositoryModule.kt
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    
    @Binds
    abstract fun bindTaskRepository(
        taskRepositoryImpl: TaskRepositoryImpl
    ): TaskRepository
    
    @Binds
    abstract fun bindSessionRepository(
        sessionRepositoryImpl: SessionRepositoryImpl
    ): SessionRepository
    
    @Binds
    abstract fun bindTimerRepository(
        timerRepositoryImpl: TimerRepositoryImpl
    ): TimerRepository
    
    @Binds
    abstract fun bindConfigurationRepository(
        configurationRepositoryImpl: ConfigurationRepositoryImpl
    ): ConfigurationRepository
    
    @Binds
    abstract fun bindStatisticsRepository(
        statisticsRepositoryImpl: StatisticsRepositoryImpl
    ): StatisticsRepository
}
```

### 網路模組

```kotlin
// di/NetworkModule.kt
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(HttpLoggingInterceptor().apply {
                level = if (BuildConfig.DEBUG) {
                    HttpLoggingInterceptor.Level.BODY
                } else {
                    HttpLoggingInterceptor.Level.NONE
                }
            })
            .addInterceptor(AuthInterceptor()) // 自定義認證攔截器
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .writeTimeout(30, TimeUnit.SECONDS)
            .build()
    }
    
    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl(BuildConfig.API_BASE_URL)
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create(
                GsonBuilder()
                    .setDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSS'Z'")
                    .create()
            ))
            .build()
    }
    
    @Provides
    @Singleton
    fun provideApiService(retrofit: Retrofit): ApiService {
        return retrofit.create(ApiService::class.java)
    }
}

// 自定義攔截器
class AuthInterceptor @Inject constructor(
    private val tokenManager: TokenManager
) : Interceptor {
    
    override fun intercept(chain: Interceptor.Chain): Response {
        val originalRequest = chain.request()
        
        val token = tokenManager.getAccessToken()
        if (token.isNullOrBlank()) {
            return chain.proceed(originalRequest)
        }
        
        val authenticatedRequest = originalRequest.newBuilder()
            .header("Authorization", "Bearer $token")
            .build()
        
        return chain.proceed(authenticatedRequest)
    }
}
```

### 服務模組

```kotlin
// di/ServiceModule.kt
@Module
@InstallIn(SingletonComponent::class)
object ServiceModule {
    
    @Provides
    @Singleton
    fun provideNotificationManager(@ApplicationContext context: Context): NotificationManager {
        return context.getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
    }
    
    @Provides
    @Singleton
    fun provideAudioManager(@ApplicationContext context: Context): AudioManager {
        return context.getSystemService(Context.AUDIO_SERVICE) as AudioManager
    }
    
    @Provides
    @Singleton
    fun provideVibrator(@ApplicationContext context: Context): Vibrator {
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
            val vibratorManager = context.getSystemService(Context.VIBRATOR_MANAGER_SERVICE) as VibratorManager
            vibratorManager.defaultVibrator
        } else {
            @Suppress("DEPRECATION")
            context.getSystemService(Context.VIBRATOR_SERVICE) as Vibrator
        }
    }
    
    @Provides
    @Singleton
    fun provideConnectivityManager(@ApplicationContext context: Context): ConnectivityManager {
        return context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
    }
    
    @Provides
    @Singleton
    fun providePowerManager(@ApplicationContext context: Context): PowerManager {
        return context.getSystemService(Context.POWER_SERVICE) as PowerManager
    }
}
```

### 工具模組

```kotlin
// di/UtilModule.kt
@Module
@InstallIn(SingletonComponent::class)
object UtilModule {
    
    @Provides
    @Singleton
    fun provideCoroutineDispatchers(): CoroutineDispatchers {
        return CoroutineDispatchers(
            io = Dispatchers.IO,
            main = Dispatchers.Main,
            default = Dispatchers.Default,
            unconfined = Dispatchers.Unconfined
        )
    }
    
    @Provides
    @Singleton
    fun provideDateHelper(): DateHelper = DateHelperImpl()
    
    @Provides
    @Singleton
    fun providePreferences(@ApplicationContext context: Context): SharedPreferences {
        return context.getSharedPreferences("pomodoro_prefs", Context.MODE_PRIVATE)
    }
    
    @Provides
    @Singleton
    fun provideGson(): Gson {
        return GsonBuilder()
            .setDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSS'Z'")
            .setPrettyPrinting()
            .create()
    }
}

// 協程調度器封裝
data class CoroutineDispatchers(
    val io: CoroutineDispatcher,
    val main: CoroutineDispatcher,
    val default: CoroutineDispatcher,
    val unconfined: CoroutineDispatcher
)
```

## 🎯 限定符（Qualifiers）

### 自定義限定符

```kotlin
// di/Qualifiers.kt
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class LocalDataSource

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class RemoteDataSource

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class CacheDataSource

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class DefaultDispatcher

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class IoDispatcher

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class MainDispatcher
```

### 限定符使用

```kotlin
// di/DataSourceModule.kt
@Module
@InstallIn(SingletonComponent::class)
abstract class DataSourceModule {
    
    @Binds
    @LocalDataSource
    abstract fun bindLocalTaskDataSource(
        localTaskDataSource: LocalTaskDataSource
    ): TaskDataSource
    
    @Binds
    @RemoteDataSource
    abstract fun bindRemoteTaskDataSource(
        remoteTaskDataSource: RemoteTaskDataSource
    ): TaskDataSource
    
    @Binds
    @CacheDataSource
    abstract fun bindCacheTaskDataSource(
        cacheTaskDataSource: CacheTaskDataSource
    ): TaskDataSource
}

// di/DispatcherModule.kt
@Module
@InstallIn(SingletonComponent::class)
object DispatcherModule {
    
    @Provides
    @DefaultDispatcher
    fun provideDefaultDispatcher(): CoroutineDispatcher = Dispatchers.Default
    
    @Provides
    @IoDispatcher
    fun provideIoDispatcher(): CoroutineDispatcher = Dispatchers.IO
    
    @Provides
    @MainDispatcher
    fun provideMainDispatcher(): CoroutineDispatcher = Dispatchers.Main
}

// Repository 中使用
@Singleton
class TaskRepositoryImpl @Inject constructor(
    @LocalDataSource private val localDataSource: TaskDataSource,
    @RemoteDataSource private val remoteDataSource: TaskDataSource,
    @CacheDataSource private val cacheDataSource: TaskDataSource,
    @IoDispatcher private val ioDispatcher: CoroutineDispatcher
) : TaskRepository {
    
    override fun getAllTasks(): Flow<List<Task>> = flow {
        // 先從快取獲取
        val cachedTasks = cacheDataSource.getTasks()
        emit(cachedTasks)
        
        // 從本地資料庫獲取
        val localTasks = localDataSource.getTasks()
        if (localTasks != cachedTasks) {
            cacheDataSource.cacheTasks(localTasks)
            emit(localTasks)
        }
        
        // 從遠端同步（如果有網路）
        try {
            val remoteTasks = remoteDataSource.getTasks()
            if (remoteTasks != localTasks) {
                localDataSource.saveTasks(remoteTasks)
                cacheDataSource.cacheTasks(remoteTasks)
                emit(remoteTasks)
            }
        } catch (e: Exception) {
            // 網路錯誤，使用本地資料
        }
    }.flowOn(ioDispatcher)
}
```

## 🔧 作用域管理

### Hilt 作用域

```kotlin
// Application 級別 - 整個應用程式生命週期
@Singleton
class GlobalConfigurationManager @Inject constructor(
    private val preferences: SharedPreferences
)

// Activity 級別 - Activity 生命週期
@ActivityScoped
class NavigationManager @Inject constructor(
    private val activity: Activity
)

// Service 級別 - Service 生命週期
@ServiceScoped
class TimerStateManager @Inject constructor()

// ViewModel 級別 - ViewModel 生命週期
@ViewModelScoped
class TimerUiStateManager @Inject constructor()

// Fragment 級別 - Fragment 生命週期
@FragmentScoped
class FragmentSpecificManager @Inject constructor()
```

### 自定義作用域

```kotlin
// di/CustomScopes.kt
@Scope
@MustBeDocumented
@Retention(AnnotationRetention.RUNTIME)
annotation class UserSessionScope

// di/UserSessionModule.kt
@Module
@InstallIn(SingletonComponent::class)
object UserSessionModule {
    
    @Provides
    @UserSessionScope
    fun provideUserSessionManager(
        userRepository: UserRepository,
        tokenManager: TokenManager
    ): UserSessionManager {
        return UserSessionManager(userRepository, tokenManager)
    }
}
```

## 🧪 測試中的依賴注入

### 測試模組

```kotlin
// test/di/TestDatabaseModule.kt
@Module
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [DatabaseModule::class]
)
object TestDatabaseModule {
    
    @Provides
    @Singleton
    fun provideTestDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.inMemoryDatabaseBuilder(
            context,
            AppDatabase::class.java
        )
        .allowMainThreadQueries()
        .build()
    }
}

// test/di/TestRepositoryModule.kt
@Module
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [RepositoryModule::class]
)
abstract class TestRepositoryModule {
    
    @Binds
    abstract fun bindTaskRepository(
        fakeTaskRepository: FakeTaskRepository
    ): TaskRepository
    
    @Binds
    abstract fun bindSessionRepository(
        fakeSessionRepository: FakeSessionRepository
    ): SessionRepository
}
```

### 假實作（Fake Implementations）

```kotlin
// test/repository/FakeTaskRepository.kt
@Singleton
class FakeTaskRepository @Inject constructor() : TaskRepository {
    
    private val tasks = mutableListOf<Task>()
    private var currentTaskId: Long? = null
    private var nextId = 1L
    
    private val _tasksFlow = MutableStateFlow(tasks.toList())
    
    override fun getAllTasks(): Flow<List<Task>> = _tasksFlow.asStateFlow()
    
    override fun getActiveTasks(): Flow<List<Task>> = _tasksFlow.map { tasks ->
        tasks.filter { !it.isCompleted }
    }
    
    override fun getCurrentTask(): Flow<Task?> = _tasksFlow.map { tasks ->
        currentTaskId?.let { id -> tasks.find { it.id == id } }
    }
    
    override suspend fun getTaskById(id: Long): Task? {
        return tasks.find { it.id == id }
    }
    
    override suspend fun insertTask(task: Task): Long {
        val newTask = task.copy(id = nextId++)
        tasks.add(newTask)
        _tasksFlow.value = tasks.toList()
        return newTask.id
    }
    
    override suspend fun updateTask(task: Task) {
        val index = tasks.indexOfFirst { it.id == task.id }
        if (index != -1) {
            tasks[index] = task
            _tasksFlow.value = tasks.toList()
        }
    }
    
    override suspend fun deleteTask(taskId: Long) {
        tasks.removeAll { it.id == taskId }
        if (currentTaskId == taskId) {
            currentTaskId = null
        }
        _tasksFlow.value = tasks.toList()
    }
    
    override suspend fun setCurrentTask(taskId: Long) {
        // 清除之前的當前任務
        tasks.forEachIndexed { index, task ->
            if (task.isCurrent) {
                tasks[index] = task.copy(isCurrent = false)
            }
        }
        
        // 設置新的當前任務
        val index = tasks.indexOfFirst { it.id == taskId }
        if (index != -1) {
            tasks[index] = tasks[index].copy(isCurrent = true)
            currentTaskId = taskId
        }
        
        _tasksFlow.value = tasks.toList()
    }
    
    override suspend fun incrementPomodoroCount(taskId: Long) {
        val index = tasks.indexOfFirst { it.id == taskId }
        if (index != -1) {
            tasks[index] = tasks[index].copy(
                completedPomodoros = tasks[index].completedPomodoros + 1
            )
            _tasksFlow.value = tasks.toList()
        }
    }
    
    // 測試用的輔助方法
    fun addTask(task: Task) {
        tasks.add(task)
        _tasksFlow.value = tasks.toList()
    }
    
    fun clearAllTasks() {
        tasks.clear()
        currentTaskId = null
        _tasksFlow.value = emptyList()
    }
    
    fun getTaskCount(): Int = tasks.size
}
```

### 測試中使用

```kotlin
// test/ui/home/HomeViewModelTest.kt
@HiltAndroidTest
@ExperimentalCoroutinesTest
class HomeViewModelTest {
    
    @get:Rule
    val hiltRule = HiltAndroidRule(this)
    
    @get:Rule
    val instantExecutorRule = InstantTaskExecutorRule()
    
    @Inject
    lateinit var fakeTaskRepository: FakeTaskRepository
    
    @Inject
    lateinit var fakeSessionRepository: FakeSessionRepository
    
    private lateinit var viewModel: HomeViewModel
    
    @Before
    fun setup() {
        hiltRule.inject()
        
        // 清理測試資料
        fakeTaskRepository.clearAllTasks()
        fakeSessionRepository.clearAllSessions()
        
        viewModel = HomeViewModel(
            // 注入依賴...
        )
    }
    
    @Test
    fun `should update current task when task is selected`() = runTest {
        // Given
        val task = Task(id = 1, name = "Test Task")
        fakeTaskRepository.addTask(task)
        
        // When
        viewModel.setCurrentTask(1)
        
        // Then
        val uiState = viewModel.uiState.value
        assertThat(uiState.currentTask?.id).isEqualTo(1)
    }
}
```

## 🔧 進階技巧

### 條件式注入

```kotlin
// di/ConditionalModule.kt
@Module
@InstallIn(SingletonComponent::class)
object ConditionalModule {
    
    @Provides
    @Singleton
    fun provideAnalyticsManager(
        @ApplicationContext context: Context
    ): AnalyticsManager {
        return if (BuildConfig.DEBUG) {
            NoOpAnalyticsManager() // 開發環境不收集分析資料
        } else {
            FirebaseAnalyticsManager(context) // 生產環境使用 Firebase
        }
    }
    
    @Provides
    @Singleton
    fun provideLogger(): Logger {
        return if (BuildConfig.DEBUG) {
            ConsoleLogger() // 開發環境輸出到控制台
        } else {
            CrashlyticsLogger() // 生產環境記錄到 Crashlytics
        }
    }
}
```

### 動態模組

```kotlin
// di/DynamicModule.kt
@Module
@InstallIn(SingletonComponent::class)
object DynamicModule {
    
    @Provides
    @Singleton
    fun provideImageLoader(@ApplicationContext context: Context): ImageLoader {
        return ImageLoader.Builder(context)
            .components {
                if (Build.VERSION.SDK_INT >= 28) {
                    add(ImageDecoderDecoder.Factory())
                } else {
                    add(GifDecoder.Factory())
                }
            }
            .memoryCache {
                MemoryCache.Builder(context)
                    .maxSizePercent(0.25)
                    .build()
            }
            .diskCache {
                DiskCache.Builder()
                    .directory(context.cacheDir.resolve("image_cache"))
                    .maxSizeBytes(50 * 1024 * 1024) // 50MB
                    .build()
            }
            .build()
    }
}
```

### 多綁定（Multibinding）

```kotlin
// di/MultibindingModule.kt
@Module
@InstallIn(SingletonComponent::class)
abstract class MultibindingModule {
    
    @Binds
    @IntoSet
    abstract fun bindTaskValidator(
        taskNameValidator: TaskNameValidator
    ): TaskValidator
    
    @Binds
    @IntoSet
    abstract fun bindPomodoroValidator(
        pomodoroCountValidator: PomodoroCountValidator
    ): TaskValidator
    
    @Binds
    @IntoMap
    @StringKey("focus")
    abstract fun bindFocusSessionHandler(
        focusSessionHandler: FocusSessionHandler
    ): SessionHandler
    
    @Binds
    @IntoMap
    @StringKey("break")
    abstract fun bindBreakSessionHandler(
        breakSessionHandler: BreakSessionHandler
    ): SessionHandler
}

// 使用多綁定
@Singleton
class CompositeTaskValidator @Inject constructor(
    private val validators: Set<@JvmSuppressWildcards TaskValidator>
) {
    
    fun validate(task: Task): Result<Unit> {
        validators.forEach { validator ->
            validator.validate(task).onFailure { return Result.failure(it) }
        }
        return Result.success(Unit)
    }
}

@Singleton
class SessionHandlerFactory @Inject constructor(
    private val handlers: Map<String, @JvmSuppressWildcards SessionHandler>
) {
    
    fun getHandler(sessionType: String): SessionHandler {
        return handlers[sessionType] ?: throw IllegalArgumentException("Unknown session type: $sessionType")
    }
}
```

## 📊 效能優化

### 延遲初始化

```kotlin
// di/LazyModule.kt
@Module
@InstallIn(SingletonComponent::class)
object LazyModule {
    
    @Provides
    @Singleton
    fun provideExpensiveService(): Lazy<ExpensiveService> {
        return lazy { ExpensiveServiceImpl() }
    }
}

// 使用延遲初始化
@Singleton
class ServiceManager @Inject constructor(
    private val expensiveService: Lazy<ExpensiveService>
) {
    
    fun doExpensiveOperation() {
        // 只有在實際需要時才會創建 ExpensiveService
        expensiveService.value.performOperation()
    }
}
```

### Provider 注入

```kotlin
@Singleton
class ViewModelFactory @Inject constructor(
    private val homeViewModelProvider: Provider<HomeViewModel>,
    private val taskViewModelProvider: Provider<TaskListViewModel>
) {
    
    fun createHomeViewModel(): HomeViewModel = homeViewModelProvider.get()
    fun createTaskViewModel(): TaskListViewModel = taskViewModelProvider.get()
}
```

## 📚 延伸閱讀

- [Hilt 官方文件](https://developer.android.com/training/dependency-injection/hilt-android)
- [Dependency Injection Principles](https://en.wikipedia.org/wiki/Dependency_injection)
- [Dagger 2 進階使用](https://dagger.dev/dev-guide/)

## 🎯 下一章預告

在下一章中，我們將探討 **測試實作**，學習：

- 單元測試設計原則
- ViewModel 和 Repository 測試
- UI 測試自動化
- 測試覆蓋率優化

準備好建構可靠的測試體系了嗎？🧪