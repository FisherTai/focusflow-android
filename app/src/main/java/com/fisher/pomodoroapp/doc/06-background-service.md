# 第 6 章：背景服務

## 📋 本章目標

- 實作 Foreground Service 進行背景計時
- 設計通知系統提供即時反饋
- 管理計時器生命週期
- 處理系統資源和電池優化

## 🔔 Foreground Service 概述

### 為什麼需要 Foreground Service？

在現代 Android 系統中，為了節省電池並提升性能，系統會嚴格限制背景任務的執行。對於番茄鐘應用，我們需要：

1. **持續計時**：即使應用在背景也要繼續計時
2. **準確通知**：在計時結束時準時提醒用戶
3. **用戶感知**：透過通知讓用戶知道服務正在運行
4. **系統允許**：Foreground Service 是被系統允許的長時間運行方式

### Foreground Service 特點

- ✅ 可以長時間運行
- ✅ 系統不會輕易殺死
- ✅ 必須顯示持續通知
- ✅ 用戶可以看到和控制

## 🛠️ TimerService 實作

### 服務基礎結構

```kotlin
// service/TimerService.kt
@AndroidEntryPoint
class TimerService : Service() {
    
    companion object {
        const val ACTION_START_TIMER = "START_TIMER"
        const val ACTION_PAUSE_TIMER = "PAUSE_TIMER"
        const val ACTION_RESET_TIMER = "RESET_TIMER"
        const val ACTION_STOP_SERVICE = "STOP_SERVICE"
        
        const val NOTIFICATION_ID = 1001
        const val CHANNEL_ID = "TIMER_CHANNEL"
        
        private const val TIMER_INTERVAL = 1000L // 1秒更新一次
    }
    
    @Inject
    lateinit var timerManager: TimerManager
    
    @Inject
    lateinit var notificationHelper: NotificationHelper
    
    private val binder = TimerBinder()
    private var isServiceRunning = false
    
    // 計時器相關
    private var countDownTimer: CountDownTimer? = null
    private var timeRemaining: Long = 25 * 60 * 1000L // 25分鐘
    private var timerState: TimerState = TimerState.IDLE
    private var currentMode: TimerMode = TimerMode.FOCUS
    
    // StateFlow 用於與 UI 通信
    private val _timerState = MutableStateFlow(
        TimerServiceState(
            timeRemaining = timeRemaining,
            timerState = timerState,
            currentMode = currentMode
        )
    )
    val timerStateFlow: StateFlow<TimerServiceState> = _timerState.asStateFlow()
    
    override fun onCreate() {
        super.onCreate()
        createNotificationChannel()
    }
    
    override fun onBind(intent: Intent?): IBinder {
        return binder
    }
    
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        when (intent?.action) {
            ACTION_START_TIMER -> startTimer()
            ACTION_PAUSE_TIMER -> pauseTimer()
            ACTION_RESET_TIMER -> resetTimer()
            ACTION_STOP_SERVICE -> stopService()
            else -> startForegroundService()
        }
        return START_NOT_STICKY
    }
    
    inner class TimerBinder : Binder() {
        fun getService(): TimerService = this@TimerService
    }
}

data class TimerServiceState(
    val timeRemaining: Long,
    val timerState: TimerState,
    val currentMode: TimerMode
)
```

### 計時器核心邏輯

