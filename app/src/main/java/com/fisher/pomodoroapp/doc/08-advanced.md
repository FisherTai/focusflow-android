# 第 8 章：進階主題

## 📋 本章目標

- 掌握 Android 應用效能優化技巧
- 實作國際化和本地化支援
- 建構無障礙友善的用戶體驗
- 探索進階架構模式和最佳實踐

## ⚡ 效能優化

### Jetpack Compose 效能優化

#### 重組（Recomposition）優化

```kotlin
// ❌ 不良實踐 - 不穩定的參數
@Composable
fun TaskItem(task: Task, onClick: () -> Unit) {
    // 每次重組都會創建新的 lambda
    Row(
        modifier = Modifier.clickable { onClick() }
    ) {
        Text(task.name)
    }
}

// ✅ 良好實踐 - 穩定的參數
@Composable
fun TaskItem(
    task: Task, 
    onClick: () -> Unit,
    modifier: Modifier = Modifier
) {
    // 使用 remember 避免不必要的重組
    val clickModifier = remember(onClick) {
        Modifier.clickable { onClick() }
    }
    
    Row(
        modifier = modifier.then(clickModifier)
    ) {
        Text(task.name)
    }
}

// 進階優化 - 使用 @Stable 和 @Immutable
@Immutable
data class TaskUIData(
    val id: Long,
    val name: String,
    val description: String,
    val isCompleted: Boolean,
    val completionPercentage: Float
)

@Stable
class TimerUiState(
    timeRemaining: Long,
    timerState: TimerState,
    currentMode: TimerMode
) {
    var timeRemaining by mutableStateOf(timeRemaining)
        private set
        
    var timerState by mutableStateOf(timerState)
        private set
        
    var currentMode by mutableStateOf(currentMode)
        private set
    
    fun updateTime(newTime: Long) {
        timeRemaining = newTime
    }
    
    fun updateState(newState: TimerState) {
        timerState = newState
    }
}
```

#### LazyList 效能優化

```kotlin
// LazyColumn 效能優化
@Composable
fun OptimizedTaskList(
    tasks: List<TaskUIData>,
    onTaskClick: (Long) -> Unit,
    modifier: Modifier = Modifier
) {
    LazyColumn(
        modifier = modifier,
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(
            items = tasks,
            key = { task -> task.id }, // 重要：提供穩定的 key
            contentType = { "task_item" } // 幫助 Compose 優化回收
        ) { task ->
            // 使用 remember 避免重複計算
            val onClick = remember(task.id, onTaskClick) {
                { onTaskClick(task.id) }
            }
            
            TaskItem(
                task = task,
                onClick = onClick,
                modifier = Modifier.animateItemPlacement() // 動畫優化
            )
        }
    }
}

// 大型清單的分頁載入
@Composable
fun PaginatedTaskList(
    lazyPagingItems: LazyPagingItems<TaskUIData>,
    onTaskClick: (Long) -> Unit
) {
    LazyColumn {
        items(
            count = lazyPagingItems.itemCount,
            key = lazyPagingItems.itemKey { it.id }
        ) { index ->
            val task = lazyPagingItems[index]
            if (task != null) {
                TaskItem(
                    task = task,
                    onClick = { onTaskClick(task.id) }
                )
            } else {
                // 載入中的占位符
                TaskItemPlaceholder()
            }
        }
    }
}
```

### 記憶體優化

#### 圖片載入優化

