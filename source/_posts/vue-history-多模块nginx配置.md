---
title: vue history 多模块nginx配置
date: 2020-07-07 09:56:56
tags:
- vue
- history
- nginx
- vue router

categories:
- 前端
- vue
- nginx
---

vue-router history模式比hash模式有很多好处，我在这里就不详细说了，主要有一点：
当我们用分享的时候，或者把url暴露给别人用的时候，很多时候会经过url编码就会导致hash的#符号被转码，然后导致整个路由错误。

### 问题
当我们使用history模式的时候一般是需要配置nginx的tryfiles的，如果在一个nginx 域名中使用了2个或者多个vue spa 的history模式则会
导致无法进入其他模块的页面，因为都会被tryfiles转到/的index.html页面。

### 解决

### 第一个模块为`/`其次都在第一个模块内
#### 第一个模块，首页模块
```
location / {
    root   /usr/vue/project1/dist/;
    index  index.html index.htm;
    try_files $uri $uri/ /index.html;
}
```

#### 第二个模块
这里的project2其实是放在project1的dist里面的。当然也放在外面更好配置，这里要注意的是，只能用alias不能用root。而这里的try_files的root则为第一个模块的root。
```
location /project2 {
    try_files $uri $uri/ /project2/$uri /project2/index.html;
    alias /usr/vue/project1/dist/project2/;
}
```
这里需要注意的是，当我们第二个或者第三个模块的进入点是在第一个里面的话则vue-router的base必须与其一致，例如：
第一个模块地址: https://www.alone.com/project1
第二个模块地址: https://www.alone.com/project1/project2
那router的base为：/project1/project2 否则无法进入路由。

### 不在第一个模块内

#### 第一个模块
```
location /project1 {
    root   /usr/vue/;
    index  index.html index.htm;
    try_files $uri $uri/ /project1/dist/index.html;
}
```
路由base配置为: /project1
访问地址: https://www.alone.com/project1

#### 第N个模块
```
location /project2 {
    alias /usr/vue/project2/dist/;
    index  index.html index.htm;
    try_files $uri $uri/ /project2/dist/index.html;
}
```
路由base配置为: /project2
访问地址: https://www.alone.com/project2

