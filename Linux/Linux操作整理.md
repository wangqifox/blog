---
title: Linux操作整理
date: 2018/05/07 14:43:00
---

# sudo命令免密码

配置`sudo`文件，通过`visudo`安全地进行设置

```
sudo visudo
```

文件最后为用户`robin`添加免密码，改成你的用户名即可。

```
# Members of the admin group may gain root privileges
robin ALL=(ALL) NOPASSWD:ALL
```
<!-- more -->

# 文件的打包、解压缩指令

> https://www.cnblogs.com/yhjoker/p/7568680.html

## 文件后缀的含义

随着压缩技术的发展，Linux环境下提供的压缩指令和格式开始变多。为了便于用户区分不同压缩文件使用的不同压缩技术，进而使用合适的指令进行操作，一般使用后缀标识文件在压缩或打包过程中所使用的压缩技术。常见的后缀有以下几种：

```
*.Z         //compress程序压缩产生的文件（现在很少使用）
*.gz        //gzip程序压缩产生的文件
*.bz2       //bzip2程序压缩产生的文件
*.zip       //zip压缩文件

*.tar       //tar程序打包产生的文件
*.tar.gz    //由tar程序打包并由gzip程序压缩产生的文件
*.tar.bz2   //由tar程序打包并由bzip2程序压缩产生的文件
```

由后缀可以看出，gzip、bzip2、tar指令是在打包和压缩过程中较为常用的指令

## 压缩命令——gzip、bzip2

gzip

gzip可以压缩产生后缀为.gz的压缩文件，也可以用于解压gzip、compress等程序压缩产生的文件。不带任何选项和参数使用gzip或只带有参数`-`时，gzip从标准输入读取输入，并在标准输出输出压缩结果。

gzip的常用指令选项如下：

```
基础格式：gzip [Options] file1 file2 file3
指令选项：（默认功能为压缩）
-c          //将输出写至标准输出，并保持原文件不变
-d          //进行解压操作
-v          //输出压缩/解压的文件名和压缩比等信息
-digit      //digit部分为数字(1-9)，代表压缩速度，digit越小，则压缩速度越快，但压缩效果越差，digit越大，则压缩速度越慢，压缩效果越好。默认为6。
```

注意，使用gzip指令压缩/解压缩文件均会使得原文件消失，即源文件会被直接解压/压缩而不保留备份。若想要保留原文件可以使用`-c`参数结合数据流重定向操作。

```
gzip exp1.txt expt2.txt         // 分别将exp1.txt和exp2.txt压缩，且不保留原文件。注意对于多个文件参数是将多个文件分别进行压缩，而不是压缩在一起。
gzip -dv exp1.gz                // 将exp1.gz解压，并显示压缩比等信息
gzip -cd exp1.gz > exp.1        // 将exp1.gz解压的结果放置在文件exp.1中，并且原压缩文件exp1.gz不会消失
```

特别注意第三条实例，`-d`指示解压缩，`-c`参数是将结果输出至标准输出，通过`>`符号，将原本输出至标准输出的解压结果重定向至exp.1中，既解压了文件，原压缩文件也没有消失。

bzip2

bzip2是采用更好压缩算法的压缩程序，一般可以提供较之gzip更好的压缩效果。其具有与gzip相似的指令选项，压缩产生.bz2后缀的压缩文件。

```
基础格式：bzip2 [Options] file1 file2 file3
指令选项：（默认功能为压缩）
-c          //将输出写至标准输出
-d          //进行解压操作
-v          //输出压缩/解压的文件名和压缩比等信息
-k          //在压缩/解压过程中保留原文件
-digit      //digit部分为数字(1-9)，代表压缩速度，digit越小，则压缩速度越快，但压缩效果越差，digit越大，则压缩速度越慢，压缩效果越好。默认为6
```

```
bzip2 exp1.txt ext2.txt     //分别将exp1.txt和exp2.txt压缩，且不保留原文件
bzip2 -dv exp1.bz2          //将exp1.bz2解压，并显示压缩比等信息
bzip2 -kd exp1.bz2          //将exp1.bz2解压，并且原压缩文件exp1.bz2不会消失
```

## 打包指令——tar

上文已经提到，gzip或bzip2带有多个文件作为参数时，执行的操作是将各个文件独立压缩，而不是将其放在一起进行压缩。这样就无法产生类似于Windows环境下的文件夹打包压缩效果。（gzip与bzip2也可以使用文件夹作为参数，使用`-f`选项，但也是将其中的每个文件独立压缩）。为了实现打包压缩的效果，可以使用命令tar进行文件的打包操作(archive)，再进行压缩。