```kotlin
// util/ImageLoader.kt
@Singleton
class OptimizedImageLoader @Inject constructor(
    @ApplicationContext private val context: Context
) {
    
    val imageLoader: ImageLoader by lazy {
        ImageLoader.Builder(context)
            .memoryCache {
                MemoryCache.Builder(context)
                    .maxSizePercent(0.25) // 使用 25% 的可用記憶體
                    .strongReferencesEnabled(true)
                    .build()
            }
            .diskCache {
                DiskCache.Builder()
                    .directory(context.cacheDir.resolve("image_cache"))
                    .maxSizeBytes(50 * 1024 * 1024) // 50MB
                    .build()
            }
            .respectCacheHeaders(false)
            .build()
    }
}

// 使用優化的圖片載入
@Composable
fun OptimizedAsyncImage(
    imageUrl: String,
    contentDescription: String?,
    modifier: Modifier = Modifier
) {
    val imageLoader = LocalContext.current.imageLoader
    
    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(imageUrl)
            .crossfade(true)
            .size(Size.ORIGINAL) // 原始尺寸，避免不必要的縮放
            .memoryCachePolicy(CachePolicy.ENABLED)
            .diskCachePolicy(CachePolicy.ENABLED)
            .build(),
        contentDescription = contentDescription,
        imageLoader = imageLoader,
        modifier = modifier,
        loading = {
            Box(
                modifier = Modifier.fillMaxSize(),
                contentAlignment = Alignment.Center
            ) {
                CircularProgressIndicator()
            }
        },
        error = {
            Box(
                modifier = Modifier.fillMaxSize(),
                contentAlignment = Alignment.Center
            ) {
                Icon(
                    imageVector = Icons.Default.Error,
                    contentDescription = "載入失敗"
                )
            }
        }
    )
}
```

#### 資料庫查詢優化

```kotlin
// dao/OptimizedTaskDao.kt
@Dao
interface OptimizedTaskDao {
    
    // 使用索引優化查詢
    @Query("""
        SELECT * FROM tasks 
        WHERE is_completed = :completed 
        ORDER BY 
            CASE WHEN :sortBy = 'priority' THEN priority END DESC,
            CASE WHEN :sortBy = 'date' THEN created_at END DESC,
            CASE WHEN :sortBy = 'name' THEN name END ASC
        LIMIT :limit OFFSET :offset
    """)
    suspend fun getTasksPaginated(
        completed: Boolean = false,
        sortBy: String = "date",
        limit: Int = 20,
        offset: Int = 0
    ): List<TaskEntity>
    
    // 使用 COUNT 查詢而不是載入所有資料
    @Query("SELECT COUNT(*) FROM tasks WHERE is_completed = 0")
    suspend fun getActiveTaskCount(): Int
    
    // 批量操作優化
    @Transaction
    @Query("UPDATE tasks SET completed_pomodoros = completed_pomodoros + 1 WHERE id IN (:taskIds)")
    suspend fun incrementPomodoroCountBatch(taskIds: List<Long>)
    
    // 使用 FTS (Full-Text Search) 進行文字搜尋
    @Query("""
        SELECT tasks.* FROM tasks 
        JOIN tasks_fts ON tasks.id = tasks_fts.rowid 
        WHERE tasks_fts MATCH :query
        ORDER BY rank
    """)
    fun searchTasks(query: String): Flow<List<TaskEntity>>
}

// 資料庫索引定義
@Entity(
    tableName = "tasks",
    indices = [
        Index(value = ["is_completed"]),
        Index(value = ["priority"]),
        Index(value = ["created_at"]),
        Index(value = ["is_current"])
    ]
)
data class TaskEntity(
    // ... 實體定義
)
```

### 背景處理優化

