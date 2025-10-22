# 第 4 章：領域層設計

## 📋 本章目標

- 理解領域層在 Clean Architecture 中的角色
- 設計和實作 Use Case（使用案例）
- 建立領域模型和業務規則
- 實作領域服務和業務邏輯

## 🏢 領域層概述

領域層是 Clean Architecture 的核心，包含：

- **業務邏輯**：應用程式的核心規則
- **領域模型**：業務實體和值物件
- **Use Cases**：應用程式的使用案例
- **Repository 介面**：資料存取抽象

### 領域層原則

1. **獨立性**：不依賴任何外部框架
2. **純粹性**：只包含業務邏輯
3. **可測試性**：容易進行單元測試
4. **可重用性**：可在不同平台重用

## 📊 領域模型設計

### 核心領域實體

```kotlin
// domain/model/Task.kt
data class Task(
    val id: Long = 0,
    val name: String,
    val description: String = "",
    val isCompleted: Boolean = false,
    val isCurrent: Boolean = false,
    val createdAt: Long = System.currentTimeMillis(),
    val updatedAt: Long = System.currentTimeMillis(),
    val estimatedPomodoros: Int = 1,
    val completedPomodoros: Int = 0,
    val priority: TaskPriority = TaskPriority.MEDIUM
) {
    init {
        require(name.isNotBlank()) { "任務名稱不能為空" }
        require(estimatedPomodoros > 0) { "預估番茄鐘數量必須大於 0" }
        require(completedPomodoros >= 0) { "完成的番茄鐘數量不能為負數" }
    }
    
    val completionPercentage: Float
        get() = if (estimatedPomodoros > 0) {
            (completedPomodoros.toFloat() / estimatedPomodoros.toFloat()).coerceIn(0f, 1f)
        } else 0f
    
    val isOverdue: Boolean
        get() = completedPomodoros > estimatedPomodoros
    
    val remainingPomodoros: Int
        get() = maxOf(0, estimatedPomodoros - completedPomodoros)
    
    fun complete(): Task = copy(
        isCompleted = true,
        updatedAt = System.currentTimeMillis()
    )
    
    fun incrementPomodoro(): Task = copy(
        completedPomodoros = completedPomodoros + 1,
        updatedAt = System.currentTimeMillis()
    )
    
    fun setAsCurrent(): Task = copy(
        isCurrent = true,
        updatedAt = System.currentTimeMillis()
    )
    
    fun clearCurrent(): Task = copy(
        isCurrent = false,
        updatedAt = System.currentTimeMillis()
    )
}

enum class TaskPriority(val displayName: String, val level: Int) {
    LOW("低", 1),
    MEDIUM("中", 2),
    HIGH("高", 3),
    URGENT("緊急", 4);
    
    companion object {
        fun fromLevel(level: Int): TaskPriority {
            return values().find { it.level == level } ?: MEDIUM
        }
    }
}
```

```kotlin
// domain/model/PomodoroSession.kt
data class PomodoroSession(
    val id: Long = 0,
    val taskId: Long? = null,
    val sessionType: SessionType,
    val startTime: Long,
    val endTime: Long? = null,
    val durationMillis: Long,
    val isCompleted: Boolean = false,
    val notes: String = "",
    val interruptions: Int = 0
) {
    init {
        require(durationMillis > 0) { "會話時長必須大於 0" }
        require(interruptions >= 0) { "中斷次數不能為負數" }
    }
    
    val actualDurationMillis: Long
        get() = endTime?.let { it - startTime } ?: 0L
    
    val isRunning: Boolean
        get() = endTime == null && !isCompleted
    
    val efficiency: Float
        get() = if (durationMillis > 0) {
            val actualDuration = actualDurationMillis
            if (actualDuration > 0) {
                (actualDuration.toFloat() / durationMillis.toFloat()).coerceIn(0f, 1f)
            } else 0f
        } else 0f
    
    val qualityScore: Float
        get() {
            val baseScore = efficiency
            val interruptionPenalty = interruptions * 0.1f
            return maxOf(0f, baseScore - interruptionPenalty)
        }
    
    fun complete(notes: String = ""): PomodoroSession = copy(
        endTime = System.currentTimeMillis(),
        isCompleted = true,
        notes = notes
    )
    
    fun addInterruption(): PomodoroSession = copy(
        interruptions = interruptions + 1
    )
}

enum class SessionType(val displayName: String, val defaultDurationMinutes: Int) {
    FOCUS("專注時間", 25),
    SHORT_BREAK("短休息", 5),
    LONG_BREAK("長休息", 15);
    
    val defaultDurationMillis: Long
        get() = defaultDurationMinutes * 60 * 1000L
}
```

