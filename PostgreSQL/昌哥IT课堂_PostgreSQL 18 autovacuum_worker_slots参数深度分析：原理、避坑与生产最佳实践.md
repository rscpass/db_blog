你好,我是昌哥,进免费技术交流群或咨询加微信: rscpass

---

先说结论：**这个参数存在，但只在 PostgreSQL 18 及以后的版本里有。**它由提交 c758119e5b 引入，PostgreSQL 13、14、15、16、17 都没有；PostgreSQL 18 官方文档和 pgPedia 的记录一致：默认 16，取值 1 到 262143，上下文 postmaster，也就是改完必须重启。

动手前先在目标实例上验一次，这句 SQL 是我这些年最省时间的习惯：

```sql
SELECT name, setting, unit, context, boot_val, min_val, max_val, pending_restart
FROM pg_settings WHERE name = 'autovacuum_worker_slots';
```

在 PG17 及更早版本上，这条语句返回 0 行——那就到此为止，别再往下调。返回 1 行，再看 context 是不是 postmaster，确认这台机器是 PG18 及以上，后面才有意义。

下面全部内容基于 PostgreSQL 18 的行为；写稿时手边没有 PG18 实例可连，数值和机制均取自 PG18 官方文档、官方提交说明与 pgPedia 参数记录，你落地前用上面那条 SQL 对一遍本机实际值。

---

**一、这个参数到底解决什么问题**

在 PG17 及以前，autovacuum_max_workers 是 postmaster 级参数，改它必须重启实例。生产库谁敢为一次调优去重启？于是多数集群的清理并发数，从建库那天起就锁死在默认的 3。

PG18 把"能起多少个 worker"这件事拆成了两层：

第一层是 autovacuum_worker_slots，在服务器启动时预留后端槽位，默认 16，取值范围 1 到 262143，上下文是 postmaster，改它必须重启。

第二层是 autovacuum_max_workers，默认 3，从 PG18 起变成 SIGHUP 级，pg_reload_conf() 就能在线改，但上限被 slots 卡死——设得比 slots 高，实例只会启动到 slots 数量的 worker，同时往日志里写一条 WARNING。

官方提交 c758119e5b 的原话是：引入 autovacuum_worker_slots 控制启动期预留多少槽位，改这个新值仍需重启，但它应该被设成你现实中可能需要的 autovacuum_max_workers 的上界；有了它，autovacuum_max_workers 就能靠 SIGHUP 生效了。

一句话记住：**槽位启动时定，并发在线调。**

这个套路在 PG 里不新：max_connections 对实际并发连接、max_wal_senders 对活跃流复制，都是"先预留上界，再按需使用"。autovacuum 走到这一步，算是补上了迟到十几年的课。

---

**二、为什么默认值是 16，而且可能不是 16**

官方文档写默认 16，但 16 不是天上掉下来的，是被内核信号量限制倒逼出来的。

c758119e5b 把 autovacuum 需要的信号量从 3 个一口气提到 16 个，结果一批 SEMMNS 默认值很低的系统直接起不来了。于是 initdb 增加了一段逻辑：像处理 max_connections、shared_buffers 那样，按需要自动下调 autovacuum_worker_slots，探测标准是 max_connections / 6 个槽位。max_connections 取默认 100 时，正好算出 16；取到最低的 25 时，算出 4——刚好还能不低于 autovacuum_max_workers 的默认值 3。

所以两个坑值得提前知道：

坑一，不同实例 SHOW 出来的值可能不一样。同样的 PG18 小版本，一台是 16，另一台是 4，多半是 initdb 阶段被内核参数压过，不是谁改坏了配置。

坑二，Linux 上 PG10 起用的是无名 POSIX 信号量，ipcs -s 什么也看不到；但在 NetBSD、OpenBSD 这类仍用 System V 信号量的平台上，SEMMNI 要至少达到 ceil(num_os_semaphores / 16)，SEMMNS 要至少是这个数乘 17。槽位不够的时候，启动阶段报的是：

```
FATAL: could not create semaphores: No space left on device
DETAIL: Failed system call was semget(...)
HINT: This error does *not* mean that you have run out of disk space.
```