```kotlin
// service/OptimizedTimerService.kt
class OptimizedTimerService : Service() {
    
    private val timerScope = CoroutineScope(
        SupervisorJob() + Dispatchers.Default
    )
    
    // 使用 WakeLock 管理器避免電池消耗
    @Inject
    lateinit var wakeLockManager: WakeLockManager
    
    // 優化的計時器實作
    private fun startOptimizedTimer(durationMillis: Long) {
        timerScope.launch {
            wakeLockManager.acquireWakeLock("timer_wakelock", 30 * 60 * 1000L) // 30分鐘
            
            try {
                var remainingTime = durationMillis
                val tickInterval = 1000L // 1秒
                
                while (remainingTime > 0 && isActive) {
                    delay(tickInterval)
                    remainingTime -= tickInterval
                    
                    // 只在需要時更新 UI
                    if (remainingTime % 1000L == 0L) {
                        updateTimerDisplay(remainingTime)
                    }
                }
                
                if (remainingTime <= 0) {
                    onTimerCompleted()
                }
            } finally {
                wakeLockManager.releaseWakeLock("timer_wakelock")
            }
        }
    }
}

// util/WakeLockManager.kt
@Singleton
class WakeLockManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val powerManager = context.getSystemService(Context.POWER_SERVICE) as PowerManager
    private val wakeLocks = mutableMapOf<String, PowerManager.WakeLock>()
    
    fun acquireWakeLock(tag: String, timeout: Long) {
        val wakeLock = powerManager.newWakeLock(
            PowerManager.PARTIAL_WAKE_LOCK,
            "PomodoroApp:$tag"
        )
        wakeLock.acquire(timeout)
        wakeLocks[tag] = wakeLock
    }
    
    fun releaseWakeLock(tag: String) {
        wakeLocks[tag]?.let { wakeLock ->
            if (wakeLock.isHeld) {
                wakeLock.release()
            }
            wakeLocks.remove(tag)
        }
    }
    
    fun releaseAllWakeLocks() {
        wakeLocks.forEach { (_, wakeLock) ->
            if (wakeLock.isHeld) {
                wakeLock.release()
            }
        }
        wakeLocks.clear()
    }
}
```

## 🌍 國際化和本地化

### 多語言資源配置

```xml
<!-- res/values/strings.xml (繁體中文) -->
<resources>
    <string name="app_name">番茄鐘</string>
    <string name="timer_focus_mode">專注時間</string>
    <string name="timer_break_mode">休息時間</string>
    <string name="button_start">開始</string>
    <string name="button_pause">暫停</string>
    <string name="button_reset">重設</string>
    
    <!-- 複數形式支援 -->
    <plurals name="completed_pomodoros">
        <item quantity="zero">尚未完成任何番茄鐘</item>
        <item quantity="one">完成了 %d 個番茄鐘</item>
        <item quantity="other">完成了 %d 個番茄鐘</item>
    </plurals>
    
    <!-- 格式化字串 -->
    <string name="timer_remaining">剩餘時間：%1$02d:%2$02d</string>
    <string name="task_progress">進度：%1$d/%2$d 個番茄鐘</string>
</resources>

<!-- res/values-en/strings.xml (英文) -->
<resources>
    <string name="app_name">Pomodoro Timer</string>
    <string name="timer_focus_mode">Focus Time</string>
    <string name="timer_break_mode">Break Time</string>
    <string name="button_start">Start</string>
    <string name="button_pause">Pause</string>
    <string name="button_reset">Reset</string>
    
    <plurals name="completed_pomodoros">
        <item quantity="zero">No pomodoros completed</item>
        <item quantity="one">Completed %d pomodoro</item>
        <item quantity="other">Completed %d pomodoros</item>
    </plurals>
    
    <string name="timer_remaining">Time remaining: %1$02d:%2$02d</string>
    <string name="task_progress">Progress: %1$d/%2$d pomodoros</string>
</resources>

<!-- res/values-ja/strings.xml (日文) -->
<resources>
    <string name="app_name">ポモドーロタイマー</string>
    <string name="timer_focus_mode">集中時間</string>
    <string name="timer_break_mode">休憩時間</string>
    <string name="button_start">開始</string>
    <string name="button_pause">一時停止</string>
    <string name="button_reset">リセット</string>
</resources>
```

### 本地化管理器

