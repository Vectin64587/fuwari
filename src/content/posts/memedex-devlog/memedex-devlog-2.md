---
title: 'MemeDex开发日记 #2'
published: 2025-07-10
description: '界面框架与底部导航栏实现'
image: './assets/9358d109b3de9c82778d779e2a81800a19d84333.jpg'
tags: [Android, Kotlin, Jetpack Compose, MemeDex]
category: 'coding'
draft: false 
---

## UI 框架大概思路

- 主界面
  
  - Home
    
    - 展示尚未分类的 meme 瀑布流
  
  - Album
    
    - 展示 meme 组 提供搜索功能
  
  - Settings

## 底部导航栏的实现

在 Compose 中，底部导航栏的实现存在不少问题。Material Design 从 Material 2 演进到 Material 3 后，`BottomNavigation` 组件已被迁移至 `NavigationBar`。然而，**Google** 官方的[文档](https://developer.android.com/develop/ui/compose/navigation#bottom-nav)至今仍在使用旧版的 `BottomNavigation` 写法，令人疑惑。接下来，我们将介绍 **Navigation** 相关的几个关键概念。

| 概念          | 目的                                                    | 类型                             |
| ----------- | ----------------------------------------------------- | ------------------------------ |
| Host        | 承载当前导航目标的 UI 元素。当用户在应用中导航时，应用会将不同的目标放进或移出自己，从而完成界面切换。 | NavHost                        |
| Graph       | 定义应用内所有导航目标及它们之间连接关系的数据结构。                            | NavGraph                       |
| Controller  | 管理目标之间导航的核心协调者。提供导航方法、深度链接处理、返回栈管理等功能。                | NavController                  |
| Destination | 导航图中的一个节点。用户导航到该节点时，主机会显示对应内容。                        | NavDestination<br>通常在构建导航图时创建。 |
| Route       | 唯一标识一个目的地及其所需数据。通过路由执行导航，路由会把你带到对应的目的地。               | 任何可序列化的数据类型                    |

首先需要获取 **`navController`** 对象，并定义应用需要导航的页面路由。这里我使用了一个预先定义的 `TopLevelRoute` 数据类来封装路由信息。由于该类的实现较为简单，这里不再展开说明。

```kotlin
val navController = rememberNavController()
val topLevelRoutes = listOf(
        TopLevelRoute("Home", "Home", Icons.Filled.Home),
        TopLevelRoute("Album", "Album", Icons.Filled.Favorite),
        TopLevelRoute("Settings", "Settings", Icons.Filled.Settings),
)
```

将 `NavigationBar` 组件作为 `bottomBar` 参数传入 `Scaffold`。

```kotlin
bottomBar = {
            NavigationBar {
                val navBackStackEntry by navController.currentBackStackEntryAsState()
                val currentDestination = navBackStackEntry?.destination
                topLevelRoutes.forEach{ topLevelRoute ->
                    NavigationBarItem(
                        icon = { Icon(topLevelRoute.icon, contentDescription = topLevelRoute.name) },
                        label = { Text(topLevelRoute.name) },
                        selected = currentDestination?.route == topLevelRoute.route,
                        onClick = {
                            navController.navigate(topLevelRoute.route){
                                popUpTo(navController.graph.findStartDestination().id){
                                    saveState = true
                                }
                                launchSingleTop =true
                                restoreState = true
                            }
                        }
                    )
                }
            }
        }
```

这里的 `navBackStackEntry` 变量委托给了 `navController` 的**扩展函数**，返回的是 `BackStackEntry` 的**栈顶**，也就是**最后导航到的页面**。另外，当导航栈顶发生改变时，会触发**重组**。  
而 `currentDestination` 则是**当前**界面的 `route`，需要用到它来判断 `NavigationBarItem` 的 `selected` 属性。  
可以看到，调用 `navController` 的 `navigate` 方法时，后面还传入了一个可选的 `builder`，作用如下：

- popUpTo(destinationId)
  
  - 在导航之前从回退栈中移除给定 `destinationId` 之上的 `entries`,确保每次导航到一个 destination 时, 都能回到根目录, 同时 `saveState` 设为 `true` 能将弹出的 `destination` 中的ui和状态进行缓存

- launchSingleTop =true
  
  - 如果目标 `destination` 已经在栈顶,那么不再创建一个新的实例,而是直接使用已存在的,确保重复点击 Tab 时不会创建多个相同实例

- restoreState = true
  
  - 恢复之前 `popUpTo` 保存的状态数据

接下来在传入的 Scaffold 的 content 中添加之前提到的 NavHost

```kotlin
NavHost(
            navController,
            startDestination = "Home",
            modifier = Modifier.padding(innerPadding)
        ){
            composable("Home"){
                HomeScreen()
            }
            composable("Album") {
                AlbumScreen()
            }
            composable("Settings"){
                SettingsScreen()
            }
}
```

底部导航栏的简单实现到这里差不多就结束了,下面是对 **`LazyVerticalGrid`** 的简单修改

```kotlin
LazyVerticalGrid(
        columns = GridCells.Fixed(2),
        horizontalArrangement = Arrangement.spacedBy(10.dp),
        verticalArrangement = Arrangement.spacedBy(10.dp)
    ) {
        items(imageList){ uri ->
            Card(
                modifier = Modifier.size(80.dp, 200.dp)
                    .padding(start = 5.dp),
                elevation = CardDefaults.cardElevation(6.dp)
            ) {
                AsyncImage(
                    model = uri,
                    contentDescription = null,
                    modifier = Modifier.fillMaxSize(),
                    contentScale = ContentScale.Crop
                )
            }
        }
    }
```

其中给 `AsyncImage` 套了 `Card`，加入了圆角和阴影，效果如下。

<img src="/images/2025-07-12-17-05-17-image.jpg" alt="效果图" data-align="center" width="300" height="636">
