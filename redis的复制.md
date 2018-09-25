redis提供了复制功能,可以将数据复制多个redis副本中. 

## 配置

复制redis数据时需要划分主节点(master) 和从节点(slave) . 复制数据是单向的只能由主节点复制到从节点.  

配置从属的节点:

+ 在配置文件中加入  `slave of [master host] [master port]` 随redis一起启动
+ 在redis-servler 启动的命令后面加入 `--slaveof [master host] [master port] `
+ 直接使用命令: `slaveof [master host] [master port]`



### 断开配置

执行 `slave of one` 断开与主节点的联系. 

断开复制的主要流程:

1. 断开与主节点的复制关系
2. 从节点晋升为主节点

使用命令`slaveof newMasterIP newMasterport` 可以切换主节点. 

切换主节点的流程: 

1. 断开与主节点的复制关系
2. 与新节点建立复制关系
3. 删除从节点所有的数据
4. 对新主节点进行复制

如果主节点设置requirepass参数,从节点也要设置和主节点相同的参数才能进行复制.

默认情况下,从节点使用 `slave-read-only=yes`配置为只读模式.  

主从节点一般配置在不同的机器上, 复制时网络延迟会影响复制. Redis提通了`repl-disable-tcp-nodelay`参数用于控制是否关闭`TCP_NODELAY` 默认情况下关闭. 

- 当关闭时.  主节点参数的命令数据都会及时的发送给从节点. 主从延迟小,增加了网络消耗.适用于主从的网络环境良好的场景.
- 当开启时. 主节点会合并较小的tcp数据包,以节省带宽. 默认发送时间间隔取决于linux内核. 一般默认为40毫秒. 



## 复制的原理 

### 复制过程

从节点执行slaveof 命令后复制过程开始运作,复制过程分为6个步骤.

1. 保存住节点(master)的信息

2. 从节点与主节点建立socket网络通讯

3. 建立网络连接后,从节点发送ping 请求进行首次通信.

   发送ping命令的目的:

   + 检查主从之间的网络是否可用 
   + 检测主节点是否可接受处理名

4. 权限验证, 如果逐渐的设置了`requirepass`参数. 需要密码验证,从节点必须配置masterauth 产生保证主节点的密码相同才能通过验证

5. 同步数据集.主从正常通信后,  首次建立连接时主节点会将所有的数据全部发送给从节点, 这个步骤耗时较长.

   > 2.8以后使用psync命令进行数据同步.  

6. 命令持续复制. 当主节点把之前的数据同步给从节点完成之后, 完成了复制的建立流程, 后面主节点会持续的将写命令发送给从节点来保持主从数据一致.



### 读写分离

对于读占比比较高的场景,可以将一部分读的流量分摊到从节点上执行,来减轻主节点的压力. 



## redis的内存

### 内存的消耗

1. 内存使用统计

   通过执行`info memory` 命令可以查看redis内存相关的指标. 

   内存统计指标:

   |         属性名          |                         说明                          |
   | :---------------------: | :---------------------------------------------------: |
   |       used_memory       | Redis分配器分配的内存总量, 内部存储所有数据内存占用量 |
   |    used_memory_human    |              以可读的方式展示used_memory              |
   |     used_memory_rss     |       操作系统显示的Redis进程占用的物理内存总量       |
   |    used_memory_peak     |                     内存使用峰值.                     |
   |     used_memory_lua     |                  Lua引擎消耗内存大小                  |
   | mem_fragmentation_ratio |   used_memory_rss/used_memory的比值, 表示内存碎片率   |
   |      mem_allocator      |                 redis使用的内存分配器                 |

   + 当mem_fragmentation_ratio>1 时. 

 

## Redis的集群







