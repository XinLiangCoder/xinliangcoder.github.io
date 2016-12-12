# Mysql 常用SQL语句集锦

> 基础篇

```
//查询时间，友好提示
$sql = "select date_format(create_time, '%Y-%m-%d') as day from table_name";
```

```
//int 时间戳类型
$sql = "select from_unixtime(create_time, '%Y-%m-%d') as day from table_name";
```

```
//一个sql返回多个总数
$sql = "select count(*) all, " ;
$sql .= " count(case when status = 1 then status end) status_1_num, ";
$sql .= " count(case when status = 2 then status end) status_2_num ";
$sql .= " from table_name";
```

```
//Update Join / Delete Join
$sql = "update table_name_1 ";
$sql .= " inner join table_name_2 on table_name_1.id = table_name_2.uid ";
$sql .= " inner join table_name_3 on table_name_3.id = table_name_1.tid ";
$sql .= " set *** = *** ";
$sql .= " where *** ";

//delete join 同上。
```

```
//替换某字段的内容的语句
$sql = "update table_name set content = REPLACE(content, 'aaa', 'bbb') ";
$sql .= " where (content like '%aaa%')";
```

```
//获取表中某字段包含某字符串的数据
$sql = "SELECT * FROM `表名` WHERE LOCATE('关键字', 字段名) ";
```

```
//获取字段中的前4位
$sql = "SELECT SUBSTRING(字段名,1,4) FROM 表名 ";
```

```
//查找表中多余的重复记录
//单个字段
$sql = "select * from 表名 where 字段名 in ";
$sql .= "(select 字段名 from 表名 group by 字段名 having count(字段名) > 1 )";
//多个字段
$sql = "select * from 表名 别名 where (别名.字段1,别名.字段2) in ";
$sql .= "(select 字段1,字段2 from 表名 group by 字段1,字段2 having count(*) > 1 )";
```

```
//删除表中多余的重复记录(留id最小)
//单个字段
$sql = "delete from 表名 where 字段名 in ";
$sql .= "(select 字段名 from 表名 group by 字段名 having count(字段名) > 1)  ";
$sql .= "and 主键ID not in ";
$sql .= "(select min(主键ID) from 表名 group by 字段名 having count(字段名 )>1) ";
//多个字段
$sql = "delete from 表名 别名 where (别名.字段1,别名.字段2) in ";
$sql .= "(select 字段1,字段2 from 表名 group by 字段1,字段2 having count(*) > 1) ";
$sql .= "and 主键ID not in ";
$sql .= "(select min(主键ID) from 表名 group by 字段1,字段2 having count(*)>1) ";
```
> 业务篇

- 连续范围问题

```
//创建测试表
CREATE TABLE `test_number` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `number` int(11) unsigned NOT NULL DEFAULT '0' COMMENT '数字',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

```
//创建测试数据
insert into test_number values(1,1);
insert into test_number values(2,2);
insert into test_number values(3,3);
insert into test_number values(4,5);
insert into test_number values(5,7);
insert into test_number values(6,8);
insert into test_number values(7,10);
insert into test_number values(8,11);
```

**实验目标：求数字的连续范围。**

**根据上面的数据，应该得到的范围。**

```
1-3
5-5
7-8
10-11
```
```
//执行Sql
select min(number) start_range,max(number) end_range
from
(
    select number,rn,number-rn diff from
    (
        select number,@number:=@number+1 rn from test_number,(select @number:=0) as number
    ) b
) c group by diff;
```

![连续范围问题](https://ntaste.github.io/image/mysql/m-1.png)


- 签到问题

```
//创建参考表(模拟数据需要用到)
CREATE TABLE `test_nums` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='参考表';
//模拟数据，插入 1-10000 连续数据.
```

```
//创建测试表
CREATE TABLE `test_sign_history` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `uid` int(11) unsigned NOT NULL DEFAULT '0' COMMENT '用户ID',
  `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '签到时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='签到历史表';
```

```
//创建测试数据
insert into test_sign_history(uid,create_time)
select ceil(rand()*10000),str_to_date('2016-12-11','%Y-%m-%d')+interval ceil(rand()*10000) minute
from test_nums where id<31;
```

```
//统计每天的每小时用户签到情况
select
    h,
    sum(case when create_time='2016-12-11' then c else 0 end) 11Sign,
    sum(case when create_time='2016-12-12' then c else 0 end) 12Sign,
    sum(case when create_time='2016-12-13' then c else 0 end) 13Sign,
    sum(case when create_time='2016-12-14' then c else 0 end) 14Sign,
    sum(case when create_time='2016-12-15' then c else 0 end) 15Sign,
    sum(case when create_time='2016-12-16' then c else 0 end) 16Sign,
    sum(case when create_time='2016-12-17' then c else 0 end) 17Sign
from
(
    select
        date_format(create_time,'%Y-%m-%d') create_time,
        hour(create_time) h,
        count(*) c
    from test_sign_history
    group by
        date_format(create_time,'%Y-%m-%d'),
        hour(create_time)
) a
group by h with rollup;
```

