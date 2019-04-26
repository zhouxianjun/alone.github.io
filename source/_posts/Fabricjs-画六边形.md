---
title: Fabricjs 画六边形
date: 2019-04-26 16:33:41
tags:
- fabric
- js
- 六边形

categories:
- 前端
- fabric
---

fabricjs，就是一个前端画板，canvas的增强版。

使用fabricjs来画六边形和canvas画其实原理是一样的，只是方法不同。

1. 设定变量： 
    - `start` 鼠标按下，开始的事件对象
    - `upObj` 在鼠标没有松开的情况下上一次绘画的对象
2. 绑定画板事件`mouse:down`设置开始事件
```javascript 1.6
canvas.on("mouse:down", o => (start = o.e));
```
3. 绑定画板事件`mouse:up`绘画结束，清空开始事件对象，并清空上一次绘画对象
```javascript 1.6
canvas.on("mouse:up", o => {
  start = null;
  upObj = null;
});
```
4. 绑定画板事件`mouse:move`绘画，也可以在`mouse:up`时绘画，但这样体验效果不好
```javascript 1.6
canvas.on("mouse:move", o => {
  if (!start) {
    return;
  }
  if (upObj) {
    canvas.remove(upObj);
  }
  const { offsetX: sx, offsetY: sy } = start;
  const { e: { offsetX: ex, offsetY: ey } } = o;
  // 计算半径，边长
  const R = Math.sqrt((ex - sx) * (ex - sx) + (ey - sy) * (ey - sy)) / 2;
  // 6条边的坐标点（60是六边形的内角度数）
  const points = Array.from({ length: 6 }).map((item, index) => {
    return {
      x: Math.cos(60 * index / 180 * Math.PI) * R,
      y: Math.sin(60 * index / 180 * Math.PI) * R
    };
  });
  upObj = new fabric.Polygon(points, {
    top: sy,
    left: sx,
    stroke: "red",
    fill: "rgba(255,255,255,0)"
  });
  canvas.add(upObj);
});
```

{% codepen VNqqEL [js,result] 600 800px zhouxianjun 0 %}
