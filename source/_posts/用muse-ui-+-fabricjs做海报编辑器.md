---
title: 用muse ui + fabricjs做海报编辑器
date: 2019-05-01 11:09:47
tags:
- fabric
- poster
- muse ui
- 海报

categories:
- 前端
- fabric
---

## 功能点
### 添加图片
> 由于画板上会有很多图片元素，并且我们需要对每个元素设置不同对圆角以及透明度，所以我们需要在添加图片的时候给当前元素生成一个唯一ID来标示它，并且把它的所有配置属性存储。

#### 图片源
```javascript 1.6
/**
* @param url 图片数据：本地地址、远程地址、base64
* @param options 图片配置：具体请查看fabric官网doc，这里注意的是，如果图片源为远程地址，则需要配置跨域访问
*/
async loadImg (url, options = {}) {
    return new Promise(resolve => {
        fabric.Image.fromURL(url, img => resolve(img), {
            ...this.optionsOfType('image'),
            ...options
        });
    });
}
```
#### 圆角
> 主要使用图片的`clipTo`的回调函数，我们可以在当前函数对当前图片画圆`arc`.这里其实就是canvas的arc函数。
```javascript 1.6
ctx => ctx.arc(0, 0, this.radius[img.id].val, 0, Math.PI * 2, true)
```
#### 透明度
> 这里只要设置当前选择图片的style（opacity）即可，然后再重新render画板。
```javascript 1.6
setActiveStyle (style) {
    const object = this.canvas.getActiveObject();
    if (!object) return;
    if (object.setSelectionStyles && object.isEditing) {
        object.setSelectionStyles(style);
    } else {
        Reflect.ownKeys(style).forEach(item => object.set(item, style[item]));
    }
    object.setCoords();
    this.canvas.requestRenderAll();
}
```
### 添加文字
> 这里采用fabric的`IText`函数创建一个文本控件。
```javascript 1.6
addText (content = 'text', options = {}) {
    const text = new fabric.IText(content, Object.assign(this.optionsOfType('i-text'), {
        left: this.canvas.width / 2,
        top: this.canvas.height / 2,
        fill: '#000000',
        scaleX: 0.5,
        scaleY: 0.5,
        hasRotatingPoint: true
    }, options));
    this.addAndSelect(text);
}
```
#### 透明
> 与图片透明度一样，设置`opacity`样式即可。
#### 字体
> 需要设置控件属性`fontFamily`。目前实现的字体：'Arial', 'Helvetica', '宋体', '黑体', '微软雅黑', '楷体', '仿宋', 'Verdana', 'Times New Roman', 'Roboto', 'Open Sans', 'Lato', 'Bellefair', 'Fresca', 'Raleway'
#### 大小
> 设置控件属性`fontSize`
#### 对齐方式
> 设置控件属性`textAlign`:left、center、right、justify
#### 颜色
> 使用第三方选择颜色组件：`vue-color`中的`Sketch`具体去npm查看。获取rgba。设置控件的`fill`样式即可。
#### 字体样式
* 粗体 - 设置属性`fontWeight`为`blob`或者`normal`
* 斜体 - 设置属性`fontStyle`为`italic`或者`normal`
* 下划线 - 设置属性`underline`为`true`或者`false`
* 删除线 - 设置属性`linethrough`为`true`或者`false`
* 上划线 - 设置属性`overline`为`true`或者`false`
#### 行高
> 设置控件属性：`lineHeight`
### 层级移动
#### 上移
> 获取当前选择控件调用控件`bringToFront`函数
#### 下移
> 获取当前选择控件调用控件`sendToBack`函数
### 背景图片
> 与添加图片一样可以从多种数据源添加。
```javascript 1.6
setBackgroundImage (img, opt = {}) {
    const { _radius, _resizeMode, _opacity } = opt;
    // 获取图片最大的半径
    this.backgroundImageRadiusMax = this.imageMaxRange(img);
    // 设置图片当前圆角半径
    this.backgroundImageRadius = _radius || this.backgroundImageRadiusMax;
    // 设置当前图片填充模式
    this.backgroundImageResizeMode = _resizeMode || this.backgroundImageResizeMode;
    // 根据当前画布大小适配图片比例
    const { width, height } = this.canvas;
    const scaleX = width / img.width;
    const scaleY = height / img.height;
    if (this.backgroundImageResizeMode === ResizeMode.stretch) {
        // 如果上拉伸填充则不保持百分比直接按画布比例缩放
        img.set({ scaleX, scaleY, left: 0, top: 0 });
    } else {
        // 如果以覆盖整个画布填充则按照最大的画布比例缩放图片宽高，反之为画布的最小比例缩放。
        const isCover = this.backgroundImageResizeMode === ResizeMode.cover;
        const scale = isCover ? Math.max(scaleX, scaleY) : Math.min(scaleX, scaleY);
        img.scale(scale).set({
            left: width / 2 - img.width * scale / 2,
            top: height / 2 - img.height * scale / 2
        });
    }
    this.canvas.setBackgroundImage(img, this.canvas.renderAll.bind(this.canvas), {
        originX: 'left',
        originY: 'top',
        opacity: _opacity || 1,
        clipTo: ctx => ctx.arc(0, 0, this.backgroundImageRadius, 0, Math.PI * 2, true)
    });
    this.currentBackgroundImg = img;
}
```
### 背景颜色
```javascript 1.6
this.canvas.setBackgroundColor(this.rgba, this.canvas.renderAll.bind(this.canvas));
```
### 删除元素
> 获取当前控件，从画布中删除
```javascript 1.6
removeActive () {
    const active = this.canvas.getActiveObject();
    if (active) {
        this.remove(active);
    } else {
        const objects = this.canvas.getActiveObjects();
        this.canvas.discardActiveObject(null);
        objects.forEach(object => this.remove(object));
    }
}
remove (obj) {
    if (obj && obj._remove !== false) {
        this.canvas.remove(obj);
        Reflect.deleteProperty(this.radius, obj.id);
    }
}
```
### 保存&导出
```javascript 1.6
async save () {
    const before = await this.beforeSave(this.canvas.getObjects().lenght, this.canvas.backgroundImage, this.canvas.backgroundColor);
    if (before === false) {
        return;
    }
    if (!fabric.Canvas.supports('toDataURL')) {
        alert('This browser doesn\'t provide means to serialize canvas to an image');
    } else {
        const json = this.canvas.toJSON(['id']);
        if (json.backgroundImage) {
            json.backgroundImage._radius = this.backgroundImageRadius;
            json.backgroundImage._resizeMode = this.backgroundImageResizeMode;
            json.backgroundImage._opacity = this.backgroundImageOpacity;
        }
        json.objects.filter(o => o.type === 'image' && Reflect.has(this.radius, o.id)).forEach(o => {
            o._radius = this.radius[o.id].val;
            Reflect.deleteProperty(o, 'id');
        });
        this.onSave(json, this.canvas.toDataURL('png'));
    }
}
```
### 加载数据
```javascript 1.6
async loadJSON (val) {
    if (val) {
        const json = typeof val === 'string' ? JSON.parse(val) : val;
        if (json.background) {
            this.canvas.setBackgroundColor(json.background, this.canvas.renderAll.bind(this.canvas));
        }
        if (json.backgroundImage) {
            const img = await this.loadImg(json.backgroundImage.src, json.backgroundImage);
            this.setBackgroundImage(img, json.backgroundImage);
        }
        if (Array.isArray(json.objects)) {
            await this.loadData(json.objects);
        }
        this.canvas.requestRenderAll();
    }
},
async loadData (data) {
    if (data) {
        data = Array.isArray(data) ? data : Array.of(data);
        for (const obj of data) {
            if (obj.type === 'image') {
                Reflect.deleteProperty(obj, 'clipTo');
                await this.addImg(obj.src, obj);
            }
            if (obj.type === 'i-text') {
                await this.addText(obj.text, obj);
            }
        }
    }
}
```

> 哎，本人懒人病发了，后面的直接贴代码了，如有不明之处或错误地方可以在当前页面评论交流，或[Github issue](https://github.com/zhouxianjun/muse-ui-fabric/issues)。
> 为了使用方便，已上传`npm` [muse-ui-fabric](https://www.npmjs.com/package/muse-ui-fabric)

{% codepen JeEboJ [js,result] 600 100% zhouxianjun 0 %}
