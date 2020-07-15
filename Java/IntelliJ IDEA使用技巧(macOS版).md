---
title: IntelliJ IDEA使用技巧(macOS版)
date: 2020/06/28 16:02:00
---

虽然我在日常开发中使用的是IntelliJ IDEA，但是基本上只是使用了一些基本的功能，很多技巧都没有使用。趁着这次给新人分享IDEA的使用，我重新学习整理了IDEA的使用技巧。

<!-- more -->

> IDEA的版本是`IntelliJ IDEA Ultimate 2020.1.1`

# 代码模板

## 文件代码模板

文件代码模板是新建文件时使用的模板，设置路径如下：`Preferences -> Editor -> File and Code Template`，也可以直接使用快捷键`command + ,`来打开`Preferences`。

![File_and_Code_Template](media/File_and_Code_Template.png)
在这里定义了IDEA内置的文件代码模板，当新建一个文件时（比如Java class），就会按照这里的模板生成指定的代码文件。

另外，我们可以在`Includes`里设置文件头：

![File_header](media/File_header.png)
设置之后创建Java类时就会在文件头上自动加上作者、新建日期等信息：

![File_header_example](media/File_header_example.png)

## 实时代码模板

IDEA提供了强大的实时代码模板功能，并且内置了很多模板，比如输入`sout`就会快速生成`System.out.println();`

配置路径如下：`Preferences -> Editor -> Live Templates`。

![Live_Templates](media/Live_Templates.png)

介绍几个常用的代码模板：

- `ifn`：生成参数为null的判断代码
- `inn`：生成参数不为null的判断代码
- `psvm`：生成main函数
- `sout`：生成`System.out.println();`
- `souf`：生成`System.out.printf("");`
- `serr`：生成`System.err.println();`
- `prsf`：生成`private static final `
- `psf`：生成`public static final `
- `psfi`：生成`public static final int `
- `psf`：生成`public static final `
- `St`：生成`String `
- `thr`：生成`throw new `

### 自定义实时算法模板

除了这些内置的代码模板以外，我们还可以自己添加定制的代码模板。

在`Live Templates`页面点击`+`，选择`Template Group`，创建代码模板组。然后点击`+`，选择`Live Template`，创建一个代码模板。

比如，我们可以创建两个代码模板：

- `prI`：生成`private Integer `
- `prS`：生成`private String `

![my_template](media/my_template.png)
除了上面简单的`Live Templates`，还可以定义稍微复杂点的模板，控制光标的位置：

```
/**
 * $VAR1$
 */
private String $VAR2$;

$END$
```

上面的模板定义了一个带注释的`private String`，光标首先会停留在`$VAR1$`处，写完注释后光标跳到`$VAR2$`处，定义好变量名最后跳到`$END$`。

![live_templates_1](media/live_templates_1.gif)


## Postfix Completion

`Postfix Completion`本质上也是一种代码模板，它可以更智能地补全代码。

它具体是做什么的，我们可以通过下面的GIF演示图来说明：

新建对象的实例：

![new_command](media/new_command.gif)

为对象实例赋值：

![var_command](media/var_command.gif)

可以看到`Postfix Completion`和实时代码模板非常像，它在我们代码片段的后面使用，可以减少鼠标的移动，从而加速我们的一些高频操作。

配置路径如下：`Preferences -> Editor -> Postfix Completion`。

![Postfix_Completion](media/Postfix_Completion.png)

IDEA内置了很多`Postfix Completion`，除了上面演示的`new`和`var`，还有下面一些比较常用的模板：

- `for`：为可遍历对象生成`foreach`循环框架
- `fori`：为一个数字生成循环框架
- `nn`和`notnull`：为对象生成不为`null`判断的框架
- `null`：为对象生成为`null`判断的框架
- `return`：在对象前面添加`return`
- `field`：为类创建成员变量
- `sout`：为对象包裹`System.out.println()`方法
- `try`：为表达式生成`try-catch`框架


# 常用快捷键

## 快捷键图

![ReferenceCardForMac](media/ReferenceCardForMac.png)

导出官方快捷键图片：

`shift + command + a`，输入`key reference`

## 代码查看

### 当前文件查找

`command + f`

### 全局查找

`shift + command + f`

### 当前文件替换

`command + r`

### 全局替换

`shift + command + r`

### 查找类(class)

`command + o`

### 查找文件(file)

`shift + command + o`

### 查找符号(symbol)

`option + command + o`

### 查找action

`shift + command + a`

### 查找class+file+symbol+action

点击两次`shift`

### 跳转到下一个 高亮错误 / 警告位置

`F2`

### 跳转到上一个高亮错误 / 警告位置

`Shift + F2`

### 查看类继承关系图

```
option + command + u
shift + option + command + u
```

### 查看函数调用层次

`ctrl + option + h`

### 展示类中变量和方法

`command + F12`

### 展开折叠代码

`command + +/-`

### 展开折叠所有代码

`shift + command + +/-`

## 代码跳转

### 查看最近文件

`command + e`

### 跳转历史光标所在处

`option + command + 方向键`

