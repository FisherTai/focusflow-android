# 第 7 章：測試實作

## 📋 本章目標

- 建立完整的測試策略
- 實作單元測試和整合測試
- 掌握 Android 測試框架
- 實現高測試覆蓋率

## 🧪 測試金字塔

### 測試層級

```
        🔺 UI Tests (E2E)
       🔺🔺 Integration Tests
      🔺🔺🔺 Unit Tests
```

- **單元測試 (70%)**：快速、獨立、可靠
- **整合測試 (20%)**：驗證元件協作
- **UI 測試 (10%)**：端到端使用者流程

### 測試原則

1. **FIRST 原則**
   - **Fast** - 快速執行
   - **Independent** - 相互獨立
   - **Repeatable** - 可重複執行
   - **Self-Validating** - 自動驗證
   - **Timely** - 及時編寫

2. **AAA 模式**
   - **Arrange** - 準備測試資料
   - **Act** - 執行測試動作
   - **Assert** - 驗證結果

## 🔧 測試環境設置

### 測試依賴配置

```kotlin
// app/build.gradle.kts - 測試相關依賴
dependencies {
    // 單元測試
    testImplementation("junit:junit:4.13.2")
    testImplementation("io.mockk:mockk:1.13.8")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
    testImplementation("app.cash.turbine:turbine:1.0.0")
    testImplementation("com.google.truth:truth:1.1.4")
    testImplementation("androidx.arch.core:core-testing:2.2.0")
    
    // Android 單元測試
    testImplementation("androidx.test:core:1.5.0")
    testImplementation("androidx.test.ext:junit:1.1.5")
    testImplementation("androidx.test:runner:1.5.2")
    testImplementation("org.robolectric:robolectric:4.11.1")
    
    // Hilt 測試
    testImplementation("com.google.dagger:hilt-android-testing:2.48")
    kaptTest("com.google.dagger:hilt-android-compiler:2.48")
    
    // Room 測試
    testImplementation("androidx.room:room-testing:2.6.0")
    
    // 儀器測試
    androidTestImplementation("androidx.test.ext:junit:1.1.5")
    androidTestImplementation("androidx.test.espresso:espresso-core:3.5.1")
    androidTestImplementation("androidx.test:rules:1.5.0")
    androidTestImplementation("androidx.test:runner:1.5.2")
    
    // Compose 測試
    androidTestImplementation(platform("androidx.compose:compose-bom:2023.10.01"))
    androidTestImplementation("androidx.compose.ui:ui-test-junit4")
    debugImplementation("androidx.compose.ui:ui-test-manifest")
    
    // Hilt Android 測試
    androidTestImplementation("com.google.dagger:hilt-android-testing:2.48")
    kaptAndroidTest("com.google.dagger:hilt-android-compiler:2.48")
}
```

### 測試配置

```kotlin
// src/test/resources/robolectric.properties
sdk=33
qualifiers=w360dp-h640dp-xhdpi

// src/test/java/com/fisher/pomodoroapp/TestApplication.kt
@HiltAndroidApp
class TestApplication : Application()
```

## 🔬 單元測試實作

### ViewModel 測試

