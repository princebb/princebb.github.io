---
layout:     post
title:      "Flutter doctor 提示 Android license status unknown解决方案"
date:       2020-03-15
author:     "Allen"
header-img: "img/code-bg.jpg"
tags:
    - Flutter
---

在安装完Flutter，运行`flutter doctor`检测依赖时提示

```
✗ Android license status unknown.
      Try re-installing or updating your Android SDK Manager.
      See https://developer.android.com/studio/#downloads or visit https://flutter.dev/setup/#android-setup for detailed instructions.
```

搜索网上的解决方案，大都让输入`flutter doctor --android-licenses` ，但我键入上述命令后，又提示

```
Android sdkmanager tool not found 
Try re-installing or updating your Android SDK,
    visit https://flutter.dev/setup/#android-setup for detailed instructions.
```

经排查，原来是最新版本的Android SDK将**Android SDK tools**包重命名成了**Android-SDK command line tools**，但是Flutter当前最新的稳定版本并没有兼容该项改动，所以有此提示，解决的方法也简单，只需要安装一个旧版本的**Android SDK Tools (Obsolete) 26.1.1**即可(取消**Hide Obsolete Packages**选择框即可见)。

![图1-1 sdk_tools](/img/flutter/sdk_tools.png)