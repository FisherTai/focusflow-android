# 第 2 章：UI 層實作

## 📋 本章目標

- 掌握 Jetpack Compose 進階開發技巧
- 應用 Material Design 3 設計系統
- 實作導航系統和狀態管理
- 建構番茄鐘應用的核心畫面

## 🎨 Jetpack Compose 基礎回顧

### 聲明式 UI 概念

```kotlin
// 傳統命令式 UI
val textView = findViewById<TextView>(R.id.timer_text)
textView.text = "25:00"
textView.setTextColor(Color.RED)

// Jetpack Compose 聲明式 UI
@Composable
fun TimerDisplay(timeText: String, isRunning: Boolean) {
    Text(
        text = timeText,
        color = if (isRunning) Color.Red else Color.Black,
        style = MaterialTheme.typography.headlineLarge
    )
}
```

### Compose 核心概念

1. **@Composable** - 可組合函數標註
2. **State** - 狀態管理
3. **Recomposition** - 重新組合
4. **Side Effects** - 副作用處理

## 🌈 Material Design 3 應用

### 主題系統設計

```kotlin
// Color.kt - 顏色系統
val md_theme_light_primary = Color(0xFF6750A4)
val md_theme_light_onPrimary = Color(0xFFFFFFFF)
val md_theme_light_primaryContainer = Color(0xFFEADDFF)
val md_theme_light_onPrimaryContainer = Color(0xFF21005D)

val md_theme_dark_primary = Color(0xFFD0BCFF)
val md_theme_dark_onPrimary = Color(0xFF381E72)
val md_theme_dark_primaryContainer = Color(0xFF4F378B)
val md_theme_dark_onPrimaryContainer = Color(0xFFEADDFF)

private val LightColors = lightColorScheme(
    primary = md_theme_light_primary,
    onPrimary = md_theme_light_onPrimary,
    primaryContainer = md_theme_light_primaryContainer,
    onPrimaryContainer = md_theme_light_onPrimaryContainer,
    // ... 更多顏色定義
)

private val DarkColors = darkColorScheme(
    primary = md_theme_dark_primary,
    onPrimary = md_theme_dark_onPrimary,
    primaryContainer = md_theme_dark_primaryContainer,
    onPrimaryContainer = md_theme_dark_onPrimaryContainer,
    // ... 更多顏色定義
)
```

### 主題應用

```kotlin
// Theme.kt - 主題定義
@Composable
fun PomodoroAppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context) else dynamicLightColorScheme(context)
        }
        darkTheme -> DarkColors
        else -> LightColors
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography,
        content = content
    )
}
```

### 主題擴展功能

```kotlin
// ThemeExtensions.kt - 主題擴展
val ColorScheme.timerRunning: Color
    @Composable get() = if (isSystemInDarkTheme()) Color(0xFF4CAF50) else Color(0xFF2E7D32)

val ColorScheme.timerPaused: Color
    @Composable get() = if (isSystemInDarkTheme()) Color(0xFFFFA726) else Color(0xFFEF6C00)

val ColorScheme.timerIdle: Color
    @Composable get() = onSurfaceVariant

@Composable
fun MaterialTheme.timerColors(): TimerColors {
    return TimerColors(
        running = colorScheme.timerRunning,
        paused = colorScheme.timerPaused,
        idle = colorScheme.timerIdle
    )
}

data class TimerColors(
    val running: Color,
    val paused: Color,
    val idle: Color
)
```

## 📱 應用架構設計

### MainActivity 設計

```kotlin
@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        enableEdgeToEdge()
        
        setContent {
            PomodoroAppTheme {
                PomodoroApp()
            }
        }
    }
}

@Composable
fun PomodoroApp() {
    val navController = rememberNavController()
    
    Scaffold(
        modifier = Modifier.fillMaxSize(),
        bottomBar = {
            BottomBar(navController = navController)
        }
    ) { innerPadding ->
        NavHost(
            navController = navController,
            startDestination = "home",
            modifier = Modifier.padding(innerPadding)
        ) {
            composable("home") { 
                HomeScreen() 
            }
            composable("tasks") { 
                TaskListScreen() 
            }
            composable("history") { 
                HistoryScreen() 
            }
        }
    }
}
```

