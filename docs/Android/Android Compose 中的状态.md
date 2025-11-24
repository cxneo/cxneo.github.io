状态表示 UI 组件中保存的数据，通常是与界面显示相关的内容。状态的变化会导致界面的重新绘制（recomposition）。例如，按钮的文本、输入框的内容、显示的列表等，都是状态的一部分。如果重新绘制的时候不保存相关状态（可以理解成变量值），那么重新绘制后的值就会恢复成初始值。

## 如何定义和使用状态？
在 Compose 中，状态通常通过 remember 和 mutableStateOf 来管理。remember 用来在组合（composing）时记住状态，而 mutableStateOf 是用来声明一个可变状态。
- remember 用于在组合时记住某个状态。这意味着，当 Compose 重新组合某个 @Composable 函数时，如果这个状态已经存在，就不会重新创建它，而是直接使用之前的值。
- mutableStateOf 用来声明一个可变的状态，它是一个封装了初始值的可变对象。你可以通过修改这个状态的值来触发 UI 更新。

**示例：按钮点击计数器**
```kotlin
@Composable
fun Counter() {
    // 记住状态值，初始值为 0
    var count by remember { mutableStateOf(0) }

    // 显示按钮和文本
    Column(
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center,
        modifier = Modifier.fillMaxSize()
    ) {
        Text(text = "Count: $count")
        Button(onClick = {
            // 每次点击按钮，count 增加 1
            count += 1
        }) {
            Text("Increase")
        }
    }
}
```
在这个例子中：
- remember 同时保证了状态 count 只会在第一次组合时创建，并且在之后的重新组合中不会丢失。mutableStateOf(0) 用来定义状态 count，它的初始值是 0。
- 每次点击按钮，count 状态增加 1，Compose 会自动重新绘制 UI 来反映新的状态。

状态变化时，Compose 会根据新的状态重新绘制 UI 组件，而无需显式调用更新方法。通过将状态与 UI 绑定，Compose 只会重绘状态发生变化的部分，而不是整个界面。

## 常见的状态用法
1. 状态在列表中的使用：在实际工程中，状态经常用于管理列表的滚动位置、选中的项目等。
```kotlin
@Composable
fun TodoList() {
    // 状态：管理 todo 列表
    var todos by remember { mutableStateOf(listOf("Learn Kotlin", "Learn Compose", "Build App")) }

    Column {
        // 显示列表
        todos.forEach { todo ->
            Text(todo)
        }

        // 按钮添加任务
        Button(onClick = {
            todos = todos + "New Task" // 添加新任务
        }) {
            Text("Add Task")
        }
    }
}
```
2. 状态与生命周期：状态也可以与组件的生命周期相关联。你可以在 LaunchedEffect 中处理一些异步任务，例如加载数据。
```kotlin
@Composable
fun LoadData() {
    // 状态：管理加载中的数据
    var data by remember { mutableStateOf("Loading...") }

    // 异步加载数据
    LaunchedEffect(Unit) {
        delay(2000)
        data = "Data Loaded!"
    }

    Text(text = data)
}
```

3. 其他常见应用场景
- 表单数据： 用状态来跟踪输入框内容、复选框选择等。 
- 页面加载状态： 用状态表示数据是否加载完成，展示加载指示器或内容。 
- 列表的滚动位置和分页： 使用状态来跟踪用户的滚动位置或当前页数。



