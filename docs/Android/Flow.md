# Kotlin Flow 教程

## 目录

1. 前置：协程 5 分钟速通（没用过协程也能看懂 Flow）
2. Flow 是什么：和 `suspend` 有啥区别？
3. 创建 Flow：`flow{}`, `flowOf`, `asFlow`
4. 收集 Flow：`collect` / `collectLatest`（含 Android 示例）
5. 中间操作符：`map / filter / debounce / flatMapLatest` 等
6. 线程切换：`flowOn`
7. 背压与性能：`buffer / conflate / collectLatest`
8. 异常处理与完成：`catch / onCompletion`
9. 冷流 vs 热流：Flow、StateFlow、SharedFlow 的区别
10. Android 实战：Room + Repository + ViewModel + UI 收集
11. 和 LiveData 的关系 & 互转
12. 常见坑 & 实战建议

---

## 1. 前置：协程 5 分钟速通

先把你写 Flow 时一定会碰到的几个协程概念过一遍。

### 1.1 三个最常见的东西

```kotlin
val scope = CoroutineScope(Dispatchers.Main)

scope.launch {
// 这里是协程体，可以调用 suspend 函数
}

suspend fun loadData(): String {
delay(1000) // 模拟网络
return "result"
}
```

- **`Dispatchers.Main`**：主线程（更新 UI）
- **`Dispatchers.IO`**：IO 线程（网络/数据库）
- **`launch`**：启动一个不关心返回值的协程
- **`async/await`**：启动一个有返回值的协程（这里先不展开）

在 Android 里最常见的是这些 scope：

- `lifecycleScope`：Activity/Fragment 的生命周期作用域
- `viewModelScope`：ViewModel 的作用域
- 自己创建：`CoroutineScope(Dispatchers.Main + SupervisorJob())`

---

## 2. Flow 是什么？和 `suspend` 有啥区别

### 2.1 一句话

> **Flow = Kotlin 协程里的「异步数据流」**  
> 可以按顺序、异步地、一条一条地把数据发出来，上游可以慢慢发，下游慢慢收。

### 2.2 和 `suspend` 的对比

- `suspend fun getUser(): User`
    - 调用一次，**给你一个结果**
- `fun getUsers(): Flow<User>`
    - 调用一次，**给你一条“数据管道”**
    - 可以持续发很多个 `User`：1 条、10 条、无限条都可以

类似类比：

| 形式                    | 比喻                     |
|-------------------------|--------------------------|
| `suspend fun`           | 点外卖，一次送到        |
| `Flow<T>`               | 订阅快递，一件件往家送  |

### 2.3 Flow 的三要素

在代码层面，你可以理解为三个阶段：

1. **创建（上游）**：`flow { emit(...) }`
2. **变换（中间操作）**：`map / filter / debounce / flatMapLatest ...`
3. **收集（下游）**：`collect { value -> ... }`

只有当你调用 `collect` 的时候，Flow 才会真正开始执行（**冷流**，后面会说）。

---

## 3. 创建 Flow：`flow{}`, `flowOf`, `asFlow`

### 3.1 `flow {}` —— 最通用的创建方式

```kotlin
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow

fun numbersFlow(): Flow<Int> = flow {
println("Flow started")
for (i in 1..5) {
emit(i)            // 发出一个值
delay(500)         // 模拟耗时
}
}
```

特点：

- `emit(value)`：发出一条数据
- 支持 `suspend` 调用（可以 `delay` / 调接口 / 读数据库）

### 3.2 `flowOf`：快速创建几个常量

```kotlin
val flow1: Flow<Int> = flowOf(1, 2, 3)
val flow2: Flow<String> = flowOf("A", "B", "C")
```

可当做简单 demo 或测试数据。

### 3.3 `asFlow`：把集合 / 范围转成 Flow

```kotlin
val flowFromList: Flow<Int> = listOf(1, 2, 3).asFlow()
val flowFromRange: Flow<Int> = (1..5).asFlow()
```

---

## 4. 收集 Flow：`collect` / `collectLatest`

### 4.1 最基本的收集：`collect`

```kotlin
scope.launch {
numbersFlow().collect { value ->
println("received: $value")
}
}
```

- `collect` 是 **挂起函数**，必须在协程里调用
- 收到一个值就执行一次 lambda