```kotlin
// TimerService.kt 續
class TimerService : Service() {
    
    private fun startTimer() {
        if (timerState == TimerState.RUNNING) return
        
        timerState = TimerState.RUNNING
        updateState()
        
        countDownTimer = object : CountDownTimer(timeRemaining, TIMER_INTERVAL) {
            override fun onTick(millisUntilFinished: Long) {
                timeRemaining = millisUntilFinished
                updateState()
                updateNotification()
            }
            
            override fun onFinish() {
                onTimerCompleted()
            }
        }.start()
        
        startForegroundService()
    }
    
    private fun pauseTimer() {
        if (timerState != TimerState.RUNNING) return
        
        countDownTimer?.cancel()
        timerState = TimerState.PAUSED
        updateState()
        updateNotification()
    }
    
    private fun resetTimer() {
        countDownTimer?.cancel()
        timeRemaining = getDefaultTimeForMode(currentMode)
        timerState = TimerState.IDLE
        updateState()
        updateNotification()
    }
    
    private fun onTimerCompleted() {
        timerState = TimerState.COMPLETED
        updateState()
        
        // 播放完成音效
        playCompletionSound()
        
        // 顯示完成通知
        showCompletionNotification()
        
        // 記錄會話
        recordSession()
        
        // 自動切換模式
        switchToNextMode()
    }
    
    private fun switchToNextMode() {
        currentMode = when (currentMode) {
            TimerMode.FOCUS -> TimerMode.BREAK
            TimerMode.BREAK -> TimerMode.FOCUS
        }
        
        timeRemaining = getDefaultTimeForMode(currentMode)
        timerState = TimerState.IDLE
        updateState()
        updateNotification()
    }
    
    private fun getDefaultTimeForMode(mode: TimerMode): Long {
        return when (mode) {
            TimerMode.FOCUS -> 25 * 60 * 1000L // 25分鐘
            TimerMode.BREAK -> 5 * 60 * 1000L  // 5分鐘
        }
    }
    
    private fun updateState() {
        _timerState.value = TimerServiceState(
            timeRemaining = timeRemaining,
            timerState = timerState,
            currentMode = currentMode
        )
    }
    
    private fun recordSession() {
        // 使用 TimerManager 記錄會話
        lifecycleScope.launch {
            timerManager.recordCompletedSession(
                mode = currentMode,
                durationMillis = getDefaultTimeForMode(currentMode)
            )
        }
    }
}
```

### 通知系統實作

