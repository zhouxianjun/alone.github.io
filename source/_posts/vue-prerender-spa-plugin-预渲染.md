---
title: vue prerender-spa-plugin 预渲染
date: 2020-06-18 13:23:25
tags:
- vue
- 预渲染
- prerender-spa-plugin

categories:
- 前端
- vue
---

在做VUE H5单页面轻应用的时候，如果不优化则进入页面会很长时间的白屏，有很多种优化，我们这里主要讲`prerender-spa-plugin`预渲染。

## 原理：
[prerender-spa-plugin](https://github.com/chrisvfritz/prerender-spa-plugin) 利用了 Puppeteer 的爬取页面的功能。 Puppeteer 是一个 Chrome官方出品的 headlessChromenode 库。它提供了一系列的 API, 可以在无 UI 的情况下调用 Chrome 的功能, 适用于爬虫、自动化处理等各种场景。它很强大，所以很简单就能将运行时的 HTML 打包到文件中。原理是在 Webpack 构建阶段的最后，在本地启动一个 Puppeteer 的服务，访问配置了预渲染的路由，然后将 Puppeteer 中渲染的页面输出到 HTML 文件中，并建立路由对应的目录。
    
例如路由：/demo/add 成品结构:

    dist
    --demo
    ----add
    ------index.html
    
## 安装
```bash
npm i -D prerender-spa-plugin
```

## 配置
在vue.config.js中配置webpack信息

1. 在顶部导入依赖
```js
const path = require('path');
const PrerenderSPAPlugin = require('prerender-spa-plugin');
```

2. 在configureWebpack节点下配置，configureWebpack为function，你也可以用chainWebpack链式配置，也可以直接在webpack配置plugins节点。
 
- configureWebpack为function
```js
config.plugins.push(new PrerenderSPAPlugin({
    // Required - The path to the webpack-outputted app to prerender.
    staticDir: path.resolve(__dirname, './dist'),
    // Required - Routes to render.
    routes: ['/demo/add']
}))
```
- chainWebpack链式
```js
config.plugin('prerender').use(PrerenderSPAPlugin, [{
    // Required - The path to the webpack-outputted app to prerender.
    staticDir: path.resolve(__dirname, './dist'),
    // Required - Routes to render.
    routes: ['/demo/add']
}]);
```

build 就能看到效果了。会在dist目录生成与路由对应的静态文件index.html。并且在你挂载的div里面有当前路由的静态元素。

##  上面说的是最简单的使用方式，在真正场景中会遇到很多问题，下面我列举几个，并给出方案。

### 当publicPath不为`/`的时候例如:`/device/`，需要在`PrerenderSPAPlugin`插件配置`indexPath`
参考issue [https://github.com/chrisvfritz/prerender-spa-plugin/issues/344#issuecomment-619475952](https://github.com/chrisvfritz/prerender-spa-plugin/issues/344#issuecomment-619475952) 
```js
new PrerenderSPAPlugin({
    // Required - The path to the webpack-outputted app to prerender.
    staticDir: path.resolve(__dirname, './dist'),
    indexPath: path.resolve(__dirname, `./dist${config.output.publicPath}index.html`),
    // Required - Routes to render.
    routes: ['/device/demo/add']
})
```
配置vue.config.js属性
```js
{
    publicPath: '/device/',
    outputPath: 'dist/device/'
}
``` 
并且会打包到dist/device/目录。

### 在预渲染的时候不初始化某些东西，例如：不想把`vConsole`预渲染。
配置 插件的render属性
```js
new PrerenderSPAPlugin({
    // Required - The path to the webpack-outputted app to prerender.
    staticDir: path.resolve(__dirname, './dist'),
    indexPath: path.resolve(__dirname, `./dist${config.output.publicPath}index.html`),
    // Required - Routes to render.
    routes: ['/device/demo/add'],
    renderer: new Renderer({
        injectProperty: '__PRERENDER_INJECTED__',
        inject: 'prerender'
    })
})
```
然后在`vConsole`初始化的地方判断
```js
const isPrerender = window.__PRERENDER_INJECTED__ === 'prerender';
if (!isPrerender) {
    // 初始化vConsole
}
```

### 想在vue挂载后渲染
配置 插件的render属性
```js
new PrerenderSPAPlugin({
    // Required - The path to the webpack-outputted app to prerender.
    staticDir: path.resolve(__dirname, './dist'),
    indexPath: path.resolve(__dirname, `./dist${config.output.publicPath}index.html`),
    // Required - Routes to render.
    routes: ['/device/demo/add'],
    renderer: new Renderer({
        renderAfterDocumentEvent: 'render-event'
    })
})
```
在你挂载的mounted钩子触发
```js
mounted () {
    document.dispatchEvent(new Event('render-event'));
}
```

### 极致预渲染，就是让script资源不影响浏览器渲染
> 先渲染完元素，再按顺序加载js资源 [tiny-loader](https://github.com/youzan/tiny-loader.js)


安装 jsdom 工具
```bash
npm i -D cheerio
```
在 PrerenderSPAPlugin 插件配置postProcess钩子
```js
config.plugins.push(new PrerenderSPAPlugin({
    // ...
    postProcess (renderedRoute) {
        const $ = cheerio.load(renderedRoute.html);
        const jsSet = new Set();
        // 先把页面的script脚本地址提取并用set去重
        $('script').each((index, s) => jsSet.add(s.attribs.src));
        // 过滤null
        const jsPaths = [ ...jsSet ].filter(s => !!s);
        const js = `
            Loader.sync(${JSON.stringify(jsPaths)});
        `;
        // 删除script标签
        $('script').remove();
        // 把js植入页面
        $(`<script type="text/javascript">${js}</script>`).insertAfter('body');
        // 把tiny-loader.js植入页面
        $(`<script src="${config.output.publicPath}tiny-loader.js" type="text/javascript"></script>`).insertAfter('body');
        renderedRoute.html = $.html();

        return renderedRoute;
    }
}));
```

### kubernetes ingress nginx 容器路由，导致丢失publicPath
> 访问路径如果不为`/`结尾，并且访问的为目录（当不做预渲染的时候不会生成路由对应的目录与html则不会有问题）nginx则会重定向到`/`结尾，会导致
> 内部重定向时跳出容器nginx从而丢失publicPath。

修改容器的nginx配置
```.env
location / {
  root   /usr/vue/device/dist/;
  if (-d $request_filename) {
    rewrite ^/(.*)([^/])$ $scheme://$host/device/$1$2/ permanent;
  }
  index  index.html index.htm;
  try_files $uri $uri/ /index.html;
}
```
其中`device`就是publicPath


## 参考

- [1] [浏览器渲染机制](https://segmentfault.com/a/1190000004292479)
- [2] [前端prerender-spa-plugin预渲染](https://segmentfault.com/a/1190000018182165?utm_source=tag-newest)
- [3] [nginx下URL末尾自动加斜杠](https://www.cnblogs.com/z-books/p/12410840.html)