```kotlin
// util/LocalizationManager.kt
@Singleton
class LocalizationManager @Inject constructor(
    @ApplicationContext private val context: Context,
    private val preferencesHelper: PreferencesHelper
) {
    
    fun getCurrentLocale(): Locale {
        val savedLanguage = preferencesHelper.getLanguage()
        return if (savedLanguage.isNotEmpty()) {
            Locale.forLanguageTag(savedLanguage)
        } else {
            Locale.getDefault()
        }
    }
    
    fun setLocale(languageTag: String) {
        preferencesHelper.setLanguage(languageTag)
        
        val locale = Locale.forLanguageTag(languageTag)
        Locale.setDefault(locale)
        
        val configuration = context.resources.configuration
        configuration.setLocale(locale)
        context.createConfigurationContext(configuration)
    }
    
    fun getSupportedLanguages(): List<SupportedLanguage> {
        return listOf(
            SupportedLanguage("zh-TW", "繁體中文", "🇹🇼"),
            SupportedLanguage("en", "English", "🇺🇸"),
            SupportedLanguage("ja", "日本語", "🇯🇵"),
            SupportedLanguage("ko", "한국어", "🇰🇷")
        )
    }
    
    fun formatDuration(durationMillis: Long): String {
        val minutes = (durationMillis / 1000) / 60
        val seconds = (durationMillis / 1000) % 60
        
        return context.getString(
            R.string.timer_remaining,
            minutes.toInt(),
            seconds.toInt()
        )
    }
    
    fun formatPomodoroCount(completed: Int, total: Int): String {
        return context.getString(R.string.task_progress, completed, total)
    }
    
    fun formatPomodoroCompletion(count: Int): String {
        return context.resources.getQuantityString(
            R.plurals.completed_pomodoros,
            count,
            count
        )
    }
}

data class SupportedLanguage(
    val code: String,
    val name: String,
    val flag: String
)
```

### 日期和時間本地化

```kotlin
// util/DateTimeFormatter.kt
@Singleton
class DateTimeFormatter @Inject constructor(
    private val localizationManager: LocalizationManager
) {
    
    private val dateFormatter by lazy {
        DateTimeFormatter.ofLocalizedDate(FormatStyle.MEDIUM)
            .withLocale(localizationManager.getCurrentLocale())
    }
    
    private val timeFormatter by lazy {
        DateTimeFormatter.ofLocalizedTime(FormatStyle.SHORT)
            .withLocale(localizationManager.getCurrentLocale())
    }
    
    private val dateTimeFormatter by lazy {
        DateTimeFormatter.ofLocalizedDateTime(FormatStyle.MEDIUM, FormatStyle.SHORT)
            .withLocale(localizationManager.getCurrentLocale())
    }
    
    fun formatDate(timestamp: Long): String {
        val instant = Instant.ofEpochMilli(timestamp)
        val localDate = instant.atZone(ZoneId.systemDefault()).toLocalDate()
        return localDate.format(dateFormatter)
    }
    
    fun formatTime(timestamp: Long): String {
        val instant = Instant.ofEpochMilli(timestamp)
        val localTime = instant.atZone(ZoneId.systemDefault()).toLocalTime()
        return localTime.format(timeFormatter)
    }
    
    fun formatDateTime(timestamp: Long): String {
        val instant = Instant.ofEpochMilli(timestamp)
        val localDateTime = instant.atZone(ZoneId.systemDefault()).toLocalDateTime()
        return localDateTime.format(dateTimeFormatter)
    }
    
    fun formatRelativeTime(timestamp: Long): String {
        val now = System.currentTimeMillis()
        val diff = now - timestamp
        
        return when {
            diff < DateUtils.MINUTE_IN_MILLIS -> "剛才"
            diff < DateUtils.HOUR_IN_MILLIS -> "${diff / DateUtils.MINUTE_IN_MILLIS} 分鐘前"
            diff < DateUtils.DAY_IN_MILLIS -> "${diff / DateUtils.HOUR_IN_MILLIS} 小時前"
            diff < DateUtils.WEEK_IN_MILLIS -> "${diff / DateUtils.DAY_IN_MILLIS} 天前"
            else -> formatDate(timestamp)
        }
    }
}
```

## ♿ 無障礙設計

### 無障礙標籤和說明