```kotlin
// test/ui/home/HomeViewModelTest.kt
@ExperimentalCoroutinesTest
class HomeViewModelTest {
    
    @get:Rule
    val instantExecutorRule = InstantTaskExecutorRule()
    
    // Mocks
    private val mockStartTimerUseCase = mockk<StartTimerUseCase>()
    private val mockPauseTimerUseCase = mockk<PauseTimerUseCase>()
    private val mockResetTimerUseCase = mockk<ResetTimerUseCase>()
    private val mockGetCurrentTaskUseCase = mockk<GetCurrentTaskUseCase>()
    
    private lateinit var viewModel: HomeViewModel
    private val testDispatcher = StandardTestDispatcher()
    
    @Before
    fun setup() {
        Dispatchers.setMain(testDispatcher)
        
        // 設置默認行為
        every { mockGetCurrentTaskUseCase() } returns flowOf(null)
        
        viewModel = HomeViewModel(
            startTimerUseCase = mockStartTimerUseCase,
            pauseTimerUseCase = mockPauseTimerUseCase,
            resetTimerUseCase = mockResetTimerUseCase,
            getCurrentTaskUseCase = mockGetCurrentTaskUseCase
        )
    }
    
    @After
    fun tearDown() {
        Dispatchers.resetMain()
    }
    
    @Test
    fun `initial state should be idle`() {
        // Given & When
        val initialState = viewModel.uiState.value
        
        // Then
        assertThat(initialState.timerState).isEqualTo(TimerState.IDLE)
        assertThat(initialState.currentMode).isEqualTo(TimerMode.FOCUS)
        assertThat(initialState.timeRemaining).isEqualTo(25 * 60 * 1000L)
    }
    
    @Test
    fun `startTimer should update state to running on success`() = runTest {
        // Given
        coEvery { mockStartTimerUseCase(SessionType.FOCUS) } returns Result.success(1L)
        
        // When
        viewModel.startTimer()
        testDispatcher.scheduler.advanceUntilIdle()
        
        // Then
        val state = viewModel.uiState.value
        assertThat(state.timerState).isEqualTo(TimerState.RUNNING)
        assertThat(state.error).isNull()
        
        coVerify { mockStartTimerUseCase(SessionType.FOCUS) }
    }
    
    @Test
    fun `startTimer should show error on failure`() = runTest {
        // Given
        val error = TimerException.AlreadyRunning
        coEvery { mockStartTimerUseCase(SessionType.FOCUS) } returns Result.failure(error)
        
        // When
        viewModel.startTimer()
        testDispatcher.scheduler.advanceUntilIdle()
        
        // Then
        val state = viewModel.uiState.value
        assertThat(state.timerState).isEqualTo(TimerState.IDLE)
        assertThat(state.error).isEqualTo(error.message)
    }
    
    @Test
    fun `pauseTimer should update state to paused`() = runTest {
        // Given
        coEvery { mockPauseTimerUseCase() } returns Result.success(Unit)
        
        // When
        viewModel.pauseTimer()
        testDispatcher.scheduler.advanceUntilIdle()
        
        // Then
        val state = viewModel.uiState.value
        assertThat(state.timerState).isEqualTo(TimerState.PAUSED)
        
        coVerify { mockPauseTimerUseCase() }
    }
    
    @Test
    fun `should observe current task changes`() = runTest {
        // Given
        val task = Task(id = 1, name = "Test Task")
        val taskFlow = MutableStateFlow<Task?>(null)
        every { mockGetCurrentTaskUseCase() } returns taskFlow
        
        // Create new ViewModel to trigger initialization
        val newViewModel = HomeViewModel(
            startTimerUseCase = mockStartTimerUseCase,
            pauseTimerUseCase = mockPauseTimerUseCase,
            resetTimerUseCase = mockResetTimerUseCase,
            getCurrentTaskUseCase = mockGetCurrentTaskUseCase
        )
        
        // When
        taskFlow.value = task
        testDispatcher.scheduler.advanceUntilIdle()
        
        // Then
        val state = newViewModel.uiState.value
        assertThat(state.currentTask).isEqualTo(task)
    }
}
```

### Repository 測試