```kotlin
// service/NotificationHelper.kt
@Singleton
class NotificationHelper @Inject constructor(
    @ApplicationContext private val context: Context
) {
    
    companion object {
        const val CHANNEL_ID = "TIMER_CHANNEL"
        const val CHANNEL_NAME = "計時器"
        const val CHANNEL_DESCRIPTION = "番茄鐘計時器通知"
        
        const val COMPLETION_CHANNEL_ID = "COMPLETION_CHANNEL"
        const val COMPLETION_CHANNEL_NAME = "完成提醒"
        const val COMPLETION_CHANNEL_DESCRIPTION = "番茄鐘完成通知"
    }
    
    private val notificationManager = context.getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
    
    init {
        createNotificationChannels()
    }
    
    private fun createNotificationChannels() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            // 計時器運行通知頻道
            val timerChannel = NotificationChannel(
                CHANNEL_ID,
                CHANNEL_NAME,
                NotificationManager.IMPORTANCE_LOW
            ).apply {
                description = CHANNEL_DESCRIPTION
                setShowBadge(false)
                setSound(null, null)
                enableVibration(false)
            }
            
            // 完成提醒通知頻道
            val completionChannel = NotificationChannel(
                COMPLETION_CHANNEL_ID,
                COMPLETION_CHANNEL_NAME,
                NotificationManager.IMPORTANCE_HIGH
            ).apply {
                description = COMPLETION_CHANNEL_DESCRIPTION
                setShowBadge(true)
                enableVibration(true)
                enableLights(true)
                lightColor = Color.GREEN
            }
            
            notificationManager.createNotificationChannels(
                listOf(timerChannel, completionChannel)
            )
        }
    }
    
    fun createTimerNotification(state: TimerServiceState): Notification {
        val intent = Intent(context, MainActivity::class.java).apply {
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
        }
        val pendingIntent = PendingIntent.getActivity(
            context, 0, intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )
        
        return NotificationCompat.Builder(context, CHANNEL_ID)
            .setContentTitle(getNotificationTitle(state))
            .setContentText(getNotificationText(state))
            .setSmallIcon(R.drawable.ic_timer)
            .setContentIntent(pendingIntent)
            .setOngoing(true)
            .setShowWhen(false)
            .addActions(state)
            .setStyle(createBigTextStyle(state))
            .build()
    }
    
    private fun getNotificationTitle(state: TimerServiceState): String {
        return when (state.currentMode) {
            TimerMode.FOCUS -> "專注時間"
            TimerMode.BREAK -> "休息時間"
        }
    }
    
    private fun getNotificationText(state: TimerServiceState): String {
        val timeText = formatTime(state.timeRemaining)
        return when (state.timerState) {
            TimerState.RUNNING -> "進行中 - $timeText"
            TimerState.PAUSED -> "已暫停 - $timeText"
            TimerState.IDLE -> "準備開始 - $timeText"
            TimerState.COMPLETED -> "已完成！"
        }
    }
    
    private fun NotificationCompat.Builder.addActions(state: TimerServiceState): NotificationCompat.Builder {
        // 開始/暫停按鈕
        when (state.timerState) {
            TimerState.IDLE, TimerState.PAUSED -> {
                addAction(createAction(
                    R.drawable.ic_play,
                    "開始",
                    TimerService.ACTION_START_TIMER
                ))
            }
            TimerState.RUNNING -> {
                addAction(createAction(
                    R.drawable.ic_pause,
                    "暫停",
                    TimerService.ACTION_PAUSE_TIMER
                ))
            }
            TimerState.COMPLETED -> {
                addAction(createAction(
                    R.drawable.ic_refresh,
                    "重新開始",
                    TimerService.ACTION_RESET_TIMER
                ))
            }
        }
        
        // 重設按鈕
        if (state.timerState != TimerState.IDLE) {
            addAction(createAction(
                R.drawable.ic_stop,
                "重設",
                TimerService.ACTION_RESET_TIMER
            ))
        }
        
        return this
    }
    
    private fun createAction(iconRes: Int, title: String, action: String): NotificationCompat.Action {
        val intent = Intent(context, TimerService::class.java).apply {
            this.action = action
        }
        val pendingIntent = PendingIntent.getService(
            context,
            action.hashCode(),
            intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )
        
        return NotificationCompat.Action.Builder(iconRes, title, pendingIntent).build()
    }
    
    private fun createBigTextStyle(state: TimerServiceState): NotificationCompat.BigTextStyle {
        val description = when (state.currentMode) {
            TimerMode.FOCUS -> "保持專注，距離休息還有 ${formatTime(state.timeRemaining)}"
            TimerMode.BREAK -> "享受休息時間，還有 ${formatTime(state.timeRemaining)}"
        }
        
        return NotificationCompat.BigTextStyle()
            .bigText(description)
    }
    
    fun showCompletionNotification(mode: TimerMode) {
        val title = when (mode) {
            TimerMode.FOCUS -> "專注時間完成！"
            TimerMode.BREAK -> "休息時間結束！"
        }
        
        val message = when (mode) {
            TimerMode.FOCUS -> "恭喜完成一個番茄鐘！該休息一下了 🍅"
            TimerMode.BREAK -> "休息時間結束，準備開始下一個番茄鐘 💪"
        }
        
        val notification = NotificationCompat.Builder(context, COMPLETION_CHANNEL_ID)
            .setContentTitle(title)
            .setContentText(message)
            .setSmallIcon(R.drawable.ic_timer_done)
            .setAutoCancel(true)
            .setPriority(NotificationCompat.PRIORITY_HIGH)
            .setDefaults(NotificationCompat.DEFAULT_ALL)
            .build()
        
        notificationManager.notify(
            System.currentTimeMillis().toInt(),
            notification
        )
    }
    
    private fun formatTime(timeInMillis: Long): String {
        val minutes = (timeInMillis / 1000) / 60
        val seconds = (timeInMillis / 1000) % 60
        return String.format("%02d:%02d", minutes, seconds)
    }
}
```

## 🎵 音效和震動

### 完成音效處理

```kotlin
// service/SoundManager.kt
@Singleton
class SoundManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    
    private var soundPool: SoundPool? = null
    private var completionSoundId: Int = 0
    
    init {
        initializeSoundPool()
    }
    
    private fun initializeSoundPool() {
        val audioAttributes = AudioAttributes.Builder()
            .setUsage(AudioAttributes.USAGE_NOTIFICATION)
            .setContentType(AudioAttributes.CONTENT_TYPE_SONIFICATION)
            .build()
        
        soundPool = SoundPool.Builder()
            .setMaxStreams(2)
            .setAudioAttributes(audioAttributes)
            .build()
        
        completionSoundId = soundPool?.load(context, R.raw.timer_complete, 1) ?: 0
    }
    
    fun playCompletionSound() {
        soundPool?.play(completionSoundId, 1f, 1f, 1, 0, 1f)
    }
    
    fun release() {
        soundPool?.release()
        soundPool = null
    }
}

// service/VibrationManager.kt
@Singleton
class VibrationManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    
    private val vibrator = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
        val vibratorManager = context.getSystemService(Context.VIBRATOR_MANAGER_SERVICE) as VibratorManager
        vibratorManager.defaultVibrator
    } else {
        @Suppress("DEPRECATION")
        context.getSystemService(Context.VIBRATOR_SERVICE) as Vibrator
    }
    
    fun vibrateCompletion() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val vibrationEffect = VibrationEffect.createWaveform(
                longArrayOf(0, 200, 100, 200, 100, 200),
                -1
            )
            vibrator.vibrate(vibrationEffect)
        } else {
            @Suppress("DEPRECATION")
            vibrator.vibrate(longArrayOf(0, 200, 100, 200, 100, 200), -1)
        }
    }
}
```

