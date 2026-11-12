# 昌哥IT课堂 | PostgreSQL autovacuum_work_mem 深度拆解：一个参数，决定你的大表会不会越清越慢

你好,我是昌哥,进免费技术交流群或咨询加微信: rscpass

## 一、先回答一个最基础的问题：这个参数到底存不存在

先说结论：**存在，而且从 PostgreSQL 9.4 一路活到了现在。**

很多 DBA 从 MySQL 转过来，找 PostgreSQL 的清理内存参数时容易犯迷糊：手册里翻到了 maintenance_work_mem，又听说 autovacuum 有独立开关，于是怀疑 autovacuum_work_mem 是某个版本的临时产物，甚至怀疑自己记混了。这里把硬信息一次给全：

- 引入版本：PostgreSQL 9.4，提交号 8693559cac
- 当前状态：PostgreSQL 18 正式版、19 开发版中依然存在，未被废弃
- 数据类型：integer，单位 kB（不写单位就按 kB 算）
- 取值范围：-1 到 2147483647
- 默认值：-1，含义是"改用 maintenance_work_mem 的值"，不是"负数内存"
- 上下文：sighup，改完 pg_ctl reload 或 SELECT pg_reload_conf(); 即生效，**不需要重启**
- 设置位置：只能在 postgresql.conf 或服务器命令行设置，会话级 SET 对它无效
- 分类：资源消耗 / 内存

在 9.4 到 19 这一长串版本里，它的默认值一次都没变过，一直是 -1。真正发生变化的是它背后的算法——这一点后面单独讲，因为它是本文最值钱的部分：**同一个参数值，在 PG 16 和 PG 17 上意义完全不同。**

想在自己的库上三十秒确认，直接查：

SELECT name, setting, unit, source, context FROM pg_settings WHERE name = 'autovacuum_work_mem';

如果 source 显示 default 且 setting 是 -1，说明你还在继承 maintenance_work_mem——也就是本文要聊的所有坑，你一个都没躲开。

## 二、它管的到底是什么内存：死元组地址清单

要理解这个参数，得先明白 VACUUM 是怎么干活的。

PostgreSQL 的 MVCC 决定了 UPDATE 和 DELETE 不会真的把旧行抹掉，只是把它们标记成"对任何事务都不可见"的死元组。VACUUM 的工作分两步：

第一步，扫堆表（heap），把所有死元组的身份标识（TID，tuple identifier）收集成一份清单，攒在内存里。

第二步，把表上的每一个索引都完整过一遍，凡是索引条目指向清单里的 TID，就顺手删掉。

关键点就在这里：**这份 TID 清单必须装得下内存。** 一个 TID 占 6 个字节，如果清单在堆扫描结束前就把内存撑满了，VACUUM 只能中断当前流程——先把手上的清单拿去把所有索引清一遍，然后清空清单，回到堆表中断的位置继续扫，再攒满、再清一遍索引……循环往复。

于是就有了那个著名的诊断信号：VACUUM 日志里的 index scans: N。

开一个 VACUUM VERBOSE 跑在目标表上，看结尾那行 INFO。如果 index scans: 1，说明一份内存吃下了全部死元组，索引只过了一遍，这是健康状态。如果显示 index scans: 4，意思就是这张表的每个索引被从头到尾读了四遍。一张六个索引、索引合计 200GB 的表，四轮就是 800GB 的读量，而不是 200GB。这才是"清理慢"背后真正的成本，而且它全部落在 I/O 上，表现出来的症状是 autovacuum 跑几个小时不结束、磁盘持续高位、业务查询跟着变慢。

autovacuum_work_mem 划定的，就是这份清单能用多少内存。理解了这一点，后面所有的调参建议和避坑点都能自己推出来。

## 三、为什么非要把它从 maintenance_work_mem 里拆出来

既然默认是 -1、继承 maintenance_work_mem，那这个参数存在的意义是什么？

因为**这两个内存的并发模型完全不是一回事。**

maintenance_work_mem 服务的是人工发起的维护操作：CREATE INDEX、REINDEX、VACUUM、ALTER TABLE 重写、ADD FOREIGN KEY、CLUSTER。这类操作的隐含假设是"同一时刻不会有几个"，所以手册明确说把它设得比 work_mem 高很多是安全的。你为了一次 REINDEX 跑得快，把它调到 4GB，逻辑上没毛病。

