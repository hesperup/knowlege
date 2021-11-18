## width：auto

auto默认值包含四种表现形式：

1. 充分利用可用空间：fill-available
2. 收缩与包裹: fit-content
3. 收缩到最小：min-content
4. 超出容器限制

## height：100%

如果父元素高度为auto，只要子元素在文档流中，百分比值会被忽略。
 使height：100%生效：

1. 设置显示的高度值



```css
html,body {
  height: 100%;
}
```

1. 使用绝对定位



```css
div {
  position: absolute;
  height: 100%;
}
```

### 外部尺寸与流体特性

1. 正常流宽度：margin/border/padding/content内容区域自动分配水平空间
2. 格式化宽度：仅出现在“绝对定位模型”中

### 内部尺寸与流体特性

1. 包裹性：包裹+自适应
2. 首选最小宽度：每个汉字宽度，每个英文字符单元
3. 最大宽度：最大的连续内联盒子的宽度

## 内联元素

可以和文字在同一行显示

### 内联盒模型

1. 内容区域
2. 内联盒子
3. 行框盒子
4. 包含盒子

### 幽灵空白节点

在HTML5文档声明中，内联元素的所有解析和渲染表现就如同每个**行框盒子**的前面有一个“**空白节点**”。这个空白节点永远透明，不占宽度，无法获取，但确实存在，表现如文本节点一样。

## 替换元素

根据是否具有可替换内容，可以把元素分为替换元素和非替换元素。
 典型的替换元素：`<img>`、`<object>`、`<video>`、`<iframe>`、`<textarea>`、`<input>`······
 替换元素特性：

1. 内容可替换
2. 内容的外观不受页面上的css影响
3. 有自己的尺寸
4. 在很多css属性上有自己的一套规则

替换元素和非替换元素的区别：

1. 替换元素和非替换元素之间只隔了一个src属性
2. 替换元素和非替换元素之间只隔了一个CSS `content`属性

## content内容生成技术

1. content辅助元素生成
2. content字符内容生成
3. content图片生成
4. content开启闭合符号生成
5. content计数器

## margin

margin合并的3种场景

1. 相邻兄弟元素margin合并
2. 父级和第一个/最后一个子元素
3. 空块级元素的margin合并

margin合并的计算规则：

1. 正正取大
2. 正负相加
3. 负负最负

`margin：auto`的填充规则

1. 如果一侧定值，一侧auto，则auto为剩余空间大小
    可用于实现左对齐或右对齐
2. 如果两侧均是auto，则平分剩余空间

注：触发`margin：auto`计算有一个前提，width和height不能是auto

margin无效情形：

1. `display`计算值`inline`的非替换元素的垂直margin无效
2. 表格中`<tr>`和`<td>`元素设置`display`计算值是`table-cell`和`table-row`的元素的margin都是无效的
3. margin合并的时候
4. 绝对定位元素非定位方向的margin值“无效”
5. 定高容器的子元素的`margin-bottom`或者定宽容器的子元素的`margin-right`的定位“失效”
6. 鞭长莫及导致的margin失效
7. 内联特性导致的margin失效

## vertical-align

`vertical-align`属性值分为4类：

1. 线类： `baseline`,`top`,`middle`,`bottom`
2. 文本类： `text-top`,`text-bottom`
3. 上下标类： `sub`,`super`
4. 数值百分比类： 如`20px`,`2em`,`20%`

`vertical-align`作用的前提： 只能应用于内联元素以及`display`值为`table-cell`的元素

## 层叠上下文和层叠水平

层叠顺序：

1. 正`z-index`
2. `z-index：0`或者`z-index:auto`或者不依赖`z-index`的层叠上下文
3. `inline`水平盒子
4. `float`浮动盒子
5. `block`块状盒子
6. 负`z-index`
7. 层叠上下文`background`/`boder`

层叠领域的黄金准则

1. 谁大谁上
2. 后来居上

**z-index不犯二准则： 对于非浮层元素，避免设定z-index值，z-index值没有任何道理需要超过2。**

## 流向的改变

1. `direction`
2. `unicode-bidi`
3. `writing-mode`



### CSS选择器的分类与优先级

**css选择器分为四类**：选择器、选择符（后代关系的空格、>、+、~、||）、伪类、伪元素（::before、::after、::first-letter等）。

**css的优先级:**
css的优先级有很多划分方法，所有的方法其实都大同小异。这里将CSS优先级划分为6个等级：

- 0级：通配选择器（*）、选择符（+、>、~、空格、||）、逻辑组合伪类(:not()、:is()等)
- 1级：标签选择器
- 2级：类选择器、属性选择器、伪类
- 3级：ID选择器
- 4级：style内联属性
- 5级：!important

CSS优先级的计算规则：0级选择器优先级数值+0，1级选择器+1，2级选择器+10，三级选择器+100

```
#foo a:not([ref=nofollow]){}     优先级数值：100+1+0+10 = 111
```

**tips1:增加CSS优先级的小技巧**
重复自身选择器，如以下选择器即提高了优先级，又不会增加耦合：
.foo.foo
或者如下写法：
.foo[class] 、 #foo[id]
**tips2: 实现类似:first-child的效果**
有如下场景，我们希望列表除了第一个第一个以外其他的都有margin-top,一般我们会这样写：