```kotlin
// ui/components/AccessibleTimerDisplay.kt
@Composable
fun AccessibleTimerDisplay(
    timeRemaining: Long,
    timerState: TimerState,
    currentMode: TimerMode,
    modifier: Modifier = Modifier
) {
    val timeText = formatTime(timeRemaining)
    val modeText = stringResource(
        when (currentMode) {
            TimerMode.FOCUS -> R.string.timer_focus_mode
            TimerMode.BREAK -> R.string.timer_break_mode
        }
    )
    
    val stateText = stringResource(
        when (timerState) {
            TimerState.RUNNING -> R.string.timer_state_running
            TimerState.PAUSED -> R.string.timer_state_paused
            TimerState.IDLE -> R.string.timer_state_idle
            TimerState.COMPLETED -> R.string.timer_state_completed
        }
    )
    
    val contentDescription = stringResource(
        R.string.timer_accessibility_description,
        modeText,
        timeText,
        stateText
    )
    
    Card(
        modifier = modifier
            .semantics {
                contentDescription = contentDescription
                // 為螢幕閱讀器提供更詳細的資訊
                stateDescription = "$modeText，$stateText"
                
                // 設定為可聚焦的重要內容
                heading()
            }
    ) {
        Column(
            modifier = Modifier.padding(24.dp),
            horizontalAlignment = Alignment.CenterHorizontally
        ) {
            Text(
                text = modeText,
                style = MaterialTheme.typography.titleMedium,
                modifier = Modifier.semantics {
                    // 隱藏此文字，因為已包含在父容器的描述中
                    invisibleToUser()
                }
            )
            
            Text(
                text = timeText,
                style = MaterialTheme.typography.displayLarge,
                modifier = Modifier.semantics {
                    // 時間變化時發出通知
                    liveRegion = LiveRegionMode.Polite
                }
            )
        }
    }
}

// 無障礙按鈕設計
@Composable
fun AccessibleTimerButton(
    text: String,
    onClick: () -> Unit,
    enabled: Boolean = true,
    modifier: Modifier = Modifier
) {
    val hapticFeedback = LocalHapticFeedback.current
    
    Button(
        onClick = {
            hapticFeedback.performHapticFeedback(HapticFeedbackType.LongPress)
            onClick()
        },
        enabled = enabled,
        modifier = modifier
            .semantics {
                // 提供更詳細的按鈕狀態說明
                if (!enabled) {
                    stateDescription = "按鈕目前無法使用"
                }
                
                // 設定最小觸控目標大小 (48dp)
                role = Role.Button
            }
            .sizeIn(minWidth = 48.dp, minHeight = 48.dp)
    ) {
        Text(text)
    }
}
```

### 顏色對比和視覺輔助

```kotlin
// ui/theme/AccessibilityTheme.kt
@Composable
fun AccessibilityAwareTheme(
    content: @Composable () -> Unit
) {
    val context = LocalContext.current
    val configuration = LocalConfiguration.current
    
    // 檢查系統設定
    val isHighContrastEnabled = remember {
        val accessibilityManager = context.getSystemService(Context.ACCESSIBILITY_SERVICE) as AccessibilityManager
        accessibilityManager.isHighTextContrastEnabled
    }
    
    val isDarkMode = isSystemInDarkTheme()
    val fontScale = configuration.fontScale
    
    // 根據無障礙設定調整顏色
    val adjustedColorScheme = when {
        isHighContrastEnabled && isDarkMode -> createHighContrastDarkColors()
        isHighContrastEnabled && !isDarkMode -> createHighContrastLightColors()
        isDarkMode -> DarkColors
        else -> LightColors
    }
    
    // 根據字體縮放調整排版
    val adjustedTypography = when {
        fontScale >= 1.3f -> createLargeTextTypography()
        fontScale >= 1.15f -> createMediumTextTypography()
        else -> Typography
    }
    
    MaterialTheme(
        colorScheme = adjustedColorScheme,
        typography = adjustedTypography,
        content = content
    )
}

private fun createHighContrastDarkColors(): ColorScheme {
    return darkColorScheme(
        primary = Color(0xFFFFFFFF),
        onPrimary = Color(0xFF000000),
        primaryContainer = Color(0xFF333333),
        onPrimaryContainer = Color(0xFFFFFFFF),
        secondary = Color(0xFFFFFF00),
        onSecondary = Color(0xFF000000),
        error = Color(0xFFFF0000),
        onError = Color(0xFFFFFFFF),
        background = Color(0xFF000000),
        onBackground = Color(0xFFFFFFFF),
        surface = Color(0xFF111111),
        onSurface = Color(0xFFFFFFFF)
    )
}

// 動態字體大小支援
@Composable
fun DynamicText(
    text: String,
    style: TextStyle,
    modifier: Modifier = Modifier
) {
    val configuration = LocalConfiguration.current
    val scaleFactor = configuration.fontScale
    
    val adjustedStyle = when {
        scaleFactor >= 1.3f -> style.copy(
            fontSize = style.fontSize * 1.1f,
            lineHeight = style.lineHeight * 1.15f
        )
        scaleFactor >= 1.15f -> style.copy(
            lineHeight = style.lineHeight * 1.1f
        )
        else -> style
    }
    
    Text(
        text = text,
        style = adjustedStyle,
        modifier = modifier
    )
}
```

