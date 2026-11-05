# 昌哥IT课堂｜PostgreSQL autovacuum_vacuum_threshold 参数深度分析：从触发公式到生产落地

你好,我是昌哥,进免费技术交流群或咨询加微信: rscpass

## 一、先说结论

`autovacuum_vacuum_threshold` 这个参数，从 PostgreSQL 8.1 引入，一直活到今天的主干版本，默认值从 9.0 到现在（含 18.x、19）始终是 **50**，一次都没变过。它小得不起眼，却在很长一段时间里被当成"autovacuum 不干活"的替罪羊，也被当成"调小它就完事了"的万能药。两种理解都是错的。

真正的事实是：它是触发公式里的**加数项**，不是主项。压垮大表的从来不是这 50，而是跟它配对的那个比例因子。你要是只盯着它调，多半白忙一场，还可能把小表拖进频繁清理的坑里。

这篇文章把它的机制、坑、以及生产环境怎么配，一次讲透。

## 二、它到底是什么

手册原文的定义很朴素：指定能在一个表上触发 VACUUM 的、被更新或被删除元组的**最小数量**。默认 50 个元组，只能在 postgresql.conf 或服务器命令行上设置，但单个表可以通过修改表存储参数（storage parameter）来覆盖。

三个属性要记牢：

**第一，它是整数，下限 0，上限 2147483647。** 设成 0 是合法的，意思是"完全按比例因子说话"，对这个表不再有任何保底门槛。

**第二，它是 sighup，不需要重启。** 改完 `pg_ctl reload` 或者 `SELECT pg_reload_conf();` 就生效。但注意一个细节：已经在跑的 autovacuum worker 不会被追溯改变，新值只影响下一轮调度判断。所以你 reload 完立刻去看，`pg_stat_user_tables` 里没什么变化是正常的，别急着再改一遍。

**第三，它能被表级参数覆盖。** 这一条是全部调优价值的落点，后面第五节会展开。

## 三、把它放回公式里看

单独看 50 没有意义，它必须放进触发条件里：

死元组数 > autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor × reltuples

`reltuples` 是 pg_class 里记录的估算行数，不是实时 count。scale_factor 默认 0.2，也就是表尺寸的 20%。

代入算几个数你就有感觉了：

一张 1 万行的配置表，阈值 = 50 + 0.2 × 10000 = 2050。要积累两千多个死元组才清一次，对这种小表来说偏松，但它数据量小，清一次也快，问题不大。

一张 1000 万行的订单表，阈值 = 50 + 0.2 × 10000000 = 200 万。得先堆出 200 万个死元组，autovacuum 才肯动手。等它跑完，索引已经胖了一圈，顺序扫描多读了不少空页，执行计划也可能已经跑偏。

一张 10 亿行的流水表，阈值是 2 亿。这个数字基本等于"永远不清理"。

看出门道了吗？**在小表上，是那个 50 在说话；在大表上，是 50 在陪跑。** 20% × reltuples 一项动辄几十万上百万，加不加这 50 完全没有区别。所以网上那些"把 autovacuum_vacuum_threshold 从 50 调到 100 解决表膨胀"的说法，逻辑上就不成立——你把加数从 50 改成 100，对一张千万行表的影响是百万分之五，测都测不出来。

## 四、PG 18 带来的变化：多了一个天花板

从 PostgreSQL 18 开始，公式外面套了一层上限：

vacuum_threshold = Min(autovacuum_vacuum_max_threshold, autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor × reltuples)

`autovacuum_vacuum_max_threshold` 默认 **100,000,000**（1 亿），设为 -1 表示不设上限，退回到老行为。它同样是 sighup，同样支持表级覆盖。

这个参数的实际意义是：**超大表上的"永远不清理"被堵住了。** 不管你的表有多大，死元组堆到 1 亿就一定会触发。它相当于给 scale_factor 的线性膨胀装了个断路器。

但别高兴太早，1 亿个死元组本身就是很可观的膨胀量了。一张 5 亿行、更新频繁的 OLTP 核心表，你未必能接受它在 1 亿死元组的状态下跑业务。所以 PG 18 只是把最坏情况兜住了，该做的表级调优一点没少。我的判断标准是：**如果 1 亿这个数字对你某张表来说仍然太宽松，那这张表就必须单独配。**

顺带提一句，PG 18 还有个 `autovacuum_worker_slots`（默认 16，启动时预留槽位），把原来必须重启才能改的 `autovacuum_max_workers` 解耦出来，允许运行时动态调整并发 worker 数。调优并发的时候要**先动 slots 再动 max_workers**，顺序反了改了不生效。

## 五、五类表，五种配法

全局参数只有一套，表却有千差万别。真正专业的做法是把全局留保守，把力气花在表级覆盖上。

