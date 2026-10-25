你好,我是昌哥,进免费技术交流群或咨询加微信: rscpass

## 前言：一个只有 PG13 之后才有的参数

先说结论，免得你白折腾：这个参数叫 `autovacuum_vacuum_insert_scale_factor`，**PostgreSQL 13 才加进来**（提交 b07642db），默认值是 0.2。所以如果你还在跑 PG 12 及更早版本，这个参数在你那里根本不存在，查 `SHOW` 会报 unrecognized configuration parameter，别怀疑是自己拼错了。

PG 13 还顺手配了一个搭档参数 `autovacuum_vacuum_insert_threshold`，默认 1000。这两个参数是成对的，后面会反复同时出现。

那为什么 PG 13 之前没有它，PG 13 之后又非加不可？这得从 autovacuum 的触发逻辑说起。

## 一、参数到底是什么：触发公式拆解

`autovacuum_vacuum_insert_scale_factor` 决定的是**因为插入而触发 autovacuum 的门槛**——它指定要把表大小的多少比例，加到 `autovacuum_vacuum_insert_threshold` 上，用来判断是否该对这个表跑一次 VACUUM。

具体算法是：

```
vacuum insert threshold = autovacuum_vacuum_insert_threshold
                       + autovacuum_vacuum_insert_scale_factor × reltuples
```

当 `pg_stat_all_tables.n_ins_since_vacuum`（上次成功 VACUUM 之后新插入的行数）**超过**上面这个阈值时，autovacuum 就把它排进清理队列。

几个关键点：

**它只数插入，不数更新和删除。** 更新和删除走的是另一套老公式，由 `autovacuum_vacuum_scale_factor` 和 `autovacuum_vacuum_threshold` 管。这套 insert 公式是完全平行的第三条腿。所以你要理解它，得把它和那两个老参数分开看，别混成一个。

**它的 context 是 sighup。** 改 `postgresql.conf` 或 `ALTER SYSTEM SET ...` 之后，不需要重启，重载配置（`pg_ctl reload` 或 `SELECT pg_reload_conf()`）就生效。这一点比很多要重启的参数友好得多。

**它的最小值是 0，最大值 100，类型是 real。** 设为 0 是合法的，表示只看 threshold 那个绝对值；设为 100 意味着要等新增行数等于表当前行数的 100 倍才触发，基本等于关掉。

**它可以作为表级存储参数覆盖。** 也就是说，全局设一个值，然后对那几张大表单独给更小的比例，这是最推荐的用法。语法是：

```sql
ALTER TABLE 表名 SET (
  autovacuum_vacuum_insert_scale_factor = 0.02,
  autovacuum_vacuum_insert_threshold = 10000
);
```

用 `toast.autovacuum_vacuum_insert_scale_factor` 和 `toast.autovacuum_vacuum_insert_threshold` 可以单独对表的 TOAST 表做调整。

## 二、为什么需要它：插入不产死元组，为什么还要 VACUUM

这个问题是理解这个参数价值的钥匙。

老逻辑的理由看起来挺合理：VACUUM 主要是回收死元组占用的空间，而 INSERT 不产生死元组，那插入为主的表就不需要 VACUUM。但这个推论其实是错的，而且错了挺久，直到错得越来越明显才被修。INSERT 多的表也需要 VACUUM，主要有两个原因。

**第一，索引只扫描靠可见性映射（visibility map）吃饭。** VACUUM 会更新堆页的 all-visible 标记，也就是维护 visibility map。优化器要判断一次索引只扫描（index-only scan）能不能安全地不回表，靠的就是查这个映射。一个长期只插入、从不被 VACUUM 的表，页上的 all-visible 位始终是脏的，于是 index-only scan 这个优化就在你毫无察觉的情况下悄悄失效了——查询明明看起来该走 index-only scan，实际却在不停回表。这种性能退化是最难排查的，因为执行计划没变，慢是慢慢变慢的。

