---
title: lua基础语法整理
date: 2018/08/01 16:23:00
---

最近在针对Redis的并发编程时用到了lua脚本，因为对lua不熟，所以在本文对lua的基本语法进行一个整理记录。

<!-- more -->

# 循环

## while循环

```lua
while(condition)
do
    statements
end
```

## for循环

### 数值for循环

```lua
for var=exp1,exp2,exp3 do
    <执行体>
end
```

var从exp1变化到exp2，每次变化以exp3为步长递增var，并执行一次"执行体"。exp3是可选的，如果不指定，默认为1。

for的三个表达式在循环开始前一次性求值，以后不再进行求值。

### 泛型for循环

泛型for循环通过一个迭代器函数来遍历所有值，类似Java中的foreach语句。

```lua
--打印数组a的所有值
for i,v in ipairs(a)
    do print(v)
end
```

i是数组的索引值，v是对应索引的数组元素值。ipairs是Lua提供的一个迭代器函数，用来迭代数组。

## repeat...until循环

repeat...until循环语句不同于for和while循环，for和while循环的条件语句在当前循环执行开始时判断，而repeat...until循环的条件语句在当前循环结束后判断。

```lua
repeat
    statements
until(condition)
```

# 流程控制

## if...elseif...else语句

```lua
if(布尔表达式1)
then
    --在布尔表达式1为true时执行该语句块
elseif(布尔表达式2)
then
    --在布尔表达式2为true时执行该语句块
elseif(布尔表达式3)
then
    --在布尔表达式3为true时执行该语句块
else
    --如果以上布尔表达式都不为true则执行该语句块
end
```

# 函数

```lua
optional_function_scope function function_name(argument1, argument2, argument3..., argumentn)
    function_body
    return result_params_comma_separated
end
```

- optional_function_scope：该参数是可选的。指定函数是全局函数还是局部函数，未设置该参数默认为全局函数，如果你需要设置函数为局部函数需要使用关键字local
- function_name：指定函数名称
- argument1, argument2, argument3..., argumentn：函数参数，多个参数以逗号隔开，函数也可以不带参数
- function_body：函数体，函数中需要执行的代码语句块
- result_params_comma_separated：函数返回值，Lua语句可以返回多个值，每个值以逗号隔开

# Lua脚本支持随机操作

Redis内嵌了Lua环境来支持用户扩展功能，但是出于数据一致性考虑，要求脚本必须是纯函数的形式，也就是说对于一段Lua脚本给定相同的参数，重复执行其结果都是相同的。

为什么要有这个限制呢？原因是Redis不仅仅是单机版的内存数据库，它还支持主从复制和持久化，执行过的Lua脚本会复制slave以及持久化到磁盘，如果重复执行得到结果不同，那么就会出现内存、磁盘、slave之间的数据不一致，在failover或者重启之后造成数据错乱影响业务。

还是以具体例子来看，假设有这么一段Lua脚本，目的很简单就是想记录下当前时间：

```lua
local now = redis.call('time')[1]
redis.call('set', 'now', now)
return redis.call('get', 'now')
```

这里使用了Redis的TIME命令来获取时间戳，然后存储到名为now的key中，但是其执行时会报错：

```
redis-cli --eval 5.lua
(error) ERR Error running script (call to f_deb4cdd62e78fe2453f1a2da456b01f4045b9f97): @user_script:2: @user_script: 2: Write commands not allowed after non deterministic commands. Call redis.replicate_commands() at the start of your script in order to switch to single commands replication mode.
```

错误提示也很明显，如果执行过非确定性命令（也就是TIME，因为时间是随机的），Redis就不允许执行写命令，以此来保证数据一致性。那如何才能实现随机写入呢？刚才的错误提示也给出了答案，使用`redis.replicate_commands()`，在执行`redis.replicate_commands()`之后，Redis就不再是把整个Lua脚本同步给slave和持久化，而是把脚本中调用Redis的写命令直接去做复制，那么slave和持久化也可以得到确定的结果。

脚本修改如下：

```lua
redis.replicate_commands()
local now = redis.call('time')[1]
redis.call('set', 'now', now)
return redis.call('get', 'now')
```

再执行就可以实现随机写入了。































> http://www.runoob.com/lua/lua-tutorial.html
> https://juejin.im/entry/5b39cfdfe51d4558b277ad01