### 值物件設計

```kotlin
// domain/model/TimerConfiguration.kt
data class TimerConfiguration(
    val focusDurationMinutes: Int = 25,
    val shortBreakDurationMinutes: Int = 5,
    val longBreakDurationMinutes: Int = 15,
    val longBreakInterval: Int = 4, // 每 4 個番茄鐘後長休息
    val autoStartBreak: Boolean = false,
    val autoStartFocus: Boolean = false,
    val enableNotifications: Boolean = true,
    val enableSounds: Boolean = true,
    val enableVibration: Boolean = true
) {
    init {
        require(focusDurationMinutes in 1..60) { "專注時間必須在 1-60 分鐘之間" }
        require(shortBreakDurationMinutes in 1..30) { "短休息時間必須在 1-30 分鐘之間" }
        require(longBreakDurationMinutes in 1..60) { "長休息時間必須在 1-60 分鐘之間" }
        require(longBreakInterval in 2..10) { "長休息間隔必須在 2-10 個番茄鐘之間" }
    }
    
    val focusDurationMillis: Long
        get() = focusDurationMinutes * 60 * 1000L
    
    val shortBreakDurationMillis: Long
        get() = shortBreakDurationMinutes * 60 * 1000L
    
    val longBreakDurationMillis: Long
        get() = longBreakDurationMinutes * 60 * 1000L
    
    fun getDurationForType(sessionType: SessionType): Long {
        return when (sessionType) {
            SessionType.FOCUS -> focusDurationMillis
            SessionType.SHORT_BREAK -> shortBreakDurationMillis
            SessionType.LONG_BREAK -> longBreakDurationMillis
        }
    }
}

// domain/model/Statistics.kt
data class DailyStatistics(
    val date: Long,
    val completedFocusSessions: Int = 0,
    val completedBreakSessions: Int = 0,
    val totalFocusTimeMillis: Long = 0,
    val totalBreakTimeMillis: Long = 0,
    val averageSessionQuality: Float = 0f,
    val completedTasks: Int = 0,
    val totalInterruptions: Int = 0
) {
    val totalSessionsCompleted: Int
        get() = completedFocusSessions + completedBreakSessions
    
    val totalTimeMillis: Long
        get() = totalFocusTimeMillis + totalBreakTimeMillis
    
    val focusEfficiency: Float
        get() = if (completedFocusSessions > 0) {
            totalFocusTimeMillis.toFloat() / (completedFocusSessions * 25 * 60 * 1000L)
        } else 0f
    
    val averageInterruptionsPerSession: Float
        get() = if (completedFocusSessions > 0) {
            totalInterruptions.toFloat() / completedFocusSessions
        } else 0f
}
```

## 🎯 Use Case 實作

### 計時器相關 Use Cases