### 4.2 `collectLatest`：只关心最新的数据

场景：上游发的很快，下游处理很慢。你只想处理**最近那一条**，中间的可以丢掉。

```kotlin
import kotlinx.coroutines.flow.collectLatest

scope.launch {
numbersFlow().collectLatest { value ->
println("start handle $value")
delay(1000)             // 处理很慢
println("end handle $value")
}
}
```

行为：

- 当新的 `value` 到来时，如果上一次处理还没结束，会**取消旧的处理**，转而处理最新的。

常见用在：

- 搜索框实时搜索（用户一直在输，不要对每一个字符发请求）

### 4.3 Android 中的收集（XML + Fragment）

```kotlin
class MyFragment : Fragment() {

    private val viewModel: MyViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        // 跟随生命周期自动取消
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.stateFlow.collect { state ->
                    // 更新 UI
                }
            }
        }
    }
}
```

---

## 5. 常用中间操作符（Flow 的灵魂）

中间操作会返回一个新的 Flow，不会触发执行，只有最终 `collect` 才会真正跑。

### 5.1 `map`：一对一转换

```kotlin
val flow = numbersFlow()
.map { it * 2 }  // 1 -> 2, 2 -> 4 ...

scope.launch {
flow.collect { println(it) }
}
```

### 5.2 `filter`：过滤

```kotlin
val evenFlow = numbersFlow()
.filter { it % 2 == 0 }

scope.launch {
evenFlow.collect { println(it) } // 2, 4
}
```

### 5.3 `take`：只取前 n 个

```kotlin
numbersFlow()
.take(3)
.collect { println(it) } // 1, 2, 3
```

### 5.4 `distinctUntilChanged`：只在值有变化时才发

常用于状态流：只有状态变化时才更新 UI。

```kotlin
val flow = flowOf(1, 1, 2, 2, 3)
.distinctUntilChanged()

// 收到：1, 2, 3
```

### 5.5 `debounce`：防抖（很常用于搜索框）

```kotlin
val searchFlow: Flow<String> = textChanges() // 假设这是输入文本变化的 Flow

searchFlow
.debounce(300)    // 300ms 内没有新输入才发出
.filter { it.isNotBlank() }
.collectLatest { keyword ->
// 只请求最后一次稳定下来的关键字
}
```

### 5.6 `flatMapLatest`：只关心最新的子流

非常适合 “关键字 -> 请求结果” 这种情况：

```kotlin
fun search(keyword: String): Flow<List<Result>> = flow { ... }

searchFlow
.debounce(300)
.flatMapLatest { keyword ->
search(keyword)  // 每个关键字返回一个 Flow
}
.collect { resultList ->
// 显示搜索结果
}
```

- 新关键字来了，旧关键字对应的请求 Flow 会被取消
- 避免发一堆错乱的请求

> 还有 `flatMapConcat`、`flatMapMerge` 等，可以先有这一个最常用的概念：**flatMapLatest = 最新优先，自动取消旧的**

---

## 6. 线程切换：`flowOn`

### 6.1 基本用法

```kotlin
fun numbersFlow(): Flow<Int> = flow {
repeat(5) {
delay(500)
emit(it)
}
}.flowOn(Dispatchers.IO)     // 把上游放到 IO 线程
```

- **`flowOn` 只影响它之前的上游代码**
- 下游 `collect` 默认在当前协程所在的线程里执行

### 6.2 推荐写法：上游做耗时，UI 线程只 collect

```kotlin
viewModelScope.launch {
repository.getUsers()      // Flow<List<User>>
.flowOn(Dispatchers.IO) // 数据库在 IO 线程
.collect { users ->     // Main 线程更新 UI
_uiState.value = users
}
}
```

---

## 7. 背压与性能：`buffer / conflate / collectLatest`

当发的快、收的慢时，会出现“背压”问题。Flow 提供了几种策略：

### 7.1 默认：顺序执行

- 上游发一条
- 下游处理完
- 上游才能发下一条

### 7.2 `buffer()`：让发和收异步

```kotlin
numbersFlow()
.buffer()          // 上下游并行，值先缓存到缓冲区
.collect { value ->
delay(1000)    // 慢收集
println(value)
}
```

收益：

- 上游不必等下游慢慢处理，整体吞吐量更高

### 7.3 `conflate()`：缓存只保留最新值