## 🏠 首頁計時器實作

### HomeScreen 結構

```kotlin
@Composable
fun HomeScreen(
    viewModel: HomeViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsState()
    
    HomeContent(
        uiState = uiState,
        onStartTimer = viewModel::startTimer,
        onPauseTimer = viewModel::pauseTimer,
        onResetTimer = viewModel::resetTimer,
        onModeSwitch = viewModel::switchMode
    )
}

@Composable
private fun HomeContent(
    uiState: HomeUiState,
    onStartTimer: () -> Unit,
    onPauseTimer: () -> Unit,
    onResetTimer: () -> Unit,
    onModeSwitch: () -> Unit
) {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(24.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        // 計時器顯示
        TimerDisplay(
            timeRemaining = uiState.timeRemaining,
            timerState = uiState.timerState,
            currentMode = uiState.currentMode
        )
        
        Spacer(modifier = Modifier.height(32.dp))
        
        // 控制按鈕
        TimerControls(
            timerState = uiState.timerState,
            onStartTimer = onStartTimer,
            onPauseTimer = onPauseTimer,
            onResetTimer = onResetTimer
        )
        
        Spacer(modifier = Modifier.height(24.dp))
        
        // 模式切換
        ModeSwitch(
            currentMode = uiState.currentMode,
            onModeSwitch = onModeSwitch
        )
        
        // 當前任務顯示
        uiState.currentTask?.let { task ->
            Spacer(modifier = Modifier.height(24.dp))
            CurrentTaskCard(task = task)
        }
    }
}
```

### 計時器顯示元件

```kotlin
@Composable
fun TimerDisplay(
    timeRemaining: Long,
    timerState: TimerState,
    currentMode: TimerMode,
    modifier: Modifier = Modifier
) {
    val timerColors = MaterialTheme.timerColors()
    
    val displayColor = when (timerState) {
        TimerState.RUNNING -> timerColors.running
        TimerState.PAUSED -> timerColors.paused
        else -> timerColors.idle
    }
    
    val animatedColor by animateColorAsState(
        targetValue = displayColor,
        animationSpec = tween(300),
        label = "timer_color"
    )
    
    Card(
        modifier = modifier.size(280.dp),
        colors = CardDefaults.cardColors(
            containerColor = MaterialTheme.colorScheme.surfaceVariant.copy(alpha = 0.3f)
        ),
        elevation = CardDefaults.cardElevation(defaultElevation = 8.dp)
    ) {
        Box(
            modifier = Modifier.fillMaxSize(),
            contentAlignment = Alignment.Center
        ) {
            Column(
                horizontalAlignment = Alignment.CenterHorizontally
            ) {
                // 模式指示器
                Text(
                    text = when (currentMode) {
                        TimerMode.FOCUS -> "專注時間"
                        TimerMode.BREAK -> "休息時間"
                    },
                    style = MaterialTheme.typography.titleMedium,
                    color = MaterialTheme.colorScheme.onSurfaceVariant
                )
                
                Spacer(modifier = Modifier.height(8.dp))
                
                // 時間顯示
                Text(
                    text = formatTime(timeRemaining),
                    style = MaterialTheme.typography.displayLarge.copy(
                        fontWeight = FontWeight.Bold
                    ),
                    color = animatedColor
                )
                
                // 狀態指示器
                if (timerState == TimerState.RUNNING) {
                    Spacer(modifier = Modifier.height(8.dp))
                    PulsingDot(color = animatedColor)
                }
            }
        }
    }
}

@Composable
private fun PulsingDot(color: Color) {
    val infiniteTransition = rememberInfiniteTransition(label = "pulsing")
    val alpha by infiniteTransition.animateFloat(
        initialValue = 0.3f,
        targetValue = 1f,
        animationSpec = infiniteRepeatable(
            animation = tween(1000),
            repeatMode = RepeatMode.Reverse
        ),
        label = "alpha"
    )
    
    Box(
        modifier = Modifier
            .size(12.dp)
            .background(
                color = color.copy(alpha = alpha),
                shape = CircleShape
            )
    )
}

private fun formatTime(timeInMillis: Long): String {
    val minutes = (timeInMillis / 1000) / 60
    val seconds = (timeInMillis / 1000) % 60
    return String.format("%02d:%02d", minutes, seconds)
}
```

