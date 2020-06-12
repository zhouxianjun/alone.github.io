---
title: 制作自己的npm包并排除第三方依赖
date: 2020-06-12 10:29:55
tags:
- npm
- npmjs
- 制作npm

categories:
- 前端
- npm
---

在工作中，我们可能会遇到第三方包所提供的功能无法满足我们的需求的时候，或者公司公共组件。我们需要制作自己的npm包。

### 前提
1. 在`npmjs.org`注册账户（公司私服则需要运维开账户）。
2. 检查本地`npm config registry`是否为`npmjs.org`或者公司私服地址。
3. 本地npm登录`npm login`输入账户/密码。然后会让你输入邮箱地址，这个为npmjs上配置的邮件地址。

### 编译npm项目 
> 可参考此项目 [https://github.com/zhouxianjun/muse-ui-fabric](https://github.com/zhouxianjun/muse-ui-fabric)

1. 使用vue cli3 创建项目。
2. 如果组件中有使用到第三方依赖包并且不想编译在自己的组件中，则需要配置vue.config.js externals
> 我这里只是做了一个例子，实际情况看你有哪些第三方依赖不想编译则自行配置

```js
{
    chainWebpack: config => {
        if (process.env.NODE_ENV === 'production') {
          config.externals({
                'lodash': {
                    commonjs: "lodash",//如果我们的库运行在Node.js环境中，import _ from 'lodash'等价于const _ = require('lodash')
                    commonjs2: "lodash",//同上
                    amd: "lodash",//如果我们的库使用require.js等加载,等价于 define(["lodash"], factory);
                    root: "_" // 这个为lodash暴露的变量
                }
          });
        }
    }
}
```
3. 在新建的项目中的components 目录下的文件全部删除（需要先把引用到的地方删除），然后把你的代码写在里面（当然你也可以不放在components中，只是我喜欢放在这里）。
4. 自己可以在App.vue中引用自己的组件测试。
5. 测通过后使用vue cli 打包
```bash
vue-cli-service build --mode production --target lib ./src/components/index.js
```

### 发布
1. 在`package.json`中把private修改为false（如果你买了私有仓库就不用改）。
2. 在`package.json`新增或修改`main`字段为：`dist/你的项目名称.umd.min.js`。
3. 发布到仓库
```bash
npm publish
```
4. 如果修改了代码想要重新发布，则必须先把`package.json`中的版本号`version`自行修改。
5. 发布后，可以再新建一个空的vue项目，直接引用你刚才发布的组件，如果你组件中有依赖第三方组件，则npm会自动下载所需要的所有依赖，
如果在使用项目中不想让组件中的依赖编译到chunk-vendors文件中，则可以在使用项目中配置externals，例如：
```js
{
    chainWebpack: config => {
          config.externals({
                'lodash': '_'
          });
    }
}
```

好了，今天就讲到这里，如有同学需要查看分析包内容则可以使用`vue ui`在任务中运行一个task然后点击分析，就可以分析每个打包后的js由哪些组成了。
