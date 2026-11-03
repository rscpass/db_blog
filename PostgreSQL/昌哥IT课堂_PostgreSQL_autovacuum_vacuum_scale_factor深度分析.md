---
title: "PostgreSQL autovacuum_vacuum_scale_factor 深度分析：从触发原理到生产级调优与避坑"
author: "阮胜昌"
date: "2026-11-03"
---

# PostgreSQL autovacuum_vacuum_scale_factor 深度分析：从触发原理到生产级调优与避坑

你好,我是昌哥,进免费技术交流群或咨询加微信: rscpass

## 一、开篇：一个被"20%"悄悄拖垮的数据库

做过 PostgreSQL 运维的同学，大概率都经历过这样的场景：业务跑得好好的，某天开始查询越来越慢、磁盘空间持续上涨，打开监控一看，某张大表的"死元组（dead tuples）"已经堆到了几千万甚至上亿。你手动跑一次 VACUUM，世界恢复了平静，可没过几天问题又卷土重来。

很多人第一反应是"autovacuum 是不是没开"，一查 `SHOW autovacuum;` 结果是 on。问题往往不在开关，而在**触发阈值**——更准确地说，是 `autovacuum_vacuum_scale_factor` 这个参数。

这篇文章，昌哥就带你把这个参数彻底讲透：它是什么、怎么算、怎么调、哪些坑绝不能踩，以及生产环境该怎么配。

## 二、先给结论：这个参数到底干什么的

PostgreSQL 使用 MVCC（多版本并发控制）机制。你每执行一次 UPDATE，数据库并不是原地修改那行数据，而是写入一个新版本，把旧版本标记为"死亡"；DELETE 也是只打一个删除标记。这些旧版本就是**死元组**。

这些死元组不会自动消失，它们会：

- 白白占用磁盘空间，造成**表膨胀（bloat）**；
- 让顺序扫描读进更多无用页面，拖慢查询；
- 撑大表对应的所有索引，导致索引也跟着膨胀；
- 让优化器的统计信息失真，生成糟糕的执行计划。

而负责回收这些死元组的"清洁工"，就是 autovacuum。它要不要在某个表上开工，核心就由一套触发公式决定，其中 `autovacuum_vacuum_scale_factor` 是关键的一员。

## 三、参数档案：官方定义的本来面目

先看一下这个参数的"官方身份证"：

- **类型**：real（浮点数）
- **默认值**：0.2，也就是表大小的 20%
- **取值范围**：0 到 100
- **生效上下文**：sighup，即修改后执行 `SELECT pg_reload_conf();` 或给 postmaster 发 SIGHUP 即可生效，不需要重启数据库
- **修改位置**：只能写在 `postgresql.conf` 或服务器命令行中
- **可被覆盖**：可以通过 ALTER TABLE 修改表的存储参数，为单张表单独设置，从而覆盖全局值

官方对它的描述是：**指定一个表尺寸的分数，在决定是否触发 VACUUM 时，将它加到 autovacuum_vacuum_threshold 上。**

## 四、核心原理：触发公式与那道"隐藏的门槛"

理解这个参数，必须把它放进完整的触发公式里看。PostgreSQL 判断是否要对某张表做 VACUUM，用的公式是：

> vacuum threshold = autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor × reltuples

其中：

- `autovacuum_vacuum_threshold` 是门槛的"地板值"，默认 50 个元组；
- `autovacuum_vacuum_scale_factor` 是比例因子，默认 0.2；
- `reltuples` 是这张表当前的估算行数。

也就是说，**自上一次 VACUUM 以来，被废弃的元组数量超过上面这个阈值，autovacuum 才会调度一次 VACUUM。**

举几个直观的例子，你就明白问题出在哪了：

**例一：一张 10 万行的小表**

10 万 × 0.2 = 2 万，加上地板值 50，约等于 2 万。积累约 2 万个死元组就触发。对一张小表来说，这个节奏还算合理。

**例二：一张 1000 万行的中等表**

1000 万 × 0.2 = 200 万。得先堆 200 万个死元组才会被清理。如果这是一张高频更新的订单表，200 万个死元组足以让查询明显变慢。

**例三：一张 10 亿行的超大表**

10 亿 × 0.2 = 2 亿。要在表里攒够 2 亿个死元组，autovacuum 才会动手。这个数量级的膨胀，往往意味着查询已经烂透了、磁盘已经告警了。

