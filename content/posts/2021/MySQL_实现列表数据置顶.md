---
title: 【MySQL】实现列表数据置顶
date: 2021-08-16 14:46:00
tags: MySQL
---

有的列表获取的业务场景，需要根据一些条件将数据置顶。

例如有一张表有 id 和 name 两个字段。数据表如下。

|  id   | name  |
|  ----  | ----  |
| 1  | 1_name |
| 2  | 2_name |
| 3  | 3_name |
| 4  | 4_name |
| 5  | 5_name |
| 6  | 6_name |
| 7  | 7_name |
| 8  | 8_name |
| 9  | 9_name |
| 10  | 10_name |

若需要将 id 为 5 和 7 的数据置顶，并且分页的 size 是 5 的话，我们的 SQL 可以这样写：
```
SELECT * FROM goods ORDER BY 
	CASE 
		WHEN id = 5 THEN -10000
		WHEN id = 7 THEN -9999 
		ELSE id 
	END
LIMIT 0, 5
```