## 常用的mixin 

### 背景图片

```
@mixin bg-image($url){
  background-image: url($url + "@2x.png");
  @media (-webkit-min-device-pixel-ratio: 3),(min-device-pixel-ratio: 3){
    background-image: url($url + "@3x.png");
  }
}
使用方法:
.icon{
    @include bg-image(logo);
}
复制代码
```

### 文本不换行

```
@mixin no-wrap(){
  text-overflow: ellipsis;
  overflow: hidden;
  white-space: nowrap;
}
使用方法:
.box {
    @include no-wrap()
}
复制代码
```

### 扩展点击区域(多用于移动端)

```
@mixin extend-click(){
  position: relative;
  &:before{
    content: '';
    position: absolute;
    top: -10px;
    left: -10px;
    right: -10px;
    bottom: -10px;
  }
}
使用方法:
.box {
    @include extend-click()
}
复制代码
```

### 多行文本溢出...

```
@mixin multiEllipsis($line:2){
    overflow : hidden;
    word-break: break-all;
    text-overflow: ellipsis;
    display: -webkit-box;
    -webkit-line-clamp: $line;
    -webkit-box-orient: vertical;
}
使用方式:
.text{
    @include multiEllipsis(3)  // 表示只显示三行,多出来的显示省略号
}
复制代码
```

### 透明度

```
@mixin opacity($opacity) {
    opacity: $opacity;
    $opacity-ie: $opacity * 100;
    filter: alpha(opacity=$opacity-ie); //IE8
}
使用方式:
.box {
    @include opacity(0.8);
}
复制代码
```

### 清除浮动

```
@mixin clearfix() {
    &:before,
    &:after {
        content: "";
        display: table;
    }
    &:after {
        clear: both;
    }
}
使用方式:
.box {
    @include clearfix()
}
复制代码
```

### 美化占位符 placeholder 样式

```
@mixin beauty-placeholder($fz, $color: #999, $align: left) {
  &:-moz-placeholder {
    font-size: $fz;
    color: $color;
    text-align: $align;
  }
  &:-ms-input-placeholder {
    font-size: $fz;
    color: $color;
    text-align: $align;
  }
  &::-webkit-input-placeholder {
    font-size: $fz;
    color: $color;
    text-align: $align;
  }
}
使用方式: 
input{
    @include beauty-placeholder(25px, #eee, left)
}
复制代码
```

### 美化文本的选中

```
@mixin beauty-select($color, $bgcolor) {
  &::selection {
    color: $color;
    background-color: $bgcolor;
  }
}
使用方式:
.text {
    @include beauty-select(#fff, #000)
}
复制代码
```

### 毛玻璃效果

```
@mixin blur($blur: 10px) {
  -webkit-filter: blur($blur);
  -moz-filter: blur($blur);
  -o-filter: blur($blur);
  -ms-filter: blur($blur);
  filter: progid:DXImageTransform.Microsoft.Blur(PixelRadius='${blur}');
  filter: blur($blur);
  *zoom: 1;
}
使用方式: 
.box {
    @include blur(10px)
}
复制代码
```

### 滤镜: 将彩色照片显示为黑白照片、保留图片层次

```
@mixin grayscale() {
  -webkit-filter: grayscale(100%);
  -moz-filter: grayscale(100%);
  -ms-filter: grayscale(100%);
  -o-filter: grayscale(100%);
  filter: grayscale(100%);
}
使用方式:
.img {
    @include grayscale()
}
复制代码
```

### 文本居中

```
@mixin center($height:100%){
    height: $height;
    line-height: $height;
    text-align: center
}
使用方式:
.text{
    color: #fff;
    @include center(30px)
}
复制代码
```

### 修改背景色等

```
@mixin background($border-radius:5px, $bg-color:#eee, $color:#fff, $font-weight:400){
    border-radius: $border-radius;
    background-color: $bg-color;
    color: $color;
    font-weight: $font-weight;
}
使用方式:
.container{
    @include background(10px, #eee, #fff, 700)
}
复制代码
```

### flex

```
@mixin flex ($direction: row, $justify-content: flex-start, $align-items: flex-start,$flex-wrap: nowrap) {
    display: flex;
    flex-direction: $direction;
    justify-content: $justify-content;
    align-items: $align-items;
    flex-wrap: $flex-wrap;
}
使用方式:
.box {
    @include flex(row,flex-start,flex-start,wrap)
}
复制代码
```

### 鼠标hover显示下划线

```
@mixin hoverLine($height:2px,$color:$color-text-primary){    position: relative;    &:hover::after{            content: '';            position: absolute;            height:$height;            width: 100%;            background-color: $color;            bottom: 0;            left: 0; } 复制代码 复制代码
} 使用方式: span{ @include hoverLine(2px,$color-white); } 
```



## 一些常用全局样式 

### 点击显示小手

```
.pointer {
    cursor: pointer;
}
复制代码
```

### 使用在字符图标上hover时的样式

```
.ibass--default{
    color: red;
    transition: color .5s;
    cursor: pointer;
    &:hover {
        color: blue;
    }
}
复制代码
```

### 文本不允许选中

```
.noselect{
  user-select: none;
}
复制代码
```

### 点击无效

```
.click--invalid {
    pointer-events: none;
}
```


