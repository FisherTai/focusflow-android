# 第 3 章：資料層實作

## 📋 本章目標

- 設計並實作 Room 資料庫
- 實作 Repository 模式管理資料存取
- 處理資料轉換和映射
- 實作資料快取和同步策略

## 💾 Room 資料庫設計

### 資料庫架構概覽

```
AppDatabase
├── TaskEntity (任務實體)
├── SessionEntity (會話實體)
└── Converters (類型轉換器)
```

### 任務實體設計

```kotlin
// entities/TaskEntity.kt
@Entity(tableName = "tasks")
data class TaskEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,
    
    @ColumnInfo(name = "name")
    val name: String,
    
    @ColumnInfo(name = "description")
    val description: String = "",
    
    @ColumnInfo(name = "is_completed")
    val isCompleted: Boolean = false,
    
    @ColumnInfo(name = "is_current")
    val isCurrent: Boolean = false,
    
    @ColumnInfo(name = "created_at")
    val createdAt: Long = System.currentTimeMillis(),
    
    @ColumnInfo(name = "updated_at")
    val updatedAt: Long = System.currentTimeMillis(),
    
    @ColumnInfo(name = "estimated_pomodoros")
    val estimatedPomodoros: Int = 1,
    
    @ColumnInfo(name = "completed_pomodoros")
    val completedPomodoros: Int = 0
)
```

### 番茄鐘會話實體

```kotlin
// entities/SessionEntity.kt
@Entity(
    tableName = "sessions",
    foreignKeys = [
        ForeignKey(
            entity = TaskEntity::class,
            parentColumns = ["id"],
            childColumns = ["task_id"],
            onDelete = ForeignKey.SET_NULL
        )
    ],
    indices = [Index(value = ["task_id"])]
)
data class SessionEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,
    
    @ColumnInfo(name = "task_id")
    val taskId: Long? = null,
    
    @ColumnInfo(name = "session_type")
    val sessionType: SessionType,
    
    @ColumnInfo(name = "start_time")
    val startTime: Long,
    
    @ColumnInfo(name = "end_time")
    val endTime: Long? = null,
    
    @ColumnInfo(name = "duration_millis")
    val durationMillis: Long,
    
    @ColumnInfo(name = "is_completed")
    val isCompleted: Boolean = false,
    
    @ColumnInfo(name = "notes")
    val notes: String = ""
)

enum class SessionType {
    FOCUS,    // 專注時間
    BREAK     // 休息時間
}
```

### DAO 介面設計

```kotlin
// dao/TaskDao.kt
@Dao
interface TaskDao {
    
    @Query("SELECT * FROM tasks ORDER BY created_at DESC")
    fun getAllTasks(): Flow<List<TaskEntity>>
    
    @Query("SELECT * FROM tasks WHERE is_completed = 0 ORDER BY created_at DESC")
    fun getActiveTasks(): Flow<List<TaskEntity>>
    
    @Query("SELECT * FROM tasks WHERE is_current = 1 LIMIT 1")
    fun getCurrentTask(): Flow<TaskEntity?>
    
    @Query("SELECT * FROM tasks WHERE id = :id")
    suspend fun getTaskById(id: Long): TaskEntity?
    
    @Insert
    suspend fun insertTask(task: TaskEntity): Long
    
    @Update
    suspend fun updateTask(task: TaskEntity)
    
    @Delete
    suspend fun deleteTask(task: TaskEntity)
    
    @Query("UPDATE tasks SET is_current = 0")
    suspend fun clearCurrentTask()
    
    @Query("UPDATE tasks SET is_current = 1 WHERE id = :taskId")
    suspend fun setCurrentTask(taskId: Long)
    
    @Transaction
    suspend fun setAsCurrentTask(taskId: Long) {
        clearCurrentTask()
        setCurrentTask(taskId)
    }
    
    @Query("UPDATE tasks SET completed_pomodoros = completed_pomodoros + 1 WHERE id = :taskId")
    suspend fun incrementPomodoroCount(taskId: Long)
}
```

