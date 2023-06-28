---
title: sql练习之连续登录问题
tags:
  - sql练习
categories:
  - 练习笔记
toc: true
abbrlink: 67cc9ac
date: 2023-03-10 16:22:38
---
[题目来源](https://mp.weixin.qq.com/s?__biz=MzU5NTc1NzE2OA==&mid=2247484833&idx=1&sn=2a965091ec5ec09fe10185fecbf92779&chksm=fe6c54bec91bdda83229a33ceefb0358f408ed24ac5048eb48dbe6911b795b1536971a98ab57&scene=21#wechat_redirect)

# 题目要求
求出连续3天登录的用户id

# 数据
```sql
CREATE TABLE if not exists one.user_login(
     id int COMMENT '用户主键',
     dt varchar(20) COMMENT '登录日期'
    );


insert into  user_login values(1001,'2021-12-12');
insert into  user_login values(1002,'2021-12-12');
insert into  user_login values(1001,'2021-12-13');
insert into  user_login values(1001,'2021-12-14');
insert into  user_login values(1001,'2021-12-16');
insert into  user_login values(1002,'2021-12-16');
insert into  user_login values(1001,'2021-12-19');
insert into  user_login values(1002,'2021-12-17');
insert into  user_login values(1001,'2021-12-20');

```

# 解题
## 解法一：自关联
```sql
select
	distinct id
from
	(
	SELECT
		id
	from
		(
		select
			a.id ,
			a.dt as dt1 ,
			b.dt as dt2
		from
			user_login a
		left join user_login b on
			a.id = b.id
			and (b.dt between DATE_SUB(a.dt, interval 2 day) and a.dt)
	) tmp1
	group by
		id,
		dt1
	having
		count(1) = 3
	)tmp2
;
```




