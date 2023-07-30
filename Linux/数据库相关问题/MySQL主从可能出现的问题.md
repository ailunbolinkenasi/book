## MySQL主从可能出现的问题


### 错误1： **Error_code: 1032; handler error HA_ERR_KEY_NOT_FOUND**
- 可能是**是主从更新时丢失数据，导致主从不一致**
```bash
Error_code: 1032; handler error HA_ERR_KEY_NOT_FOUND
```
1. 在主库上执行`show master status\\G;`
```bash
# 找到有问题的Pos定位 212338698
Could not execute Delete_rows event on table chnenergy-prod.sys_dept; Can't find record in 'sys_dept', Error_code: 1032; handler error HA_ERR_KEY_NOT_FOUND; the event's master log mysql-bin.000005, end_log_pos 212338698
```
2. 通过`mysqlbinlog`分析这个Pos点在干什么内容
```bash
mysqlbinlog --no-defaults -v -v --base64-output=DECODE-ROWS /var/lib/mysql/binlog/mysql-bin.000005 | grep -A '10' 212338698
```
3. 这个时候你需要去查询一下主库上是否有binlog中的数据,如果主库有从库没有的话只需要补全数据然后跳过报错
```sql
mysql> insert into t1 values(); # 根据自己的报错语句进行操作
mysql> stop slave ;
mysql> set global sql_slave_skip_counter=1;
mysql> start slave;
mysql> show slave status\G;
```

4. 如果你想直接复制当前Master的Pos

```sql
CHANGE MASTER TO MYSQL_LOG_FILE='binglog' MYSQL_LOG_POS='便宜量'
```

