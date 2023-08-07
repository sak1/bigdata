# Pg 与mysql 在虚拟机下基于存储过程的插值的性能与空间对比



## 一、环境与前置条件

电脑：MacBook Pro (15-inch, 2016)、2.7 GHz 四核Intel Core i7、MacOS

虚拟机：Virtualbox、CentOS7.0

数据库：Postgresql 14 、 mysql5.7

上述软件已安装，数据目录已初始化，可正常运行。

Shell 登录 CentOS

显示[root@namenode ~]# 

## 二、Pg 操作

进入pg 安装目录（如:/usr/pgsql-14/bin）启动

```bash
./psql
create database abc;
CREATE DATABASE
postgres=# \c abc;
```

1、建t_user 表，用于存储模拟数据

```
CREATE TABLE t_user(
  id SERIAL PRIMARY KEY,
  first_name VARCHAR(20) NOT NULL,
  last_name VARCHAR(20) NOT NULL,
  sex VARCHAR(5) NOT NULL,
  score INTEGER NOT NULL,
  copy_id INTEGER NOT NULL
);
```

2、建一个模拟生成百万用户数据的过程，用于生成数据并向t_user 插值；

```
CREATE OR REPLACE FUNCTION add_user_innodb(num INT) RETURNS VOID AS $$
DECLARE
    rowid INT := 0;
    firstname VARCHAR(10);
    name1 VARCHAR(10);
    name2 VARCHAR(10);
    lastname VARCHAR(10) := '';
    sex INTEGER;
    score INTEGER;
BEGIN
    WHILE rowid < num LOOP
        firstname := SUBSTRING(md5(RANDOM()::TEXT), 1, 4);
        name1 := SUBSTRING(md5(RANDOM()::TEXT), 1, 4);
        name2 := SUBSTRING(md5(RANDOM()::TEXT), 1, 4);
        sex := FLOOR(RANDOM() * 2);
        score := FLOOR(40 + RANDOM() * 60);
        rowid := rowid + 1;

IF ROUND(RANDOM()) = 0 THEN 
        lastname := name1;
    ELSE
        lastname := name1 || name2;
    END IF;

INSERT INTO t_user_innodb(first_name, last_name, sex, score, copy_id) VALUES (firstname, lastname, sex, score, rowid);  

END LOOP;

   END;
$$ LANGUAGE plpgsql;
```

3、打开时间显示

```sql
timing on
```

4、调用存储过程执行，设置参数为百万

```bash
abc=# select add_user(1000000);
add_user 
(0 行记录)

时间：25062.998 ms (00:25.063)
```

用时 35 分钟 27 秒



5、检查数据生情况，正常

```sql
abc=# select * from t_user;
  id  | first_name | last_name | sex | score | copy_id 
---------+------------+-----------+-----+-------+---------

​    1 | 753d    | 4712   | 0  |  50 |    1
​    2 | 60c2    | 456de196 | 1  |  82 |    2
​    3 | ac1e    | 2b96   | 1  |  93 |    3
​    4 | 3c98    | 03e897c2 | 1  |  75 |    4
​    5 | d3e6    | bdd2ebde | 1  |  71 |    5
​    6 | 0dcb    | 9752   | 0  |  57 |    6
​    7 | b9a2    | 5eded0b4 | 0  |  82 |    7
​    8 | 373c    | cf6b   | 0  |  93 |    8
​    9 | 25ef    | 6d6ff9a9 | 1  |  65 |    9
​    10 | 7d55    | e7890b82 | 1  |  48 |   10
```

6、查询所用尺寸

```bash
abc=# SELECT pg_size_pretty(pg_table_size('t_user'));
 pg_size_pretty 
 108 MB
(1 行记录)
```



## 三、mysql 操作

进入mysql安装目录（如:/usr/sbin/mysqld）启动

```bash
mysql_server start
mysql -uroot -p -S /usr/sbin/mysqld/tmp/mysql.sock
```

1、建t_user 表，用于存储模拟数据

```sql
use test
CREATE TABLE t_user(
  id INT NOT NULL AUTO_INCREMENT,
  first_name VARCHAR(20) NOT NULL,
  last_name VARCHAR(20) NOT NULL,
  sex VARCHAR(5) NOT NULL,
  score INT NOT NULL,
  copy_id INT NOT NULL,
  PRIMARY KEY (`id`)
) engine=innodb;
```

2、建一个模拟生成百万用户数据的过程，用于生成数据并向t_user 插值；

```sql
create PROCEDURE add_user(in num INTEGER)
BEGIN
DECLARE rowid INTEGER DEFAULT 0;
DECLARE firstname VARCHAR(10);
DECLARE name1 VARCHAR(10);
DECLARE name2 VARCHAR(10);
DECLARE lastname VARCHAR(10) DEFAULT '';
DECLARE sex CHAR(1);
DECLARE score CHAR(2);
WHILE rowid < num DO
SET firstname = SUBSTRING(md5(rand()),1,4); 
SET name1 = SUBSTRING(md5(rand()),1,4); 
SET name2 = SUBSTRING(md5(rand()),1,4); 
SET sex=FLOOR(0 + (RAND() * 2));
SET score= FLOOR(40 + (RAND() *60));
SET rowid = rowid + 1;
IF ROUND(RAND())=0 THEN 
SET lastname =name1;
END IF;
IF ROUND(RAND())=1 THEN
SET lastname = CONCAT(name1,name2);
END IF;
insert INTO t_user(first_name,last_name,sex,score,copy_id) VALUES (firstname,lastname,sex,score,rowid);  
END WHILE;
END //
DELIMITER ;
```

3、调用存储过程执行，设置参数为百万

```sql
call add_user(1000000);
Query OK, 1 row affected (35 min 27.70 sec)
```

用时 35 分钟 27 秒

4、检查数据生情况，正常

```sql
select * from t_user;

+---------+------------+-----------+-----+-------+---------+
| id   | first_name | last_name | sex | score | copy_id |
+---------+------------+-----------+-----+-------+---------+
|    1 | e833    | 70e1fb14 | 1  |  51 |    1 |
|    2 | deb0    | 992711   | 0  |  73 |    2 |
|    3 | 50b8    | 4c6569c0 | 0  |  54 |    3 |
|    4 | 65c5    | f3c4e267 | 1  |  86 |    4 |
|    5 | c691    | aa004ea1 | 0  |  90 |    5 |
|    6 | 0108    | a964aa80 | 1  |  69 |    6 |
|    7 | 1275    | efe1afda | 0  |  44 |    7 |
|    8 | 01ac    | 66a8b6c7 | 0  |  72 |    8 |
|    9 | 4a3a    | 72c5ad2f | 1  |  97 |    9 |
|   10 | d1ca    | d8c98d45 | 1  |  42 |   10 |
```

5、查询所用尺寸

```sql
SHOW TABLE STATUS LIKE 't_user';

显示其中的data length为:48824320 

```

尺寸为：48824320 bytes / (1024 * 1024) ≈ 46.56 MB

## 结论

插值pg 比 mysql 快 85 倍，存储空间mysql 不到pg 的一半。

排除字段影响情况下，存储空间成本较低的情况下用 PG，支出极有限的情况用mysql