但 autovacuum 不吃这套假设。autovacuum launcher 会按 autovacuum_max_workers（默认 3）拉起多个 worker，**每个 worker 各自独立地申请一份 autovacuum_work_mem**。于是那个被无数生产事故验证过的算式出现了：

峰值内存 = autovacuum_work_mem × autovacuum_max_workers

按默认继承算：maintenance_work_mem = 1GB、workers = 3，光是后台自动清理就可能吃掉 3GB，这还没算任何一条人工 REINDEX。有人只是想让建索引快一点，把 maintenance_work_mem 抬到 4GB，结果等于悄悄给每个 autovacuum worker 也发了 4GB 的额度，12GB 就这么被背景进程预定了。等到业务高峰内存吃紧、OOM killer 开始挑进程下手的时候，没人会想到根源是一次建索引调优。

这就是 autovacuum_work_mem 的设计初衷：**把前台一次性维护的额度，和后台常驻进程的额度分开管。** 前台该大方大方，后台该收紧收紧，两条线互不干扰。

顺便把另外两个容易混的内存参数也钉一下，避免张冠李戴：

work_mem 是单个查询里单个排序/哈希节点用的，而且会乘上会话数、算子个数、并行 worker 数——500 个连接配 256MB work_mem，理论峰值能到 125GB，这是另一个经典的 OOM 现场，但和本文主题无关。

maintenance_work_mem 是单条维护操作独占的额度，并行建索引的 worker 之间是**分摊**这份额度，不是各拿一份，所以并行度上升不会放大这块内存。

autovacuum_work_mem 是唯一一个会**随 worker 数量线性放大**的维护类内存。记住这条区别，你就抓住了 PostgreSQL 内存规划的主线。

## 四、PG 17 分水岭：同一个参数，两套玩法

这是本文最重要的部分，也是网上大部分中文资料已经过时、照着配会出事的地方。

**PostgreSQL 16 及更早：扁平数组，1GB 天花板**

早期实现里，TID 清单就是一个扁平数组，元素个数用 32 位计数，于是产生了两道硬限制：

一是无论你把 autovacuum_work_mem 调到多大，实际生效上限就是 1GB；二是 1GB 大约只够装 1.79 亿个死元组的 TID。

这意味着：如果某张表在两次清理之间积累的死元组超过 1.79 亿，多轮索引扫描就是**必然发生**的，调内存救不回来，唯一的解法是把这张表的触发阈值调低，让它清得更勤、每次攒得少。

**PostgreSQL 17 起：TIDStore + 自适应基数树**

PG 17 用基于自适应基数树（Adaptive Radix Tree）的 TIDStore 替换了扁平数组。变化是根本性的：1GB 上限消失，同样一份内存能装下多得多的 TID，大多数负载下多轮索引清理从"常见"变成了"罕见"。

所以调参建议要按版本分开说，别再拿一份配置通吃：

PG 16 及更早：生产上重更新的大表，显式设 autovacuum_work_mem = 1GB 是合理的，这是你的天花板；但对那些两次清理之间死元组会突破 1.5 亿的表，正解是调表的 autovacuum_vacuum_scale_factor，不是继续加内存。

PG 17 及以后：保持 -1 让它在多数场景下继承就够了，基数树本来就用得省。真正需要单独设低值的场景是——你把 autovacuum_max_workers 调到了 6 甚至更高，这时该考虑给 autovacuum_work_mem 一个低于 maintenance_work_mem 的明确值，把总内存封住，把省下来的空间留给前台操作。

一句话记住：**16 及以前是"内存不够导致多轮"，17 之后是"多轮发生了就该去查为什么"。**

## 五、怎么算出你该给多少：一个能当场用的公式

不用拍脑袋，也不用照抄别人的数值。

第一步，找出你最大的那张"热表"在两次清理之间能攒多少死元组：

SELECT relname, n_live_tup, n_dead_tup,
round(n_dead_tup::numeric / GREATEST(n_live_tup, 1) * 100, 2) AS dead_pct,
last_autovacuum, autovacuum_count
FROM pg_stat_user_tables
WHERE n_live_tup > 0
ORDER BY n_dead_tup DESC LIMIT 20;

第二步，套 AWS RDS 官方文档给的那个极简估算：**死元组数 × 6 字节 = 单轮清完所需内存**。这条公式的权威性很足，RDS 的自动清理内存调优就是按它给的：

SELECT relname AS table_name, n_dead_tup, pg_size_pretty(n_dead_tup * 6) AS estimated_memory
FROM pg_stat_all_tables
ORDER BY n_dead_tup DESC LIMIT 10;