**第二，也是更致命的，事务 ID 冻结。** 插入的元组带着 xmin，xmin 早晚得被冻结，否则就有事务 ID 回卷（wraparound）的风险。在一个纯插入、老逻辑永远不触发的表上，唯一会来给它做冻结的，就是 anti-wraparound vacuum。而这家伙有个很坏的习惯：它第一次出场往往就是在业务高峰期，一次性把所有积压的冻结工作全做完，而且**它不会因为你阻塞了并发就主动让位**——遇到需要 ACCESS EXCLUSIVE 锁的短操作（比如 DDL），它会把对方堵住，然后那个被堵住的 DDL 又堵住后面所有访问这张表的请求，业务直接停摆。

PG 13 加这两个参数，核心动机就是把这个冻结工作**摊平到平时做**，别等最后一起爆。从效果上说，默认值 0.2 + 1000 的组合，对大多数场景已经比老逻辑好太多。

## 三、PG 18 的一个改进：阈值会随冻结进度自动调整

如果你在 PG 18 上，触发公式比上面那个要复杂一点。PG 18 把 scale factor 那部分乘上了一个系数——表里**还没被冻结**的比例：

```
vacuum insert threshold = autovacuum_vacuum_insert_threshold
  + autovacuum_vacuum_insert_scale_factor
    × reltuples
    × (1 - relallfrozen / relpages)
```

这个改动很有意义。一张表在早期生命周期里大部分页都还没冻结，所以 insert 驱动的 VACUUM 会跑得比较勤——这时候正好有冻结的活要干，多做几次不亏。等到表里大部分页都稳定冻结了，阈值会自动抬高，insert 触发的 VACUUM 就变少了。

带来的直接好处是：**大且基本静态的表，那种"表已经稳定了还被反复 VACUUM"的无效开销大幅减少**。如果你还在 PG 17 及更早版本，享受不到这个优化，只能靠手动把 scale factor 调大来压低频率。这也是我建议有能力就升 PG 18 的理由之一。

## 四、优化建议：全局不动，表级下手

**第一原则：全局保持默认，别在 postgresql.conf 里调它。** 0.2 这个默认值比老的那套 `autovacuum_vacuum_scale_factor` 的默认 0.2 + 50 要合理——插入场景本来就该更激进一点。你一改全局，影响的是全库所有表，很可能把小表的触发频率搞乱。

**第二，只对真正插入密集的大表做表级覆盖。** 典型的候选是事件表、日志表、审计流水、时序数据这些 append-only 或 append-mostly 的表。推荐把 scale factor 设到 0.01 ~ 0.02 之间，同时把 threshold 放大：

```sql
ALTER TABLE events SET (
  autovacuum_vacuum_insert_scale_factor = 0.02,
  autovacuum_vacuum_insert_threshold = 10000
);
```

为什么 threshold 要跟着放大？因为你要避免的是"表短暂安静一下，insert 计数器刚归零，紧接着又插入几百行就再次触发"这种抖动。给一个足够大的 threshold，是给表一个喘息的余量。

**第三，记得和清理强度预算一起看。** scale factor 只管"多脏才动手"，`autovacuum_vacuum_cost_limit` 才管"每轮干多快"。默认的成本预算偏保守，对于写入追不上清理的表，可以把 cost_limit 适当调高，让它在有限的时间窗口里干更多活。

**第四，把它和老那套参数一起审。** 很多人的 autovacuum 配置只调了 `autovacuum_vacuum_scale_factor`，完全没碰 insert 这一对，结果更新不膨胀了，插入侧的 index-only scan 和冻结摊平却一直没改善。两边要一起看。

## 五、避坑：这几个坑我一个个给你标出来

**坑一：把 -1 当成"优化"。** `autovacuum_vacuum_insert_threshold = -1` 会禁用该表的 insert 触发 vacuum。几乎永远没有理由这么做。知道它存在的意义，只是为了能认出别人犯的这个错。

**坑二：忘了它是表级参数，改完以为没生效。** 你以为设的是全局，其实是某个表自己的存储参数在起作用，或者反过来。排查时先看表级：

```sql
SELECT c.relname,
       c.reloptions,
       tc.reloptions AS toast_reloptions
FROM pg_class c
LEFT JOIN pg_class tc ON tc.oid = c.reltoastrelid
WHERE c.relname = '你的表名';
```

再看全局：