## 🔗 服務與 UI 通信

### ServiceConnection 管理

```kotlin
// service/TimerServiceConnection.kt
@Singleton
class TimerServiceConnection @Inject constructor(
    @ApplicationContext private val context: Context
) {
    
    private var timerService: TimerService? = null
    private var isServiceBound = false
    
    private val _isConnected = MutableStateFlow(false)
    val isConnected: StateFlow<Boolean> = _isConnected.asStateFlow()
    
    private val _timerState = MutableStateFlow<TimerServiceState?>(null)
    val timerState: StateFlow<TimerServiceState?> = _timerState.asStateFlow()
    
    private val serviceConnection = object : ServiceConnection {
        override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
            val binder = service as TimerService.TimerBinder
            timerService = binder.getService()
            isServiceBound = true
            _isConnected.value = true
            
            // 開始觀察服務狀態
            observeServiceState()
        }
        
        override fun onServiceDisconnected(name: ComponentName?) {
            timerService = null
            isServiceBound = false
            _isConnected.value = false
        }
    }
    
    private fun observeServiceState() {
        timerService?.let { service ->
            CoroutineScope(Dispatchers.Main).launch {
                service.timerStateFlow.collect { state ->
                    _timerState.value = state
                }
            }
        }
    }
    
    fun bindService() {
        if (!isServiceBound) {
            val intent = Intent(context, TimerService::class.java)
            context.bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE)
        }
    }
    
    fun unbindService() {
        if (isServiceBound) {
            context.unbindService(serviceConnection)
            isServiceBound = false
            _isConnected.value = false
        }
    }
    
    fun startTimer() {
        val intent = Intent(context, TimerService::class.java).apply {
            action = TimerService.ACTION_START_TIMER
        }
        context.startService(intent)
    }
    
    fun pauseTimer() {
        val intent = Intent(context, TimerService::class.java).apply {
            action = TimerService.ACTION_PAUSE_TIMER
        }
        context.startService(intent)
    }
    
    fun resetTimer() {
        val intent = Intent(context, TimerService::class.java).apply {
            action = TimerService.ACTION_RESET_TIMER
        }
        context.startService(intent)
    }
}
```

### ViewModel 整合

```kotlin
// ui/home/HomeViewModel.kt - 服務整合部分
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val timerServiceConnection: TimerServiceConnection,
    // ... 其他依賴
) : ViewModel() {
    
    init {
        // 綁定服務
        timerServiceConnection.bindService()
        
        // 觀察服務狀態
        observeServiceState()
    }
    
    private fun observeServiceState() {
        viewModelScope.launch {
            timerServiceConnection.timerState.collect { serviceState ->
                serviceState?.let { state ->
                    _uiState.update { currentState ->
                        currentState.copy(
                            timeRemaining = state.timeRemaining,
                            timerState = state.timerState,
                            currentMode = state.currentMode
                        )
                    }
                }
            }
        }
    }
    
    fun startTimer() {
        timerServiceConnection.startTimer()
    }
    
    fun pauseTimer() {
        timerServiceConnection.pauseTimer()
    }
    
    fun resetTimer() {
        timerServiceConnection.resetTimer()
    }
    
    override fun onCleared() {
        super.onCleared()
        timerServiceConnection.unbindService()
    }
}
```

## 🔋 電池優化處理

### 電池優化白名單