第三步，用总内存预算反推。先决定你愿意让 autovacuum 总共吃多少——经验上是物理内存的 10% 到 20%，再除以 worker 数：

单 worker 额度 ≈ （物理内存 × autovacuum 预算占比）÷ autovacuum_max_workers

举个例子，64GB 内存、预算 12%（约 7.7GB）、workers = 4，那么单 worker 大约 1.9GB，取 2GB 是舒服的。但如果同样这台机器你只给 autovacuum 留 5%（约 3.2GB）而 workers 开到 8，单 worker 就只剩 400MB——这时候该做的是把 workers 降下来，而不是硬着头皮把单 worker 压到不够用。

第四步，用真实进程反证。改完 reload 之后，直接看 worker 的实际驻留内存：

ps -o pid,rss,cmd -C postgres | grep autovacuum

改前改后各采一次，比任何理论推算都直观。

还有一条分配原则来自 RDS 的官方建议，值得抄进你的运维手册：**小表多、大表少的库，倾向多 worker、少内存；有 100GB 以上大表的库，倾向少 worker、多内存。** 因为大表真正卡住的是单轮能不能装下，小表卡的是排队等不到 worker。

## 六、避坑清单：这八个坑我见人踩过

坑一：以为它是会话级参数，SET 完发现没效果。

它只能在 postgresql.conf 或服务器命令行设置，上下文是 sighup。ALTER SYSTEM 写进去再 SELECT pg_reload_conf(); 是对的，SET autovacuum_work_mem = '1GB' 只对当前会话里手工执行的 VACUUM 有意义，对后台 worker 不起作用。

坑二：改完忘记 reload，或者只 reload 了一个节点。

主从、Patroni 集群、读写分离环境下，postgresql.conf 是各管各的。主库改了备库没改，备库上因为 hot_standby_feedback 引发的那类清理压力就完全没被覆盖。改完务必在每台机器上查一遍 pg_settings 的 source 列，确认显示的是 configuration file。

坑三：单位写成裸数字，结果小了 1024 倍。

不写单位时按 kB 计算。autovacuum_work_mem = 1 不是 1GB，是 1MB。永远写成带单位的 1GB、512MB，这是零成本的保险。

坑四：设成 0 试图"关掉"。

0 不是"关闭"，而是非法/无意义取值（PG 17 起最小值放宽到 64kB，但 0 依然不是你的朋友）。想关掉自动清理用的是 autovacuum_enabled = off，那是表级参数，而且除非你非常清楚自己在做什么，否则别碰。

坑五：只加内存，不调触发阈值。

这是最普遍的一类误配置。一张十亿行的表用默认的 autovacuum_vacuum_scale_factor = 0.2，意味着要攒到两亿条死元组才触发清理——你把内存加到 1GB 依然清不动，因为问题不在单轮容量，而在触发太晚。顺序必须是：**先调阈值让清理更勤，再看单轮内存够不够。** 反过来做，内存加了、I/O 更猛了、死元组照样堆。

坑六：把 maintenance_work_mem 抬高的收益，误以为对 autovacuum 也成立。

前面提过那个实测：200 万行的表建索引，maintenance_work_mem 从 4MB 到 64MB 到 1GB，耗时分别是 874ms、1004ms、946ms——差异全在跑动噪声范围内，建索引根本不吃这个内存。但 VACUUM 吃，而且是刚需。所以把两者拆开设，让 maintenance_work_mem 服务于偶尔的人工操作、autovacuum_work_mem 服务于常驻后台，是唯一合理的姿势。

坑七：算总账时漏掉了人工维护并发。

峰值不是 workers × autovacuum_work_mem 就完事了。如果此刻还有人在跑 REINDEX、还有一次部署在并发建两个索引，实际占用要往上叠。粗算公式是：

峰值 ≈ shared_buffers + autovacuum_max_workers × autovacuum_work_mem + 并发人工维护数 × maintenance_work_mem + 连接数 × work_mem + 操作系统与 page cache 预留

把这几项加一遍，很多人会当场发现自己那台 128GB 的机器其实早就超卖了。

坑八：只看单 worker 够不够，不看总账会不会 OOM。

内存太小的症状很明确：autovacuum 耗时极长、磁盘 I/O 明显抬高、index scans 大于 1。内存太大的症状更凶险：多个 worker 同时吃满，物理内存迅速见底，开始 swap 甚至触发 OOM killer，整个实例连坐。调参的方向永远是**两头夹**：大到避免多轮，小到总量安全，中间那点空间就是用 autovacuum_max_workers 去切的。