**这就是 `autovacuum_vacuum_scale_factor` 最大的问题：它让大表的清理门槛随表尺寸线性放大，表越大越来不及。** 一个"一刀切"的 20%，对小表宽松无害，对大表却是灾难性的懒惰。

另外补充一点 PostgreSQL 18 之后的变化：新版引入了 `autovacuum_vacuum_max_threshold` 作为上限，默认把触发门槛的上限卡在约 1 亿死元组，缓解了超大表"等到天荒地老"的问题。所以在 PG18 上，对绝大多数表可以少做一些表级调优；但在 PG17 及更早版本，针对大表的 scale_factor 调优是**刚需**。

## 五、调优建议：全局别乱动，表级精准调

### 建议一：保持全局默认值，把调优下沉到表级

这是昌哥最想强调的一点。**不要一上来就把全局的 scale_factor 从 0.2 改成 0.05。** 这样做会让所有表都更频繁地触发 VACUUM，包括那些几乎不写的小表、只读的维度表、历史归档表——它们的清理成本被白白放大了。

正确的姿势是：**找出写多、膨胀快、对性能敏感的那几张热点表，单独给它们设置。**

```sql
ALTER TABLE public.orders SET (
    autovacuum_vacuum_scale_factor = 0.02,
    autovacuum_vacuum_threshold    = 1000
);
```

这条语句做了两件事：

- 把比例从 20% 降到 2%，让清理更早介入；
- 把地板值从 50 提高到 1000，避免极小表的清理过于频繁。

两者必须**配套调整**。只把 scale_factor 调得很小、却不提高 threshold，结果就是小表被 VACUUM 反复打扰，徒增开销。

### 建议二：不同业务形态，给不同的力度

**高频更新的 OLTP 热表（订单表、账户余额表、库存表）**

建议把 scale_factor 设到 0.01 ~ 0.02，threshold 设到 1000 ~ 5000，让死元组在可控范围内就被清理掉。

```sql
ALTER TABLE public.accounts SET (
    autovacuum_vacuum_scale_factor = 0.01,
    autovacuum_vacuum_threshold    = 1000,
    autovacuum_analyze_scale_factor = 0.005,
    autovacuum_analyze_threshold    = 1000
);
```

**只追加写的日志/流水表（审计日志、操作日志）**

这类表的"废弃"主要是插入驱动的，应当使用 PG13 引入的插入相关参数：

```sql
ALTER TABLE public.audit_log SET (
    autovacuum_vacuum_insert_scale_factor = 0.05,
    autovacuum_vacuum_insert_threshold    = 1000
);
```

注意：纯 INSERT 表默认并不会轻易触发 VACUUM，只有开启并合理配置 insert threshold 才会生效。如果你的版本里没有 insert_scale_factor，说明版本较老，需要先评估升级。

**超大但写很少的历史/归档表**

这类表不需要激进清理，保持默认甚至略松即可，避免无意义地消耗 I/O。

### 建议三：调完别走，用数据验证

设置完之后，一定要回过头检查配置是否真的生效、是否真的起作用：

```sql
SELECT relname, reloptions
FROM pg_class
WHERE relname IN ('orders', 'accounts', 'audit_log');
```

这条查询会列出表上设置的存储参数。如果你在这里看不到你设置的值，说明要么没提交成功，要么你改错了对象。

### 建议四：清理得勤，也得给得起"体力"

很多团队只调低了 scale_factor，却忘了 autovacuum 是"干一会儿歇一会儿"的。触发得更频繁，但它每次能干多少活，由成本限制参数控制。

在 SSD/NVMe 这种现代存储上，默认的成本参数往往过于保守（它们当初是为机械盘校准的）。可以考虑：

- `autovacuum_vacuum_cost_limit`：默认 200，在 SSD 上可以提到 1000 ~ 2000，让 worker 每轮多干点活；
- `autovacuum_vacuum_cost_delay`：PG13 之后默认 2ms，PG12 及更早是 20ms，可以适当降到 1 ~ 2ms；
- `autovacuum_max_workers`：默认 3 个，如果你有成几十张高写表在互相排队，可以提到 5 ~ 6 个。

**但这里有个坑**：从 PostgreSQL 14 开始，每个 worker 各自拥有一份 cost_limit 预算；而在 PG14 之前，cost_limit 是所有 worker 共享的。所以在老版本上盲目加 worker 却不提高 cost_limit，结果是"每个 worker 都被限得更死"，越调越慢。加 worker 必须和提 cost_limit 一起做。

