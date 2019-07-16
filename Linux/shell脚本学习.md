---
title: shell脚本学习
date: 2019/06/14 09:25:00
---

本文是对shell脚本学习的整理

<!-- more -->

## 变量

### 定义变量

定义变量时，变量名不加美元符号：

```
var="hello world"
```

注意，变量名和等号之间不能有空格。

### 使用变量

使用一个定义过的变量，只要在变量名前面加美元符号即可：

```
var="hello world"
echo $var
echo ${var}
```

### 只读变量

使用`readonly`命令可以将变量定义为只读变量，只读变量的值不能被改变。

```
var="hello"
readonly var
var="world"
```

执行上面的命令会报错：`var: readonly variable`

### 删除变量

使用`unset`命令可以删除变量：

```
unset var
```

## 字符串

字符串可以用单引号也可以用双引号来表示

单引号字符串的限制：

- 单引号里的任何字符都会原样输出，单引号字符串中的变量是无效的
- 单引号字符串中不能出现单独一个的单引号（对单引号使用转义符后也不行）

双引号的优点：

- 双引号里可以有变量
- 双引号里可以出现转义字符

### 获取字符串长度

```
string="abcd"
echo ${#string} #输出4
```

### 提取字符串

以下实例从字符串第2个字符开始截取4个字符：

```
string="hello world"
echo ${string:1:4}  # 输出ello
```

### 查找子字符串

查找字符`l`或`h`的位置（哪个字母先出现就计算哪个）

```
string="hello world"
echo `expr index "$string" lh`  # 输出1
```

## 数组

数组中可以存放多个值。shell只支持一维数组，初始化时不需要定义数组大小。

shell数组用括号来表示，元素用"空格"符号分隔开：

```
array_name=(value1 ... valueN)
```

### 读取数据

读取数组元素的一般格式是：

`${array_name[index]}`

### 获取数组中的所有元素

使用`@`或`*`可以获取数组中的所有元素

```
my_array[0]=A
my_array[1]=B
my_array[2]=C
my_array[3]=D

echo "数组的元素为: ${my_array[*]}"
echo "数组的元素为: ${my_array[@]}"
```

### 获取数组的长度

获取数组长度的方法与获取字符串长度的方法相同

```
my_array[0]=A
my_array[1]=B
my_array[2]=C
my_array[3]=D

echo "数组元素个数为: ${#my_array[*]}"
echo "数组元素个数为: ${#my_array[@]}"
```

## map

### 定义一个空map

`declare -A map=()`

### 定义并初始化map

`declare -A map=(["100"]="1" ["200"]="2")`

### 输出所有key

`echo ${!map[@]}`

### 输出所有value

`echo ${map[@]}`

### 添加值

`map["300"]="3"`

### 输出key对应的值

`echo ${map["100"]}`

### 遍历输出map

```
for key in ${!map[@]}
do
    echo ${map[$key]}
done
```

## 传递参数

在执行shell脚本时，向脚本传递参数，脚本内获取参数的格式为：`$n`。n代表一个数字，1为执行脚本的第一个参数，2为执行脚本的第二个参数，以此类推。其中`$0`为执行的文件名。

另外，还有几个特殊字符用来处理参数：


| 参数处理 | 说明  |
| --- | --- |
| `$#` | 传递到脚本的参数个数 |
| `$*` | 以一个单字符串显示所有向脚本传递的参数 |
| `$$` | 脚本运行的当前进程ID号 |
| `$!` | 后台运行的最后一个进行的ID号 |
| `$@` | 与`$*`相同，但是使用时加引号，并在引号中返回每个参数 |
| `$-` | 显示shell使用的当前选项 |
| `$?` | 显示最后命令的退出状态。0表示没有错误，其他任何值表明有错误 |

**`$*`与`$@`区别**

- 相同点：都是引用所有参数
- 不同点：只有在双引号中体现出来。假设脚本运行时写了三个参数`1 2 3`，则`$*`等价于`"1 2 3"`，而`$@`等价于`"1" "2" "3"`（传递了三个参数）

## 判断

1. 文件

    1. `-a`、`-e`：文件存在
    2. `-f`：是否为普通文件
    3. `-d`：是否为目录文件
    4. `-b`：是否为块设备文件
    5. `-c`：是否为字符设备文件
    6. `-S`：是否是socket文件
    7. `-p`：是否是FIFO文件或者是命名管道文件
    8. `-L`：是否为链接文件

