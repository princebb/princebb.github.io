---
layout:     post
title:      "Flutter绘制可拖拽多边形"
date:       2020-08-20
author:     "Allen"
header-img: "img/code-bg.jpg"
tags:
    - Flutter
---


近期在flutter_map开源地图插件的基础上实现了一个可拖拽polygon的功能，觉得还是蛮有意思的，故而记录一下。

先展示一下功能吧

![field](/img/flutter/field.gif)

1.在地图上可以根据手指落下的位置绘制点

2.对绘制的点进行拖拽调整，不论封闭与否

3.当大于3个点时，点击第一个点会进行闭合操作

4.线段若相交，则变红

比较郁闷的是flutter_map(0.10.1+1)并没有提供可进行拖拽的marker，所以，为实现这个功能，我们需要先扩展一个drag_marker类，因为是保密类项目，所以只能说下大概的过程，代码细节等可以参考flutter_map的其它组件的实现。将marker用手势进行包裹，然后实现onPanDown,onPanEnd,OnTap事件。在这里使用了Listener，原因是因为flutter_map内部实现了LongPress的监听，但是并没有对使用者暴露关闭的开关，所以当我们手指在marker上按下后并停留一会儿，再在marker上移动的话，并不会接收到move事件，因为事件会向上冒泡被flutter_map消费掉，所以为了解决这个问题，包裹了原始事件Listerner，对move进行监听。

```dart
Listener(
  onPointerMove: onPointerMove,
  child: GestureDetector(
    onPanDown: onPanDown,
    onPanEnd: onPanEnd,
    onTap: () {
      //marker onTap logic
      ...
    },
    child: Stack(children: [
      Positioned(
        width: marker.width,
        height: marker.height,
        left: xxx,
        top: xxx,
        child: xxx,
      ),
    ]),
  ),
);
```

再说一下如何创建marker之间的中间点吧，对绘制的点进行循环遍历，取当前index的点和下一点，进行如下计算，然后再创建一个marker加入到dragMarkers列表中。

```dart
var intermediatePoint = LatLng(
    point1.latitude +
        (point2.latitude - point1.latitude) / 2,
    point1.longitude +
        (point2.longitude - point1.longitude) / 2);
```

marker扩展写好后，其它的功能也就水到渠成了，通过监听marker的移动，实时刷新point的位置，然后更新marker和Polyline即可。线段的闭合，就是在定义marker的时候，回传点击marker的index。如果点击的是第一个marker，则在points列表中add一个firstPoint坐标点即可。

接下来说下线段相交的判断。

如果point的点数小于3个，是不需要处理相交的。一个细节需要注意下，因为之前闭合时，为了让收尾相连，所以多添加了一个first点，在相交判断时，需要拷贝一个新的list，并removeLast()。

相交算法使用的是网上的快速排斥实验以及跨立实验。

我们整个项目使用Bloc进行状态的全局管理，所以当线段相交时，通知到widget中，改变Polyline(flutter_map自带widget)的线段颜色。