```kotlin
// domain/usecase/timer/StartTimerUseCase.kt
class StartTimerUseCase @Inject constructor(
    private val timerRepository: TimerRepository,
    private val sessionRepository: SessionRepository,
    private val taskRepository: TaskRepository
) {
    suspend operator fun invoke(sessionType: SessionType): Result<Long> {
        return try {
            // 檢查是否有正在運行的會話
            val runningSession = sessionRepository.getRunningSession()
            if (runningSession != null) {
                return Result.failure(TimerException.AlreadyRunning)
            }
            
            // 獲取當前任務（如果是專注時間）
            val currentTask = if (sessionType == SessionType.FOCUS) {
                taskRepository.getCurrentTask().firstOrNull()
            } else null
            
            // 獲取計時器配置
            val config = timerRepository.getConfiguration()
            val duration = config.getDurationForType(sessionType)
            
            // 建立新會話
            val session = PomodoroSession(
                taskId = currentTask?.id,
                sessionType = sessionType,
                startTime = System.currentTimeMillis(),
                durationMillis = duration
            )
            
            // 保存會話
            val sessionId = sessionRepository.insertSession(session)
            
            // 啟動計時器
            timerRepository.startTimer(sessionId, duration)
            
            Result.success(sessionId)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

// domain/usecase/timer/PauseTimerUseCase.kt
class PauseTimerUseCase @Inject constructor(
    private val timerRepository: TimerRepository,
    private val sessionRepository: SessionRepository
) {
    suspend operator fun invoke(): Result<Unit> {
        return try {
            // 獲取當前運行的會話
            val runningSession = sessionRepository.getRunningSession()
                ?: return Result.failure(TimerException.NoRunningTimer)
            
            // 暫停計時器
            timerRepository.pauseTimer()
            
            // 更新會話狀態（如需要）
            // 這裡可以記錄暫停時間等資訊
            
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

// domain/usecase/timer/CompleteSessionUseCase.kt
class CompleteSessionUseCase @Inject constructor(
    private val sessionRepository: SessionRepository,
    private val taskRepository: TaskRepository,
    private val statisticsRepository: StatisticsRepository
) {
    suspend operator fun invoke(sessionId: Long, notes: String = ""): Result<Unit> {
        return try {
            // 獲取會話
            val session = sessionRepository.getSessionById(sessionId)
                ?: return Result.failure(TimerException.SessionNotFound)
            
            // 完成會話
            val completedSession = session.complete(notes)
            sessionRepository.updateSession(completedSession)
            
            // 如果是專注會話，更新任務進度
            if (session.sessionType == SessionType.FOCUS && session.taskId != null) {
                val task = taskRepository.getTaskById(session.taskId)
                if (task != null) {
                    val updatedTask = task.incrementPomodoro()
                    taskRepository.updateTask(updatedTask)
                }
            }
            
            // 更新統計資料
            statisticsRepository.updateDailyStatistics(completedSession)
            
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}
```

### 任務管理 Use Cases

```kotlin
// domain/usecase/task/AddTaskUseCase.kt
class AddTaskUseCase @Inject constructor(
    private val taskRepository: TaskRepository,
    private val taskValidator: TaskValidator
) {
    suspend operator fun invoke(
        name: String,
        description: String = "",
        estimatedPomodoros: Int = 1,
        priority: TaskPriority = TaskPriority.MEDIUM
    ): Result<Long> {
        return try {
            // 驗證任務資料
            taskValidator.validateTaskData(name, estimatedPomodoros)
                .onFailure { return Result.failure(it) }
            
            // 建立任務
            val task = Task(
                name = name.trim(),
                description = description.trim(),
                estimatedPomodoros = estimatedPomodoros,
                priority = priority
            )
            
            // 保存任務
            val taskId = taskRepository.insertTask(task)
            
            Result.success(taskId)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

// domain/usecase/task/SetCurrentTaskUseCase.kt
class SetCurrentTaskUseCase @Inject constructor(
    private val taskRepository: TaskRepository
) {
    suspend operator fun invoke(taskId: Long): Result<Unit> {
        return try {
            // 檢查任務是否存在
            val task = taskRepository.getTaskById(taskId)
                ?: return Result.failure(TaskException.TaskNotFound)
            
            // 檢查任務是否已完成
            if (task.isCompleted) {
                return Result.failure(TaskException.TaskAlreadyCompleted)
            }
            
            // 設置為當前任務
            taskRepository.setCurrentTask(taskId)
            
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

// domain/usecase/task/CompleteTaskUseCase.kt
class CompleteTaskUseCase @Inject constructor(
    private val taskRepository: TaskRepository,
    private val statisticsRepository: StatisticsRepository
) {
    suspend operator fun invoke(taskId: Long): Result<Unit> {
        return try {
            // 獲取任務
            val task = taskRepository.getTaskById(taskId)
                ?: return Result.failure(TaskException.TaskNotFound)
            
            // 完成任務
            val completedTask = task.complete()
            taskRepository.updateTask(completedTask)
            
            // 如果是當前任務，清除當前狀態
            if (task.isCurrent) {
                taskRepository.clearCurrentTask()
            }
            
            // 更新統計資料
            statisticsRepository.incrementCompletedTasks()
            
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}
```