2. 文件的权限

    1. `-r`：是否存在且对当前用户可读
    2. `-w`：是否存在且对当前用户可写
    3. `-x`：是否存在且对当前用户可执行
    4. `-u`：文件存在且该文件设置了`setuid`位
    5. `-g`：文件存在且该文件设置了`setgid`位
    6. `-k`：文件存在且设置了`sticky bit`（粘滞位）
    7. `-O`：文件存在且该文件的userID和执行命令的用户相同
    8. `-G`：文件存在且该文件的groupID和执行命令的用户相同
    9. `-t`：文件的FD（文件描述符）是否打开并且指向某个终端
    10. `-s`：文件大小是否大于0

3. 文件之间的比较

    1. `-nt`：(newer than)file1是否比file2新
    2. `-ot`：(older than)file1是否比file2旧
    3. `-ef`：file1和file2是否是同一个文件，主要用于判断两个硬链接文件是否指向同一个文件

4. 整数之间的比较

    1. `-eq`：两数值相等(equal)
    2. `-ne`：两数值不相等(not equal)
    3. `-gt`：n1大于n2(greater than)
    4. `-lt`：n1小于n2(less than)
    5. `-ge`：n1大于等于n2(greater than or equal)
    6. `-le`：n1小于等于n2(less than or equal)

5. 字符串

    1. `-n`：字符串长度不为0
    2. `-z`：字符串长度为0
    3. `=`：两个字符串相等
    4. `!=`两个字符串不相等

6. 多重条件

    1. `-a`：(and)两状况同时成立
    2. `-o`：(or)两状况任何一个成立
    3. `!`：判断结果取反

## shell脚本睡眠

- `sleep 1`：睡眠1秒
- `sleep 1s`：睡眠1秒
- `sleep 1m`：睡眠1分
- `sleep 1h`：睡眠1小时

## 流程控制

### if

**只包含if**

```
if condition
then
    command1
    command2
    ...
    commandN
fi
```

**if else**

```
if condition
then
    command1
    command2
    ...
    commandN
else
    command
fi
```

**if else-if else**

```
if condition1
then
    command1
elif condition2
then
    command2
else
    commandN
fi
```

### for

```
for var in item1 item2 ... itemN
do
    command1
    command2
    ...
    commandN
done
```

### while

```
while condition
do
    command
done
```

### 无限循环

```
while :
do
    command
done
```

```
while true
do
    command
done
```

```
for (( ; ; ))
```

### until循环

until循环执行一系列命令直到条件为true时停止。

until循环与while循环在处理方式上刚好相反

```
until condition
do
    command
done
```

### case

case语句为多选择语句。可以用case语句匹配一个值与一个模式，如果匹配成功，执行相匹配的命令。

```
case 值 in
模式1)
    command1
    command2
    ...
    commandN
    ;;
模式2)
    command1
    command2
    ...
    commandN
    ;;
esac
```

取值后面必须为单词`in`，每一模式必须以右括号结束。取值可以为变量或常数。匹配发现取值符合某一模式后，期间所有命令开始执行直至`;;`。

取值将检测匹配的每一个模式。一旦模式匹配，则执行完匹配模式相应命令后不再继续其他模式。如果无一匹配模式，使用星号`*`捕获该值，再执行后面的命令。

### 跳出循环

- `break`：跳出当前的循环
- `continue`：跳过本次循环

### 注意事项

- shell的流程控制不可为空。如果else分支没有语句执行，就不要写这个else。

## 几种括号的区别

