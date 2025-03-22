---
layout: post
title: "Go微服存储过程+事件,实现数据备份"
date: 2022-4-17
tags: [Go]
comments: true
author: mazezen
---


遇到一个需求。需要每天凌晨三点实现对指定的几张表执行备份（备份前一天的数据）。并且写到备份库里，并对现有库中删除掉。每天的单子量非常大,如果再加上备份读写 mysql 会比较慢

刚开始通过go协程开四个协程实现备份，一个小时备份了 不到50万太慢了。所以改用存储过程+事件的方式实现。经测试

530万的数据量 备份需要大概13分钟。



创建demo 数据库

创建 o_a_order表

创建 o_b_order表

创建 o_c_order表

创建 o_match_order表



 备份的表

创建 o_a_order_backup表

创建 o_b_order_backup表

创建 o_c_order_backup表

创建 o_match_order_backup表



1. 查看mysql 事件调度器是否打开

```mysql
show variables like '%event_scheduler%';
```

2. 开启event_scheduler sql指令

```mysql
SET GLOBAL event_scheduler = ON;

SET @@global.event_scheduler = ON;

SET GLOBAL event_scheduler = 1;

SET @@global.event_scheduler = 1;

```

3. o_a_order

```mysql
-- 存储过程 o_a_order
DELIMITER |

DROP PROCEDURE IF EXISTS a_backup |

CREATE PROCEDURE a_backup()

BEGIN

INSERT INTO o_a_order_backup SELECT * FROM o_a_order WHERE created_at < CAST(CAST(SYSDATE()AS DATE)AS DATETIME);

DELETE FROM o_a_order WHERE created_at < CAST(CAST(SYSDATE()AS DATE)AS DATETIME);

END

|


-- 定时器 omp_a_order
SET GLOBAL event_scheduler = 1;

CREATE EVENT IF NOT EXISTS a_backup

ON SCHEDULE EVERY 1 DAY


ON COMPLETION PRESERVE

DO CALL a_backup();


-- 启动定时器 omp_a_order
ALTER EVENT a_backup ON
COMPLETION PRESERVE ENABLE;

```

4. o_b_order

```mysql
-- 存储过程 o_b_order
DELIMITER |

DROP PROCEDURE IF EXISTS b_backup |

CREATE PROCEDURE b_backup()

BEGIN

INSERT INTO o_b_order_backup SELECT * FROM o_b_order WHERE created_at < CAST(CAST(SYSDATE()AS DATE)AS DATETIME);

DELETE FROM o_b_order WHERE created_at < CAST(CAST(SYSDATE()AS DATE)AS DATETIME);

END

|

---- 定时器 omp_b_order
SET GLOBAL event_scheduler = 1;

CREATE EVENT IF NOT EXISTS b_backup

ON SCHEDULE EVERY 1 DAY

ON COMPLETION PRESERVE

DO CALL b_backup();

-- 启动定时器 omp_b_order
ALTER EVENT b_backup ON
COMPLETION PRESERVE ENABLE;
```

5. o_c_order

```mysql
DELIMITER |

DROP PROCEDURE IF EXISTS c_backup |

CREATE PROCEDURE c_backup()

BEGIN

INSERT INTO o_c_order_backup SELECT * FROM o_c_order WHERE created_at < CAST(CAST(SYSDATE()AS DATE)AS DATETIME);

DELETE FROM o_c_order WHERE created_at < CAST(CAST(SYSDATE()AS DATE)AS DATETIME);

END

|

---- 定时器 o_c_order
SET GLOBAL event_scheduler = 1;

CREATE EVENT IF NOT EXISTS c_backup

ON SCHEDULE EVERY 1 DAY

ON COMPLETION PRESERVE

DO CALL c_backup();

-- 启动定时器 omp_c_order
ALTER EVENT c_backup ON
COMPLETION PRESERVE ENABLE;
```

6. o_match_order

```mysql
-- 存储过程 o_match_order
DELIMITER |

DROP PROCEDURE IF EXISTS m_order_backup |

CREATE PROCEDURE m_order_backup()

BEGIN

INSERT INTO o_match_order_backup SELECT * FROM o_match_order WHERE created_at < CAST(CAST(SYSDATE()AS DATE)AS DATETIME);

DELETE FROM o_match_order WHERE created_at < CAST(CAST(SYSDATE()AS DATE)AS DATETIME);

END

|

---- 定时器 o_match_order
SET GLOBAL event_scheduler = 1;

CREATE EVENT IF NOT EXISTS match_order_backup

ON SCHEDULE EVERY 1 DAY

ON COMPLETION PRESERVE

DO CALL m_order_backup();

-- 启动定时器 o_match_order
ALTER EVENT m_order_backup ON
COMPLETION PRESERVE ENABLE;
```





```mysql
show events;


SHOW PROCEDURE STATUS LIKE 'a_backup';
SHOW PROCEDURE STATUS LIKE 'b_backup';
SHOW PROCEDURE STATUS LIKE 'c_backup';
SHOW PROCEDURE STATUS LIKE 'match_backup';


drop event a_backup;
drop event b_backup;
drop event c_backup;
drop event match_order_backup;
```