```kotlin
// dao/SessionDao.kt
@Dao
interface SessionDao {
    
    @Query("SELECT * FROM sessions ORDER BY start_time DESC")
    fun getAllSessions(): Flow<List<SessionEntity>>
    
    @Query("""
        SELECT * FROM sessions 
        WHERE DATE(start_time / 1000, 'unixepoch') = DATE(:date / 1000, 'unixepoch')
        ORDER BY start_time DESC
    """)
    fun getSessionsByDate(date: Long): Flow<List<SessionEntity>>
    
    @Query("""
        SELECT * FROM sessions 
        WHERE task_id = :taskId 
        ORDER BY start_time DESC
    """)
    fun getSessionsByTask(taskId: Long): Flow<List<SessionEntity>>
    
    @Query("""
        SELECT COUNT(*) FROM sessions 
        WHERE session_type = :type AND is_completed = 1
        AND DATE(start_time / 1000, 'unixepoch') = DATE(:date / 1000, 'unixepoch')
    """)
    suspend fun getCompletedSessionsCount(type: SessionType, date: Long): Int
    
    @Insert
    suspend fun insertSession(session: SessionEntity): Long
    
    @Update
    suspend fun updateSession(session: SessionEntity)
    
    @Delete
    suspend fun deleteSession(session: SessionEntity)
    
    @Query("UPDATE sessions SET is_completed = 1, end_time = :endTime WHERE id = :sessionId")
    suspend fun completeSession(sessionId: Long, endTime: Long)
}
```

### 類型轉換器

```kotlin
// database/Converters.kt
class Converters {
    
    @TypeConverter
    fun fromSessionType(sessionType: SessionType): String {
        return sessionType.name
    }
    
    @TypeConverter
    fun toSessionType(sessionType: String): SessionType {
        return SessionType.valueOf(sessionType)
    }
}
```

### 資料庫實作

```kotlin
// database/AppDatabase.kt
@Database(
    entities = [
        TaskEntity::class,
        SessionEntity::class
    ],
    version = 1,
    exportSchema = false
)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
    
    abstract fun taskDao(): TaskDao
    abstract fun sessionDao(): SessionDao
    
    companion object {
        const val DATABASE_NAME = "pomodoro_database"
    }
}
```

## 🔄 Repository 模式實作

### Repository 介面定義

```kotlin
// domain/repository/TaskRepository.kt
interface TaskRepository {
    fun getAllTasks(): Flow<List<Task>>
    fun getActiveTasks(): Flow<List<Task>>
    fun getCurrentTask(): Flow<Task?>
    suspend fun getTaskById(id: Long): Task?
    suspend fun insertTask(task: Task): Long
    suspend fun updateTask(task: Task)
    suspend fun deleteTask(taskId: Long)
    suspend fun setCurrentTask(taskId: Long)
    suspend fun incrementPomodoroCount(taskId: Long)
}

// domain/repository/SessionRepository.kt
interface SessionRepository {
    fun getAllSessions(): Flow<List<PomodoroSession>>
    fun getSessionsByDate(date: Long): Flow<List<PomodoroSession>>
    fun getSessionsByTask(taskId: Long): Flow<List<PomodoroSession>>
    suspend fun getCompletedSessionsCount(type: SessionType, date: Long): Int
    suspend fun insertSession(session: PomodoroSession): Long
    suspend fun updateSession(session: PomodoroSession)
    suspend fun deleteSession(sessionId: Long)
    suspend fun completeSession(sessionId: Long)
}
```

### Repository 實作