### 控制按鈕元件

```kotlin
@Composable
fun TimerControls(
    timerState: TimerState,
    onStartTimer: () -> Unit,
    onPauseTimer: () -> Unit,
    onResetTimer: () -> Unit,
    modifier: Modifier = Modifier
) {
    Row(
        modifier = modifier,
        horizontalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        // 開始/暫停按鈕
        when (timerState) {
            TimerState.IDLE, TimerState.PAUSED -> {
                FilledTonalButton(
                    onClick = onStartTimer,
                    modifier = Modifier.size(width = 120.dp, height = 48.dp)
                ) {
                    Icon(
                        imageVector = Icons.Default.PlayArrow,
                        contentDescription = "開始"
                    )
                    Spacer(modifier = Modifier.width(8.dp))
                    Text("開始")
                }
            }
            TimerState.RUNNING -> {
                FilledTonalButton(
                    onClick = onPauseTimer,
                    modifier = Modifier.size(width = 120.dp, height = 48.dp)
                ) {
                    Icon(
                        imageVector = Icons.Default.Pause,
                        contentDescription = "暫停"
                    )
                    Spacer(modifier = Modifier.width(8.dp))
                    Text("暫停")
                }
            }
            TimerState.COMPLETED -> {
                FilledTonalButton(
                    onClick = onStartTimer,
                    modifier = Modifier.size(width = 120.dp, height = 48.dp)
                ) {
                    Icon(
                        imageVector = Icons.Default.Refresh,
                        contentDescription = "重新開始"
                    )
                    Spacer(modifier = Modifier.width(8.dp))
                    Text("重新開始")
                }
            }
        }
        
        // 重設按鈕
        if (timerState != TimerState.IDLE) {
            OutlinedButton(
                onClick = onResetTimer,
                modifier = Modifier.size(width = 100.dp, height = 48.dp)
            ) {
                Icon(
                    imageVector = Icons.Default.Stop,
                    contentDescription = "重設"
                )
                Spacer(modifier = Modifier.width(8.dp))
                Text("重設")
            }
        }
    }
}
```

## 📋 任務管理畫面

### TaskListScreen 實作