```kotlin
// test/repository/TaskRepositoryTest.kt
@ExperimentalCoroutinesTest
class TaskRepositoryTest {
    
    @get:Rule
    val instantExecutorRule = InstantTaskExecutorRule()
    
    private lateinit var database: AppDatabase
    private lateinit var taskDao: TaskDao
    private lateinit var taskMapper: TaskMapper
    private lateinit var repository: TaskRepositoryImpl
    
    @Before
    fun setup() {
        // 使用內存資料庫
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
    fun `getAllTasks should return all tasks in descending order`() = runTest {
        // Given
        val task1 = Task(name = "Task 1")
        val task2 = Task(name = "Task 2")
        
        repository.insertTask(task1)
        Thread.sleep(1) // 確保時間戳不同
        repository.insertTask(task2)
        
        // When
        val tasks = repository.getAllTasks().first()
        
        // Then
        assertThat(tasks).hasSize(2)
        assertThat(tasks[0].name).isEqualTo("Task 2") // 最新的在前面
        assertThat(tasks[1].name).isEqualTo("Task 1")
    }
    
    @Test
    fun `setCurrentTask should update task current status`() = runTest {
        // Given
        val task1 = Task(name = "Task 1")
        val task2 = Task(name = "Task 2")
        
        val task1Id = repository.insertTask(task1)
        val task2Id = repository.insertTask(task2)
        
        // When
        repository.setCurrentTask(task1Id)
        
        // Then
        val currentTask = repository.getCurrentTask().first()
        val allTasks = repository.getAllTasks().first()
        
        assertThat(currentTask?.id).isEqualTo(task1Id)
        assertThat(allTasks.find { it.id == task1Id }?.isCurrent).isTrue()
        assertThat(allTasks.find { it.id == task2Id }?.isCurrent).isFalse()
    }
    
    @Test
    fun `setCurrentTask should clear previous current task`() = runTest {
        // Given
        val task1 = Task(name = "Task 1")
        val task2 = Task(name = "Task 2")
        
        val task1Id = repository.insertTask(task1)
        val task2Id = repository.insertTask(task2)
        
        repository.setCurrentTask(task1Id)
        
        // When
        repository.setCurrentTask(task2Id)
        
        // Then
        val allTasks = repository.getAllTasks().first()
        
        assertThat(allTasks.find { it.id == task1Id }?.isCurrent).isFalse()
        assertThat(allTasks.find { it.id == task2Id }?.isCurrent).isTrue()
    }
    
    @Test
    fun `incrementPomodoroCount should increase completed pomodoros`() = runTest {
        // Given
        val task = Task(name = "Test Task", completedPomodoros = 2)
        val taskId = repository.insertTask(task)
        
        // When
        repository.incrementPomodoroCount(taskId)
        
        // Then
        val updatedTask = repository.getTaskById(taskId)
        assertThat(updatedTask?.completedPomodoros).isEqualTo(3)
    }
}
```

### Use Case 測試

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
        val priority = TaskPriority.HIGH
        val expectedTaskId = 1L
        
        every { 
            mockTaskValidator.validateTaskData(taskName, estimatedPomodoros) 
        } returns Result.success(Unit)
        
        coEvery { mockTaskRepository.insertTask(any()) } returns expectedTaskId
        
        // When
        val result = addTaskUseCase(taskName, description, estimatedPomodoros, priority)
        
        // Then
        assertThat(result.isSuccess).isTrue()
        assertThat(result.getOrNull()).isEqualTo(expectedTaskId)
        
        coVerify {
            mockTaskRepository.insertTask(
                match { task ->
                    task.name == taskName &&
                    task.description == description &&
                    task.estimatedPomodoros == estimatedPomodoros &&
                    task.priority == priority
                }
            )
        }
    }
    
    @Test
    fun `should fail when task validation fails`() = runTest {
        // Given
        val invalidName = ""
        val validationError = TaskException.InvalidTaskName("任務名稱不能為空")
        
        every { 
            mockTaskValidator.validateTaskData(invalidName, any()) 
        } returns Result.failure(validationError)
        
        // When
        val result = addTaskUseCase(invalidName)
        
        // Then
        assertThat(result.isFailure).isTrue()
        assertThat(result.exceptionOrNull()).isEqualTo(validationError)
        
        coVerify(exactly = 0) { mockTaskRepository.insertTask(any()) }
    }
    
    @Test
    fun `should handle repository exception`() = runTest {
        // Given
        val taskName = "Test Task"
        val repositoryException = RuntimeException("Database error")
        
        every { 
            mockTaskValidator.validateTaskData(taskName, any()) 
        } returns Result.success(Unit)
        
        coEvery { mockTaskRepository.insertTask(any()) } throws repositoryException
        
        // When
        val result = addTaskUseCase(taskName)
        
        // Then
        assertThat(result.isFailure).isTrue()
        assertThat(result.exceptionOrNull()).isEqualTo(repositoryException)
    }
}
```

### StateFlow 測試

```kotlin
// test/util/FlowTestExtensions.kt
@ExperimentalCoroutinesTest
suspend fun <T> Flow<T>.test(
    timeout: Duration = 1.seconds,
    validate: suspend FlowTurbine<T>.() -> Unit
) {
    return this.test(timeout, validate)
}