```kotlin
// data/repository/TaskRepositoryImpl.kt
@Singleton
class TaskRepositoryImpl @Inject constructor(
    private val taskDao: TaskDao,
    private val taskMapper: TaskMapper
) : TaskRepository {
    
    override fun getAllTasks(): Flow<List<Task>> {
        return taskDao.getAllTasks().map { entities ->
            entities.map { taskMapper.entityToDomain(it) }
        }
    }
    
    override fun getActiveTasks(): Flow<List<Task>> {
        return taskDao.getActiveTasks().map { entities ->
            entities.map { taskMapper.entityToDomain(it) }
        }
    }
    
    override fun getCurrentTask(): Flow<Task?> {
        return taskDao.getCurrentTask().map { entity ->
            entity?.let { taskMapper.entityToDomain(it) }
        }
    }
    
    override suspend fun getTaskById(id: Long): Task? {
        return taskDao.getTaskById(id)?.let { 
            taskMapper.entityToDomain(it) 
        }
    }
    
    override suspend fun insertTask(task: Task): Long {
        val entity = taskMapper.domainToEntity(task)
        return taskDao.insertTask(entity)
    }
    
    override suspend fun updateTask(task: Task) {
        val entity = taskMapper.domainToEntity(task)
        taskDao.updateTask(entity)
    }
    
    override suspend fun deleteTask(taskId: Long) {
        val task = taskDao.getTaskById(taskId)
        task?.let { taskDao.deleteTask(it) }
    }
    
    override suspend fun setCurrentTask(taskId: Long) {
        taskDao.setAsCurrentTask(taskId)
    }
    
    override suspend fun incrementPomodoroCount(taskId: Long) {
        taskDao.incrementPomodoroCount(taskId)
    }
}
```

```kotlin
// data/repository/SessionRepositoryImpl.kt
@Singleton
class SessionRepositoryImpl @Inject constructor(
    private val sessionDao: SessionDao,
    private val sessionMapper: SessionMapper
) : SessionRepository {
    
    override fun getAllSessions(): Flow<List<PomodoroSession>> {
        return sessionDao.getAllSessions().map { entities ->
            entities.map { sessionMapper.entityToDomain(it) }
        }
    }
    
    override fun getSessionsByDate(date: Long): Flow<List<PomodoroSession>> {
        return sessionDao.getSessionsByDate(date).map { entities ->
            entities.map { sessionMapper.entityToDomain(it) }
        }
    }
    
    override fun getSessionsByTask(taskId: Long): Flow<List<PomodoroSession>> {
        return sessionDao.getSessionsByTask(taskId).map { entities ->
            entities.map { sessionMapper.entityToDomain(it) }
        }
    }
    
    override suspend fun getCompletedSessionsCount(
        type: SessionType, 
        date: Long
    ): Int {
        return sessionDao.getCompletedSessionsCount(type, date)
    }
    
    override suspend fun insertSession(session: PomodoroSession): Long {
        val entity = sessionMapper.domainToEntity(session)
        return sessionDao.insertSession(entity)
    }
    
    override suspend fun updateSession(session: PomodoroSession) {
        val entity = sessionMapper.domainToEntity(session)
        sessionDao.updateSession(entity)
    }
    
    override suspend fun deleteSession(sessionId: Long) {
        val session = sessionDao.getAllSessions().first()
            .find { it.id == sessionId }
        session?.let { sessionDao.deleteSession(it) }
    }
    
    override suspend fun completeSession(sessionId: Long) {
        sessionDao.completeSession(sessionId, System.currentTimeMillis())
    }
}
```

## 🔄 資料轉換和映射

### Domain 模型定義

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
    val completedPomodoros: Int = 0
) {
    val completionPercentage: Float
        get() = if (estimatedPomodoros > 0) {
            (completedPomodoros.toFloat() / estimatedPomodoros.toFloat()).coerceIn(0f, 1f)
        } else 0f
}

// domain/model/PomodoroSession.kt
data class PomodoroSession(
    val id: Long = 0,
    val taskId: Long? = null,
    val sessionType: SessionType,
    val startTime: Long,
    val endTime: Long? = null,
    val durationMillis: Long,
    val isCompleted: Boolean = false,
    val notes: String = ""
) {
    val actualDurationMillis: Long
        get() = endTime?.let { it - startTime } ?: 0L
        
    val isRunning: Boolean
        get() = endTime == null && !isCompleted
}
```

### 資料映射器

```kotlin
// data/mapper/TaskMapper.kt
@Singleton
class TaskMapper @Inject constructor() {
    
