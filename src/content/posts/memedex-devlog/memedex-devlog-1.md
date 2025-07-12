---
title: 'MemeDex开发日记 #1'
published: 2025-06-08
description: 'not a tutorial'
image: './assets/3de79bf76f91decdad97db4732529a3fb169f134.jpg'
tags: [Android, Kotlin, Jetpack Compose, MemeDex]
category: 'coding'
draft: false 
---

# 一.权限申请

Android 的权限申请一直是让人很头疼的问题,特别是从 Android 10开始,存储权限几乎是每个大版本都会更新,但好在万能的开源社区已经提供了现成的解决方案 [Accompanist](https://github.com/google/accompanist).尽管如此,针对不同安卓版本适配图片读取逻辑还是不可避免的,秉持着先做一坨垃圾出来的原则, 这里优先适配 Andriod 14

## 1. Android 14

1.先在 build.gradle 里添加Accompanist的依赖

```kotlin
…
dependencies {
    …
    implementation (libs.accompanist.permissions)
}
…
```

2.在 Manifest 中声明权限

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <uses-permission android:name="android.permission.READ_MEDIA_IMAGES"/>
    …
    </application>

</manifest>
```

3.申请权限

这里还是做垃圾原则,直接在 `LaunchedEffect` 中处理权限申请逻辑.这种实现方式的特点是：

1. 仅在 Composable 首次组合时执行一次

2. 通过调用 `PermissionState.launchPermissionRequest()` 即可进行权限申请

```kotlin
fun HomeScreen(){
    val context = LocalContext.current
    val readMediaImagePermission = rememberPermissionState(android.Manifest.permission.READ_MEDIA_IMAGES)


    LaunchedEffect(Unit) {
        if (!readMediaImagePermission.status.isGranted){
            readMediaImagePermission.launchPermissionRequest()
        }
        else{
            …
        }
    }
}
```

在 **Accompanist** 的帮助下,申请权限的代码就只需要以上寥寥几行(当然还没有考虑到各种包括但不限于被权限申请被拒绝的情况~~,这些当然是统统交给未来的我啦~~)

# 二.图片读取

在完成权限申请后,下一步是读取设备中的图片.针对 **Android 14**,如果未申请 **MANAGE_EXTERNAL_STORAGE** (访问全部文件权限),官方推荐使用 **MediaStore API** 来安全访问媒体文件,这里甩一个 Google 官方文档的链接 [访问共享存储空间中的媒体文件](https://developer.android.com/training/data-storage/shared/media)

```kotlin
withContext(Dispatchers.IO){
                val collection = MediaStore.Images.Media.EXTERNAL_CONTENT_URI
                val projection = arrayOf(
                    MediaStore.Images.Media._ID,
                    MediaStore.Images.Media.DISPLAY_NAME,
                    MediaStore.Images.Media.DATE_ADDED
                )
                val query = context.contentResolver.query(
                    collection,
                    projection,
                    null,
                    null,
                    null
                )
                val list = mutableListOf<Uri>()

                query?.use { cursor ->
                    val idColumn = cursor.getColumnIndexOrThrow(MediaStore.Images.Media._ID)
                    while (cursor.moveToNext()) {
                        val id = cursor.getLong(idColumn)
                        val contentUri = ContentUris.withAppendedId(collection, id)
                    }
                }
}
```

图片读取操作在**协程**中执行,通过 `query` 方法传入参数可指定：

- **媒体类型**（如仅查询图片）

- **返回列**（筛选需要的字段）

- **排序方式**（如按日期降序）

- 等等

当前实现仅使用了前两项配置,最终获取所有可用图片的 `Uri`,供后续展示使用.

# 三.图片展示

目前计划使用瀑布流展示图片,简单搭建脚手架后,直接使用 LazyVerticalGrid,没错Compose 已经内置了现成的瀑布流布局~~,让人怀念起给 RecyclerView 写 Adapter 的峥嵘岁月(~~ 图片加载库选择了与 Kotlin 和协程兼容性更好的 **Coil**

```kotlin
LazyVerticalGrid(
        columns = GridCells.Fixed(2),
        horizontalArrangement = Arrangement.spacedBy(10.dp),
        verticalArrangement = Arrangement.spacedBy(10.dp)
    ) {
        items(imageList){ uri ->
            AsyncImage(
                model = uri,
                contentDescription = null
            )
        }
    }
```

# 四.效果

<img src="/images/2025-06-09-00-21-16-image.jpg" alt="效果图" data-align="center">

可以看出产物十分吻合开发理念(
