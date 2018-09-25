redis除了提供五种数据结构外,还提供一些功能.

## 慢查询分析

redis提供了慢查询日志来帮助定位系统中存在的慢操作. 系统命令在执行前后统计每条命令的执行时间, 当超过预设的阈值时,就记录该命令的相关信息. 

### 命令执行

redis客户端执行命令的流程:

![](http://ww1.sinaimg.cn/large/005JpQbVgy1fvlrvvit3zj30nu0d2q3l.jpg)



redis客户端执行命令分为4个步骤:

1. 发送命令
2. 命令排队
3. 执行命令
4. 返回结果

redis的慢查询日志只统计 执行命令的时间. 



### 慢查询的两个配置参数

对应redis的慢查询功能有两个问题:

+ 预设阈值如何设置? 
+ 慢查询记录存放在哪儿 ?

慢查询相关的两个参数:

+ `slowlog-log-lower-than` 预设阈值

+ `slowlog-max-len `  慢日志最大记录条数

> 使用  
>
> `config set slowlog-log-slower-than 20000 `
>
> `config set slowlog-max-len 1000 `
>
> `config rewrite`   命令可以配置产生并将其写入配置文件



**获取慢查询日志**

`slowlog get n`  返回n条慢查询日志

返回结果:

```
1) (integer) 100
   2) (integer) 1537859984
   3) (integer) 13
   4) 1) "SLOWLOG"
      2) "get"
      3) "1"
   5) "127.0.0.1:49426"
   6) ""
```

返回结果由4部分组成:

1. 慢查询日志的标识id 
2. 发生时间戳
3. 命令耗时
4. 执行命令和参数

**获取慢日志列表当前长度**

`slowlog len`

**重置慢日志查询记录**

`slowlog reset`



## redis Shell

### redis-cli

参数 : 

+ -r (repeate) 命令执行多次  执行三次ping : `redis-cli -r 3 ping ` 
+ -i (interval) 隔几秒执行一次命令,必须和 -r 一起使用. `redis-cli -r 3 -i 1 ping`

+ -x 标准输入. 读取数据作为redis-cli的最后一个参数. 例如: `echo "wolrd" | redis-cli  -x set hello` 
+ -c (cluster) 连接 redis cluster节点时候使用 -c 选项可以防止 moved 和 ask异常
+ -a (auth)  如果redis 配置了密码 使用 -a 输入密码,后不需要 使用auth 命令输入密码. 
+ --scan --pattern 用于扫描指定模式的键 相当于 scan命令
+ --slave 把当前客户端模拟成redis节点的从节点. 可以用来获取当前Redis节点的更新操作

+ --rdb 请求Redis实例生成并发送RDB持久化文件.  例如:`redis-cli --red test.rdb`
+ --pipe 将命令封装成 redis 通讯协定的数据格式, 批量发送给Redis. 
+ --bigkeys 使用scan命令对redis的键进行采样 . 从中找到内存占比较大的键值.

+ --eval 指定执行的Lua脚本
+ -- latency 测试延迟
  + --latency 测试客户端到目标Redis的网络延迟 例如: `redis-cli -h server.host  --latency`
  + --latency-history  分时段形式了解延迟信息, 默认15秒输出一次, 使用-i指定时间间隔 . `redis-cli -h server.host  --latency-history -i 1` 
  + --lantency-dist  使用统计图表的方式展示数据
+ --state 实时获取redis的重要统计信息 
+ --raw和--noraw  --no-raw 命令返回的结果必须是原始格式. --raw  返回格式化后的结果



### redis-server

+ --test-memory 检测系统能否稳定的分配指定容量的内存给redis. 

  例:`redis-server --test-memory 1024`

### redis-benchmark

redis-benchmark 为redis做基准测试. 

1. -c (clent) 客户端的并发数 (默认50)
2. -n 客户端请求总量

例:`redis-beanchmark -c  100 -n 20000` 表示100个客户端同时请求redis 一共执行2000次 . 

redis-benchmark会对各类数据结构的命令进行测试.  

3. -q 只显示 requests per second 
4. -r 随机的插入更多的键 `redis-benchmark -c 100 -n 20000 -r 10000`
5. -P 表示每个请求的pipeline数据量
6. -k 是否使用 keepalinve. 1.使用, 0 不使用 默认1 
7. -t 指定命令进行基准测试
8. --cvs 按照csv格式输出 



## Pipeline

pipeline(流水线)机制: 将一组Redis命令进行组装,通过一次RTT传输给Redis,Redis执行完这些命令后将执行的结果按照顺序返回给客户端.

> Round Trip Time 返还时间 简写 :RTT



原生批量执行和Pipline执行的对比:

+ 原生批量执行具有原子性
+ 原型批量命令是一个命令对应多个key Pipeline 支持多个命令
+ 原生批量执行是Redis服务端支持实现的. 而pipeline 需要服务端和客户端共同完成



## 事务与Lua