### 跳转到指定行

`command + l`

### 跳转到指定文件夹

点击两次`shift`，再输入`/文件夹名`

### 从方法使用处跳转到方法定义

`command + 鼠标左键`

`command + B`

### 从方法使用处跳转到方法的实现

`option + command + 鼠标左键`

`option + command + B`

### 接口跳转到实现

`option + command + B`

### 实现跳转到接口

`command + U`

### 书签(bookmarks)

#### 设置书签

`F3`

#### 设置带有标记的（mnemonic）书签

便于记忆的书签指的是，我们在设置书签时可以添加一个数字或者字母来标记这个书签。便于之后书签之间的跳转。

`option + F3`

#### 带有标记的书签（mnemonic bookmark）之间的跳转

`ctrl + 数字/字母`

#### 显示所有的书签

`command + F3`

![bookmarks](media/bookmarks.gif)

### 收藏列表

#### 打开收藏夹

`command + 2`

#### 添加到收藏列表

`option + shift + f`

## 代码编辑

### 删除当前行

`command + back space`

### 复制当前行

`command + d`

### 自动代码提示

`option + enter`

非常智能，不同场景下有不同的提示。

### 变量大小写切换

`shift + command + u`

### 代码格式化

`option + command + l`

### 代码自动收尾

`shift + command + enter`

比如一行代码完成后输入`shift + command + enter`，会在这行代码末尾加上`;`，光标定位到末尾。

代码收尾不仅仅是添加`;`：

![complete_statement](media/complete_statement.gif)

这里IDEA会为`if`表达式自动加上大括号。

### 对选中的代码弹出环绕选项

`option + command + t`

![surround-with](media/surround-with.gif)


### 插入自定义动态代码模板

`command + j`

![command+j](media/command+j.gif)

### 生成代码(构造函数、`Getter/Setter`方法、`equals`方法、`hashCode`方法、`toString`方法)

```
command + n
ctrl + enter
```

![command+n](media/command+n.gif)

### 上下移动选中的代码

`option + shift + 上/下键`

![option+shift+up:down](media/option+shift+up:down.gif)

### 选中所有相同的代码

`ctrl + command + G`

![ctrl+command+g](media/ctrl+command+g.gif)

### 在当前目录下新建文件

`ctrl + option + n`

### 复制当前文件

`F5`

### 移动当前文件

`F6`

### 复制文件名

`command + c`

### 复制完整的文件路径

`shift + command + c`

### 查看剪切板内容

`shift + command + v`

## 重构

### 打开重构对话框

`ctrl + T`

### 重命名变量、文件、文件夹

`shift + F6`

### 改变变量的类型

`shift + command + F6`

### 修改方法的签名

`command + F6`

### 快速提取变量

`option + command + v`

### 快速提取常量

`option + command + c`

### 快速提取成员变量

`option + command + f`

### 快速提取变量到方法参数

`option + command + p`

### 快速提取方法

`option + command + m`

## git的集成

### 查看最近修改

`ctrl + shift + option + 上/下键`

### 撤销修改

`option + command + z`

## 代码运行

### 运行光标所在上下文的方法

`ctrl + shift + r`

### 调试光标所在上下文的方法

`ctrl + shift + d`

### 打开历史运行的方法列表

`ctrl + option + r`

### 打开历史调试的方法列表

`ctrl + option + d`

## 断点调试

### 添加/关闭断点

`command + F8`

### 查看所有断点

`shift + command + F8`

### 设置条件断点

`shift + command + F8`

### 表达式求职

`option + F8`

### 单步运行

`F8`

### 单步跟踪

`F7`

### 运行到光标位置

`option + F9`

### resume

```
option + command + r
F9
```

### 设置变量的值

`F2`

# IntelliJ IDEA提供的注释

IntelliJ IDEA提供了一些注释。IDEA会扫描并分析这些使用注释的代码，如果有问题就会提示用户。注解如下：

- `@Nls`：表示带注释的代码是需要本地化的字符串。
- `@NonNls`：表示带注释的代码是用户不可见的字符串，它不需要本地化，并且不包含需要本地化的字符串。当你使用`@NonNls`注释代码时，本地化工具将跳过次元素和其他的字符串。
- `@PropertyKey`：表示方法参数接收必须是特定`resource bundle`中的有效属性。当参数不是`resource bundle`中的有效属性，IDEA会将其显示为错误。
- `@TestOnly`：表示该方法或者构造器只能在测试代码中被调用
- `@Contract`：用户可以在该注释中指定方法必须遵守的规则。一旦规则被打破，IDEA就会报错
- `@Nullable`：表示变量、参数、返回值可以为`null`，IDEA会提示你处理该参数为`null`的情况
- `@NotNull`：表示变量、参数、返回值不可以为`null`，IDEA会在调用处提示你处理`null`的情况




> https://www.jetbrains.com/help/idea/discover-intellij-idea.html
> https://juejin.im/post/5d47f4e851882534d152a206
> https://atips.cn/idea/
> http://blog.didispace.com/intellij-idea-some-features-sam-1/