# CSS 水平居中与垂直居中的16个方法



![文章结构](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/3/18/169913195af002db~tplv-t2oaga2asx-watermark.awebp)



# 一、水平居中

## 1.1 行内元素

```
.parent {
    text-align: center;
}
复制代码
```

## 1.2 块级元素

### 1.2.1 块级元素一般居中方法

```
.son {
    margin: 0 auto;
}
复制代码
```

### 1.2.2 子元素含 float

```
.parent{
    width:fit-content;
    margin:0 auto;
}

.son {
    float: left;
}
复制代码
```

### 1.2.3 Flex 弹性盒子

1） flex 2012版

```
.parent {
    display: flex;
    justify-content: center;
}
复制代码
```

2）flex 2009版

```
.parent {
    display: box;
    box-orient: horizontal;
    box-pack: center;
}
复制代码
```

### 1.2.4 绝对定位

1）transform

```
.son {
    position: absolute;
    left: 50%;
    transform: translate(-50%, 0);
}
复制代码
```

2）left: 50%

```
.son {
    position: absolute;
    width: 宽度;
    left: 50%;
    margin-left: -0.5*宽度
}
复制代码
```

3）left/right: 0

```
.son {
    position: absolute;
    width: 宽度;
    left: 0;
    right: 0;
    margin: 0 auto;
}
复制代码
```

## 小结

以上是 CSS 水平居中的 8 种方法。

# 二、垂直居中

## 2.1 行内元素

```
.parent {
    height: 高度;
}

.son {
    line-height: 高度;
}
复制代码
```

**注：① 子元素 line-height 值为父元素 height 值。② 单行文本。**

## 2.2 块级元素

### 2.2.1 行内块级元素

```
.parent::after, .son{
    display:inline-block;
    vertical-align:middle;
}
.parent::after{
    content:'';
    height:100%;
}
复制代码
```

适应 IE7。

### 2.2.2 table

```
.parent {
  display: table;
}
.son {
  display: table-cell;
  vertical-align: middle;
}
复制代码
```

**优点**

- 元素高度可以动态改变, 不需再CSS中定义, 如果父元素没有足够空间时, 该元素内容也不会被截断。

**缺点**

- IE6~7, 甚至IE8 beta中无效。

### 2.2.3 Flex 弹性盒子

1）flex 2012版

```
.parent {
    display: flex;
    align-items: center;
}
复制代码
```

**优点**

- 内容块的宽高任意, 优雅的溢出。
- 可用于更复杂高级的布局技术中。

**缺点**

- IE8/IE9不支持。
- 需要浏览器厂商前缀。
- 渲染上可能会有一些问题。

2）flex 2009版

```
.parent {
    display: box;
    box-orien: vertical;
    box-pack: center;
}
复制代码
```

**优点**

- 实现简单, 扩展性强。

**缺点**

- 兼容性差, 不支持IE。

### 2.2.4 绝对定位

1）transform

```
.son {
    position: absolute;
    top: 50%;
    transform: translate( 0, -50%);
}
复制代码
```

**优点**

- 代码少。

**缺点**

- IE8不支持, 属性需要追加浏览器厂商前缀, 可能干扰其他 transform 效果, 某些情形下会出现文本或元素边界渲染模糊的现象。

2）top: 50%

```
.son {
    position: absolute;
    top: 50%;
    height: 高度;
    margin-top: -0.5高度;
}
复制代码
```

**优点**

- 适用于所有浏览器。

**缺点**

- 父元素空间不够时, 子元素可能不可见(当浏览器窗口缩小时,滚动条不出现时).如果子元素设置了overflow:auto, 则高度不够时, 会出现滚动条。

3）top/bottom: 0;

```
.son {
    position: absolute;
    top: 0;
    bottom: 0;
    margin: auto 0;
}
复制代码
```

**优点**

- 简单。

**缺点**

- 没有足够空间时, 子元素会被截断, 但不会有滚动条。

## 小结

以上是 CSS 垂直居中的 8 种方法及其优缺点。

# 三、小结



![小结](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/3/18/1699132195c5d0af~tplv-t2oaga2asx-watermark.awebp)

以上总结了水平居中、垂直居中各8个共16种方法。



其中，

- flex
- 绝对定位

同时适用于水平居中和垂直居中。