```kotlin
numbersFlow()
.conflate()
.collect { value ->
delay(1000) // 处理很慢
println(value)
}
```

行为：

- 缓冲区只保留**最新的**值，中间那些处理不过来的被丢弃
- 适合 UI 场景（你只关心当前最新状态）

### 7.4 `collectLatest` 再重复一下

- 类似 `conflate` + “取消旧任务” 的组合：
    - 新值来 → 取消旧的 lambda 执行 → 执行新的

---

## 8. 异常处理与完成：`catch / onCompletion`

### 8.1 普通 `try/catch`

```kotlin
scope.launch {
try {
flowThatMayThrow().collect { value ->
// ...
}
} catch (e: Exception) {
// 统一处理异常
}
}
```

### 8.2 `catch` 操作符（推荐）

```kotlin
import kotlinx.coroutines.flow.catch
import kotlinx.coroutines.flow.onCompletion

flowThatMayThrow()
.catch { e ->
// 只捕获上游（之前）抛出的异常
emit(defaultValue) // 可以选择发一个兜底值
}
.onCompletion { cause ->
if (cause == null) {
println("正常完成")
} else {
println("异常结束: $cause")
}
}
.collect { value ->
// ...
}
```

注意：

- `catch {}` 只会捕获位于它之前的上游异常
- `onCompletion` 无论正常结束还是异常都会调用（`cause` 为 null 表示正常）

---

## 9. 冷流 vs 热流：Flow、StateFlow、SharedFlow

### 9.1 冷流（Cold Flow）

普通的 `Flow` 都是冷的：

- 每次 `collect` = 从头到尾重新跑一遍
- 没有订阅就不工作（**懒执行**）

```kotlin
val f = numbersFlow() // 还没执行

f.collect { ... }     // 第一次，从 1 到 5
f.collect { ... }     // 第二次，又从 1 到 5
```

### 9.2 热流（Hot Flow）

**一直活着、一直在发数据**，和有没有订阅没关系。

典型就是：

- `StateFlow<T>`：保存“当前最新状态”
- `SharedFlow<T>`：事件广播器，可多订阅者

#### 9.2.1 StateFlow：类似 LiveData + 初始值 + 同步读

定义：

```kotlin
private val _uiState = MutableStateFlow(UiState())
val uiState: StateFlow<UiState> = _uiState
```

使用：

```kotlin
_uiState.value = _uiState.value.copy(loading = true)

// 立即读取当前值
val currentState = _uiState.value
```

特点：

- 必须有一个**初始值**
- 永远会保存一个最新值，新订阅者会立刻收到当前值
- 非常适合 ViewModel → UI 的“状态”传递

#### 9.2.2 SharedFlow：事件/多订阅场景

简单理解：**没有“当前值”，你只拿得到你订阅之后的那些发射**（配置可调）。

定义：

```kotlin
private val _events = MutableSharedFlow<UiEvent>()
val events = _events.asSharedFlow()
```

使用：

```kotlin
viewModelScope.launch {
_events.emit(UiEvent.ShowToast("Hello"))
}
```

收集：

```kotlin
lifecycleScope.launch {
viewModel.events.collect { event ->
when (event) {
is UiEvent.ShowToast -> showToast(event.message)
}
}
}
```

适合：

- 一次性的 UI 事件：Toast、导航、Dialog 等

---

## 10. Android 实战：Room + Repository + ViewModel + UI

给你一个比较贴近真实的完整链路。

### 10.1 Room DAO 返回 Flow

```kotlin
@Dao
interface UserDao {
@Query("SELECT * FROM user")
fun getUsers(): Flow<List<User>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertUsers(users: List<User>)
}
```

### 10.2 Repository 中组合网络 + 数据库

```kotlin
class UserRepository(
private val api: UserApi,
private val dao: UserDao,
) {

    // 暴露一个冷流，UI 层一直收集
    fun observeUsers(): Flow<List<User>> =
        dao.getUsers()
            .flowOn(Dispatchers.IO)

    // 主动刷新
    suspend fun refreshUsers() {
        val remote = api.getUsers()            // 网络请求
        dao.insertUsers(remote)                // 写入数据库
    }
}
```

### 10.3 ViewModel 暴露 StateFlow