// 使用 Turbine 測試 StateFlow
@Test
fun `should emit correct states when timer starts and completes`() = runTest {
    // Given
    val mockTimerService = mockk<TimerService>()
    val timerStateFlow = MutableStateFlow(
        TimerServiceState(
            timeRemaining = 25 * 60 * 1000L,
            timerState = TimerState.IDLE,
            currentMode = TimerMode.FOCUS
        )
    )
    
    every { mockTimerService.timerStateFlow } returns timerStateFlow
    
    // When & Then
    timerStateFlow.test {
        // 初始狀態
        val initialState = awaitItem()
        assertThat(initialState.timerState).isEqualTo(TimerState.IDLE)
        
        // 開始計時
        timerStateFlow.value = timerStateFlow.value.copy(timerState = TimerState.RUNNING)
        val runningState = awaitItem()
        assertThat(runningState.timerState).isEqualTo(TimerState.RUNNING)
        
        // 計時完成
        timerStateFlow.value = timerStateFlow.value.copy(timerState = TimerState.COMPLETED)
        val completedState = awaitItem()
        assertThat(completedState.timerState).isEqualTo(TimerState.COMPLETED)
    }
}
```

## 🔗 整合測試

### 資料庫整合測試

```kotlin
// androidTest/database/TaskDaoTest.kt
@RunWith(AndroidJUnit4::class)
class TaskDaoTest {
    
    private lateinit var database: AppDatabase
    private lateinit var taskDao: TaskDao
    
    @Before
    fun createDb() {
        val context = ApplicationProvider.getApplicationContext<Context>()
        database = Room.inMemoryDatabaseBuilder(context, AppDatabase::class.java)
            .allowMainThreadQueries()
            .build()
        taskDao = database.taskDao()
    }
    
    @After
    fun closeDb() {
        database.close()
    }
    
    @Test
    fun insertAndGetTask() = runTest {
        // Given
        val task = TaskEntity(
            name = "Test Task",
            description = "Test Description",
            estimatedPomodoros = 3
        )
        
        // When
        val taskId = taskDao.insertTask(task)
        val retrievedTask = taskDao.getTaskById(taskId)
        
        // Then
        assertThat(retrievedTask?.name).isEqualTo("Test Task")
        assertThat(retrievedTask?.description).isEqualTo("Test Description")
        assertThat(retrievedTask?.estimatedPomodoros).isEqualTo(3)
    }
    
    @Test
    fun setCurrentTaskShouldClearPreviousCurrentTask() = runTest {
        // Given
        val task1 = TaskEntity(name = "Task 1")
        val task2 = TaskEntity(name = "Task 2")
        
        val task1Id = taskDao.insertTask(task1)
        val task2Id = taskDao.insertTask(task2)
        
        // When
        taskDao.setAsCurrentTask(task1Id)
        taskDao.setAsCurrentTask(task2Id)
        
        // Then
        val allTasks = taskDao.getAllTasks().first()
        val task1Current = allTasks.find { it.id == task1Id }?.isCurrent
        val task2Current = allTasks.find { it.id == task2Id }?.isCurrent
        
        assertThat(task1Current).isFalse()
        assertThat(task2Current).isTrue()
    }
}
```

### Repository 整合測試

```kotlin
// androidTest/repository/TaskRepositoryIntegrationTest.kt
@HiltAndroidTest
@RunWith(AndroidJUnit4::class)
class TaskRepositoryIntegrationTest {
    