## 六、参数避坑指南：这些雷区千万别踩

### 坑一：VACUUM 在跑，死元组却纹丝不动

这是最迷惑人的场景。日志里 autovacuum 跑得很勤，可 `n_dead_tup` 就是不见下降。

原因几乎都不是参数没调好，而是**有东西挡住了最旧的快照（xmin horizon）**。VACUUM 只能回收那些"不可能再被任何事务看到"的死元组。以下任何一个都能让 horizon 被卡住：

- 一个忘了提交的"idle in transaction"会话；
- 一条跑了很久的分析 SQL；
- 一个已经不再使用、却还活着的复制槽（replication slot）；
- 一个长时间未提交的预备事务（prepared transaction）；
- 开启了 `hot_standby_feedback` 的备库上一条长查询。

这种情况下，你就是把 scale_factor 调到 0.001、把 cost_limit 调到无穷大，也回收不了一个死元组。**调参数之前，永远先排除"阻塞源"。**

预防手段是设置一个全局的超时，让"忘了提交"的会话被自动踢掉：

```sql
ALTER SYSTEM SET idle_in_transaction_session_timeout = '2min';
SELECT pg_reload_conf();
```

### 坑二：表里死元组堆成山，只因为有个长事务

一个跑了 5 分钟的事务，就能让整个数据库的回收停滞 5 分钟以上。所以生产环境一定要持续监控：

- `pg_stat_activity` 里 state 为 "idle in transaction" 且持续很久的会话；
- `pg_replication_slots` 里不再使用却还存在的复制槽，尤其是带 xmin 的槽。

这是 autovacuum 调优里最容易忽略、也最致命的一点。

### 坑三：只在全局下调 scale_factor，导致全库被过度清理

前面说过，全局改小会让所有表都更频繁地 VACUUM。VACUUM 不是免费的：它要扫表、要脏页、要写 WAL、还要清理每一个索引。一张有 12 个索引的表，在索引清理阶段的开销大约是单索引的 12 倍。对这样的表更频繁地触发 VACUUM，是有真实代价的，不是"越勤越好"。

**结论：全局默认 0.2 别动，热点表单独设置。**

### 坑四：忘了给更新频繁的表留 HOT 更新的空间

如果 UPDATE 没有修改任何被索引的列，PostgreSQL 可以做 HOT（Heap-Only Tuple）更新，完全不动索引，产生的索引膨胀极小，对频繁清理也更友好。

要做到这一点，可以给高更新表设置一个小于 100 的 `fillfactor`，比如 70 或 80，为页内更新留出空间：

```sql
ALTER TABLE public.accounts SET (fillfactor = 80);
```

配合 HOT 更新，你的 autovacuum 调优会事半功倍。判断 HOT 更新是否生效，看 `pg_stat_user_tables` 里的 `n_tup_hot_upd` 是否够高。

### 坑五：拿 VACUUM FULL 当"优化工具"用

很多人在网上看到"VACUUM FULL 能回收空间"就把它写进定时任务，这是很糟糕的做法。VACUUM FULL 会**全程持有 ACCESS EXCLUSIVE 锁**，期间整张表任何读写都做不了，而且它不压缩索引，甚至可能让索引更大。

**生产环境几乎不要用 VACUUM FULL。**

如果表已经严重膨胀、确实需要回收磁盘空间，请用 `pg_repack` 做在线重组，它不需要长时间持有排他锁。前提是表有主键或唯一非空索引，并且临时预留约一倍表大小的磁盘空间。

### 坑六：只看表膨胀，不看事务 ID 回卷风险

autovacuum 除了回收死元组，还有第二份生死攸关的工作——**冻结（freeze）旧元组，防止事务 ID 回卷**。

PostgreSQL 的事务 ID 是 32 位，用满约 21 亿后会回卷。如果没及时冻结，旧数据在事务视角下会"跑到未来去"，等同于数据损坏，PostgreSQL 会强制进入紧急模式甚至停库自保。

所以要持续监控 `age(relfrozenxid)`，建议对接近 `autovacuum_freeze_max_age`（默认 2 亿）一定比例的表提前告警，并单独给超大表安排维护窗口做手动 freeze。

## 七、生产环境最佳配置建议

