1. **避免使用 select \***
   - 查询时需要将 * 解析为所有字段，**增加查询解析器的成本**
   - 查询出不需要的字段时浪费 **CPU** 和**内存资源**
   - select * 查询一般不走覆盖索引
   - 文本数据和大字段数据传输增加网络消耗
2. **使用小表驱动大表**
   - Join Buffer （连接缓冲区）是优化器用于处理连接查询操作时的临时缓冲区。简单来说当我们需要比较两个或多个表的数据进行Join操作时，Join Buffer可以帮助 MYSQL 临时存储结果，以减少磁盘读取和CPU负担，提高查询效率。需要注意的是每个join都有一个单独的缓冲区
   - Block nested-loop join（BNL算法）会将驱动表数据加载到 join buffer 里面，然后再批量与非驱动表进行匹配；如果驱动表数据量较大，join buffer无法一次性装载驱动表的结果集，将会分阶段与被驱动表进行批量数据匹配，会增加被驱动表的扫描次数，从而降低查询效率。所以开发中要遵守小表驱动大表的原则。
3. **优先考虑连接查询而非子查询**
   - 子查询需要执行**两次**数据库查询，一次外部查询，一次嵌套子查询
   - 连接查询可以利用表上的索引，子查询不行，另外子查询会使用临时表或内存表，因此连接查询可以更快访问表中的数据且减少资源的消耗
4. **在无需排除重复项的时候优先使用 union all 而非 union**
   - union all：获取所有数据但是数据不去重，包含重复数据
   - union：获取所有数据且数据去重，不包含重复数据
5. **对 group by 字段创建 索引**
   - B+ 树索引 以树形结构存储数据，适用于范围查询和精确查询，支持**有序数据的快速查找**、排序和聚合操作
6. **减少 join 表数量**
   - 查询效率下降：多表JOIN查询数据对比时间边长
   - 系统负载增加：JOIN操作需要进行大量的计算，因此会导致系统负载增加。
   - 维护难度加大：在一个连接了多个表的查询中，如果需要修改其中一个表的结构或内容，就可能会需要同时修改其他表的结构或内容。
7. **使用批量 插入和删除 代替 单个 操作**
   - 逐个操作会频繁与数据库进行交互，损耗性能（但是不建议一次批量操作太多数据，数据太多数据库响应也会很慢，建议每批操作控制在 500 **以内**）
8. **使用 limit 约束数据量**
   - 提高查询效率：限制返回的数据行数，减轻系统负担，提高查询效率
   - 避免过度提取数据，对于大型数据库系统，提取大量数据可能导致系统崩溃
   - 优化分页查询。分页查询需要查询所有所有的数据才会进行分页处理，使用 limit 可以只查询需要的数据行，缩短查询时间



### **扩展**

#### 百万级表 Limit 翻页越往后越慢怎么办？

``` sql
select * from table_name limit 10000,10
```

这句 SQL 的执行逻辑是

1. 从数据表中读取第N条数据添加到数据集中
2. 重复第一步直到N=10000+10
3. 根据 offset 抛弃前面 10000 条数
4. 返回剩余的 10 条数据

解决思路：先查找出需要数据的索引列（假设为 id ），再通过索引列查找出需要的数据。

```sql
select * from table_name where id in (
	select id from table_name 
    where (user = XXX)
) limit 10000,10
```

再次优化

```sql
select * from table_name where id inner join (
	select id from table_name 
    where (user = XXX)
) limit 10000,10
```