tar指令可以将文件打包成文件档案(archive)存储在磁盘/磁带中，打包操作一般伴随着压缩操作，也可以使用tar指令对打包压缩后的文件解压。

```
基本格式：tar [Options] file_archive
常用命令参数：
// 指定tar进行的操作，以下三个选项不能出现在同一条命令中
-c              //创建一个新的打包文件(archive)
-x              //对打包文件(archive)进行解压操作
-t              //查看打包文件(archive)的内容，主要是构成打包文件(archive)的文件名

// 指定支持的压缩/解压文件，操作取决于前面的参数，若为创建(-c)，则进行压缩，若为解压(-x)，则进行解压，不加下列参数时，则为单纯的打包操作
-z              //使用gzip进行压缩/解压，一般使用.tar.gz后缀
-j              //使用bzip2进行压缩/解压，一般使用.tar.bz2后缀

// 指定tar指令使用的文件，若没有压缩操作，则以.tar作为后缀
-f filename     //-f后面接操作使用的文件，用空格隔开，且中间不能有其他参数，推荐放在参数集最后或单独作为参数
                //文件作用取决于前面的参数，若为创建(-c)，则-f后为创建的文件的名字(路径)，若为(-x/-t)，则-f后为待解压/查看的打包压缩文件名
                
// 其他辅助选项
-v              //详细显示正在处理的文件名
-C Dir          //将解压文件放置在-C指定的目录下
-p(小写)          //保留文件的权限和属性，在备份文件时较有用
-P(大写)          //保留原文件的绝对路径，即不会拿掉文件路径开始的根目录
--exclude=file  //排除不进行打包的文件
```

常见的tar指令操作如下：

```
压缩：
tar -cvjpf etc.tar.bz2 /etc     //-c为创建一个打包文件，相应的-f后面接创建的文件的名称，使用了.tar.bz2后缀，-j标志使用bzip2压缩，最后面为具体的操作对象/etc目录

查看：
tar -tvjf etc.tar.bz2           //-t为查看操作，则-f对应所查看的文件的名称，文件后缀显示使用bzip2进行压缩，所以加入-j选项，-v会显示详细的权限信息

解压：
tar -xvjf etc.tar.bz2           //-x为解压操作，则-f指定的是解压使用的文件，文件后缀显示使用bzip2进行压缩，所以加入-j选项，即使用bzip2解压
                                //若只解压指定打包文件中的一个文件，在上述指令的最后加上带解压文件名作为参数即可
```

注意：使用tar打包的文件会保存原有的文件路径，并默认去除了所有成员文件路径的根目录。

这样做的目的在于，当用户在某一目录如`/home/haha`目录下进行解压操作时，tar会将解压出来的文件路径与当前目录拼接，即为`/home/haha/etc/emacs`，从而将文件解压在当前目录下。（如果还有印象，目录名也可以使用-C选项指定）但若是打包压缩时不去除文件路径的根目录，则会按照存储的绝对路径如`/etc/emacs`解压文件，可能将`/etc`文件下的相应文件覆盖掉，当然在进行备份和恢复时该操作是有效的。tar提供`-P`选项来保留文件路径的根目录。

## zip文件相关命令——unzip

unzip命令与之前的tar指令类似，具有对zip文件进行查看、测试和解压的功能

```
基本格式：unzip [Options] file[.zip]     //不接任何Options时，默认将指定的file文件解压至当前文件夹
可同时接受多个文件参数
常用命令参数：
//压缩文件内容查看
-Z          //以形如ls -l的格式显示目标文件内容，实际原理是命令第一个参数为-Z时，其余参数会被视为zipinfo的参数，并产生对应效果
-Zl         //仅显示压缩文件内容的文件名，更多显示可查看zipinfo命令的man帮助
-l          //显示压缩文件中包括时间、占用空间和文件名等信息，内容上较-Z更简单

//文件测试
-t          //在内容中解压文件并进行文件的完整性校验(CRC校验)

//解压缩参数，注意unzip默认即为解压操作
-f          //注意与tar命令不同，unzip指定-f参数时，则将磁盘上已经存在且内容新于对应磁盘文件的压缩内容解压出来
-n          //解压缩时不覆盖已存在的文件(而是跳过)
-q          //安静模式，仅解压缩而不输出详细信息
-d dir      //将文件解压至dir指定的文件夹中
```

可以使用unzip命令对zip文件进行相关的操作：

```
查看压缩文件的所有文件名（注意-Z选项表示之后所有的参数被视为zipinfo的参数并输出相应结果）
unzip -Z1 file.zip

测试文件的完整性
unzip -t file.zip

将文件解压至当前用户的主目录
unzip -q file.zip -d ~
```