### 鍵盤導航支援

```kotlin
// ui/components/KeyboardNavigableTaskList.kt
@Composable
fun KeyboardNavigableTaskList(
    tasks: List<TaskUIData>,
    onTaskSelected: (Long) -> Unit,
    modifier: Modifier = Modifier
) {
    var focusedIndex by remember { mutableStateOf(0) }
    val focusRequester = remember { FocusRequester() }
    
    LazyColumn(
        modifier = modifier
            .focusRequester(focusRequester)
            .onKeyEvent { keyEvent ->
                when (keyEvent.key) {
                    Key.DirectionDown -> {
                        if (focusedIndex < tasks.size - 1) {
                            focusedIndex++
                        }
                        true
                    }
                    Key.DirectionUp -> {
                        if (focusedIndex > 0) {
                            focusedIndex--
                        }
                        true
                    }
                    Key.Enter, Key.NumPadEnter -> {
                        if (tasks.isNotEmpty()) {
                            onTaskSelected(tasks[focusedIndex].id)
                        }
                        true
                    }
                    else -> false
                }
            }
    ) {
        itemsIndexed(tasks) { index, task ->
            TaskItem(
                task = task,
                isSelected = index == focusedIndex,
                onClick = { onTaskSelected(task.id) },
                modifier = Modifier
                    .focusable()
                    .background(
                        if (index == focusedIndex) {
                            MaterialTheme.colorScheme.primaryContainer
                        } else {
                            Color.Transparent
                        }
                    )
            )
        }
    }
    
    // 自動聚焦到列表
    LaunchedEffect(Unit) {
        focusRequester.requestFocus()
    }
}
```

## 🔧 進階架構模式

### MVI (Model-View-Intent) 模式