## 七、生产环境推荐配置：按机器规格分档

下面是可直接落地的起步值，单位是带单位的明确值，不是 -1。前提是 autovacuum 保持开启（默认就是开的，别关）。

第一档：8GB 到 16GB 内存，中小 OLTP，表数量多、单表不大

autovacuum_max_workers = 3
autovacuum_work_mem = 256MB
maintenance_work_mem = 512MB
autovacuum_naptime = 30s
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = 500

思路：worker 不多、单 worker 给到 256MB 足够覆盖千万级死元组，总账控制在 1GB 以内，别去碰物理内存的红线。

第二档：32GB 到 64GB 内存，混合负载，有几张大热表

autovacuum_max_workers = 4
autovacuum_work_mem = 1GB
maintenance_work_mem = 2GB
autovacuum_naptime = 20s
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = 1000

思路：这是最主流的一档。1GB 正好顶到 PG 16 及以前的有效天花板，多轮索引扫描的概率压到很低；四个 worker 合计 4GB，在 64GB 上完全可控。PG 17 及以后如果嫌 1GB 浪费，可以降到 512MB 观察 index scans 是否仍为 1。

第三档：128GB 以上，大表密集、更新频繁，或有批处理/ETL 冲击

autovacuum_max_workers = 6
autovacuum_work_mem = 2GB
maintenance_work_mem = 4GB
autovacuum_naptime = 15s
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = 2000

思路：worker 上到 6，就必须给 autovacuum_work_mem 一个明确值把它封住——这时候**绝不能让它继承 4GB 的 maintenance_work_mem**，6 × 4GB = 24GB 纯后台开销，这个代价没人付得起。2GB × 6 = 12GB，在 128GB 上占比不到 10%，安全。

三条跨档位的通用纪律：

一，表级覆盖优先于全局加码。真正热的那一两张表，用 ALTER TABLE 单独给更激进的 scale_factor 和 threshold，别为了它们把全局参数推高，让几百张冷表跟着陪跑：

ALTER TABLE orders SET (autovacuum_vacuum_scale_factor = 0.03, autovacuum_vacuum_threshold = 1000);

二，cost_delay 从默认 20ms 降到 2ms 这一手，通常比加内存更能改善"清理追不上写入"。它是节流阀，不是内存旋钮，别混为一谈。

三，大版本升级时把这一项重新评一次。从 16 升到 17，TIDStore 换来的是同样的活只要更少的内存，原来的 1GB 很可能可以往下收，收下来的部分还给业务。

## 八、改完之后怎么验证：三个信号

第一个信号，看日志里的 index scans。开 log_autovacuum_min_duration = 0 跑一个维护窗口，抓 VACUUM 的 INFO 行，index scans: 1 就是钱花对了；大于 1 就说明内存还是不够，或者该去调阈值了。

第二个信号，看 pg_stat_progress_vacuum。长清理跑着的时候查这张视图，能直接看到当前阶段和进度。PG 17 上如果还频繁出现多轮索引清理，那属于反常现象，值得单独排查——基数树时代它本该罕见。

第三个信号，看死元组走势和 worker 的实际 RSS。调优后正常情况下死元组占比应稳定在个位数以内，last_autovacuum 在合理间隔内持续刷新；同时用前面那条 ps 命令确认 worker 的驻留内存确实跟着配置走了。

顺序上也提一句：先改触发阈值类参数观察一周死元组走势，再动 cost_delay 和 cost_limit 观察单次清理时长，最后才调 worker 数和内存看整体负载。反着来的结果是每次改动的效果都分不清是谁带来的。

## 九、收尾

autovacuum_work_mem 是 PostgreSQL 里少有的"改起来成本极低、收益却能直接看见"的参数：一条 ALTER SYSTEM、一次 reload、不用重启、不用改表结构。但它也是最容易改错方向的一个——因为它真正的杠杆不在内存本身，而在于你会不会**同时**算清三件事：单轮清单装不装得下、worker 数量会不会把总账推过红线、以及这张表是不是根本该靠调阈值而不是加内存来解决。

再叠上 PG 17 那次底层换血，同一个数值在两个大版本上意义已经不同。所以别抄配置，按公式算、按日志验、按版本分档，这三个动作做完，你手上的 autovacuum 就不会再成为半夜告警的来源。

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
