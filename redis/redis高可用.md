### 持久化机制 RDB和AOF
##### RDB：Redis DataBase
> 把某一时刻的数据以快照的形式保存在磁盘上。

1. 触发方式
    - save命令：阻塞执行save命令，其它命令不能进行处理，直到RDB过程完成，但不会消耗额外内存
    - bgsave： fork创建子进程异步进行快照，fork阶段阻塞，但一般时间很短，子进程需要消耗内存
    - 自动触发：配置文件中进行设置
2. 优劣
    - 优
        - 文件紧凑，全量备份，适合备份和灾难恢复
        - 使用bgsave进行RDB时主进程不需要进行任何IO操作
        - 大数据集恢复时更快
        - 适合冷备和复制传输
    - 劣
        - bgsave快照持久化期间修改的数据不会被保存，可能丢失数据
        - 持久化速度比较慢

3. bgsave
    - fork，创建一个子进程，子进程共享父类的内存数据
    - copyonwrite：主进程fork子进程后，内核将主进程中所有的内存页的权限都设置read-only。当主进程接收到写命令时，
    检测到内存页是只读的，于是触发页异常中断（page-fault），陷入内核的一个中断例程。在中断例程中，内核就会把触发的异常
    页复制一份（仅复制异常页，而不是内存的全部数据），于是主进程和子进程各自持有独立的一份。
        - 减少空间资源的浪费，降低复杂性，提高性能
        - 对于大多redis节点而言，写请求一般都是远小于读请求的，所以复制的内存空间不会很大。    

##### AOF,Append Only File
> 日志记录，每执行一个命令都追加到日志文件中

1. 触发方式
    - 每次同步：，同步持久化，每次执行之后会立即记录到磁盘，性能较差但是数据完整性比较好，主线程执行，主线程阻塞
    - 每秒同步：异步操作，每秒记录，性能较好但是数据丢失的可能性更大，子进程执行，主线程不会阻塞
    - 不同步：每次aof时都不进行刷入磁盘的同步操作，只有当满足redis被关闭、aof功能被关闭，系统刷新缓存等原因时，才会进行刷入
    磁盘，且主进程会被阻塞
    
2. 优劣
    - 优
        - 数据丢失的可能性更小
        - AOF顺序写入磁盘，写入性能非常高，文件不容易破损
        - AOF文件可读性高
    - 劣
        - AOF日志文件通常会更大
        - 对性能影响比较大

3. 还原
    - 打开AOF文件
    - 循环读取Redis命令并执行Redis命令      
        
4. 文件重写 
    - 目的：减小文件体积，提高恢复效率
    - 使用： 
        - 手动执行bgrewriteaof命令
        - 自动触发，默认情况下当前AOF文件比最后一次AOF重写的大小大一倍就会触发
    - 流程：
        - fork出一条子进程将整个内存中的数据内容用命令的方式重写入一个新的aof文件
        - 在重写期间，主进程除了需要将修改命令写入AOF文件外，还需要写入AOF重写缓存。
        - 子进程完成重写后，发送信号给主进程，主进程阻塞将AOF重写缓存中的内容写入新AOF文件，然后修改新的AOF文件覆盖旧文件。
    - 重点关注
        - 文件重写并没有读取旧的aof文件，而是与快照类似
        - 在整个AOF重写过程中，只有最后的写入缓存和对新AOF文件的改名操作会造成主进程的阻塞
        - 在AOF和RDB都开启时，Redis在重启的时候会默认使用AOF去重新构建数据，因为AOF的数据是比RDB完整的        

##### 混合持久化
> 同时结合RDB持久化和AOF持久化混合写入AOF文件，结合两者优点，快速加载的同时避免丢失过多的数据
> 缺点是AOF文件中的RDB部分是压缩格式



### 高可用
#### 主从复制
> 主节点负责读写，从节点负责读

- 主从复制流程
    - 从节点连接主节点，使用psync命令发送主节点的replication ID和复制偏移量
    - 主节点根据偏移量可以进行增量同步，或者全量同步

- 全量同步
    - 主节点开始执行bgsave命令开启一个后台进程生成rdb文件，主线程使用缓冲区记录此后执行的写命令
    - 后台进程保存完成时，主节点向从节点发送快照文件
    - 从节点将接收到的快照文件保存在磁盘上，然后进行加载
    - 主节点发送缓冲区中的写命令
    - 从服务器执行主服务同步过来的写命令
    
- 增量同步
    - 是从节点初始化后开始正常工作时的同步方式
    - 主节点发送命令保持对slave的更新，包括客户端的**写入、key的过期或被逐出**等修改主节点数据的操作
    
- backlog积压缓冲区
    - 由主节点维护的一个固定长度的FIFO队列，用于缓存已经传播出去的命令。
    - 当主节点进行命令传播时，不仅发送给从节点，还会将命令写入到复制积压缓冲区中。     
    - 全量复制时，主节点的修改命令会临时存放到backlog中等待完成全量复制后增量发送到从节点，必须为此保留足够的空间
    - 默认大小是1MB
    