```kotlin
// ui/mvi/MviViewModel.kt
abstract class MviViewModel<I : UiIntent, S : UiState, E : UiEffect> : ViewModel() {
    
    private val initialState: S by lazy { createInitialState() }
    abstract fun createInitialState(): S
    
    private val _uiState: MutableStateFlow<S> = MutableStateFlow(initialState)
    val uiState: StateFlow<S> = _uiState.asStateFlow()
    
    private val _uiEffect: MutableSharedFlow<E> = MutableSharedFlow()
    val uiEffect: SharedFlow<E> = _uiEffect.asSharedFlow()
    
    private val intentFlow = MutableSharedFlow<I>()
    
    init {
        subscribeToIntents()
    }
    
    private fun subscribeToIntents() {
        viewModelScope.launch {
            intentFlow
                .distinctUntilChanged()
                .collect { intent ->
                    handleIntent(intent)
                }
        }
    }
    
    fun sendIntent(intent: I) {
        viewModelScope.launch {
            intentFlow.emit(intent)
        }
    }
    
    protected abstract suspend fun handleIntent(intent: I)
    
    protected fun updateState(reducer: S.() -> S) {
        _uiState.value = _uiState.value.reducer()
    }
    
    protected suspend fun sendEffect(effect: E) {
        _uiEffect.emit(effect)
    }
}

// MVI 實作範例
sealed interface TimerIntent : UiIntent {
    object StartTimer : TimerIntent
    object PauseTimer : TimerIntent
    object ResetTimer : TimerIntent
    object SwitchMode : TimerIntent
}

data class TimerUiState(
    val timeRemaining: Long = 25 * 60 * 1000L,
    val timerState: TimerState = TimerState.IDLE,
    val currentMode: TimerMode = TimerMode.FOCUS,
    val isLoading: Boolean = false,
    val error: String? = null
) : UiState

sealed interface TimerEffect : UiEffect {
    object TimerCompleted : TimerEffect
    data class ShowError(val message: String) : TimerEffect
    object PlaySound : TimerEffect
}

class TimerMviViewModel @Inject constructor(
    private val startTimerUseCase: StartTimerUseCase,
    private val pauseTimerUseCase: PauseTimerUseCase,
    private val resetTimerUseCase: ResetTimerUseCase
) : MviViewModel<TimerIntent, TimerUiState, TimerEffect>() {
    
    override fun createInitialState(): TimerUiState = TimerUiState()
    
    override suspend fun handleIntent(intent: TimerIntent) {
        when (intent) {
            is TimerIntent.StartTimer -> handleStartTimer()
            is TimerIntent.PauseTimer -> handlePauseTimer()
            is TimerIntent.ResetTimer -> handleResetTimer()
            is TimerIntent.SwitchMode -> handleSwitchMode()
        }
    }
    
    private suspend fun handleStartTimer() {
        updateState { copy(isLoading = true) }
        
        startTimerUseCase(uiState.value.currentMode.toSessionType())
            .onSuccess {
                updateState { 
                    copy(
                        timerState = TimerState.RUNNING,
                        isLoading = false,
                        error = null
                    )
                }
            }
            .onFailure { exception ->
                updateState { 
                    copy(
                        isLoading = false,
                        error = exception.message
                    )
                }
                sendEffect(TimerEffect.ShowError(exception.message ?: "未知錯誤"))
            }
    }
    
    // 其他 intent 處理方法...
}
```

### 模組化架構

```kotlin
// feature/timer/di/TimerModule.kt
@Module
@InstallIn(SingletonComponent::class)
abstract class TimerFeatureModule {
    
    @Binds
    abstract fun bindTimerRepository(
        timerRepositoryImpl: TimerRepositoryImpl
    ): TimerRepository
    
    @Binds
    @IntoSet
    abstract fun bindTimerFeature(
        timerFeature: TimerFeatureImpl
    ): AppFeature
}

// core/feature/AppFeature.kt
interface AppFeature {
    val name: String
    val isEnabled: Boolean
    
    suspend fun initialize()
    suspend fun cleanup()
}

// feature/timer/TimerFeatureImpl.kt
@Singleton
class TimerFeatureImpl @Inject constructor(
    private val timerService: TimerService,
    private val notificationHelper: NotificationHelper
) : AppFeature {
    
    override val name: String = "Timer"
    override val isEnabled: Boolean = true
    
    override suspend fun initialize() {
        notificationHelper.createNotificationChannels()
        // 其他初始化邏輯
    }
    
    override suspend fun cleanup() {
        timerService.stopSelf()
        // 清理資源
    }
}
```

## 📊 分析和監控

### 自定義分析框架