### 統計相關 Use Cases

```kotlin
// domain/usecase/statistics/GetDailyStatisticsUseCase.kt
class GetDailyStatisticsUseCase @Inject constructor(
    private val statisticsRepository: StatisticsRepository,
    private val dateHelper: DateHelper
) {
    suspend operator fun invoke(date: Long = System.currentTimeMillis()): Result<DailyStatistics> {
        return try {
            val normalizedDate = dateHelper.normalizeToDay(date)
            val statistics = statisticsRepository.getDailyStatistics(normalizedDate)
            Result.success(statistics)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

// domain/usecase/statistics/GetWeeklyStatisticsUseCase.kt
class GetWeeklyStatisticsUseCase @Inject constructor(
    private val statisticsRepository: StatisticsRepository,
    private val dateHelper: DateHelper
) {
    suspend operator fun invoke(weekStartDate: Long = System.currentTimeMillis()): Result<List<DailyStatistics>> {
        return try {
            val weekRange = dateHelper.getWeekRange(weekStartDate)
            val statistics = statisticsRepository.getStatisticsInRange(
                weekRange.first,
                weekRange.second
            )
            Result.success(statistics)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}
```

## 🔍 領域服務

### 計時器規則服務

```kotlin
// domain/service/TimerRuleService.kt
@Singleton
class TimerRuleService @Inject constructor(
    private val configurationRepository: ConfigurationRepository
) {
    
    suspend fun determineNextSessionType(
        completedFocusSessions: Int
    ): SessionType {
        val config = configurationRepository.getConfiguration()
        
        return if (completedFocusSessions > 0 && 
                   completedFocusSessions % config.longBreakInterval == 0) {
            SessionType.LONG_BREAK
        } else {
            SessionType.SHORT_BREAK
        }
    }
    
    suspend fun shouldAutoStartNext(sessionType: SessionType): Boolean {
        val config = configurationRepository.getConfiguration()
        
        return when (sessionType) {
            SessionType.FOCUS -> config.autoStartFocus
            SessionType.SHORT_BREAK, SessionType.LONG_BREAK -> config.autoStartBreak
        }
    }
    
    fun validateSessionDuration(durationMillis: Long): Result<Unit> {
        return when {
            durationMillis <= 0 -> Result.failure(
                TimerException.InvalidDuration("會話時長必須大於 0")
            )
            durationMillis > 2 * 60 * 60 * 1000L -> Result.failure(
                TimerException.InvalidDuration("會話時長不能超過 2 小時")
            )
            else -> Result.success(Unit)
        }
    }
}

// domain/service/TaskPriorityService.kt
@Singleton
class TaskPriorityService {
    
    fun calculateTaskScore(task: Task): Float {
        val priorityWeight = task.priority.level * 0.3f
        val urgencyWeight = calculateUrgencyScore(task) * 0.4f
        val progressWeight = (1f - task.completionPercentage) * 0.3f
        
        return priorityWeight + urgencyWeight + progressWeight
    }
    
    private fun calculateUrgencyScore(task: Task): Float {
        // 基於任務建立時間計算緊急程度
        val daysSinceCreation = (System.currentTimeMillis() - task.createdAt) / (24 * 60 * 60 * 1000L)
        return when {
            daysSinceCreation > 7 -> 1.0f  // 超過一週
            daysSinceCreation > 3 -> 0.7f  // 超過三天
            daysSinceCreation > 1 -> 0.4f  // 超過一天
            else -> 0.1f
        }
    }
    
    fun getSuggestedNextTask(tasks: List<Task>): Task? {
        return tasks
            .filter { !it.isCompleted }
            .maxByOrNull { calculateTaskScore(it) }
    }
}
```