    @get:Rule
    val hiltRule = HiltAndroidRule(this)
    
    @Inject
    lateinit var database: AppDatabase
    
    @Inject
    lateinit var taskRepository: TaskRepository
    
    @Before
    fun init() {
        hiltRule.inject()
    }
    
    @After
    fun cleanup() {
        database.clearAllTables()
    }
    
    @Test
    fun repositoryIntegrationTest() = runTest {
        // Given
        val task = Task(
            name = "Integration Test Task",
            description = "Testing repository integration",
            estimatedPomodoros = 5
        )
        
        // When
        val taskId = taskRepository.insertTask(task)
        taskRepository.setCurrentTask(taskId)
        
        // Then
        val currentTask = taskRepository.getCurrentTask().first()
        val allTasks = taskRepository.getAllTasks().first()
        
        assertThat(currentTask?.id).isEqualTo(taskId)
        assertThat(currentTask?.name).isEqualTo("Integration Test Task")
        assertThat(allTasks).hasSize(1)
        assertThat(allTasks[0].isCurrent).isTrue()
    }
}
```

## 🎨 UI 測試

### Compose UI 測試

```kotlin
// androidTest/ui/HomeScreenTest.kt
@HiltAndroidTest
@RunWith(AndroidJUnit4::class)
class HomeScreenTest {
    
    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)
    
    @get:Rule(order = 1)
    val composeTestRule = createAndroidComposeRule<MainActivity>()
    
    @Before
    fun init() {
        hiltRule.inject()
    }
    
    @Test
    fun homeScreen_displaysTimerCorrectly() {
        composeTestRule.setContent {
            PomodoroAppTheme {
                HomeScreen()
            }
        }
        
        // 驗證計時器顯示
        composeTestRule
            .onNodeWithText("25:00")
            .assertIsDisplayed()
        
        composeTestRule
            .onNodeWithText("專注時間")
            .assertIsDisplayed()
        
        // 驗證開始按鈕存在
        composeTestRule
            .onNodeWithText("開始")
            .assertIsDisplayed()
            .assertIsEnabled()
    }
    
    @Test
    fun clickStartButton_startsTimer() {
        composeTestRule.setContent {
            PomodoroAppTheme {
                HomeScreen()
            }
        }
        
        // 點擊開始按鈕
        composeTestRule
            .onNodeWithText("開始")
            .performClick()
        
        // 驗證按鈕變為暫停
        composeTestRule
            .onNodeWithText("暫停")
            .assertIsDisplayed()
    }
    
    @Test
    fun timerRunning_showsPulsingIndicator() {
        composeTestRule.setContent {
            val uiState = HomeUiState(
                timerState = TimerState.RUNNING,
                timeRemaining = 24 * 60 * 1000L, // 24:00
                currentMode = TimerMode.FOCUS
            )
            
            HomeContent(
                uiState = uiState,
                onStartTimer = { },
                onPauseTimer = { },
                onResetTimer = { },
                onModeSwitch = { }
            )
        }
        
        // 驗證運行狀態顯示
        composeTestRule
            .onNodeWithText("24:00")
            .assertIsDisplayed()
        
        composeTestRule
            .onNodeWithText("暫停")
            .assertIsDisplayed()
    }
}
```

### 任務列表 UI 測試

```kotlin
// androidTest/ui/TaskListScreenTest.kt
@HiltAndroidTest
@RunWith(AndroidJUnit4::class)
class TaskListScreenTest {
    
    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)
    
    @get:Rule(order = 1)
    val composeTestRule = createAndroidComposeRule<MainActivity>()
    
    @Inject
    lateinit var fakeTaskRepository: FakeTaskRepository
    
    @Before
    fun init() {
        hiltRule.inject()
        fakeTaskRepository.clearAllTasks()
    }
    
    @Test
    fun taskListScreen_displaysTasksCorrectly() {
        // Given
        val tasks = listOf(
            TaskUIData(id = 1, name = "Task 1", description = "Description 1"),
            TaskUIData(id = 2, name = "Task 2", description = "Description 2", isCompleted = true)
        )
        
        composeTestRule.setContent {
            PomodoroAppTheme {
                TaskListContent(
                    uiState = TaskListUiState(tasks = tasks),
                    onAddTask = { },
                    onDeleteTask = { },
                    onToggleTask = { },
                    onSetCurrentTask = { }
                )
            }
        }
        
        // Then
        composeTestRule
            .onNodeWithText("Task 1")
            .assertIsDisplayed()
        
        composeTestRule
            .onNodeWithText("Task 2")
            .assertIsDisplayed()
        
        composeTestRule
            .onNodeWithText("任務清單")
            .assertIsDisplayed()
    }
    
    @Test
    fun addTaskButton_opensDialog() {
        composeTestRule.setContent {
            PomodoroAppTheme {
                TaskListScreen()
            }
        }
        
        // 點擊新增按鈕
        composeTestRule
            .onNodeWithContentDescription("新增任務")
            .performClick()
        
        // 驗證對話框出現
        composeTestRule
            .onNodeWithText("新增任務")
            .assertIsDisplayed()
        
        composeTestRule
            .onNodeWithText("取消")
            .assertIsDisplayed()
        
        composeTestRule
            .onNodeWithText("確認")
            .assertIsDisplayed()
    }
    
    @Test
    fun swipeTask_showsActionButtons() {
        val task = TaskUIData(id = 1, name = "Swipe Test Task")
        
        composeTestRule.setContent {
            PomodoroAppTheme {
                SwipeableTaskItem(
                    task = task,
                    onToggle = { },
                    onDelete = { },
                    onSetCurrent = { }
                )
            }
        }
        
        // 執行滑動操作
        composeTestRule
            .onNodeWithText("Swipe Test Task")
            .performTouchInput {
                swipeLeft()
            }
        
        // 驗證操作按鈕出現
        composeTestRule
            .onNodeWithContentDescription("設為目前任務")
            .assertIsDisplayed()
        
        composeTestRule
            .onNodeWithContentDescription("刪除任務")
            .assertIsDisplayed()
    }
}
```

### 端到端測試

```kotlin
// androidTest/e2e/PomodoroFlowTest.kt
@HiltAndroidTest
@RunWith(AndroidJUnit4::class)
class PomodoroFlowTest {
    
    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)
    
    @get:Rule(order = 1)
    val composeTestRule = createAndroidComposeRule<MainActivity>()
    
    @Before
    fun init() {
        hiltRule.inject()
    }
    
    @Test
    fun completePomodoroCycle_addsToHistory() {
        // 1. 新增任務
        composeTestRule
            .onNodeWithText("任務")
            .performClick()
        
        composeTestRule
            .onNodeWithContentDescription("新增任務")
            .performClick()
        
        composeTestRule
            .onNodeWithText("任務名稱")
            .performTextInput("E2E Test Task")
        
        composeTestRule
            .onNodeWithText("確認")
            .performClick()
        
        // 2. 設置為當前任務
        composeTestRule
            .onNodeWithText("E2E Test Task")
            .performTouchInput { swipeLeft() }
        
        composeTestRule
            .onNodeWithContentDescription("設為目前任務")
            .performClick()
        
        // 3. 回到首頁開始計時
        composeTestRule
            .onNodeWithText("首頁")
            .performClick()
        
        composeTestRule
            .onNodeWithText("開始")
            .performClick()
        
        // 4. 驗證計時器運行
        composeTestRule
            .onNodeWithText("暫停")
            .assertIsDisplayed()
        
        // 5. 檢查歷史記錄（假設有快速完成功能用於測試）
        composeTestRule
            .onNodeWithText("歷史")
            .performClick()
        
        // 驗證歷史記錄中有相關資料...
    }
}
```

## 📊 測試覆蓋率

### 覆蓋率配置

```kotlin
// app/build.gradle.kts
android {
    buildTypes {
        debug {
            isTestCoverageEnabled = true
        }
    }
    
    testOptions {
        unitTests {
            isIncludeAndroidResources = true
            isReturnDefaultValues = true
        }
        
        animationsDisabled = true
    }
}