```kotlin
// analytics/AnalyticsManager.kt
interface AnalyticsManager {
    fun trackEvent(event: AnalyticsEvent)
    fun trackScreenView(screenName: String)
    fun setUserProperty(key: String, value: String)
    fun trackError(throwable: Throwable, context: String? = null)
}

@Singleton
class AnalyticsManagerImpl @Inject constructor(
    private val firebaseAnalytics: FirebaseAnalytics,
    private val crashlytics: FirebaseCrashlytics
) : AnalyticsManager {
    
    override fun trackEvent(event: AnalyticsEvent) {
        val bundle = Bundle().apply {
            event.parameters.forEach { (key, value) ->
                when (value) {
                    is String -> putString(key, value)
                    is Long -> putLong(key, value)
                    is Double -> putDouble(key, value)
                    is Boolean -> putBoolean(key, value)
                }
            }
        }
        
        firebaseAnalytics.logEvent(event.name, bundle)
    }
    
    override fun trackScreenView(screenName: String) {
        firebaseAnalytics.logEvent(FirebaseAnalytics.Event.SCREEN_VIEW) {
            param(FirebaseAnalytics.Param.SCREEN_NAME, screenName)
            param(FirebaseAnalytics.Param.SCREEN_CLASS, screenName)
        }
    }
    
    override fun setUserProperty(key: String, value: String) {
        firebaseAnalytics.setUserProperty(key, value)
    }
    
    override fun trackError(throwable: Throwable, context: String?) {
        crashlytics.apply {
            context?.let { setCustomKey("error_context", it) }
            recordException(throwable)
        }
    }
}

// 分析事件定義
sealed class AnalyticsEvent(
    val name: String,
    val parameters: Map<String, Any> = emptyMap()
) {
    object TimerStarted : AnalyticsEvent(
        name = "timer_started",
        parameters = mapOf("timer_mode" to "focus")
    )
    
    object TimerCompleted : AnalyticsEvent(
        name = "timer_completed",
        parameters = mapOf("duration_minutes" to 25)
    )
    
    data class TaskCreated(
        val taskName: String,
        val estimatedPomodoros: Int
    ) : AnalyticsEvent(
        name = "task_created",
        parameters = mapOf(
            "task_name_length" to taskName.length,
            "estimated_pomodoros" to estimatedPomodoros
        )
    )
}
```

### 效能監控

```kotlin
// monitoring/PerformanceMonitor.kt
@Singleton
class PerformanceMonitor @Inject constructor(
    private val analyticsManager: AnalyticsManager
) {
    
    fun trackStartupTime() {
        val startupTime = System.currentTimeMillis() - ProcessStartTime.get()
        analyticsManager.trackEvent(
            AnalyticsEvent.AppStartup(startupTime)
        )
    }
    
    fun trackDatabaseOperation(operation: String, duration: Long) {
        if (duration > 100) { // 超過 100ms 的操作
            analyticsManager.trackEvent(
                AnalyticsEvent.SlowDatabaseOperation(operation, duration)
            )
        }
    }
    
    fun trackMemoryUsage() {
        val runtime = Runtime.getRuntime()
        val memoryInfo = MemoryInfo(
            totalMemory = runtime.totalMemory(),
            freeMemory = runtime.freeMemory(),
            maxMemory = runtime.maxMemory()
        )
        
        analyticsManager.trackEvent(
            AnalyticsEvent.MemoryUsage(memoryInfo)
        )
    }
}

// 效能裝飾器模式
class PerformanceTrackingRepository<T : Any>(
    private val delegate: T,
    private val performanceMonitor: PerformanceMonitor,
    private val repositoryName: String
) : T by delegate {
    
    suspend fun <R> trackOperation(
        operationName: String,
        operation: suspend () -> R
    ): R {
        val startTime = System.currentTimeMillis()
        return try {
            operation()
        } finally {
            val duration = System.currentTimeMillis() - startTime
            performanceMonitor.trackDatabaseOperation(
                "$repositoryName.$operationName",
                duration
            )
        }
    }
}
```

## 📚 延伸閱讀

- [Android Performance Best Practices](https://developer.android.com/topic/performance)
- [Android Accessibility Guide](https://developer.android.com/guide/topics/ui/accessibility)
- [Android App Bundle](https://developer.android.com/guide/app-bundle)
- [Kotlin Coroutines Best Practices](https://developer.android.com/kotlin/coroutines/coroutines-best-practices)

## 🎯 總結

恭喜你完成了番茄鐘應用程式的完整教學系列！透過這八個章節，你已經學會了：

1. **基礎設置** - 專案環境和配置
2. **架構設計** - Clean Architecture 和 MVVM
3. **UI 開發** - Jetpack Compose 進階技巧
4. **資料管理** - Room 資料庫和 Repository 模式
5. **業務邏輯** - Domain 層和 Use Case 設計
6. **依賴注入** - Hilt 框架進階應用
7. **背景服務** - Foreground Service 和通知系統
8. **測試實作** - 完整的測試策略
9. **進階主題** - 效能優化、國際化、無障礙設計

現在你已經具備了建構現代化 Android 應用程式的完整技能！🚀