    fun entityToDomain(entity: TaskEntity): Task {
        return Task(
            id = entity.id,
            name = entity.name,
            description = entity.description,
            isCompleted = entity.isCompleted,
            isCurrent = entity.isCurrent,
            createdAt = entity.createdAt,
            updatedAt = entity.updatedAt,
            estimatedPomodoros = entity.estimatedPomodoros,
            completedPomodoros = entity.completedPomodoros
        )
    }
    
    fun domainToEntity(domain: Task): TaskEntity {
        return TaskEntity(
            id = domain.id,
            name = domain.name,
            description = domain.description,
            isCompleted = domain.isCompleted,
            isCurrent = domain.isCurrent,
            createdAt = domain.createdAt,
            updatedAt = domain.updatedAt,
            estimatedPomodoros = domain.estimatedPomodoros,
            completedPomodoros = domain.completedPomodoros
        )
    }
    
    fun domainToUiData(domain: Task): TaskUIData {
        return TaskUIData(
            id = domain.id,
            name = domain.name,
            description = domain.description,
            isCompleted = domain.isCompleted,
            isCurrent = domain.isCurrent,
            completionPercentage = domain.completionPercentage,
            estimatedPomodoros = domain.estimatedPomodoros,
            completedPomodoros = domain.completedPomodoros
        )
    }
}

// data/mapper/SessionMapper.kt
@Singleton
class SessionMapper @Inject constructor() {
    
    fun entityToDomain(entity: SessionEntity): PomodoroSession {
        return PomodoroSession(
            id = entity.id,
            taskId = entity.taskId,
            sessionType = entity.sessionType,
            startTime = entity.startTime,
            endTime = entity.endTime,
            durationMillis = entity.durationMillis,
            isCompleted = entity.isCompleted,
            notes = entity.notes
        )
    }
    
    fun domainToEntity(domain: PomodoroSession): SessionEntity {
        return SessionEntity(
            id = domain.id,
            taskId = domain.taskId,
            sessionType = domain.sessionType,
            startTime = domain.startTime,
            endTime = domain.endTime,
            durationMillis = domain.durationMillis,
            isCompleted = domain.isCompleted,
            notes = domain.notes
        )
    }
}
```

## 📊 資料快取策略

### 記憶體快取實作

```kotlin
// data/cache/MemoryCache.kt
@Singleton
class MemoryCache @Inject constructor() {
    
    private val taskCache = mutableMapOf<Long, Task>()
    private val sessionCache = mutableMapOf<Long, PomodoroSession>()
    
    // Task 快取操作
    fun cacheTask(task: Task) {
        taskCache[task.id] = task
    }
    
    fun getCachedTask(id: Long): Task? {
        return taskCache[id]
    }
    
    fun removeCachedTask(id: Long) {
        taskCache.remove(id)
    }
    
    fun clearTaskCache() {
        taskCache.clear()
    }
    
    // Session 快取操作
    fun cacheSession(session: PomodoroSession) {
        sessionCache[session.id] = session
    }
    
    fun getCachedSession(id: Long): PomodoroSession? {
        return sessionCache[id]
    }
    
    fun removeCachedSession(id: Long) {
        sessionCache.remove(id)
    }
    
