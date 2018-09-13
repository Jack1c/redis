# redis的持久化

持久化能避免因进程的退出导致的数据丢失的问题. 当下次启动时使用之前持久化的文件能恢复之前的数据. 



## RDB持久化

RDB持久化是将当前redis实例的进程数据生成快照保存到硬盘. 

### 触发方式

1. 手动触发

   `save ` 命令

   执行save命令时会阻塞当前Redis实例,知道持久化完成. 对内存比较大的实例可能会造成长时间的阻塞. 

   `bgsave命令` 

   redis进程会先fork操作创建子进程, 持久化操作由子进程负责, 完成后子进程自动结束. 阻塞过程只发生在fork阶段,一般时间很短.  redis内部所有涉及到RDB持久化的操作都使用 `bgsave` 的方式. 

2. 自动触发

+ 使用save相关的配置时, 例如 `save m n`, 表示m秒内数据集在n次修改时,自动触发 `bgsave`
+ 从redis节点执行全量复制操作时, 主节点自动执行 `bgsave` 生成RDB文件,发送给从节点
+ 执行debug reload命令重新加载 redis时 会自动触发 save操作
+ 默认情况下 执行 `shutdown`命令时, 如果没有开启AOF持久化,就会自动执行bgsave.

### 执行流程

![](http://ww1.sinaimg.cn/large/005JpQbVgy1fv7yghlsefj30ti0m6jsa.jpg)

1. 执行`bgsave`命令,redis实例判断是否存在正在执行的子进程, 比如 RDB/AOF进程, 如果存在直接返回. 

2. 执行fork创建子进程, 在进行时redis实例会阻塞. 

   >  info stats命令查看latest_fork_usec选项，可以获取最近一个fork操作的耗 时，单位为微秒。

3. redis实例完成fork操作后, 返回 信息 redis不再阻塞, 开始响应其他命令 

4. 子进程创建RDB文件. 根据redis实例进程内存生成临时快照文件. 完成后对原有的文件进行原子性替换. 

   > 执行`lastsave `命令可以查看最后一次生成RDB的时间

5. 子进程给redis实例发送信号通知实例化完成,redis实例更新统计信息. 具体



## AOF持久化