这条 HINT 已经说明了一切——它真的在跟你说"不是磁盘满了"。在容器、老版本 BSD、精简内核的云主机上把 slots 往上调之前，先算一遍信号量够不够，别等重启失败再回滚。在 Linux 上，这个数可以通过 kernel.sem 这类内核参数放宽。

槽位本身很便宜：它只是共享内存里若干条目加一组信号量，日常多预留十几个的代价可以近似看成零。这也是后面"往宽了留"这条建议的底气。

---

**三、最容易踩的五个坑**

**坑一：把 slots 当成加速旋钮。**

slots 只决定上限，一个 worker 干活的快慢跟它毫无关系。真正决定速度的是触发阈值和成本限流。单纯把 slots 从 16 调到 32、max_workers 从 3 调到 12，然后发现清理还是慢——这是最常见的误操作。

**坑二：加了 worker 却没加成本预算，整体反而更慢。**

autovacuum 的成本预算是全局共享的。三个 worker 并发时每个分到三分之一的 limit，十二个就是十二分之一。默认 autovacuum_vacuum_cost_limit 是 -1，意思是用 vacuum_cost_limit 的值，也就是 200；autovacuum_vacuum_cost_delay 默认 -1，落到 worker 上有效值约 2ms。按命中脏页代价 20 算，一个 worker 刷掉大约 10 个页就得睡一觉。

所以正确的动作是三件套一起动：worker 数上去，autovacuum_vacuum_cost_limit 同步上调（比如 1000），再观察 log_autovacuum_min_duration 里的实际耗时。只做第一件，等于把一块饼切给更多人。

**坑三：忘了 slots 是启动期参数。**

ALTER SYSTEM SET autovacuum_worker_slots = 32 加 SELECT pg_reload_conf() 不会报错，但也不会生效，pending_restart 会变成 true。这类"静默不生效"在紧急故障里最坑人，因为你会以为已经扩容了。改完务必查一眼：

```sql
SELECT name, setting, pending_restart FROM pg_settings
WHERE name IN ('autovacuum_worker_slots','autovacuum_max_workers');
```

**坑四：拿 autovacuum_max_workers 去治单张大表的膨胀。**

一张表同一时刻只会被一个 autovacuum worker 处理。worker 从 3 加到 12，对"一张 2TB 的表清不完"没有任何帮助，只是让别的表也排不上了。这种场景该做的是表级 ALTER TABLE ... SET (autovacuum_vacuum_scale_factor = 0.01) 加独立的 cost 参数，或者干脆在维护窗口手工 VACUUM。

顺带说清另一件事：PG18 之前，大表要堆到 死元组数 > autovacuum_vacuum_threshold（默认 50）+ reltuples × autovacuum_vacuum_scale_factor（默认 0.1，即 10%）才会触发自动清理。十亿行的表意味着要积到约一亿条死元组才动手，这是设计上的硬伤。PG18 补了 autovacuum_vacuum_max_threshold（默认一亿条），给死元组数量上了硬顶，大表调优应该先盯它，而不是盯 worker 数。

**坑五：误以为 worker 多了就一定跑得动。**

清理会被两样东西挡住：一是长事务和老的快照，死元组对它们仍可见，扫了也删不掉；二是等待锁。truncating heap 阶段要拿 ACCESS EXCLUSIVE 锁，等的时候 wait_event 显示的是 VacuumTruncate，而且这个阶段是不可中断的。排查时把 pg_stat_progress_vacuum 和 pg_stat_activity、pg_blocking_pids 连起来看，比反复调参数有用得多。

**额外提醒一句 PG19 的新坑：**PG19 引入了 autovacuum_max_parallel_workers，让单个 autovacuum 可以用并行 worker 加速索引清理，但这些并行 worker 取自 max_parallel_workers 这个池子——和业务并行查询抢的是同一份资源。三个 autovacuum 各带四个并行 worker，就是十二个槽位被清理占掉。这个功能默认关闭，绝大多数集群保持关闭就是最优解，只对确实有几 TB 级、宽 jsonb 加 GIN 索引的表，按表级 autovacuum_parallel_workers = 2 精细打开。

---

**四、生产环境怎么配**