- 当多个从节点与主节点同时进行全量同步时，可能导致主节点IO剧增宕机
- 主从复制在master主节点端是非阻塞的，可以同时处理客户顿命令以及主从同步
- 主从复制在从节点也是非阻塞的，可以配置从节点在初次同步时使用旧版本的数据集提供查询服务，也可以向客户端返回错误。
从节点加载新数据集时会阻塞，删除旧数据可以使用其它线程
- 全量同步时间过长时
- 当多个从节点向主节点发送同步请求，会执行一个单独的后台保存，以便于为多个 slave 服务。？？
- 磁盘性能低的话可以进行无盘复制，
- 可写的从节点不会将数据传播到与该节点相连的子从节点，子从节点只会接收到最顶层的master的复制流 
- 可配置在只有当N个节点在M时间内同步成功才被实际写入
- 在执行lua时，redis默认不允许动态的、不确定性的变量存在，如果存在，需要开启命令复制模式，所有的写命令都会包装在multi...exec里
- 在执行lua时，redis默认是复制整个脚本的内容，定义为whole scripts replication，而只复制写命令的模式定义为script effects replication

- 过期key的同步
    - 不能依赖主从使用同步时钟，一是无法使用同步时钟，二是会导致数据集不一致的问题
    - 从节点不对key进行过期处理，主节点中key过期后发送del命令同步到所有的子节点
    - 针对无法及时同步master的del命令的情况，从节点使用逻辑时钟判断key是否已经过期，保证跟主节点的一致性读取操作，但不进行删除操作
    - 不对设置了存活时间的键值进行特殊处理，不保证主从的强一致性
        - 使用expire设置过期时间的取决于主从同步延迟，延迟越短，主从不一致的时间就越短
        - 使用expireat设置过期时间的取决于主从节点各自的本地时钟的误差，误差越大，不一致可能越大
    - 在lua脚本执行过程中，master的时间是相当于被冻结的，以脚本执行开始时间对key进行过期处理，也要保证把相同的lua脚本发送到从节点
    
- Replication ID
    - 标记一个指定的历史数据集
    - master重启或者一个从节点升级为master都会生成一个新的replication ID
    - 从节点与主节点连接后获取到replication ID
    - 具有相同replication ID的节点的数据是相同的，但不一定是在同一时间
    - 从节点升级为主节点的过程中，会存在两个Replication ID
        - 升级为主节点时，从节点需要保留上一个主节点的replication ID以及复制偏移量offset
        - 其它从节点使用旧的主节点 replication ID 和offset进行重同步时，新的主节点会匹配两个replication ID，
        发生故障转移时从节点连接新的主节点时不需要进行全量同步
    ```
    redismodule.h
    typedef struct RedisModuleReplicationInfo {
        uint64_t version;       /* Not used since this structure is never passed
                                   from the module to the core right now. Here
                                   for future compatibility. */
        int master;             /* true if master, false if replica */
        char *masterhost;       /* master instance hostname for NOW_REPLICA */
        int masterport;         /* master instance port for NOW_REPLICA */
        char *replid1;          /* Main replication ID */
        char *replid2;          /* Secondary replication ID */
        uint64_t repl1_offset;  /* Main replication offset */
        uint64_t repl2_offset;  /* Offset of replid2 validity */
    } RedisModuleReplicationInfoV1;
    ```
    - 使用新的replication ID是因为可能发生网络分区等原因进行的故障转移，继续使用同一个replication ID违背了
    同一个replication ID具有相同数据集的约定。
### 哨兵
> Sentinel,是一种分布式架构，包含多个Sentinel节点和Redis数据节点。每个Sentinel节点会对数据节点和其余
> Sentinel节点进行监控，当它发现节点不可达时，会对节点做下线标识。当被标识节点是主节点时，还会同Sentinel
> 节点进行协商，当大多数Sentinel节点都认为主节点不可达时，会选举出一个Sentinel节点来完成自动故障转移工作。

- 功能
    - 监控：监控节点是否正常
    - 提醒：节点异常时可以向管理员或者应用程序发送通知
    - 自动故障迁移：节点异常时，会进行自动故障迁移
    - 注册中心：节点连接sentinel询问主点，发生故障转移时会进行报告推送
    
- 分布式
    - 大多数sentinel节点任务

- 领导者选举
> Raft算法，假设s1(sentinel-1)最先完成客观下线，它会向其余Sentinel节点发送命令，请求成为领导者；
> 收到命令的Sentinel节点如果没有同意过其他Sentinel节点的请求，那么就会同意s1的请求，否则拒绝；
> 如果s1发现自己的票数已经大于等于某个值，那么它将成为领导者。

- 故障转移
    - 领导者Sentinel在从节点列表中选出一个节点作为新的主节点
    - 选择的从节点是与主节点复制相似度最高的从节点
    - 让其它从节点成为新的主节点的从节点
    - 将原来的主节点更新为从节点，并保持关注，当其恢复后命令他去复制新的主节点

    
### 集群
