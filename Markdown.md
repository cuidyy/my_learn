# markdown学习

### 1、标题语法

`#` 标题(是多少级标题就添加多少个#号,之间有一个空格)

### 2、段落语法  

使用空白行将一行或多行文本进行分隔  
HTML语法：`<p>`text`</p>`

### 3、换行语法

一行的末尾添加两个空格，然后按回车键   
行尾添加反斜杠\\  
HTML语法：`<br>`

### 4、强调语法

##### 粗体：
`**`bold text`**` or `__`bold text`__`  
HTML语法: `<strong>`bold text`</strong>`

##### 斜体： 
`*`Ltalic text`*` or `_`Ltalic text`_`  
HTML语法：`<em>`bold text`</em>`

要同时用粗体和斜体突出显示文本，单词或短语的前后各添加三个星号或下划线。

### 5、引用语法

要创建块引用，在段落前添加一个 `>` 符号  
块引用可以包含多个段落。为段落之间的空白行添加一个 `>` 符号。  
块引用可以嵌套。在要嵌套的段落前添加一个 `>>` 符号。

### 6、列表语法

##### 有序列表
数字`.` text  
HTML语法:`<ol>` `<li>`first term`</li>` `</ol>`

##### 无序列表

在每个列表项前面添加 `-`、 `*` 或 `+`  
HTML语法：`<ul>` `<li>`first term`</li>` `</ul>`

要在保留列表连续性的同时在列表中添加另一种元素，请将该元素缩进四个空格或一个制表符

### 7、代码语法

将单词或短语表示为代码，将其包裹在反引号`` ` ` ``中  
HTML语法:`<code>`text`</code>`

转义反引号：将单词或短语包裹在双反引号中。

### 8、分隔线语法
单独一行上使用三个或多个星号`***`、破折号 `---` 或下划线 `___` ，且不包含其他内容。

### 9、链接语法

超链接  
``[超链接显示名](超链接地址 "超链接title")``
HTML语法：``<a href="超链接地址" title="超链接title">超链接显示名</a>``

网址和Email地址  
``<https://markdown.com.cn>``

### 10、图片语法

``![图片alt](图片链接 "图片title")``  
HTML语法：``<img src="图片链接" alt="图片alt" title="图片title">``

给图片增加链接，请将图像的Markdown 括在方括号中，然后将链接添加在圆括号中。
``[![图片alt](图片链接 "图片title")](链接)``