```
1：.cs:not(:first-child){margin-top:1em;}
2：.cs{margin-top:1em};
.cs:first-child{margin-top:0}
```

下面我们利用兄弟选择器：

```
.cs+.cs{margin-top: 1em}
```

**tips3：实现前面兄弟选择符的效果**
如下场景：输入框聚焦时，前面文字高亮显示
1：flex布局；
2：float浮动；
3：absolute绝对定位；
4：direction属性实现，代码如下：

```
<div class="cs-direction">
  <input class="cs-input"><label class="cs-label">用户名：</label>
</div>

.cs-direction{
  direction: rtl
}
.cs-direction .cs-input,.cs-direction .cs-label{
  direction: ltr;
}
.cs-label{
  display:inline-block;
}
:focus + .cs-label{
 color: darkblue;
 text-shadow: 0 0 1px;
}
```

**注意：使用direction属性时，针对的必须为内联元素，因此，对于块级元素需进行转化为内联**

### 属性选择器

- [attr]
- [attr = ''val']
- [attr ~= 'val']
- [attr ^= 'val']
- [attr $= 'val']
- [attr *= 'val']

### 用户行为伪类

**1：手型经过伪类:hover（支持所有HTML元素）**

一个简单的应用场景：当鼠标经过a标签时，'显示' 文字显示,因为:hover是即时的，所以通过transition对visibility进行过渡（trantion对display不起作用）起到时延作用。

```
  <a href class="icon-delete" data-title="显示">删除</a>

.icon-delete{
    display: inline-block;
    height: 20px;
    position: relative;
  }
  .icon-delete::before{
    content: attr(data-title);
    position: absolute;
    bottom: 30px;
    left: 0;
  }
  .icon-delete::before,
  .icon-delete::after {
    transition: visibility 0s 0.2s;
    visibility: hidden;
  }
  .icon-delete:hover::before,
  .icon-delete:hover::after {
    visibility: visible;
  }
```

纯hover显示浮层的体验问题：纯hover显示浮层确实很方便，但是如果鼠标坏了，或者是触屏设备通过Tab来切换，如果是一个下拉列表，那么久完全瘫痪了，我们可以通过增加:focus来优化css，上述代码可以优化为：

```
<a href class="icon-delete" data-title="删除">删除</a>

 .icon-delete{
    display: inline-block;
    height: 20px;
    position: relative;
  }
  .icon-delete::before{
    content: attr(data-title);
    position: absolute;
    bottom: 30px;
    left: 0;
  }
  .icon-delete::before,
  .icon-delete::after {
    transition: visibility 0s 0.2s;
    visibility: hidden;
  }
  .icon-delete:hover::before,
  .icon-delete:hover::after {
    visibility: visible;
  }
.icon-delete:focus::before,
.icon-delete:focus::after {
    visibility: visible;
    transition: none;
}
```

但是，上述并没有解决所有的问题，如果浮层的内部有链接或者按钮的话，使用:focus浮层内的连接或者按钮是无法被点击的，因为Tab在切换焦点元素的时候，浮层会失焦而迅速隐藏。这时需要另外一个伪类:focus-within来解决,详细我们在下面讨论。


**2:激活伪类:active**
:active可以用于设置元素激活状态的样式，可通过点击鼠标主键，也可以通过手指或者触控笔点击触摸屏触发激活状态。具体表现为 点击按下触发，点击抬起取消。**:active支持所有HTML元素**

:active的不足之处：

- IE浏览器下，:active无法冒泡。即当一个元素与父级都设置:active样式时，点击子元素，父元素的:active不会被触发
- IE浏览器，html、body用:active设置背景后，背景色无法还原；
- 移动端safari浏览器下，:active伪类默认无效

**按钮的通用:active样式技巧(主要应用移动端)**
在桌面端可以通过:hover来反馈状态变化，移动端只能通过:active来反馈，移动端有很多需求点击反馈链接跟按钮，有如下通用技巧：

- 使用box-shadow(兼容IE9，该方法对非对称闭合标签无效)

```
 [href]:active,button:active{
    box-shadow:  0 0 0 999px rgba(0,0,0,.05);
  }
```

- 使用linear-gradient线性渐变(兼容IE10以上)

```
[href]:active,button:active,[type=reset]:active,[type=button]:active,[type=submit]:active{
  background-image: linear-gradient(rgba(0,0,0,.05),rgba(0,0,0,.05));
}
```

**3:焦点伪类:focus(支持IE8+)**
:focus只能匹配特定的元素，包括：

- 非disabled状态的表单元素；
- 包含href的<a>元素
- <area>、<summary>

那么如何让普通的元素也可以用:focus伪类呢？
**设置HTML tabindex属性，如下写法：**

```
 <div tabindex='-1'>内容</div>
 <div tabindex='0'>内容</div>
 <div tabindex='1'>内容</div>
```

如果期望<div>元素被Tab索引，且被点击的时候触发:focus伪类样式，设置tabindex='0';如果期望不希望被索引，只在点击的时候触发:focus，则设置为-1。
如下，实现点击一个图标显示大图的交互：

```
<span class="cs-small" tabindex='0'></span><span class="cs-big"></span>

.cs-small{
    display: inline-block;
    width: 10px;
    height: 10px;
    background: green;
  };
  .cs-big{
    display: inline-block;
    width: 50px;
    height: 50px;
    background: green;
    margin-left: 10px;
    display: none;
  };
  :focus + .cs-big{
    display: inline-block;
  }
```

实际上，使用tabindex也并不是那么完美，在ios safari浏览器下，元素处于focus状态后，除非有其他可聚焦的元素转移焦点，否则会一直保持聚焦状态。解决方法只需要再套个父盒子，设置tabindex=‘-1’，同时该复盒子需要取消outline样式（outline：0 none;）

**4:整体焦点伪类:focus-within(移动端以及非IE浏览器)**

：focus-within功能通:focus类似，但是:focus是对当前元素而言，:focus-within在当前元素或者当前元素的子元素处于聚焦状态都会匹配。
例：:focus-within 实现无障碍访问的下拉列表，解决上面:focus在浮层里也有链接无法Tab切换的问题：

```
<div class="cs-bar">
        <div class="cs-details">
          <a href='javascript:void(0)' class="cs-summary">我的消息</a >
          <div class="cs-datalist">
            <a href class="cs-datalist-a">我的回答<sup class="cs-datalist-sup">12</sup>
            </a >
            <a href class="cs-datalist-a">我的私信</a >
          </div>
        </div>
      </div>

cs-bar {
    background-color: #e3e4e5;
    color: #888;
    padding-left: 40px;
    margin-bottom: 200px;
  }
  .cs-details {
    display: inline-block;
    text-align: left;
  }
  .cs-summary {
    display: inline-block;
    padding: 5px 28px;
    text-indent: -15px;
    user-select: none;
    position: relative;
    z-index: 1;
  }

  .cs-datalist {
    display: none;
    position: absolute;
    min-width: 100px;
    border: 1px solid #ddd;
    background-color: #fff;
    margin-top: -1px;
  }

  .cs-datalist-a {
    display: block;
    padding: 5px 10px;
    transition: background-color 0.2s, color 0.2s;
    color: inherit;
  }
  .cs-datalist-a:hover {
    background-color: #f5f5f5;
  }
  .cs-datalist-a:active {
    background-color: #f0f0f0;
    color: #555;
  }
  .cs-datalist-sup {
    position: absolute;
    color: #cd0000;
    font-size: 12px;
    margin-top: -0.25em;
    margin-left: 2px;
  }
   .cs-details:focus-within .cs-summary,
   .cs-summary:hover {
     background-color: #fff;
   }
   .cs-details:focus-within .cs-datalist {
     display: block;
   }
```

### URL定位伪类

**:link伪类（基本被a标签取代，不做讨论）**

**:visited伪类（怪癖最多的CSS伪类）**

- 1：支持:visited的CSS属性有限
  目前仅支持以下CSS： color、background-color、border-color、border-bottom-color、border-top-color、border-left-color、border-right-color、column-rule-color、outline-color。类似::before跟::after也不支持。
  例：实现访问过的连接文字后面加上一个visited的字样

```
<a href=''>文字<small></small></a>

  small{
    color: white;
  }
  small:after{
    content: 'visited';
  }
  a:visited small{
    color: lightskyblue;
  }
```

- 2:没有半透明，该伪类控制颜色时，加透明度无效
- 3：只能重置，不能凭空设置
  如下：

```
a{
 color: blue;
}
a:visited{
 color: red;
 background-color: gray;
}
```

以上代码设置只会使color生效。

- 4:无法获取:visited设置和呈现的色值
  该伪类设置的色值，用js的getComputedStyle()无法获取。

**:any-link伪类（IE9+）**
两大特性：
1.匹配所有设置了href的连接元素，包括<a>、<link>、<area>
2.匹配所有匹配:link伪类或者:visited伪类的元素（:link只匹配没有访问过的连接）
**目标伪类:target**
URL锚点可以跟页面中元素的id匹配进行锚定，例：

```
<div>
       <p id='cs-first'>第一行,id:cs-first</p>
       <p id='cs-second'>第二行,id:cs-second</p>
       <p id='cs-last'>第三行,id:cs-last</p>
     </div>

p:target{
    font-size: bold;
    color: skyblue;
  }
```

通过链接的hash值就可以匹配到对应的元素

### 树结构伪类

*:root伪类*
在html中:root等同于html元素；
*:empty伪类*

- :empty用来匹配空标签元素
- :empty伪类可以匹配前后闭合的替换元素（button、textarea等）；
- :empty伪类匹配非闭合的元素（input、img等）

**若元素有注释或者空格、换行等都是不能匹配到的**
**::before、::after都可以插入内容，但是不会影响：empty的匹配**

**:empty的实际应用**

- 隐藏空元素。
  比如像一个动态列表有内容的时候可能会加margin、padding等布局，但是没有内容的时候就会空出一大块，这时候就可以用:empty伪类在匹配到空的时候隐藏掉。
- 字段缺失只能提示
  如在一个动态详情列表中，如果对应的字段没有内容，则在后面显示 '暂无'的字样，代码如下：

```
  <dl>
   <dt>姓名：</dt><dd>张三</dd><br/>
   <dt>年龄：</dt><dd></dd><br/>
   <dt>身高：</dt><dd>178cm</dd>
  <dl>

  dt,dd{
    display: inline-block;
  }
  dd:empty:before{
    content: '暂无';
  }
```

*子索引伪类*

###### :first-child、:last-child

###### :only-child

###### :nth-child()伪类和:nth-last-child()伪类

适用于动态匹配场景，可以匹配制定索引序号的元素。参数可以是关键字（odd/even）,也可以是函数符号。
函数符号形式如下：

1. An+B: A、B必须为整数，n前面可以有负数，n为从1开始的自然序列（0，1，2…..），第一个子元素匹配序号为1.
2. 一些示例：

- tr:nth-child(odd),匹配1、3、5…..行；
- tr:nth-child(even),匹配2、4、6…..；
- li:nth-child(1),匹配第一个；
- li:nth-child(n+4):nth-child(-n+10): 匹配4-10个li

###### :first-of-type伪类和:last-of-type伪类

表示当前标签类型元素第一个和最后一个

###### :only-of-type

###### :nth-of-type()伪类和:nth-last-of-type伪类

:nth-of-type与:nth-child的不同之处：:nth-of-type匹配的是所有相同标签的兄弟元素，而另一个无视标签类型。

```
<div>
  <h3>标题一</h3>
  <p>段落一</p>
  <p>段落二</p>
  <h3>标题二</h3>
  <p>段落三</p>
  <p>段落四</p>
</div>

  p:nth-child(2n+1){
    color: red;
  }
  p:nth-child(4n){
    color: darkblue;
  }
  p:nth-of-type(2n+1){
    color: skyblue;
  }
```

图中，第2、5行为浅蓝色，第三行为红色。

### 逻辑组合伪类

**否定伪类:not()**
:not()主要的应用场景为重置CSS
例：input:not(:disabled):not(:read-only)()
表示匹配所有不处于禁用，也不是只读状态的input

例： 列表一行5个，每个都向右有空隙，每一行的最后一个没有空隙；

```
 li:not(:nth-of-type(5n+1)){
   margin-right: 10px;
 }
```

**任意匹配伪类:is()**
例：

```
  .cs-a img,.cs-b img,.cs-c img,.cs-d img{
    border: 1px solid skyblue;
  }
  可以简化为如下：

  :is(.cs-a,.cs-b,.cs-c,.cs-d) img{
    border: 1px solid skyblue;
  }
```

**:where()、:has()**

### 输入伪类

*输入控件状态:*

###### 可用状态与禁用状态伪类 :enabled和:disabled (IE9+)

- 对于表单元素来说，:enable跟:disabled是完全对立的。但是有一个例外<a>元素，在chrome浏览器下，带href的a元素是可以匹配:enabled伪类的，但是却无法匹配:disabled伪类，其他浏览器也有各种各样的问题，不展开讨论。
- 设置tabindex属性的元素不能匹配:enabled伪类；
- 元素设置visibility:hidden跟display:none的能够匹配:enabled伪类跟:disabled伪类。

###### 读写特性伪类:read-only和:read-write

readonly与disabled的区别：
设置readonly的输入框不能输入内容，但是可以被表单提交；
设置disabled的输入框不能输入内容美也不能被表单提交。

###### 占位符显示伪类:placeholder-shown（除IE）

表示当输入框的placeholder内容显示的时候，匹配输入框
由于其只在内容为空值状态的时候才显示，所以我们可以借助placeholder-shown伪类来判断一个输入框是否有值。
例： 输入框没有值得时候显示空值提示

```
input:placeholder-shown + small:before{
  content: '尚未输入验证码';
  color: red;
}
```

###### 默认选项伪类default

*输入值状态:*

###### 选中选项伪类:checked

1. :checked与[checked]的比较：

- :checked只能匹配标准的表单控件元素，[checked]可以与任意元素匹配；
- [checked]是非实时的，因此不建议用此属性选择器；
- 伪类可以从祖先元素继承；

###### 不确定值伪类:indeterminate

*输入值验证:*

###### 有效验证伪类:valid和:invalid

###### 范围验证伪类:in-range和:out-of-range

###### 可选性伪类:required和:optional

用户交互伪类：user-invalid和空值伪类：blank 





第 12章 其他伪类选择器 181
12．1 与作用域相关的伪类 181
12．1．1 参考元素伪类：scope 181
12．1．2 Shadow树根元素伪类：host 183
12．1．3 Shadow树根元素匹配伪类：host() 184
12．1．4 Shadow树根元素上下文匹配伪类：host-context() 185
12．2 与全屏相关的伪类：fullscreen 187
12．3 了解语言相关伪类 188
12．3．1 方向伪类：dir() 189
12．3．2 语言伪类：lang() 190





### 输入框聚焦，文字高亮

#### flex方案(推荐)




用户：



这一方法主要是通过flex-direction:row-reverse调换元素的水平呈现顺序来实现DOM位置和视觉位置不一样。书上说，这个方法的唯一问题是兼容性了，用户群是外部用户的桌面慎用，移动端无碍。不是很懂前面那句话的意思，真人实测，chrome,firefox,opera,IE11,Edge都可以使用。毕竟都9021年了。

```
<div class="cs-flex">
    <input class="cs-input" /><label class="cs-label">用户：</label>
</div>
<style type="text/css">
.cs-flex {
    display: inline-flex;
    flex-direction: row-reverse;
}
.cs-input {
    width: 200px;
}
.cs-label {
    width: 64px;
}
:focus ~ .cs-label {
    color: darkblue;
    text-shadow: 0 0 10px;
}
</style>
```



#### float浮动实现



用户：



这个方案主要就是把输入框浮动到右侧了，兼容性极佳。不兼容IE8的话，可以使用calc来计算输入框的宽度。书中说这个不能实现多个元素的前面选择符效果。也对，每多一个，你就要重新计算宽度。

```
<div class="cs-float"><input class="cs-input" /><label class="cs-label">用户：</label></div>
<style>
.cs-float {
    width: 268px;
}
.cs-float .cs-input {
    float: right;
    width: 200px;
}
.cs-float .cs-label {
    display: block;
    overflow: hidden;
    line-height: normal;
}
:focus ~ .cs-label {
    color: darkblue;
    text-shadow: 0 0 10px;
}
</style>
```



#### absolute绝对定位



用户：



这个方案兼容性不错，但是你会发现一个问题，对DOM结构要求比较高，页面元素一多起来，这个的控制成本就比较高了。

```
<div class="cs-absolute"><input class="cs-input" /><label class="cs-label">用户：</label></div>
<style type="text/css">
.cs-absolute {
    width: 268px;
    position: relative;
}
.cs-absolute .cs-input {
    width: 200px;
    margin-left: 64px;
}
.cs-absolute .cs-label {
    position: absolute;
    left: 0;
}
.cs-absolute:focus ~ .cs-label {
    color: darkblue;
    text-shadow: 0 0 10px;
}
</style>
```



#### direction属性实现



用户：



借助direction属性改变文档流的顺序可以轻松实现DOM位置和视觉位置的调换。唯一不足的就是它必须针对内联元素。

```
<div class="cs-direction"><input class="cs-input" /><label class="cs-label">用户：</label></div>
<style type="text/css">
.cs-direction {
    direction: rtl;
    display: inline-block;
}
.cs-direction .cs-input,
.cs-direction .cs-label {
    direction: ltr;
}
.cs-direction .cs-label{
    display: inline-block;
}
.cs-direction:focus ~ .cs-label {
    color: darkblue;
    text-shadow: 0 0 10px;
}
</style>
```



#### :focus-within 实现(推荐)



用户：



:focus-within整体焦点伪类。只要不兼容IE就行.它是正常的DOM流，简直不能太完美

```
<div class="cs-normal"><label class="cs-label">用户：</label><input class="cs-input" /></div>
<style>
.cs-normal:focus-within .cs-label{
    color: darkblue;
    text-shadow: 0 0 1px;
}
</style>
```



### css属性选择器搜索过滤

借助属性选择器来辅助我们实现搜索过滤效果，如通讯录、城市列表，这样做性能高，代码少



- 重庆市
- 哈尔滨市
- 长春市
- 兰州市
- 北京市
- 杭州市
- 长沙市
- 沈阳市
- 成都市
- 合肥市
- 天津市
- 西安市
- 武汉市
- 济南市
- 广州市
- 南京市
- 上海市
- 昆明市
- 郑州市
- 贵阳市
- 西宁市
- 海口市
- 南昌市
- 香港 特区
- 澳门 特区

这里只有最简单的功能展示，没有其它的过渡啊，动画啊之类的，所以用户体验不会很好。这一点，肯定就需要各位自己去写啦。

```
<input id="css-search" type="search" placeholder="输入城市名称或拼音" />
<ul class="filter"><li data-search="重庆市chongqing">重庆市</li><li data-search="哈尔滨市haerbing">哈尔滨市</li><li data-search="长春市changchun">长春市</li><li data-search="兰州市lanzhou">兰州市</li><li data-search="北京市beijing">北京市</li><li data-search="杭州市hangzhou">杭州市</li><li data-search="长沙市changsha">长沙市</li><li data-search="沈阳市shenyang">沈阳市</li><li data-search="成都市chengdu">成都市</li><li data-search="合肥市hefei">合肥市</li><li data-search="天津市tianjin">天津市</li><li data-search="西安市xian">西安市</li><li data-search="武汉市wuhan">武汉市</li><li data-search="济南市jinan">济南市</li><li data-search="广州市guangzhou">广州市</li><li data-search="南京市nanjing">南京市</li><li data-search="上海市shanghai">上海市</li><li data-search="昆明市kunming">昆明市</li><li data-search="郑州市zhangzhou">郑州市</li><li data-search="贵阳市guiyang">贵阳市</li><li data-search="西宁市xining">西宁市</li><li data-search="海口市haikou">海口市</li><li data-search="南昌市nanchang">南昌市</li><li data-search="香港 特区xianggang">香港 特区</li><li data-search="澳门 特区aomen">澳门 特区</li></ul>

<script type="text/javascript">
    let eleStyle = document.createElement('style');
    document.head.appendChild(eleStyle);
    // 文本框输入
    let input = document.querySelector('#css-search');
    input.addEventListener('input', function(){
        let value = this.value.trim();
        eleStyle.innerHTML = value ? '[data-search]:not([data-search*="'+ value + '"]){ display: none;}' : '';
    });
</script>
```



### :hover显示优化

通过增加延时来优化体验。这里模拟一个tooltip提示。



删除



完整代码如下，可以看看那个画三角形的。嘿嘿嘿！
![hover优化代码](https://zhanghang12135.github.io/2019/11/28/%E3%80%8Acss%E9%80%89%E6%8B%A9%E5%99%A8%E4%B8%96%E7%95%8C%E3%80%8B%E6%80%BB%E7%BB%93/hover-del.png)
当然这里，还可以添加:focus来优化体验，这里我就不写了。将最后的:hover换成:focus即可

### :target交互布局技术

:target是IE9及以上版本的浏览器都支持的一个伪类，与URL地址中的锚点定位强关联的伪类。
事先申明这种技术的应用于，js被禁用或者页面还没加载完全的清空下使用，这种方案就是一种兜底的方案。

#### 展开与收起效果

**性感真人，在线忧伤**



无论我们最后生疏到什么样子，曾经对你的好都是真的。希望你不后悔认识我，也是真的快乐过。就算终有一散，但别辜负相遇。如果能回到从前，我会选择不认识你。不是我后悔，是我不能面对现在的结局。

[阅读更多](https://zhanghang12135.github.io/2019/11/28/《css选择器世界》总结/#articleMore)



上述css的实现原理是把锚链元素放在最前面，然后通过兄弟选择符`~`来控制对应元素的显影变化
![:target实现收缩展开](https://zhanghang12135.github.io/2019/11/28/%E3%80%8Acss%E9%80%89%E6%8B%A9%E5%99%A8%E4%B8%96%E7%95%8C%E3%80%8B%E6%80%BB%E7%BB%93/target-1.png)

#### 选项卡效果

这里的hexo框架不知道做了什么，我直接写html会和预期的不一样,我就直接放张鑫旭大佬的demo效果图了
![选项卡效果](https://zhanghang12135.github.io/2019/11/28/%E3%80%8Acss%E9%80%89%E6%8B%A9%E5%99%A8%E4%B8%96%E7%95%8C%E3%80%8B%E6%80%BB%E7%BB%93/target-panel.gif)
利用display:none元素作为锚链元素，利用兄弟选择器控制状态变化,就没有页面跳动的糟糕体验了。不过呢，这种技术肯定不是最完美的。没有任何事物是十全十美的。所以啊，推荐再此方案上在加上js的控制。这样就完美的多了.
推荐思路：

> 1. 默认按照：target伪类交互技术实现，与一个类名标志量关联。
> 2. js也正常实现交互。js加载完成之后移除类名标志量，交互交给js.

```
<div class="cs-tab-x">
    <i id="tabPanel2" class="cs-tab-anchor-2" hidden></i>
    <i id="tabPanel3" class="cs-tab-anchor-3" hidden></i>
    <div class="cs-tab"><a href="#tabPanel1" class="cs-tab-li">选项卡1</a><a href="#tabPanel2" class="cs-tab-li">选项卡2</a><a href="#tabPanel3" class="cs-tab-li">选项卡3</a></div>
    <div class="cs-panel"><div class="cs-panel-li">面板内容1</div><div class="cs-panel-li">面板内容2</div><div class="cs-panel-li">面板内容3</div></div>
</div>
<style type="text/css">
.cs-tab-x{
    width: 300px;
    margin: auto;
    text-align: left;
}
.cs-tab{
    display: inline-block;
}
.cs-tab-li{
    display: inline-block;
    background-color: #f0f0f0;
    color: #333;
    padding: 5px 10px;
}
.cs-tab-anchor-2:not(:target) + :not(:target) ~ .cs-tab .cs-tab-li:first-child,
.cs-tab-anchor-2:target ~ .cs-tab .cs-tab-li:nth-of-type(2),
.cs-tab-anchor-3:target ~ .cs-tab .cs-tab-li:nth-of-type(3) {
    background-color: deepskyblue;
    color: #fff;
}
.cs-panel-li {
    display: none;
    padding: 20px;
    border: 1px solid #ccc;
}
.cs-tab-anchor-2:not(:target) + :not(:target) ~ .cs-panel .cs-panel-li:first-child,
.cs-tab-anchor-2:target ~ .cs-panel .cs-panel-li:nth-of-type(2),
.cs-tab-anchor-3:target ~ .cs-panel .cs-panel-li:nth-of-type(3) {
    display: block;
}
</style>
```

### :placeholder-show的应用

#### Material Design 风格占位符交互

利用:placeholder-shown伪类匹配输入框中没有输入的时候，来实现。老实说，我真的被这种思路惊艳到了。



邮箱

用户名

评论



这里的思路是这样的，首先让placeholder透明，然后让兄弟元素label采用绝对定位去占那个位置，然后输入框或得焦点的时候以及:placeholder伪类没有匹配的时候，将label缩小并移动到上方。添加transitiong过渡就可以了。关键部分，我用红框标出来了。
![示例完整代码](https://zhanghang12135.github.io/2019/11/28/%E3%80%8Acss%E9%80%89%E6%8B%A9%E5%99%A8%E4%B8%96%E7%95%8C%E3%80%8B%E6%80%BB%E7%BB%93/placeholder-1.png)

#### 空值判断



由于placeholder内容只在空值状态的时候才显示，因此可借助:placeholder-shown伪类来判断一个输入框中是否有值。

```
<input class="if-empty" placeholder=" " /><small></small>
<style type="text/css">
.if-empty:placeholder-shown ~ small::before {
    content: '尚未输入内容';
    color: red;
    font-size: 87.5%
}
</style>
```



### :default的实际应用

:default伪类只能用作用在表单元素上，表示处于默认状态的表单元素。
推荐标记 用法



支付宝

微信

云闪付



:default伪类的匹配不受之后checked属性值变化影响，所以增加的`（推荐）`字符会一直在微信后面。如果以后需求变了，直接更改将checked更改就好。这里的，你可以直接F12之后更改，就可以看到，`推荐`会跟随变化。

```
<p><input class="default-radio" type="radio" name="pay" id="pay0"/><label for="pay0">支付宝</label></p>
<p><input class="default-radio" type="radio" name="pay" id="pay1" checked/><label for="pay1">微信</label></p>
<p><input class="default-radio" type="radio" name="pay" id="pay2"/><label for="pay2">云闪付</label></p>
<style>
.default-radio:default + label::after {
    content: '(推荐)';
}
</style>
```



### 单复选框元素显隐技术

由于单选框和复选框的选中行为是由点击事件触发的，因此配合兄弟选择符，可以选择不适用js实现多种点击交互行为，如展开收起，选项卡切换或者多级下拉列表等。不过呢，这里不是我要讲的重点，详细的可以参照书籍。接下来的，就是可以立足于实际开发的一下技巧。

#### 自定义单复选框

浏览器原生的单复选框常常和设计风格不搭，需要自定义,最好的实现方法就是借助原生单复选框再配合其它伪类

单选框1单选框2单选框3 disabled

复选框1复选框2复选框3

![自定义单复选框样式](https://zhanghang12135.github.io/2019/11/28/%E3%80%8Acss%E9%80%89%E6%8B%A9%E5%99%A8%E4%B8%96%E7%95%8C%E3%80%8B%E6%80%BB%E7%BB%93/diy-input.png)

#### 开关效果

常见的开关效果，其本质就是一个复选框，分为`打开`和`关闭`两个状态。原理相同，我这里就直接上效果和代码了



话不多说，直接上代码。这里的一些间距什么的，可能会有不同的显示。需要您自己再去适配了。嘿嘿o(*￣▽￣*)ブ
![开关按钮代码](https://zhanghang12135.github.io/2019/11/28/%E3%80%8Acss%E9%80%89%E6%8B%A9%E5%99%A8%E4%B8%96%E7%95%8C%E3%80%8B%E6%80%BB%E7%BB%93/switch.png)

#### 标签/列表/素材的选择

无论是单选还是多选，选择标签，还是选择图片，都可以借助:checked伪类纯css实现.

科技体育军事娱乐动漫音乐电影生活

您已选择个话题



配合CSS计数器，可直接显示选中标签元素的个数。

```
<input type="checkbox" class="topic" id="topic1"/><label for="topic1" class="cs-topic">科技</label>
<input type="checkbox" class="topic" id="topic2"/><label for="topic2" class="cs-topic">体育</label>
<input type="checkbox" class="topic" id="topic3"/><label for="topic3" class="cs-topic">军事</label>
<input type="checkbox" class="topic" id="topic4"/><label for="topic4" class="cs-topic">娱乐</label>
<input type="checkbox" class="topic" id="topic5"/><label for="topic5" class="cs-topic">动漫</label>
<input type="checkbox" class="topic" id="topic6"/><label for="topic6" class="cs-topic">音乐</label>
<input type="checkbox" class="topic" id="topic7"/><label for="topic7" class="cs-topic">电影</label>
<input type="checkbox" class="topic" id="topic8"/><label for="topic8" class="cs-topic">生活</label>
<p>您已选择<span class="cs-topic-counter"></span>个话题</p>
<style type="text/css">
.topic{
    position: absolute;
    clip: rect(0,0,0,0)
}
.cs-topic {
    display: inline-block;
    width: 64px;
    margin-top: 5px;
    margin-right: 5px;
    padding: 5px;
    border: 1px solid silver;
    text-align: center;
    cursor: pointer;
}
.cs-topic:hover {
    border-color: gray;
}
:checked + .cs-topic {
    border-color: deepskyblue;
    background-color: azure;
}
/* 选择计数器 */
body {
    counter-reset: topicCounter;
}
:checked + .cs-topic {
    counter-increment: topicCounter;
}
.cs-topic-counter::before {
    color: red;
    margin: 0 2px;
    content: counter(topicCounter);
}
</style>
```

![img](https://zhanghang12135.github.io/2019/11/28/%E3%80%8Acss%E9%80%89%E6%8B%A9%E5%99%A8%E4%B8%96%E7%95%8C%E3%80%8B%E6%80%BB%E7%BB%93/bg-1.jpg)

 

![img](https://zhanghang12135.github.io/2019/11/28/%E3%80%8Acss%E9%80%89%E6%8B%A9%E5%99%A8%E4%B8%96%E7%95%8C%E3%80%8B%E6%80%BB%E7%BB%93/bg-2.jpg)

 

![img](https://zhanghang12135.github.io/2019/11/28/%E3%80%8Acss%E9%80%89%E6%8B%A9%E5%99%A8%E4%B8%96%E7%95%8C%E3%80%8B%E6%80%BB%E7%BB%93/bg-3.jpg)

 

![img](https://zhanghang12135.github.io/2019/11/28/%E3%80%8Acss%E9%80%89%E6%8B%A9%E5%99%A8%E4%B8%96%E7%95%8C%E3%80%8B%E6%80%BB%E7%BB%93/bg-4.jpg)

 

![img](https://zhanghang12135.github.io/2019/11/28/%E3%80%8Acss%E9%80%89%E6%8B%A9%E5%99%A8%E4%B8%96%E7%95%8C%E3%80%8B%E6%80%BB%E7%BB%93/bg-5.jpg)

 

![img](https://zhanghang12135.github.io/2019/11/28/%E3%80%8Acss%E9%80%89%E6%8B%A9%E5%99%A8%E4%B8%96%E7%95%8C%E3%80%8B%E6%80%BB%E7%BB%93/bg-6.jpg)



![单选图片](https://zhanghang12135.github.io/2019/11/28/%E3%80%8Acss%E9%80%89%E6%8B%A9%E5%99%A8%E4%B8%96%E7%95%8C%E3%80%8B%E6%80%BB%E7%BB%93/img-radio.png)

### 输入值验证

借助h5的required和pattern等验证相关属性，以及有效验证伪类:valid 验证成功和:invalid验证失败。不过呢，这两个伪类是在页面开始加载时就会触发。新出了一个:user-invalid伪类，不过还处于实验阶段，暂时没有浏览器支持。不过我们可以借助js,来达到同样的效果。就是在用户产生提交表单的行为发生后，为元素添加特定的类名，来触发浏览器内置验证开启。

验证码：

不要在意这里的那个阴影，那是前面的那个影响的。没什么太大的影响。我就不修改前面的东西了。

```
<form id="csForm" novalidate>
    <p>验证码：<input class="cs-input" placeholder=" " required pattern="\w{4,6}" />
    <span class="cs-valid-tips"></span></p>
    <input type="submit" value="提交"/>
</form>

<style type="text/css">
.valid .cs-input:invalid {
    border-color: red;
}
.valid .cs-input:valid + .cs-valid-tips::before {
    content: '√';
    color: green;
}
.valid .cs-input:invalid + .cs-valid-tips::before {
    content: '不符合要求';
    color: red;
}
.valid .cs-input:placeholder-shown + .cs-valid-tips::before {
    content: '尚未输入值';
}
</style>

<script type="text/javascript">
    document.querySelector('#csForm').addEventListener('submit', function(event){
        this.classList.add('valid');
        event.preventDefault();
    })
</script>
```



有几点需要提醒一下：

> 如果你想知道所有的表单都验证通过，可以使用form表单的原生的checkValidity()方法，返回整个表单是否通过验证的布尔值
> 另外，想要表单验证时及时的，可以监听form的input事件或者focus事件。
> 在IE浏览器中，会出现无法重绘的bug,大致就是输入已经合法了，但是后面的提示内容没有实时的变化。解决办法就是手动触发重绘。改变父元素的样式啊，或者设置无关紧要的类名。下面的是张鑫旭大佬写的一个补丁，放在页面的任意位置即可。
>
> ```
> if(typeof document.msHidden != 'undefined' || !history.pushState) {
>     document.addEventListener('input', function (event) {
>         if(event.target && /^input|textarea$/i.test(event.target.tagName)) {
>             event.target.parentElement.className = event.target.parentElement.className;
>         }
>     })
> }
> ```

### 必填，可选

:required伪类用来匹配设置了required属性的表单元素，表示这个表单元素必填或者必写。:optional伪类可以看成是:required伪类的对立面。只要表单元素没有设置required属性都可以匹配:optional伪类.



问卷调查

1. 1-3年3-5年5年以上**你从事前端几年了？**
2. 1-3小时3-5小时5小时以上**你每周多少业余时间用来学习前端技术？**
3. 1-3本3-5本5本以上**你每年买多少本前端相关书籍？**
4. **有什么其它想说的？**





大致说下这里的思路，随后兄弟选择符，后置标题。利用table-caption后置标题视觉上提前显示。然后再利用css计数器去实现编号。可选性自动标记效果，以后想要修改可选性，只需要修改表单元素的`required`属性即可,文案信息自动同步，维护更简单。这里的，你也可以按F12直接修改，看看效果。
![调查问卷代码](https://zhanghang12135.github.io/2019/11/28/%E3%80%8Acss%E9%80%89%E6%8B%A9%E5%99%A8%E4%B8%96%E7%95%8C%E3%80%8B%E6%80%BB%E7%BB%93/required.png)

### 花样表格

这里的话，我就不弄效果了。直接上代码吧，比较简单。

```
/* 偶数行 */
tr:nth-child(even){
    background-color: #eee;
}
/* 前三个元素 */
tr:nth-child(-n + 3) td {
    background: bisque;
}
/* 第4到10个元素 */
tr:nth-child(n + 4):nth-child(-n + 10) td {
    background: lightcyan;
}
```