下面给的是一套**可直接落地的分层方案**，不是让你照搬数字，而是照着这个思路去配。

### 第一层：全局参数（postgresql.conf）

核心原则——**保守、通用、留余量**：

- `autovacuum` 保持 on，绝对不要关；
- `autovacuum_vacuum_scale_factor` 保持默认 0.2，不要全局下调；
- `autovacuum_vacuum_threshold` 保持默认 50；
- `autovacuum_max_workers`：根据库里热点表数量设 4 ~ 6；
- `autovacuum_vacuum_cost_limit`：SSD 上设 1000 ~ 2000；
- `autovacuum_vacuum_cost_delay`：PG13+ 保持 2ms 或降到 1ms；
- `autovacuum_work_mem` / `maintenance_work_mem`：给足，尤其索引多的表；
- `idle_in_transaction_session_timeout`：必须设，建议 1 ~ 5 分钟；
- `log_autovacuum_min_duration`：调优初期设 0 记录每次运行，稳定后调大，避免日志爆炸。

改完 `sighup` 类参数记得执行 `SELECT pg_reload_conf();`，改 `autovacuum_max_workers` 则需要重启。

### 第二层：表级存储参数（ALTER TABLE）

先通过监控找出写量最大的前 10 ~ 20 张表（按 `n_tup_upd + n_tup_del` 排序），只对它们做定制：

- 高频更新 OLTP 表：`scale_factor = 0.01 ~ 0.02`，`threshold = 1000 ~ 5000`；
- 只追加的日志表：用 `autovacuum_vacuum_insert_scale_factor = 0.02 ~ 0.05`；
- 高索引表：谨慎调低，配合 HOT 更新与 fillfactor；
- 超大只读/归档表：保持默认或略松。

并且建议把这些 ALTER TABLE 写进建表的迁移脚本里，让参数随表一起发布，而不是出了问题才临时补。

### 第三层：监控与告警

光配不监控，等于没配。至少要盯住这几项：

- 各表死元组数量和死元组占比 `n_dead_tup / (n_live_tup + n_dead_tup)`，占比持续上涨超过 15% ~ 20% 就该告警；
- `last_autovacuum` 是否长时间未更新；
- `pg_stat_progress_vacuum`，看当前 autovacuum 卡在哪个阶段；
- `age(relfrozenxid)`，事务 ID 回卷风险要提前预警；
- 活跃 autovacuum worker 数与 `autovacuum_max_workers` 的比值，判断是否 worker 不足；
- `pg_replication_slots` 与 idle in transaction 会话。

每次调参后观察 24 ~ 72 小时，看死元组曲线是否被压下去、业务延迟是否改善，再决定下一步。**调优是一个持续迭代的过程，不是一锤子买卖。**

## 八、一张速查清单（按顺序做）

昌哥帮你把整个调优流程浓缩成几步，照着走就不容易乱：

第一步，确认 autovacuum 是开启的，确认目标表确实有死元组在堆积；

第二步，先查有没有长事务、idle in transaction 会话、废弃的复制槽挡住 xmin horizon，有就先清掉；

第三步，找出写量最大的热点表，只对它们做表级 ALTER TABLE，把 scale_factor 降到 0.01 ~ 0.02，同时把 threshold 提到 1000 以上；

第四步，根据磁盘类型调整 cost_limit 与 cost_delay，让 worker 干得动活；

第五步，如果热点表多、worker 在排队，再提高 `autovacuum_max_workers`，注意老版本要同步提高 cost_limit；

第六步，设置 idle_in_transaction_session_timeout 做预防；

第七步，建立死元组占比、last_autovacuum、事务 ID age 三套监控告警；

第八步，24 ~ 72 小时后回看曲线，再决定是否继续调整。

## 九、总结

`autovacuum_vacuum_scale_factor` 这个参数，说白了就是决定"这张表允许脏到什么程度才打扫"。PostgreSQL 默认的 20%，在十年前的小库时代是合理的；放到今天动辄几十亿行、秒级更新的业务里，就成了大表膨胀的隐形推手。

掌握它的关键只有三句话：

**一，理解触发公式，明白大表为什么总是来不及清理；**

**二，全局保守、表级精准，绝不一刀切；**

**三，调参之前先排除长事务与复制槽的阻塞，调参之后用监控验证效果。**

把这三句话记住，你就已经超过了绝大多数只会照抄参数模板的 DBA。

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