**第一类，高频更新的中大型热表**（订单、会话、任务队列、状态机）。这是唯一值得重点投入的对象。推荐把比例因子压到 1%~2%，同时把 threshold 抬起来当"防抖"用：

ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.02,
  autovacuum_vacuum_threshold = 1000,
  autovacuum_analyze_scale_factor = 0.01,
  autovacuum_analyze_threshold = 200
);

注意这个组合的逻辑：比例因子压小了，阈值就必须抬起来，否则表稍微动一动就触发一次清理，等于给自己制造 IO 风暴。1000 万行的表按这套配，触发点在 50 万 + 1000，比默认的 200 万紧了四倍，但又不至于过敏。

**第二类，超大表（数亿行以上）。** 思路可以反过来：比例因子给很小甚至给 0，完全用绝对数值说话，这样触发点可预测、不随表增长漂移：

ALTER TABLE big_events SET (
  autovacuum_vacuum_scale_factor = 0,
  autovacuum_vacuum_threshold = 50000
);

这种配法的代价是表长到一定程度后清理频率会相对变高，要配合 cost_limit 一起看。

**第三类，小表（几千行以内）。** 别动。默认 50 + 20% 对小表基本就是"50 说了算"，已经很灵敏了。你把它调更小，换来的是 autovacuum 每几十秒扫一遍这张表，纯浪费 IO。

**第四类，只插不改的表**（日志、流水、审计）。它的麻烦不是死元组，是**清理一直不触发**。因为死元组永远是 0，公式永远不成立，表就一直不 vacuum，冻结进度也一直不往前走，直到 `autovacuum_freeze_max_age`（默认 2 亿）把它强制拖去清理，一次性干一大票活。从 13 开始这类表要用插入侧的参数：

ALTER TABLE access_log SET (
  autovacuum_vacuum_insert_scale_factor = 0.02,
  autovacuum_vacuum_insert_threshold = 5000
);

**第五类，几乎不动的静态归档表。** 反过来，把全局那套相对激进的值在这里放松，把 IO 预算省给热表。

## 六、避坑：八个真实翻车点

**坑一：只调 threshold 不调 scale_factor。** 前面算过了，大表上完全是无效功。这是最高频的误操作。

**坑二：把全局 scale_factor 一刀切调小。** 0.2 改 0.02 听起来很美，但它是**对所有库所有表**生效的，包括那些几千行、一整天不动一张的配置表。autovacuum launcher 每轮都要扫一遍候选表，全库跟着空转。正确顺序永远是：先找出写压力最大的那十几张表，只给它们做表级覆盖。

**坑三：忘了 HOT 更新率。** 如果更新不涉及索引列，PostgreSQL 会走 HOT（Heap-Only Tuple）更新，完全不碰索引。HOT 比例高的表，你敢把触发点压得很低；HOT 比例低、索引又多的表（十二个索引就意味着索引清理阶段大概十二倍的开销），压低触发点是在给自己加负担。动手前先看 `pg_stat_user_tables` 里的 `n_tup_hot_upd`，这个数不看你就是在盲调。

**坑四：阈值调到很低，清理频率爆炸。** autovacuum 不是免费的，它读页、脏页、产 WAL、跟业务抢 IO。触发点压得太狠，会出现"上一轮还没跑完下一轮又排上"的局面，worker 队列越排越长。

**坑五：以为调了阈值就能清干净。** 阈值只决定"什么时候启动"，不决定"能清掉多少"。如果 `n_dead_tup` 在 autovacuum 跑完之后不降，问题根本不在参数上，而在有东西压着回收位点：

**长事务和 idle in transaction**。PG 用全局最老活跃事务的 xmin 画一条线，比它新的死元组一律不能碰。一个挂着不动的 `BEGIN` 就能让整个实例的清理全部失效。查法：

SELECT pid, state, now() - xact_start AS xact_age, age(backend_xmin) AS xmin_age, left(query, 80)
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL OR state = 'idle in transaction'
ORDER BY xact_start NULLS LAST;

**悬挂的复制槽**。这个最阴。备库挂了、CDC 连接器停了，槽还在，`hot_standby_feedback = on` 时它会一直顶着 xmin，WAL 也一起堆着不放。查 `pg_replication_slots` 里 `active = false` 的行。

**遗留的两阶段提交**。查 `pg_prepared_xacts`，正常应该是空表。有残留几分钟以上的，多半是某个 XA/JTA 事务管理器忘了收尾，它可以永久卡住清理。

**坑六：只看 n_dead_tup = 0 就放心。** 这个计数是估算值，靠统计采集和 autovacuum 运行来刷新。读到 0 可能只是"很久没更新这个计数器了"，不代表真的没死元组。判断要看趋势，不看单点。

**坑七：用 VACUUM FULL 救急。** 它重写整张表、全程持有 ACCESS EXCLUSIVE 锁，几百 GB 的表能锁上几个小时。生产环境要压缩空间用 pg_repack，或者低峰期分段做。另外 PG 18 起有全局 `vacuum_truncate` 开关，关掉它可以避免尾部截断那一下排他锁打到只读备库上造成查询卡顿。

