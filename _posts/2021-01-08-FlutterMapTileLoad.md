---
layout:     post
title:      "Flutter map瓦片加载及计算原理"
date:       2021-01-08
author:     "Allen"
header-img: "img/code-bg.jpg"
tags:
    - Flutter
---


前阵子做地图的离线功能时，需要按显示区域下载地图区块的Tiles图片，所以对[Flutter_map](https://github.com/fleaflet/flutter_map)的Tiles加载逻辑进行了一番研究。在此，特地整理一下。

在这里，先复习一下两个小学地理学的概念吧。

##### 纬度和经度

如果将世界想象成一个围绕其轴自旋转的球体，那么北极位于最顶部，南极位于最底部，赤道则是围绕在其中间的一个假象线。

沿着赤道南北两边，画出许多和赤道平行的圆圈，叫做“纬圈”；同理，从北极到南极点也可以画出许多与赤道垂直的圆圈，叫做“经圈”。

![graticule](/img/flutter/graticule.png)

而纬度可以简单理解成与赤道形成的夹角，处于赤道为0°，赤道以北的纬度叫北纬N，以南的纬度叫南纬S，南或北极点最高为90°。经度则是从本初子午线(格林尼治天文台的经线)开始测量的角度，以东叫东经E，以西叫西经W。

经度和纬度最大的区别在于，纬度在球体上距离始终相等，但经度的线在赤道距离最远，在两极靠的更近。

![latitude_longitude](/img/flutter/latitude_longitude.png)

##### 墨卡托投影

将3D地球上的点转换为2D平面的方法称为地图投影，而Flutter_map采用的是Google Maps的墨卡托地图投影，选择墨卡托投影最大的好处在于南北方向是直上直下的，东西方向也是左右直线的，而不是像其他投影一样，会随着移动偏离或者弯曲地图。墨卡托投影最大的缺点在于，距离赤道越远，国家的规模大小会被严重扭曲。

试想一下你剥橘子皮的场景，结合下图，应该就能理解我表述的意思了。可以看到，如果将地图铺平，越靠近南北两极，拉伸的幅度越大。

##### ![unwrapping_globe](/img/flutter/unwrapping_globe.png)Tiles加载

在Flutter_map中，地图的加载参照了[Google Map](https://developers.google.com/maps/documentation/javascript/coordinates)的做法，不以整张图片进行下载，而是将图片分解为一个个较小的图块，彼此拼接起来的。为什么这么做呢？主要原因还是图片的尺寸大小。在Google Map最高缩放级别下，即使进行了图像压缩，图片的大小也将超过5亿像素平方，超过了25000TB，如果是浏览器进行下载，至少需要花费6年以上。并且，该方案也会增加服务器的负载，为每个用户生成合适的屏幕尺寸的图片将消耗很大的资源，并且不同用户间的复用率也很低，平移几个像素的地图下载一个新的图片，对于用户体验也不怎么友好。而使用Tiles的好处就在于每个人都可以共享同一个图片，服务器也可以进行缓存，平移几个像素也只用下载几个相应的图片即可。

根据不同的缩放等级，Tiles的数量也有所不同。在最缩小的缩放级别(Zoom Level 0)下，整个地图由单个256 x 256像素的正方形图块组成。每增加一个缩放级别，地图在每个方向上的大小都会加倍缩放时，即替换为4个更详细的图块(2x2)，每个图块仍然只有256 x 256像素。由图可见，虽然总宽度和高度在每个缩放级别仅增加一倍，但是面积确增加的很快，在Zoom Level 21时，已经包含4万亿个图块。

![tiles_zoom_level](/img/flutter/tiles_zoom_level.png)

##### Tiles计算

在Flutter Map中，使用坐标系进行计算，还需要理解3个坐标概念：world coordinates(世界坐标)、pixel coordinates(像素坐标)、 tile coordinates(瓦片坐标)。

世界坐标与Zoom Level没有关系，用于经纬度与地图上的当前位置之间进行转换。它们是相对于Zoom Level 0的单个图块计算的，纬度和经度映射到该图块上的x和y像素。

```dart
static const TILE_SIZE = 256;

WorldCoordinate project(LatLng latLng) {
  var siny = Math.sin((latLng.latitude * Math.pi) / 180);
  // Truncating to 0.9999 effectively limits latitude to 89.189. This is
  // about a third of a tile past the edge of the world tile.
  siny = Math.min(Math.max(siny, -0.9999), 0.9999);
  return WorldCoordinate(TILE_SIZE * (0.5 + latLng.longitude / 360),
      TILE_SIZE * (0.5 - Math.log((1 + siny) / (1 - siny)) / (4 * Math.pi)));
}

class WorldCoordinate {
  final double x;
  final double y;
  WorldCoordinate(this.x, this.y);
}
```

像素坐标是特定Zoom Level下经纬度的具体像素位置，可以通过世界坐标乘以缩放级别来计算。

```dart
CustomPoint toPixelCoordinate(LatLng latLng, int zoom) {
  final scale = 1 << zoom;
  final worldCoordinate = project(latLng);
  final x = (worldCoordinate.x * scale).floor();
  final y = (worldCoordinate.y * scale).floor();
  return CustomPoint(x, y);
}
```

瓦片坐标则是由客户端告诉服务器请求的具体图块。从左上角开始为(0,0)，往右和下方依次递增。瓦片坐标也与Zoom Level有关，计算方式则是像素坐标除以图块大小取整即可。![tile_coordinates](/img/flutter/tile_coordinates.png)

```dart
Coords toTileCoords(LatLng latLng, int zoom) {
  final scale = 1 << zoom;
  final worldCoordinate = project(latLng);
  final x = ((worldCoordinate.x * scale) / TILE_SIZE).floor();
  final y = ((worldCoordinate.y * scale) / TILE_SIZE).floor();
  return Coords(x, y)..z = zoom;
}
```

通过对上述知识点的理解，相信阅读Flutter map的[tile_layer](https://github.com/fleaflet/flutter_map/blob/master/lib/src/layer/tile_layer.dart)也不存在什么难度了。