- `$()`和<code>\`\`</code>：`$()`和<code>\`\`</code>都是用来做命令替换的。

    比如，`version=$(uname -r)`和<code>version=\`uname -r\`</code>都可以使version得到内核的版本号
    
- `${}`：`${}`用于变量替换。一般情况下，`$var`和`${var}`并没有啥不一样。但是`${}`会比较精确地界定变量名称的范围。

    匹配替换模式：
    
    | 变量设定方式 | 说明  |
    | --- | --- |
    | `${var#pattern}` | 从左开始匹配，将符合的最短数据删除 |
    | `${var##pattern}` | 从左开始匹配，将符合的最长数据删除 |
    | `${var%pattern}` | 从右开始匹配，将符合的最短数据删除 |
    | `${var%%pattern}` | 从右开始匹配，将符合的最长数据删除 |

    注意：只有`pattern`为通配符模式时，最短最长匹配才有效

    几种特殊的替换模式：
    
    | 变量设定方式 | str没有设定 | str为空字符串 | str已设定为非空字符串 |
    | --- | --- | --- | --- |
    | `var=${str-expr}` | var=expr | var= | var=$str |
    | `var=${str:-expr}` | var=expr | var=expr | var=$str |
    | `var=${str+expr}` | var= | var=expr | var=expr |
    | `var=${str:+expr}` | var= | var= | var=expr |
    | `var=${str=expr}` | str=expr var=expr | str不变 var= | str不变 var=$str |
    | `var=${str:=expr}` | str=expr var=expr | str=expr var=expr | str不变 var=$str |
    | `var=${str?expr}` | expr输出至stderr | var= | var=$str |
    | `var=${str:?expr}` | expr输出至stderr | expr输出至stderr | var=$str |

- `$[]`和`$(())`：它们是一样的，都是进行数学运算的。支持`+ - * / %`。注意，bash只能作整数运算，对于浮点数是当做字符串处理的。
- `[]`：test命令的另一种形式

    注意：
    
    1. 必须在左括号的右侧和右括号的左侧各加一个空格，否则会报错
    2. test命令使用标准的数学比较符号来表示字符串的比较
    3. 大于符号或小于符号必须要转义，否则会被理解成重定向

- `(())`和`[[]]`：它们分别是`[]`的针对数学比较表达式和字符串表达式的加强版

    其中`(())`，不需要再将表达式里面的大小于符号转义，除了可以使用标准的数据运算符外，还增加了以下符号：
    
    
    | 符号 | 描述 |
    | --- | --- |
    | `val++` | 后增 |
    | `val--` | 后减 |
    | `++val` | 先增 |
    | `--val` | 先减 |
    | `!` | 逻辑求反 |
    | `~` | 位求反 |
    | `**` | 幂运算 |
    | `<<` | 左位移 |
    | `>>` | 右位移 |
    | `&` | 位布尔和 |
    | `|` | 位布尔或 |
    | `&&` | 逻辑和 |
    | `||` | 逻辑或 |

## 输入/输出重定向

大多数UNIX系统命令从你的终端接受输入并将产生的输出发送回到您的终端。一个命令通常从一个叫标准输入的地方读取输入，默认情况下，这恰好是你的终端。同样，一个命令通常将其输出写入到标准输出，默认情况下，这也是你的终端。


| 命令 | 说明 |
| --- | --- |
| command > file | 将输出重定向到file |
| command < file | 将输入重定向到file |
| command >> file | 将输出以追加的方式重定向到file |
| n > file | 将文件描述符为n的文件重定向到file |
| n >> file | 将文件描述符为n的文件以追加的方式重定向到file |
| n >& m | 将输出文件m和n合并 |
| n <& m | 将输入文件m和n合并 |
| << tag | 将开始标记tag和结束标记tag之间的内容作为输入 |

### 重定向深入讲解

一般情况下，每个Unix/Linux命令运行时都会打开3个文件：

- 标准输入文件(stdin)：stdin的文件描述符为0，Unix程序默认从stdin读取数据
- 标准输出文件(stdout)：stdout的文件描述符为1，Unix程序默认向stdout输出数据
- 标准错误文件(stderr)：stderr的文件描述符为2，Unix程序会向stderr流中写入错误信息

默认情况下，`command > file`将stdout重定向到file，`command < file`将stdin重定向到file。

如果希望stderr重定向到file，可以这样写：`command 2 > file`。

如果希望stderr追加到file文件末尾，可以这样写：`command 2 >> file`。2表示标准错误文件(stderr)。

如果希望将stdout和stderr合并后重定向到file，可以这样写：

```
command > file 2>&1
或者
command >> file 2>&1
```

如果希望对stdin和stdout都重定向，可以这样写：

`command < file1 > file2`

command命令将stdin重定向到file1，将stdout重定向到file2。

### Here Document

Here Document是shell中的一种特殊的重定向方式，用来将输入重定向到一个交互式shell脚本或程序。

```
command << delimiter
    document
delimiter
```

它的作用是将两个delimiter之间的内容(document)作为输入传递给command

注意：

- 结尾的delimiter一定要顶格写，前面不能有任何字符，后面也不能有任何字符，包括空格和tab缩进
- 开始的delimiter前后的空格会被忽略掉

### /dev/null文件

如果希望执行某个命令，但又不希望在屏幕上显示输出结果，那么可以将输出重定向到`/dev/null`：`command > /dev/null`

`/dev/null`是一个特殊文件，写入到它的内容都会被丢弃；如果尝试从该文件读取内容，那么什么也读不到。但是`/dev/null`文件非常有用，将命令的输出重定向到它，会起到**禁止输出**的效果。

如果希望屏蔽stdout和stderr，可以这样写：

`comamnd > /dev/null 2>&1`

## 使用IFS设置分隔符分割字符串

IFS(internal field separator)用于指定分隔符，示例如下：

```
while IFS=';' read -ra ADDR; 
do
    for i in "${ADDR[@]}"; do
        # 处理 "$i"
    done
done <<< "$IN"
```