    fun clearSessionCache() {
        sessionCache.clear()
    }
}
```

### 智能快取 Repository

```kotlin
// data/repository/CachedTaskRepository.kt
@Singleton
class CachedTaskRepository @Inject constructor(
    private val taskDao: TaskDao,
    private val taskMapper: TaskMapper,
    private val memoryCache: MemoryCache
) : TaskRepository {
    
    override suspend fun getTaskById(id: Long): Task? {
        // 先檢查快取
        memoryCache.getCachedTask(id)?.let { return it }
        
        // 從資料庫獲取
        val entity = taskDao.getTaskById(id)
        return entity?.let { 
            val task = taskMapper.entityToDomain(it)
            // 快取結果
            memoryCache.cacheTask(task)
            task
        }
    }
    
    override suspend fun updateTask(task: Task) {
        // 更新資料庫
        val entity = taskMapper.domainToEntity(task)
        taskDao.updateTask(entity)
        
        // 更新快取
        memoryCache.cacheTask(task)
    }
    
    override suspend fun deleteTask(taskId: Long) {
        val task = taskDao.getTaskById(taskId)
        task?.let { 
            taskDao.deleteTask(it)
            // 清除快取
            memoryCache.removeCachedTask(taskId)
        }
    }
    
    // 其他方法實作...
}
```

## 🔄 資料同步

### 離線優先策略

```kotlin
// data/sync/DataSyncManager.kt
@Singleton
class DataSyncManager @Inject constructor(
    private val localTaskRepository: TaskRepository,
    private val remoteTaskRepository: RemoteTaskRepository, // 預留遠端資料源
    private val connectivityManager: ConnectivityManager
) {
    
    private val _syncState = MutableStateFlow(SyncState.IDLE)
    val syncState: StateFlow<SyncState> = _syncState.asStateFlow()
    
    suspend fun syncTasks() {
        if (!isNetworkAvailable()) {
            _syncState.value = SyncState.OFFLINE
            return
        }
        
        _syncState.value = SyncState.SYNCING
        
        try {
            // 同步邏輯（預留）
            // 1. 上傳本地變更
            // 2. 下載遠端變更
            // 3. 解決衝突
            
            _syncState.value = SyncState.SUCCESS
        } catch (e: Exception) {
            _syncState.value = SyncState.ERROR
        }
    }
    
    private fun isNetworkAvailable(): Boolean {
        val networkInfo = connectivityManager.activeNetworkInfo
        return networkInfo?.isConnected == true
    }
}

enum class SyncState {
    IDLE,
    SYNCING,
    SUCCESS,
    ERROR,
    OFFLINE
}
```

## 🧪 資料層測試

### Repository 測試

```kotlin
// test/repository/TaskRepositoryTest.kt
@ExperimentalCoroutinesTest
class TaskRepositoryTest {
    
    @get:Rule
    val instantExecutorRule = InstantTaskExecutorRule()
    
    private lateinit var database: AppDatabase
    private lateinit var taskDao: TaskDao
    private lateinit var repository: TaskRepository
    private lateinit var taskMapper: TaskMapper
    
    @Before
    fun setup() {
        database = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            AppDatabase::class.java
        ).allowMainThreadQueries().build()
        
        taskDao = database.taskDao()
        taskMapper = TaskMapper()
        repository = TaskRepositoryImpl(taskDao, taskMapper)
    }
    
    @After
    fun teardown() {
        database.close()
    }
    
    @Test
    fun `insertTask should return task id`() = runTest {
        // Given
        val task = Task(name = "Test Task", description = "Test Description")
        
        // When
        val taskId = repository.insertTask(task)
        
        // Then
        assertThat(taskId).isGreaterThan(0)
    }
    
    @Test
    fun `getAllTasks should return all tasks`() = runTest {
        // Given
        val task1 = Task(name = "Task 1")
        val task2 = Task(name = "Task 2")
        repository.insertTask(task1)
        repository.insertTask(task2)
        
        // When
        val tasks = repository.getAllTasks().first()
        
        // Then
        assertThat(tasks).hasSize(2)
        assertThat(tasks.map { it.name }).containsExactly("Task 2", "Task 1") // 按建立時間降序
    }
}
```

## 📚 延伸閱讀

- [Room Persistence Library](https://developer.android.com/training/data-storage/room)
- [Repository Pattern](https://developer.android.com/codelabs/android-room-with-a-view)
- [Data Caching Strategies](https://developer.android.com/topic/architecture/data-layer)

## 🎯 下一章預告

在下一章中，我們將探討 **領域層設計**，學習：

- Use Case 設計和實作
- 業務邏輯封裝
- 領域模型設計
- 領域服務實作

準備好建構清晰的業務邏輯層了嗎？🏢