## ⚠️ 異常處理

### 領域異常定義

```kotlin
// domain/exception/DomainExceptions.kt
sealed class DomainException(message: String, cause: Throwable? = null) : Exception(message, cause)

sealed class TimerException(message: String) : DomainException(message) {
    object AlreadyRunning : TimerException("計時器已在運行中")
    object NoRunningTimer : TimerException("沒有正在運行的計時器")
    object SessionNotFound : TimerException("找不到指定的會話")
    class InvalidDuration(message: String) : TimerException(message)
}

sealed class TaskException(message: String) : DomainException(message) {
    object TaskNotFound : TaskException("找不到指定的任務")
    object TaskAlreadyCompleted : TaskException("任務已完成")
    class InvalidTaskName(message: String) : TaskException(message)
    class InvalidEstimation(message: String) : TaskException(message)
}

sealed class ValidationException(message: String) : DomainException(message) {
    class EmptyField(fieldName: String) : ValidationException("$fieldName 不能為空")
    class InvalidRange(fieldName: String, min: Int, max: Int) : 
        ValidationException("$fieldName 必須在 $min 到 $max 之間")
}
```

## 🧪 領域層驗證

### 輸入驗證服務

```kotlin
// domain/service/TaskValidator.kt
@Singleton
class TaskValidator {
    
    fun validateTaskData(name: String, estimatedPomodoros: Int): Result<Unit> {
        return try {
            validateTaskName(name)
                .onFailure { return Result.failure(it) }
            
            validateEstimatedPomodoros(estimatedPomodoros)
                .onFailure { return Result.failure(it) }
            
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    private fun validateTaskName(name: String): Result<Unit> {
        return when {
            name.isBlank() -> Result.failure(
                TaskException.InvalidTaskName("任務名稱不能為空")
            )
            name.length > 100 -> Result.failure(
                TaskException.InvalidTaskName("任務名稱不能超過 100 個字元")
            )
            else -> Result.success(Unit)
        }
    }
    
    private fun validateEstimatedPomodoros(estimated: Int): Result<Unit> {
        return when {
            estimated <= 0 -> Result.failure(
                TaskException.InvalidEstimation("預估番茄鐘數量必須大於 0")
            )
            estimated > 50 -> Result.failure(
                TaskException.InvalidEstimation("預估番茄鐘數量不能超過 50")
            )
            else -> Result.success(Unit)
        }
    }
}

// domain/service/ConfigurationValidator.kt
@Singleton
class ConfigurationValidator {
    
    fun validateConfiguration(config: TimerConfiguration): Result<Unit> {
        return try {
            validateDuration("專注時間", config.focusDurationMinutes, 5, 90)
                .onFailure { return Result.failure(it) }
            
            validateDuration("短休息時間", config.shortBreakDurationMinutes, 1, 30)
                .onFailure { return Result.failure(it) }
            
            validateDuration("長休息時間", config.longBreakDurationMinutes, 5, 60)
                .onFailure { return Result.failure(it) }
            
            validateLongBreakInterval(config.longBreakInterval)
                .onFailure { return Result.failure(it) }
            
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    private fun validateDuration(name: String, duration: Int, min: Int, max: Int): Result<Unit> {
        return if (duration in min..max) {
            Result.success(Unit)
        } else {
            Result.failure(ValidationException.InvalidRange(name, min, max))
        }
    }
    
    private fun validateLongBreakInterval(interval: Int): Result<Unit> {
        return if (interval in 2..10) {
            Result.success(Unit)
        } else {
            Result.failure(ValidationException.InvalidRange("長休息間隔", 2, 10))
        }
    }
}
```