**坑八：为了省 IO 把 autovacuum 关掉。** 这是通往"database is not accepting commands to avoid wraparound"的直达路径——整库进保护性只读，只能单用户模式救火。宁可降低频率、收紧表级触发，也永远不要在生产库上 `autovacuum = off`。

## 七、配套参数：阈值只是调度，速度是另一件事

阈值决定"什么时候开工"，cost 参数决定"干多快"。两者不配套，调了阈值也是白调——你会发现清理是按时启动了，但一张表跑两小时还没跑完，下一轮又排上了。

代价模型的大致口径：读一个页记 1 点代价，弄脏一个页记 20 点，每条索引条目另有开销。累计超过 `autovacuum_vacuum_cost_limit`（默认 200）就睡 `autovacuum_vacuum_cost_delay`（PG 13 起默认 2ms，更早版本是 20ms）。这套默认值是按机械盘标定的，你的库多半跑在 SSD 甚至 NVMe 上。

SSD 环境的参考起点：cost_limit 给到 1000~2000，cost_delay 给 1~2ms；NVMe 上有人直接给 0（完全不限流），但这个要看存储能不能扛住，共享存储和 IOPS 本来就紧的云盘不建议。有个实测参照：一张 100GB 的表，默认参数下 autovacuum 跑完要两个多小时，cost_delay 调到 0 之后 20 分钟搞定。

再往上是并发和内存：

`autovacuum_max_workers` 默认 3，多表高并发场景常常不够，可以提到 4~6。但有个坑必须知道：**cost_limit 是在 worker 之间共享的预算**（PG 14 起每个 worker 有自己的额度，具体行为随版本变），只加 worker 不加 cost_limit，结果是每个 worker 被限得更狠，越加越慢。

`autovacuum_naptime` 默认 1 分钟，意思是 launcher 一分钟才看一轮。写压力大的库可以降到 15~30 秒，响应会明显快。但同样地，只降 naptime 不提 cost_limit，结果只是"把同样跑得慢的清理启动得更频繁"。

`autovacuum_work_mem` 默认 -1，表示继承 `maintenance_work_mem`。这个值太小的话，大表索引清理要扫好多遍。给到 512MB~1GB 是比较常见的起点；worker 数调上去的时候，记得显式把它算清楚，别让总内存随 worker 数失控。PG 17 已经去掉了这块内存 1GB 的上限。

还有一个纯收益的开关：`log_autovacuum_min_duration = 1000`（甚至设 0 全记）。有了它，哪张表清了多久、被什么延迟了，PG 18 还会在 VERBOSE 输出里直接告诉你延迟原因和时长，调优就从猜变成看了。PG 18 的 `pg_stat_all_tables` 也新增了 total_autovacuum_time 之类的累计计数器，可以直接对比调优前后的总耗时。

## 八、生产环境推荐配置

前提是全局保持克制，攻击性只放在表级。下面这套适配 32GB 内存、8 核、SSD、以 OLTP 为主的常见规格，你可以按自己机器等比缩放：

autovacuum = on
autovacuum_naptime = 30s
autovacuum_max_workers = 4
autovacuum_work_mem = 512MB
autovacuum_vacuum_threshold = 100
autovacuum_vacuum_scale_factor = 0.1
autovacuum_analyze_threshold = 200
autovacuum_analyze_scale_factor = 0.05
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = 2000
log_autovacuum_min_duration = 1000
track_cost_delay_timing = on

PG 18 及以上，额外加一条，给超大表装断路器：

autovacuum_vacuum_max_threshold = 50000000

如果要用 6 个 worker，PG 18 上记得先 `autovacuum_worker_slots = 8` 再调 max_workers。

全局只做温和优化的理由很简单：**它影響的是你不认识、也没精力逐个看的那几百张表。** 真正的热表，用第五节的表级覆盖去打。

落地节奏建议分三步走。第一步，先把长事务、悬挂复制槽、遗留 prepared xact 这三个"压制源"清掉，这一步通常就能让一批表的死元组自己降下来。第二步，按死元组绝对数和死元组比例两个维度排出前 20 张表，逐个做表级覆盖。第三步，观察一周，看 `last_autovacuum` 是否在推进、`n_dead_tup` 是否不再单调上涨、业务高峰有没有 IO 排队，再决定要不要动 cost 参数。

监控上至少要挂三个告警：任意库 `age(datfrozenxid)` 超过 8 亿（离 20 亿的强制线还有余量）；核心表死元组比例持续高于 10%；以及表体积的周环比异常增长。

## 九、一句话总结

`autovacuum_vacuum_threshold` 是触发公式的**保底项**，它的价值在于保护小表不被频繁清理，而不是拯救大表。大表的问题在 scale_factor，超大表的问题在 PG 18 之前的公式本身没有上限。调优的顺序永远是：先清压制源，再做表级覆盖，最后才动全局和 cost 参数。把顺序搞反了，改一百遍 threshold 也治不好膨胀。

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