```kotlin
@Composable
fun TaskListScreen(
    viewModel: TaskListViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsState()
    
    TaskListContent(
        uiState = uiState,
        onAddTask = viewModel::addTask,
        onDeleteTask = viewModel::deleteTask,
        onToggleTask = viewModel::toggleTaskCompletion,
        onSetCurrentTask = viewModel::setCurrentTask
    )
}

@Composable
private fun TaskListContent(
    uiState: TaskListUiState,
    onAddTask: (String) -> Unit,
    onDeleteTask: (Long) -> Unit,
    onToggleTask: (Long) -> Unit,
    onSetCurrentTask: (Long) -> Unit
) {
    var showAddDialog by remember { mutableStateOf(false) }
    
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp)
    ) {
        // 標題和新增按鈕
        Row(
            modifier = Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.SpaceBetween,
            verticalAlignment = Alignment.CenterVertically
        ) {
            Text(
                text = "任務清單",
                style = MaterialTheme.typography.headlineMedium
            )
            
            FloatingActionButton(
                onClick = { showAddDialog = true },
                modifier = Modifier.size(56.dp)
            ) {
                Icon(
                    imageVector = Icons.Default.Add,
                    contentDescription = "新增任務"
                )
            }
        }
        
        Spacer(modifier = Modifier.height(16.dp))
        
        // 任務列表
        LazyColumn(
            verticalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            items(
                items = uiState.tasks,
                key = { it.id }
            ) { task ->
                SwipeableTaskItem(
                    task = task,
                    onToggle = { onToggleTask(task.id) },
                    onDelete = { onDeleteTask(task.id) },
                    onSetCurrent = { onSetCurrentTask(task.id) },
                    modifier = Modifier.animateItemPlacement()
                )
            }
        }
    }
    
    // 新增任務對話框
    if (showAddDialog) {
        AddTaskDialog(
            onDismiss = { showAddDialog = false },
            onConfirm = { taskName ->
                onAddTask(taskName)
                showAddDialog = false
            }
        )
    }
}
```

### 滑動操作任務項目

```kotlin
@Composable
fun SwipeableTaskItem(
    task: TaskUIData,
    onToggle: () -> Unit,
    onDelete: () -> Unit,
    onSetCurrent: () -> Unit,
    modifier: Modifier = Modifier
) {
    val swipeableState = rememberSwipeableState(initialValue = 0)
    val sizePx = with(LocalDensity.current) { 80.dp.toPx() }
    
    Box(
        modifier = modifier
            .fillMaxWidth()
            .height(72.dp)
    ) {
        // 背景操作按鈕
        Row(
            modifier = Modifier
                .fillMaxSize()
                .padding(horizontal = 16.dp),
            horizontalArrangement = Arrangement.End,
            verticalAlignment = Alignment.CenterVertically
        ) {
            // 設為目前任務按鈕
            IconButton(
                onClick = onSetCurrent,
                modifier = Modifier
                    .background(
                        MaterialTheme.colorScheme.primary,
                        CircleShape
                    )
                    .size(48.dp)
            ) {
                Icon(
                    imageVector = Icons.Default.PlayArrow,
                    contentDescription = "設為目前任務",
                    tint = MaterialTheme.colorScheme.onPrimary
                )
            }
            
            Spacer(modifier = Modifier.width(8.dp))
            
            // 刪除按鈕
            IconButton(
                onClick = onDelete,
                modifier = Modifier
                    .background(
                        MaterialTheme.colorScheme.error,
                        CircleShape
                    )
                    .size(48.dp)
            ) {
                Icon(
                    imageVector = Icons.Default.Delete,
                    contentDescription = "刪除任務",
                    tint = MaterialTheme.colorScheme.onError
                )
            }
        }
        
        // 任務項目
        Card(
            modifier = Modifier
                .fillMaxSize()
                .swipeable(
                    state = swipeableState,
                    anchors = mapOf(
                        0f to 0,
                        -sizePx * 2 to 1
                    ),
                    thresholds = { _, _ -> FractionalThreshold(0.3f) },
                    orientation = Orientation.Horizontal
                )
                .offset {
                    IntOffset(swipeableState.offset.value.roundToInt(), 0)
                },
            elevation = CardDefaults.cardElevation(defaultElevation = 2.dp)
        ) {
            TaskItem(
                task = task,
                onToggle = onToggle
            )
        }
    }
}

@Composable
fun TaskItem(
    task: TaskUIData,
    onToggle: () -> Unit,
    modifier: Modifier = Modifier
) {
    Row(
        modifier = modifier
            .fillMaxSize()
            .clickable { onToggle() }
            .padding(16.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        // 完成狀態指示器
        Checkbox(
            checked = task.isCompleted,
            onCheckedChange = { onToggle() }
        )
        
        Spacer(modifier = Modifier.width(12.dp))
        
        // 任務內容
        Column(modifier = Modifier.weight(1f)) {
            Text(
                text = task.name,
                style = MaterialTheme.typography.bodyLarge,
                textDecoration = if (task.isCompleted) TextDecoration.LineThrough else null,
                color = if (task.isCompleted) {
                    MaterialTheme.colorScheme.onSurfaceVariant
                } else {
                    MaterialTheme.colorScheme.onSurface
                }
            )
            
            if (task.description.isNotEmpty()) {
                Text(
                    text = task.description,
                    style = MaterialTheme.typography.bodyMedium,
                    color = MaterialTheme.colorScheme.onSurfaceVariant
                )
            }
        }
        
        // 目前任務指示器
        if (task.isCurrent) {
            Icon(
                imageVector = Icons.Default.PlayArrow,
                contentDescription = "目前任務",
                tint = MaterialTheme.colorScheme.primary
            )
        }
    }
}
```