```kotlin
data class UserUiState(
val loading: Boolean = false,
val users: List<User> = emptyList(),
val error: String? = null
)

class UserViewModel(
private val repository: UserRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(UserUiState())
    val uiState: StateFlow<UserUiState> = _uiState

    init {
        observeUsers()
        refresh()
    }

    private fun observeUsers() {
        viewModelScope.launch {
            repository.observeUsers()
                .onEach { users ->
                    _uiState.update { it.copy(users = users, loading = false) }
                }
                .catch { e ->
                    _uiState.update { it.copy(error = e.message, loading = false) }
                }
                .launchIn(this) // 也可以用 launchIn + onEach/catch 组合
        }
    }

    fun refresh() {
        viewModelScope.launch {
            _uiState.update { it.copy(loading = true, error = null) }
            try {
                repository.refreshUsers()
            } catch (e: Exception) {
                _uiState.update { it.copy(error = e.message, loading = false) }
            }
        }
    }
}
```

### 10.4 Fragment 中收集（基于 ViewBinding/XML）

```kotlin
class UserFragment : Fragment() {

    private val viewModel: UserViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        // 收集状态
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    progressBar.isVisible = state.loading
                    recyclerView.isVisible = state.users.isNotEmpty()
                    // adapter.submitList(state.users)
                    state.error?.let { showToast(it) }
                }
            }
        }

        swipeRefreshLayout.setOnRefreshListener {
            viewModel.refresh()
        }
    }
}
```

---

## 11. Flow 和 LiveData：啥关系 & 互转

### 11.1 对比

**LiveData：**

- Android 专属
- 感知生命周期
- 线程限制（更新要在主线程）
- API 较老，拓展性一般

**Flow / StateFlow：**

- Kotlin 协程家族的一员，**平台无关**
- 操作符丰富（map/filter/debounce/flatMapLatest... 一整个生态）
- 更容易和协程/挂起函数整合
- 通过 `lifecycleScope`/`repeatOnLifecycle` 实现生命周期感知

现在官方更推荐在新代码中使用 Flow + StateFlow，LiveData 多是旧项目兼容。

### 11.2 互转

- Flow → LiveData

```kotlin
val liveData = flow.asLiveData()
```

- LiveData → Flow

```kotlin
val flow = liveData.asFlow()
```

一般情况下，在 **ViewModel 内部** 用 Flow / StateFlow，**对外给 UI 直接暴露 StateFlow** 就足够了。

---

## 12. 常见坑 & 实战建议

### 12.1 别在上游做大量主线程耗时

**反例：**

```kotlin
fun heavyFlow(): Flow<Int> = flow {
// 这里默认在调用 collect 的线程执行
for (i in 1..1000000) {
// 大量计算，阻塞主线程
...
emit(i)
}
}
```

**建议：**

- 用 `flowOn(Dispatchers.Default/IO)` 把上游丢到后台线程

```kotlin
fun heavyFlow(): Flow<Int> =
flow { ... }
.flowOn(Dispatchers.Default)
```

### 12.2 `flowOn` 的位置很重要

- `flow { ... }.map { ... }.flowOn(IO)`：上游 flow + map 都在 IO
- `flow { ... }.flowOn(IO).map { ... }`：只有 flow 在 IO，map 在当前线程

通常你想把所有耗时操作放到 `flowOn` 前面。

### 12.3 别忘了生命周期 / 取消

- 不要自己手写 endless scope 而不取消
- 在 Android 中尽量使用：
    - `viewModelScope`（ViewModel）
    - `lifecycleScope + repeatOnLifecycle`（Fragment/Activity）
- 这些会替你自动取消协程，防止内存泄漏

### 12.4 UI 层多用 `StateFlow` / `SharedFlow`，少直接裸 `Flow`

建议一个实践模式：

- Repository：返回 `Flow<T>`
- ViewModel：
    - 长期状态：`StateFlow<UiState>`
    - 一次性事件：`SharedFlow<UiEvent>`
- UI 层：只收集 StateFlow/SharedFlow

### 12.5 调试 Flow

- 可以用 `onEach { println(it) }` 在链路中打印调试
- 或者 `onStart { ... }` 在 Flow 启动时打印日志

```kotlin
flow
.onStart { println("start collecting") }
.onEach { println("value: $it") }
.onCompletion { println("completed") }
.collect()
```

---