```kotlin
// util/BatteryOptimizationHelper.kt
@Singleton
class BatteryOptimizationHelper @Inject constructor(
    @ApplicationContext private val context: Context
) {
    
    fun isIgnoringBatteryOptimizations(): Boolean {
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            val powerManager = context.getSystemService(Context.POWER_SERVICE) as PowerManager
            powerManager.isIgnoringBatteryOptimizations(context.packageName)
        } else {
            true
        }
    }
    
    fun requestIgnoreBatteryOptimizations(activity: Activity) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M && !isIgnoringBatteryOptimizations()) {
            val intent = Intent().apply {
                action = Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS
                data = Uri.parse("package:${context.packageName}")
            }
            
            try {
                activity.startActivity(intent)
            } catch (e: ActivityNotFoundException) {
                // 處理某些設備不支援的情況
                openBatteryOptimizationSettings(activity)
            }
        }
    }
    
    private fun openBatteryOptimizationSettings(activity: Activity) {
        try {
            val intent = Intent(Settings.ACTION_IGNORE_BATTERY_OPTIMIZATION_SETTINGS)
            activity.startActivity(intent)
        } catch (e: ActivityNotFoundException) {
            // 最後的備用方案
            val intent = Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).apply {
                data = Uri.parse("package:${context.packageName}")
            }
            activity.startActivity(intent)
        }
    }
}
```

## 📊 服務生命週期管理

### 服務狀態監控

```kotlin
// service/ServiceLifecycleManager.kt
@Singleton
class ServiceLifecycleManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    
    private val _serviceRunning = MutableStateFlow(false)
    val serviceRunning: StateFlow<Boolean> = _serviceRunning.asStateFlow()
    
    fun startTimerService() {
        val intent = Intent(context, TimerService::class.java)
        context.startForegroundService(intent)
        _serviceRunning.value = true
    }
    
    fun stopTimerService() {
        val intent = Intent(context, TimerService::class.java).apply {
            action = TimerService.ACTION_STOP_SERVICE
        }
        context.startService(intent)
        _serviceRunning.value = false
    }
    
    fun isServiceRunning(): Boolean {
        val activityManager = context.getSystemService(Context.ACTIVITY_SERVICE) as ActivityManager
        @Suppress("DEPRECATION")
        return activityManager.getRunningServices(Integer.MAX_VALUE)
            .any { it.service.className == TimerService::class.java.name }
    }
}
```

## 🧪 服務測試

### 服務單元測試

```kotlin
// test/service/TimerServiceTest.kt
@ExperimentalCoroutinesTest
class TimerServiceTest {
    
    @get:Rule
    val instantExecutorRule = InstantTaskExecutorRule()
    
    private lateinit var service: TimerService
    private lateinit var context: Context
    
    @Before
    fun setup() {
        context = ApplicationProvider.getApplicationContext()
        service = TimerService()
    }
    
    @Test
    fun `service should start timer correctly`() = runTest {
        // Given
        val startIntent = Intent(context, TimerService::class.java).apply {
            action = TimerService.ACTION_START_TIMER
        }
        
        // When
        service.onStartCommand(startIntent, 0, 1)
        
        // Then
        val state = service.timerStateFlow.value
        assertThat(state.timerState).isEqualTo(TimerState.RUNNING)
    }
    
    @Test
    fun `service should pause timer correctly`() = runTest {
        // Given - 先啟動計時器
        service.onStartCommand(
            Intent(context, TimerService::class.java).apply {
                action = TimerService.ACTION_START_TIMER
            }, 0, 1
        )
        
        // When - 暫停計時器
        service.onStartCommand(
            Intent(context, TimerService::class.java).apply {
                action = TimerService.ACTION_PAUSE_TIMER
            }, 0, 1
        )
        
        // Then
        val state = service.timerStateFlow.value
        assertThat(state.timerState).isEqualTo(TimerState.PAUSED)
    }
}
```

## 📚 延伸閱讀

- [Android Foreground Services](https://developer.android.com/guide/components/foreground-services)
- [Notification Channels](https://developer.android.com/training/notify-user/channels)
- [Background Processing](https://developer.android.com/guide/background)

## 🎯 下一章預告

在下一章中，我們將探討 **測試實作**，學習：

- 單元測試設計和實作
- ViewModel 和 Repository 測試
- UI 測試自動化
- 測試覆蓋率優化

準備好建構可靠的測試體系了嗎？🧪