## 📚 Repository 介面

### 領域層 Repository 介面

```kotlin
// domain/repository/TimerRepository.kt
interface TimerRepository {
    suspend fun startTimer(sessionId: Long, durationMillis: Long)
    suspend fun pauseTimer()
    suspend fun resumeTimer()
    suspend fun stopTimer()
    suspend fun getConfiguration(): TimerConfiguration
    suspend fun updateConfiguration(config: TimerConfiguration)
    fun getTimerState(): Flow<TimerState>
}

// domain/repository/ConfigurationRepository.kt
interface ConfigurationRepository {
    suspend fun getConfiguration(): TimerConfiguration
    suspend fun updateConfiguration(config: TimerConfiguration)
    suspend fun resetToDefaults()
}

// domain/repository/StatisticsRepository.kt
interface StatisticsRepository {
    suspend fun getDailyStatistics(date: Long): DailyStatistics
    suspend fun getStatisticsInRange(startDate: Long, endDate: Long): List<DailyStatistics>
    suspend fun updateDailyStatistics(session: PomodoroSession)
    suspend fun incrementCompletedTasks()
    suspend fun clearStatistics()
}
```

## 🧪 領域層測試

### Use Case 測試範例

```kotlin
// test/domain/usecase/AddTaskUseCaseTest.kt
@ExperimentalCoroutinesTest
class AddTaskUseCaseTest {
    
    private val mockTaskRepository = mockk<TaskRepository>()
    private val mockTaskValidator = mockk<TaskValidator>()
    private val addTaskUseCase = AddTaskUseCase(mockTaskRepository, mockTaskValidator)
    
    @Test
    fun `should add task successfully when data is valid`() = runTest {
        // Given
        val taskName = "Test Task"
        val description = "Test Description"
        val estimatedPomodoros = 3
        val expectedTaskId = 1L
        
        every { mockTaskValidator.validateTaskData(taskName, estimatedPomodoros) } returns Result.success(Unit)
        coEvery { mockTaskRepository.insertTask(any()) } returns expectedTaskId
        
        // When
        val result = addTaskUseCase(taskName, description, estimatedPomodoros)
        
        // Then
        assertThat(result.isSuccess).isTrue()
        assertThat(result.getOrNull()).isEqualTo(expectedTaskId)
        
        coVerify {
            mockTaskRepository.insertTask(
                match { task ->
                    task.name == taskName &&
                    task.description == description &&
                    task.estimatedPomodoros == estimatedPomodoros
                }
            )
        }
    }
    
    @Test
    fun `should fail when task validation fails`() = runTest {
        // Given
        val invalidName = ""
        val validationError = TaskException.InvalidTaskName("任務名稱不能為空")
        
        every { mockTaskValidator.validateTaskData(invalidName, any()) } returns Result.failure(validationError)
        
        // When
        val result = addTaskUseCase(invalidName)
        
        // Then
        assertThat(result.isFailure).isTrue()
        assertThat(result.exceptionOrNull()).isEqualTo(validationError)
        
        coVerify(exactly = 0) { mockTaskRepository.insertTask(any()) }
    }
}
```

## 📚 延伸閱讀

- [Domain-Driven Design](https://domainlanguage.com/ddd/)
- [Clean Architecture Use Cases](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Result Pattern in Kotlin](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-result/)

## 🎯 下一章預告

在下一章中，我們將探討 **依賴注入**，學習：

- Hilt 框架深入應用
- 模組設計和作用域管理
- 測試中的依賴注入
- 效能優化技巧

準備好建構靈活的依賴管理系統了嗎？🔌