# 1 概念
```
BFC：Box是css布局的对象和基本单位，BFC就是页面上的一个隔离的独立容器，容器里面的子元素不会影响到外面的元素，反之也是。

```

# 规则

```
1 内部的BOX会在垂直的方向上一个接一个的放置
2 属于同一个BFC的两个相邻的BOX的margin会重叠，不同的就不会
3 是页面独立容器
4 bfc区域不会和float box重叠
5 计算bfc高度 浮动元素也参与计算
```
# 触发条件

```
根元素
float 不为none
position为absolute 或fixed
overflow不为 visible
display 为inline-block table-cell table-caption 、flex 、inline-flex
```

# 应用场景

```
1 清除内部浮动，触发父元素的BFC，会包含float元素
防止浮动导致父元素高度坍塌，父级设置overflow：hidden
元素float：right
2 阻止元素被浮动元素覆盖，各自是独立的渲染区域；
3 分属不同bfc 防止margin重叠，给父级套个overflow：hidden
4 自适应两栏布局
```

# 创建BFC

```
overflow: hidden / auto /scroll, 不能是visible;( 最常用的创建方式 )
display:flex; display:grid;
position:absolute / fixed;
float:left / right , 不能 none; */

```