我的默认模板很简单：autovacuum_worker_slots 保持 16 不改，autovacuum_max_workers 保持 3 不改，先把可观测性和表级阈值做对。16 个槽位对绝大多数负载都够，用掉 3 个和预留 16 个的差距，在内存上根本看不见。

需要往上走的时候，按这个顺序判断。

第一种，多库或多租户实例：几百个 schema、几十个 database，launcher 按库轮转调度，worker 天然不够分。这时把 slots 提到 24 或 32，max_workers 先放到 6，观察一周再决定要不要继续加。

第二种，真正的大实例：内存 512GB 以上的机器，可以参照云数据库那套按内存比例放大的思路，上限落到 32 左右。再往上，收益会被 I/O 和成本预算吃干净。

第三种，小规格实例：1 到 2 核、4GB 内存的云主机，slots 保持默认甚至被 initdb 压到 4 都无所谓，真正要收紧的是 max_workers（2 就够）和 cost 参数，别让清理把仅有的 IOPS 抢光。

配套要一起做的四件事：

一是打开日志。log_autovacuum_min_duration = 1000（毫秒），没有这条，后面所有调优都是猜。

二是把成本预算调到匹配存储。SSD 上加 autovacuum_vacuum_cost_limit 到 1000 到 2000，delay 不用动；云盘、NFS、SAN 这类高延迟存储，优先调 io_method（默认 worker）和 io_workers（默认 3，大系统可以放 6 到 12），靠异步预读把等待时间盖掉。

三是把大表的触发阈值改成表级。全局 scale_factor 别动——调低会让一堆本来健康的小表跟着空转。只对热点表做 ALTER TABLE，scale_factor 降到 0.01 到 0.05，threshold 按实际写入量给个绝对值。

四是让冻结均匀发生。PG18 的 eager freezing 会把 freeze 工作摊到每次普通 vacuum 里，避免将来某个时刻集中爆发 I/O 风暴，前提是 autovacuum_max_freeze_age 别被谁改得太激进。

---

**五、改之前和改之后，各看什么**

改之前，先确认问题不是"worker 不够"而是"阈值没到"或"被锁挡住"。查一遍各表的死元组规模和最近一次 autovacuum 时间，n_dead_tup 高得离谱而 last_autovacuum 是空的，才是真的没排上。

改之后，用三个地方验证：pg_settings 确认参数真的生效（pending_restart 必须为 false）；日志里 autovacuum 行的 elapsed time 和 dead tuples 有没有变化；pg_stat_progress_vacuum 里这张表是不是被处理完了。PG18 还给这张视图加了 delay_time 字段，配合 track_cost_delay_timing 打开，能直接看到 worker 有多少时间花在"被限流睡觉"上——这是判断该加 worker 还是该加预算的唯一硬证据。

最后把顺序钉死：**先加观测，再改阈值，再调预算，最后才动并发。**slots 只是这条链上最后、也最不需要急于动手的一环。

---

字里行间，幸得你驻足品读。我是昌哥，欢迎关注与留言，共赴一场思想的碰撞。
技能提升|培训认证|数据救援|运维保障 找昌哥 微信：rscpass

-----------------------END-----------------------
各大平台都可以找到我：
————————————————————
公众号：昌哥知识星球
技术博客:http://www.linuxmysql.com
墨天轮：https://www.modb.pro/u/427810
CSDN ：https://blog.csdn.net/rscpass
51CTO: https://blog.51cto.com/u_16068254
博客园：https://home.cnblogs.com/u/rscpass
知乎:https://www.zhihu.com/people/shukuinfo
掘金：https://juejin.cn/user/2801995051703454
百家号：https://author.baidu.com/home/1780697309880431

作者：阮胜昌
拥有:MySQL8.0 OCP、Oracle OCP、TIDB PCTA/PCTP/PCSD、Kingbase KCP,软考中级数据库系统工程师、RHCE7.0等行业认证
擅长主流数据库MySQL、Oracle、PostgreSQL的备份恢复,SQL调优、监控运维、故障应急处理等
可提供的技术服务：
1.数据库故障处理/疑难杂症远程支援
2.MySQL/PG/Oracle/SQLSERVER数据库技术服务 

欢迎关注我的博客:http://www.linuxmysql.com 一起学习新知识！
技能提升|培训认证|数据救援|运维保障  微信：rscpass