```sql
SHOW autovacuum_vacuum_insert_scale_factor;
SHOW autovacuum_vacuum_insert_threshold;
```

**坑三：以为设完立刻见效。** 它要等下一次 autovacuum 轮询才评估，轮询间隔由 `autovacuum_naptime` 控制（默认 1 分钟）。所以改完别急着下结论说"没效果"，等几个 nap 周期再查 `pg_stat_user_tables`。

**坑四：光看参数，不看有没有活干。** autovacuum 能不能及时执行，还受制于 `autovacuum_max_workers`、`autovacuum_vacuum_cost_delay`、以及 autovacuum 能不能抢到 slot。在极端繁忙的实例上，即使阈值被冲破，worker 不够用也得排队。这时候调 scale factor 是治不了病的，得看 worker 数量和成本延迟。

**坑五：PG 18 上误解了"为什么频率变了"。** 升级到 18 后你会发现有些表的 insert 触发 VACUUM 次数变了，这是新公式里那个"未冻结比例"系数在起作用，是正常现象，不是参数漂移，别手贱去改回来。

**坑六：用 0 来"关闭"。** 设成 0 只是表示不按表比例加成，threshold 那 1000（或你设的绝对值）还在，所以不是关闭。真要关，得用 -1。

**坑七：忽视 TOAST 表。** 宽表、存了大字段的表，TOAST 里的膨胀和冻结压力可能比主表还大。前面提到的 `toast.autovacuum_vacuum_insert_scale_factor` 就是给 TOAST 表单独调的入口，别漏了。

## 六、生产环境最佳配置建议

结合上面这些，我给一套可以直接拿去用的思路。

**第一步，建一个巡检习惯。** 把下面这条 SQL 挂进日常监控，抓出那些"插入多、却长期没被 VACUUM"的表：

```sql
SELECT schemaname, relname,
       n_live_tuples,
       n_ins_since_vacuum,
       last_vacuum, last_autovacuum,
       age(relfrozenxid) AS xid_age
FROM pg_stat_all_tables
JOIN pg_class ON pg_class.relname = pg_stat_all_tables.relname
WHERE n_ins_since_vacuum > 1000000
   OR age(relfrozenxid) > 400000000
ORDER BY n_ins_since_vacuum DESC;
```

`n_ins_since_vacuum` 这一列本身就是 PG 13 之后才有的，它直接告诉你这张表离上次 VACUUM 以来插入了多少行。配合 `age(relfrozenxid)` 看事务 ID 年龄，能同时发现冻结风险。

**第二步，对候选表做表级覆盖。** 识别出 append-only 的大表后，用前面那个 `ALTER TABLE ... SET` 给小比例、大 threshold。不要动全局默认值。

**第三步，配套设置成本预算。** 对写入压力大、清理追不上的表，把 `autovacuum_vacuum_cost_limit` 单独抬高，让它在同一轮里干更多活。

**第四步，保证 autovacuum 有足够资源。** 确认 `autovacuum_max_workers` 够用，别让大量表排队等一个 worker。同时也别忘了 `autovacuum_freeze_max_age` 这条防线——对特别关键、又特别怕被 anti-wraparound 突袭的插入表，给它单独设一个小一些的 `autovacuum_freeze_max_age`，让冻结更早、更分散地发生，是老版本 PG 上最有效的兜底手段。

**第五步，监控事务 ID 年龄是底线。** 这套 insert 参数的最终目的之一就是避免 anti-wraparound vacuum 突然闯入业务高峰。所以 `age(datfrozenxid)` 的告警必须长期开着，这比任何单一参数调优都重要。

**一句话总结：** 这个参数是 PG 13 给 append-only 表的"定期体检"开关。它解决的不是空间回收，而是 visibility map 维护和事务 ID 冻结摊平这两件 INSERT 场景下被长期忽视的事。用法就八个字——**全局不动，表级下调**；判断依据就一条——看 `n_ins_since_vacuum`。

如果你还在 PG 12 上，这篇文章对你而言就是一份升级路线图，而不是调参清单。真到了那种全库表都被 anti-wraparound 堵死的凌晨，你会希望自己早就升了。

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