# 软连接与硬链接

## 软连接与硬链接的区别

从使用的角度讲，两者没有任何区别，都与正常的文件访问方式一样，支持读写，如果是可执行文件的话也可以直接执行。

区别在底层的原理上。

我们首先在自己的一个工作目录下创建一个文件，然后对这个文件进行链接的创建：

```
$ touch myfile && echo "This is a plain text file." > myfile
$ cat myfile

This is a plain text file.
```

然后我们对它创建一个硬链接，并查看一下当前目录：

```
$ ln myfile hard
$ ls -li

25869085 -rw-r--r--  2 unixzii  staff  27  7  8 17:39 hard
25869085 -rw-r--r--  2 unixzii  staff  27  7  8 17:39 myfile
```

在`ls`结果的最左边一列，是文件的inode值，你可以简单把它想成C语言中的指针。它指向了物理硬盘的一个区块，事实上文件系统会维护一个引用计数，只要有文件指向这个区块，它就不会从硬盘上消失。

这两个文件就如同一个文件一样，`inode`值相同，都指向同一个区块。

然后我们修改一下刚才创建的hard链接文件：

```
$ echo "New line" >> hard
$ cat myfile

This is a plain text file.
New line
```

可以看到，这两个文件果真就是一个文件。

下面看看软连接（也就是符号连接）和它有什么区别。

```
$ ln -s myfile soft
$ ls -li

25869085 -rw-r--r--  2 unixzii  staff  36  7  8 17:45 hard
25869085 -rw-r--r--  2 unixzii  staff  36  7  8 17:45 myfile
25869216 lrwxr-xr-x  1 unixzii  staff   6  7  8 17:47 soft -> myfile
```

你会发现，这个软连接的`inode`不一样，并且它的文件属性上也有一个`l`的flag，这就说明它与之前我们创建的两个文件根本不是一个类型。

下面我们试着删除myfile文件，然后分别输出软硬链接的文件内容。

```
$ rm myfile
$ cat hard

This is a plain text file.
New line

$ cat soft

cat: soft: No such file or directory
```

之前的硬链接没有丝毫地影响，因为它`inode`所指向的区块由于有一个硬链接在指向它，所以这个区块仍然有效，并且可以访问到。

然而软连接的`inode`所指向的内容实际上是保存了一个绝对路径，当用户访问这个文件时，系统会自动将其替换成其所指的文件路径，然而这个文件已经被删除了，所以自然就会显示无法找到该文件了。

为验证这一猜想，我们再向这个软连接写点东西：

```
$ echo "Something" >> soft
$ ls

hard   myfile soft
```

可以看到，刚才删除的myfile文件竟然又出现了！这就说明，当我们写入访问软连接时，系统自动将其路径替换为其所代表的绝对路径，并直接访问那个路径了。

## 硬链接

### 建立硬链接

```
ln [options] source_file target_file
ln [options] source_file... target_file
source_file是待建立链接文件的文件，target_file是新创建的链接文件

-f  建立时，将同名档案删除
-i  删除前进行询问
```

## 软连接

### 建立软连接

```
ln -s source_file target_file
source_file是待建立链接文件的文件，target_file是新创建的链接文件
```

### 删除软连接

假设我们有dir文件夹，通过`ln -s dir dir2`命令建立了一个dir2的软连接。

删除软连接的正确方式是`rm dir2`，如果我们使用`rm -rf dir2/`命令删除的则是dir中实际存在的数据，可能会有严重的后果。


> https://www.jianshu.com/p/dde6a01c4094
> https://www.cnblogs.com/xiaochaohuashengmi/archive/2011/10/05/2199534.html
> http://blog.51cto.com/kusorz/1876315

# 查看端口占用情况

> http://lazybios.com/2015/03/netstat-notes/

## netstat

`netstat`用来查看系统当前网络状态信息，包括端口、连接情况等，常用方式如下：

`netstat -atunlp`，各参数含义如下：

- `-t`：显示tcp端口
- `-u`：显示udp端口
- `-l`：仅显示监听套接字（LISTEN状态的套接字）
- `-p`：显示进程标识符和程序名称，每个套接字/端口都属于一个程序
- `-n`：不进行DNS解析
- `-a`：显示所有连接的端口

## lsof

`lsof`的作用是列出当前系统打开文件(list open files)，不过通过`-i`参数也能查看端口的连接情况，`-i`后跟冒号端口可以查看指定端口信息，直接`-i`是系统当前所有打开的端口

`lsof -i:22 #查看22端口连接情况`