// 覆蓋率報告任務
tasks.register("jacocoTestReport", JacocoReport::class) {
    dependsOn("testDebugUnitTest", "createDebugCoverageReport")
    
    reports {
        xml.required.set(true)
        html.required.set(true)
    }
    
    val fileFilter = listOf(
        "**/R.class",
        "**/R\$*.class",
        "**/BuildConfig.*",
        "**/Manifest*.*",
        "**/*Test*.*",
        "android/**/*.*",
        "**/databinding/**",
        "**/di/**" // 依賴注入通常不需要測試覆蓋率
    )
    
    val debugTree = fileTree("${buildDir}/intermediates/javac/debug") {
        exclude(fileFilter)
    }
    
    val mainSrc = "${project.projectDir}/src/main/java"
    
    classDirectories.setFrom(debugTree)
    sourceDirectories.setFrom(files(mainSrc))
    executionData.setFrom(fileTree(buildDir) {
        include("**/*.exec", "**/*.ec")
    })
}
```

### 測試覆蓋率目標

- **整體覆蓋率**: > 80%
- **業務邏輯覆蓋率**: > 90%
- **UI 元件覆蓋率**: > 70%
- **Repository 覆蓋率**: > 85%

## 🛠️ 測試工具和技巧

### 自定義測試 Rules

```kotlin
// test/util/TimberTestRule.kt
class TimberTestRule : TestWatcher() {
    override fun starting(description: Description) {
        Timber.plant(object : Timber.Tree() {
            override fun log(priority: Int, tag: String?, message: String, t: Throwable?) {
                println("[$priority] $tag: $message")
            }
        })
    }
    
