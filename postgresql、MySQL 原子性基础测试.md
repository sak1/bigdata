# Postgresql、MySQL 原子性基础测试

**测试结果：**通过对MySQL 与 PostgreSQL 各六项原子性测试，前 5 项表现一致，仅第六项“死锁”测试出现的位置不同， MySQL 默认采用了“REPEATABLE READ”隔离级别，PostgreSQL 默认为READ COMMITTED（读已提交）隔离级别。**初步结论：两数据库原子性无本质差异。**

**测试环境：**VMbox、CentOS7.9　Postgresql14.9、5.5.68-MariaDB

当编写数据库原子性测试用例时，你需要使用SQL语句来模拟不同的操作和情景。以下是一些使用PostgreSQL语法的具体SQL示例，用于测试不同操作的原子性：

**1. 插入操作原子性测试：**

```sql

-- 测试成功插入一条记录
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL
);

INSERT INTO users (username, email) VALUES ('testuser', 'test@example.com');


INSERT 0 1
-- 测试插入操作被中断不会插入任何记录
BEGIN;
INSERT INTO users (username, email) VALUES ('interrupted', 'interrupted@example.com');
ROLLBACK;

```

![image-20230812125330989](/Users/mac/Library/Application Support/typora-user-images/image-20230812125330989.png)

成功：没有新纪录出现；

![image-20230812225415285](/Users/mac/Library/Application Support/typora-user-images/image-20230812225415285.png)

**2. 更新操作原子性测试：**

```

-- 测试成功更新一条记录
UPDATE users SET email = 'test@example.com' WHERE username ='testuser';

-- 测试更新操作被中断不会部分更新记录
BEGIN;
UPDATE users SET email = 'new@example.com' WHERE username ='testuser';
ROLLBACK;
```

![image-20230812130003097](/Users/mac/Library/Application Support/typora-user-images/image-20230812130003097.png)

成功：更新操作被中断记录未更新

![image-20230812225527221](/Users/mac/Library/Application Support/typora-user-images/image-20230812225527221.png)

**3. 删除操作原子性测试：**

```
INSERT INTO users (username, email) VALUES ('testuser1', 'test@example.com');

-- 测试成功删除一条记录
DELETE FROM users WHERE username ='testuser1';

-- 测试删除操作被中断不会部分删除记录
BEGIN;
DELETE FROM users WHERE username ='testuser';
ROLLBACK;
```

![image-20230812130708085](/Users/mac/Library/Application Support/typora-user-images/image-20230812130708085.png)

成功：删除操作被中断不会部分删除记录

![image-20230812225648045](/Users/mac/Library/Application Support/typora-user-images/image-20230812225648045.png)

**4. 事务原子性测试：**

```

-- 测试事务中的操作全部成功
BEGIN;
INSERT INTO users (username, email) VALUES ('testuser2', 'test2@example.com');
UPDATE users SET email= 'test2@example.com' WHERE username = 'testuser2';
COMMIT;

-- 测试事务中的操作被中断不会部分生效
BEGIN;
INSERT INTO users (username, email)  VALUES  ('testuser3', 'test3@example.com');
UPDATE users SET email= 'test3@example.com' WHERE username = 'testuser3';
ROLLBACK;
```

![image-20230812163701610](/Users/mac/Library/Application Support/typora-user-images/image-20230812163701610.png)

![image-20230812225758325](/Users/mac/Library/Application Support/typora-user-images/image-20230812225758325.png)

**5. 并发原子性测试：**

```
-- 创建多个会话执行插入操作，只有一个能成功
-- 会话1：
BEGIN;
INSERT INTO users (username, email) VALUES ('testuser5', 'test5@example.com');
-- 会话2（同时执行）：
BEGIN;
INSERT INTO users (username, email) VALUES ('testuser5', 'test6@example.com');-- 这条会失败
ROLLBACK;
-- 会话1（继续执行）：
COMMIT;
```

![image-20230812165125698](/Users/mac/Library/Application Support/typora-user-images/image-20230812165125698.png)

![image-20230812230047700](/Users/mac/Library/Application Support/typora-user-images/image-20230812230047700.png)

**6. 死锁测试：**

```
-- 创建两个事务引发死锁
-- A 会话1：
BEGIN;
UPDATE users SET email = 'test5a@example.com' WHERE username = 'testuser5';
-- B C会话2（同时执行）：
BEGIN;
UPDATE users SET email = 'test2a@example.com' WHERE username = 'testuser2';　－mysql 死锁
-- C 会话1（继续执行）：
UPDATE users SET email = 'test2a@example.com' WHERE username = 'testuser2'; -- pg死锁
-- D 会话2（继续执行）：
UPDATE users SET email = 'test5a@example.com' WHERE username = 'testuser5'; -- pg、mysql死锁
```

![image-20230812170534101](/Users/mac/Library/Application Support/typora-user-images/image-20230812170534101.png

![image-20230812231230745](/Users/mac/Library/Application Support/typora-user-images/image-20230812231230745.png)

成功：

![image-20230812230436996](/Users/mac/Library/Application Support/typora-user-images/image-20230812230436996.png)

**PG隔离级别**

![image-20230812165334416](/Users/mac/Library/Application Support/typora-user-images/image-20230812165334416.png)



这些SQL示例是用于测试数据库原子性的基本情况。具体的测试用例应该根据你的数据模型和需求进行调整和扩展。

**mariaDB隔离级别**

![image-20230812231417219](/Users/mac/Library/Application Support/typora-user-images/image-20230812231417219.png)



MariaDB 支持的隔离级别：

1. READ UNCOMMITTED（读未提交）
2. READ COMMITTED（读已提交）
3. REPEATABLE READ（可重复读）：
4. SERIALIZABLE（串行化）：