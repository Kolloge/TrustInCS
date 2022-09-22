- redo log日志：重做日志InnoDB引擎特有的。一般用于MySQL异常重启后恢复数据
  - 记录的是物理日志：某页的某个字段修改后的值（page id  record offset （filed，value））类似于这种格式
  - 循环写
- binlog日志：归档日志MySQL的Server层实现的，因此所有存储引擎都可以使用。一般用于恢复到某一时刻数据
  - 追加写，一个文件写满之后，会创建一个新页
  - 记录的是逻辑日志：有三种模式：
    - row模式：记录的是每行数据的变化
    - statement：记录的就是sql
    - mixed：前两种模式的混合：
  - statement模式下的日志，用来恢复数据的时候，会使原来sql里的一些函数生成的数值发生变化（如：uuid(),now()）导致数据不一致
  - row模式下会使日志量翻倍
  - 因此mixed模式就是要使记录日志时可以根据不同sql，自动选择适用的日志格式
  - redo log和 binlog都开启的情况下：
  - 为了保证数据的一致性，更新数据时采用两段提交的方式：数据库更新数据时，先把要更新的那行数据所在的数据页读到内存中（如果已存在直接返回），找到对应的数据，执行更新操作生成一行新的数据，然后插入到内存中。
  在更新数据的同时更新redo log，此时redo log处于prepare状态，然后再写binlog，binlog写入磁盘后将redo log的状态修改为commit。
这样做的好处就是，如果MySQL崩溃重启后，在恢复数据时，可以根据redo log的状态和binlog的内容 来抉择部分数据是需要更新还是回滚。