    override fun finished(description: Description) {
        Timber.uprootAll()
    }
}

// test/util/CoroutineTestRule.kt
@ExperimentalCoroutinesTest
class CoroutineTestRule(
    private val testDispatcher: TestDispatcher = StandardTestDispatcher()
) : TestWatcher() {
    
    override fun starting(description: Description) {
        Dispatchers.setMain(testDispatcher)
    }
    
    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}
```

### 測試輔助函數

```kotlin
// test/util/TestHelpers.kt
object TestHelpers {
    
    fun createTestTask(
        id: Long = 1,
        name: String = "Test Task",
        description: String = "Test Description",
        isCompleted: Boolean = false,
        estimatedPomodoros: Int = 3
    ): Task {
        return Task(
            id = id,
            name = name,
            description = description,
            isCompleted = isCompleted,
            estimatedPomodoros = estimatedPomodoros
        )
    }
    
    fun createTestSession(
        id: Long = 1,
        taskId: Long? = null,
        sessionType: SessionType = SessionType.FOCUS,
        isCompleted: Boolean = false
    ): PomodoroSession {
        return PomodoroSession(
            id = id,
            taskId = taskId,
            sessionType = sessionType,
            startTime = System.currentTimeMillis(),
            durationMillis = sessionType.defaultDurationMillis,
            isCompleted = isCompleted
        )
    }
    
    fun runTestWithTimeout(
        timeout: Duration = 5.seconds,
        block: suspend TestScope.() -> Unit
    ) = runTest(timeout = timeout, testBody = block)
}
```

## 📚 延伸閱讀

- [Android Testing Fundamentals](https://developer.android.com/training/testing/fundamentals)
- [Testing Jetpack Compose](https://developer.android.com/jetpack/compose/testing)
- [Kotlin Coroutines Testing](https://kotlinlang.org/docs/coroutines-testing.html)

## 🎯 下一章預告

在下一章中，我們將探討 **進階主題**，學習：

- 效能優化技巧
- 記憶體管理策略
- 國際化和本地化
- 無障礙設計

準備好深入探索 Android 開發的進階技術了嗎？🚀