## 🧭 導航系統設計

### 底部導航列

```kotlin
@Composable
fun BottomBar(
    navController: NavHostController,
    modifier: Modifier = Modifier
) {
    val items = listOf(
        BottomNavItem("home", "首頁", Icons.Default.Home),
        BottomNavItem("tasks", "任務", Icons.Default.Assignment),
        BottomNavItem("history", "歷史", Icons.Default.History)
    )
    
    val navBackStackEntry by navController.currentBackStackEntryAsState()
    val currentRoute = navBackStackEntry?.destination?.route
    
    NavigationBar(
        modifier = modifier
    ) {
        items.forEach { item ->
            NavigationBarItem(
                icon = {
                    Icon(
                        imageVector = item.icon,
                        contentDescription = item.label
                    )
                },
                label = { Text(item.label) },
                selected = currentRoute == item.route,
                onClick = {
                    navController.navigate(item.route) {
                        popUpTo(navController.graph.findStartDestination().id) {
                            saveState = true
                        }
                        launchSingleTop = true
                        restoreState = true
                    }
                }
            )
        }
    }
}

data class BottomNavItem(
    val route: String,
    val label: String,
    val icon: ImageVector
)
```

## 📊 狀態管理最佳實踐

### ViewModel 設計

```kotlin
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val startTimerUseCase: StartTimerUseCase,
    private val pauseTimerUseCase: PauseTimerUseCase,
    private val resetTimerUseCase: ResetTimerUseCase,
    private val getCurrentTaskUseCase: GetCurrentTaskUseCase
) : ViewModel() {
    
    private val _uiState = MutableStateFlow(HomeUiState())
    val uiState: StateFlow<HomeUiState> = _uiState.asStateFlow()
    
    init {
        observeCurrentTask()
        observeTimerState()
    }
    
    private fun observeCurrentTask() {
        viewModelScope.launch {
            getCurrentTaskUseCase()
                .collect { task ->
                    _uiState.update { 
                        it.copy(currentTask = task) 
                    }
                }
        }
    }
    
    private fun observeTimerState() {
        // 觀察計時器狀態變化
    }
    
    fun startTimer() {
        viewModelScope.launch {
            startTimerUseCase()
                .onSuccess {
                    _uiState.update { 
                        it.copy(timerState = TimerState.RUNNING) 
                    }
                }
                .onFailure { error ->
                    _uiState.update { 
                        it.copy(error = error.message) 
                    }
                }
        }
    }
    
    // 其他方法...
}
```

## 📚 延伸閱讀

- [Jetpack Compose 官方文件](https://developer.android.com/jetpack/compose)
- [Material Design 3](https://m3.material.io/)
- [State and Jetpack Compose](https://developer.android.com/jetpack/compose/state)

## 🎯 下一章預告

在下一章中，我們將深入探討 **資料層實作**，學習：

- Room 資料庫設計和實作
- Repository 模式深入應用
- 資料快取和同步策略
- 資料轉換和映射

準備好建構強大的資料存取層了嗎？💾