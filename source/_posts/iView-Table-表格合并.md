---
title: iView Table 表格合并
date: 2019-04-28 20:10:55
tags:
- iView
- table
- merge
- 合并单元格

categories:
- 前端
- iView
---

在项目开发中，总是有些地方需要把单元格合并的，比如订单列表：同一个订单号，不同的产品，则需要合并订单号等。
但是官方目前只支持表头的合并，还未支持单元格的合并，有人在Github上提了[Issues](https://github.com/iview/iview/issues/2751),
但是官方一直没有解决，so。。。我就自己写了，一个方法，思路是：
1. 获取到Vue中iView到Table实例；
2. 获取当前所有行数据（class为`ivu-table-row`）;
```javascript 1.6
const rows = table?.$refs?.tbody?.$el?.getElementsByClassName('ivu-table-row');
```
3. 循环每条数据，判断当前行数据的相邻数据是否相等，例如：[{id: 1}, {id: 1}]则代表第一行和第二行的id值为相邻数据，反之：[{id: 1},{id: 2},{id: 3}]则不算相邻数据。
```javascript 1.6
/**
 * 最后一个相邻数据的下标
 * @param data 数据
 * @param value 值
 * @param field 字段
 * @param index 当前数据下标
 * @returns {number} 下标
 */
const lastAdjacentIndex = (data, value, field, index) => {
    let last = 0;
    // 取出大于当前下标的所有数据
    const slice = data.slice(index);
    // 循环每个数据判断是否相等
    for (let i = 0; i < slice.length; i++) {
        if (slice[i][field] === value) {
            last = index + i;
        } else {
            // 只要有一个不相等则直接返回
            return last;
        }
    }
    return last;
};
```
4. 把相邻数据的下面所有行的当前列删除。
```javascript 1.6
// colSpanItem为已经删除列的行数据：key为行下标，值为列下标+1
const lastIndex = lastAdjacentIndex(data, item[field], field, index);
const rowSpan = lastIndex - index + 1;
if (rowSpan > 1) {
    for (let i = 1; i < rowSpan; i++) {
        rows[index + i].querySelector(`td:nth-of-type(${colIndex + 1 - (colSpanItem[index + i] || 0)})`).remove();
      colSpanItem[index + i] = colIndex + 1;
    }
}
```
5. 然后对当前行的当前列设置rowspan属性，有几个相邻数据就为几。
```javascript 1.6
rows[index].querySelector(`td:nth-of-type(${colIndex + 1 - (colSpanItem[index] || 0)})`).setAttribute('rowspan', rowSpan);
```
具体请看下面的例子：

{% codepen pBqmVm [js,result] 330 0 zhouxianjun 0 %}