![统计每天的每小时用户签到情况](https://ntaste.github.io/image/mysql/m-2.png)

```
//统计每天的每小时用户签到情况(当某个小时没有数据时，显示0)
select
    h ,
    sum(case when create_time='2016-12-11' then c else 0 end) 11Sign,
    sum(case when create_time='2016-12-12' then c else 0 end) 12Sign,
    sum(case when create_time='2016-12-13' then c else 0 end) 13Sign,
    sum(case when create_time='2016-12-14' then c else 0 end) 14Sign,
    sum(case when create_time='2016-12-15' then c else 0 end) 15Sign,
    sum(case when create_time='2016-12-16' then c else 0 end) 16Sign,
    sum(case when create_time='2016-12-17' then c else 0 end) 17Sign
from
(
    select b.h h,c.create_time,c.c from
     (
        select id-1 h from test_nums where id<=24
     ) b
     left join
     (
        select
         date_format(create_time,'%Y-%m-%d') create_time,
         hour(create_time) h,
         count(*) c
        from test_sign_history
        group by
         date_format(create_time,'%Y-%m-%d'),
         hour(create_time)
      ) c on (b.h=c.h)
) a
group by h with rollup;
```

![统计每天的每小时用户签到情况(当某个小时没有数据时，显示0)](https://ntaste.github.io/image/mysql/m-3.png)

```
//统计每天的用户签到数据和每天的增量数据
select
        type,
        sum(case when create_time='2016-12-11' then c else 0 end) 11Sign,
        sum(case when create_time='2016-12-12' then c else 0 end) 12Sign,
        sum(case when create_time='2016-12-13' then c else 0 end) 13Sign,
        sum(case when create_time='2016-12-14' then c else 0 end) 14Sign,
        sum(case when create_time='2016-12-15' then c else 0 end) 15Sign,
        sum(case when create_time='2016-12-16' then c else 0 end) 16Sign,
        sum(case when create_time='2016-12-17' then c else 0 end) 17Sign
from
(
        select b.create_time,ifnull(b.c-c.c,0) c,'Increment' type from
        (
            select
             date_format(create_time,'%Y-%m-%d') create_time,
             count(*) c
            from test_sign_history
            group by
             date_format(create_time,'%Y-%m-%d')
        ) b
        left join
        (
            select
             date_format(create_time,'%Y-%m-%d') create_time,
             count(*) c
            from test_sign_history
            group by
             date_format(create_time,'%Y-%m-%d')
        ) c on(b.create_time=c.create_time+ interval 1 day)
    union all
        select
         date_format(create_time,'%Y-%m-%d') create_time,
         count(*) c,
         'Current'
        from test_sign_history
        group by
         date_format(create_time,'%Y-%m-%d')
) a
group by type
order by case when type='Current' then 1 else 0 end desc;
```

![统计每天的用户签到数据和每天的增量数据](https://ntaste.github.io/image/mysql/m-4.png)

```
//模拟不同的用户签到了不同的天数
insert into test_sign_history(uid,create_time)
select uid,create_time + interval ceil(rand()*10) day from test_sign_history,test_nums
where test_nums.id <10 order by rand() limit 150;
```

```
//统计签到天数相同的用户数量
select
    sum(case when day=1 then cn else 0 end) 1Day,
    sum(case when day=2 then cn else 0 end) 2Day,
    sum(case when day=3 then cn else 0 end) 3Day,
    sum(case when day=4 then cn else 0 end) 4Day,
    sum(case when day=5 then cn else 0 end) 5Day,
    sum(case when day=6 then cn else 0 end) 6Day,
    sum(case when day=7 then cn else 0 end) 7Day,
    sum(case when day=8 then cn else 0 end) 8Day,
    sum(case when day=9 then cn else 0 end) 9Day,
    sum(case when day=10 then cn else 0 end) 10Day
from
(
    select c day,count(*) cn
    from
    (
        select uid,count(*) c from test_sign_history group by uid
    ) a
    group by c
) b;
```

![统计签到天数相同的用户数量](https://ntaste.github.io/image/mysql/m-5.png)

```
//统计每个用户的连续签到时间
select * from (
    select d.*,
    @ggid := @cggid,
    @cggid := d.uid,
    if(@ggid = @cggid, @grank := @grank + 1, @grank := 1) grank
    from
    (
        select uid,min(c.create_time) begin_date ,max(c.create_time) end_date,count(*) count from
        (
            select
            b.*,
            @gid := @cgid,
            @cgid := b.uid,
            if(@gid = @cgid, @rank := @rank + 1, @rank := 1) rank,
            b.diff-@rank flag from (
                select
                distinct
                uid,
                date_format(create_time,'%Y-%m-%d') create_time,
                datediff(create_time,now()) diff
                from test_sign_history order by uid,create_time
            ) b, (SELECT @gid := 1, @cgid := 1, @rank := 1) as a
        ) c group by uid,flag
        order by uid,count(*) desc
    ) d,(SELECT @ggid := 1, @cggid := 1, @grank := 1) as e
)f
where grank=1;
```

![统计签到天数相同的用户数量](https://ntaste.github.io/image/mysql/m-6.png)

Thanks ~

> 相关备注

- 遇到技术问题，可以给我的微信公众号留言，看到后我会及时回复。
- Demo 可以提供思路，具体情况依照各自实际需求进行开发。

微信名称：IT小圈儿，微信号：ToFeelings。

![IT小圈儿](https://ntaste.github.io/image/qr.jpg)
