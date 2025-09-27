# TiDB 基本操作和 SQL 使用教程

## 1. 连接 TiDB 数据库

### 1.1 使用 MySQL 客户端连接

#### 1.1.1 基本连接命令

```bash
# 基本连接语法
mysql -h <host> -P <port> -u <username> -p

# 本地连接（默认）
mysql -h 127.0.0.1 -P 4000 -u root

# 指定数据库连接
mysql -h 127.0.0.1 -P 4000 -u root -p my_database

# 使用配置文件连接
mysql --defaults-file=my.cnf
```

#### 1.1.2 配置文件示例

创建 `my.cnf` 文件：

```ini
[client]
host = 127.0.0.1
port = 4000
user = root
password = your_password
default-character-set = utf8mb4
```

### 1.2 连接字符串示例

#### 1.2.1 JDBC 连接字符串

```java
// Java JDBC 连接
String url = "jdbc:mysql://127.0.0.1:4000/test_db?user=root&password=&useSSL=false&useUnicode=true&characterEncoding=UTF-8";
Connection conn = DriverManager.getConnection(url);
```

#### 1.2.2 Python 连接示例

```python
# 使用 pymysql
import pymysql

connection = pymysql.connect(
    host='127.0.0.1',
    port=4000,
    user='root',
    password='',
    database='test_db',
    charset='utf8mb4',
    cursorclass=pymysql.cursors.DictCursor
)

# 使用 sqlalchemy
from sqlalchemy import create_engine
engine = create_engine('mysql+pymysql://root@127.0.0.1:4000/test_db')
```

#### 1.2.3 Go 连接示例

```go
// 使用 go-sql-driver/mysql
import "database/sql"
import _ "github.com/go-sql-driver/mysql"

db, err := sql.Open("mysql", "root@tcp(127.0.0.1:4000)/test_db")
if err != nil {
    log.Fatal(err)
}
defer db.Close()
```

## 2. 数据库基础操作

### 2.1 数据库管理

#### 2.1.1 创建和删除数据库

```sql
-- 创建数据库
CREATE DATABASE my_database;

-- 指定字符集和排序规则
CREATE DATABASE my_database
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;

-- 查看数据库列表
SHOW DATABASES;

-- 查看数据库信息
SHOW CREATE DATABASE my_database;

-- 使用数据库
USE my_database;

-- 删除数据库（谨慎操作）
DROP DATABASE my_database;

-- 检查数据库是否存在再删除
DROP DATABASE IF EXISTS my_database;
```

#### 2.1.2 数据库配置查看

```sql
-- 查看当前数据库
SELECT DATABASE();

-- 查看数据库大小
SELECT
    table_schema as 'Database',
    SUM(data_length + index_length) / 1024 / 1024 as 'Size (MB)'
FROM information_schema.tables
GROUP BY table_schema;

-- 查看表数量
SELECT table_schema, COUNT(*) as table_count
FROM information_schema.tables
GROUP BY table_schema;
```

### 2.2 表的基本操作

#### 2.2.1 创建表

```sql
-- 基本表创建
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- 带注释的表
CREATE TABLE products (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL COMMENT '产品名称',
    price DECIMAL(10,2) NOT NULL COMMENT '产品价格',
    category_id INT COMMENT '分类ID',
    stock INT DEFAULT 0 COMMENT '库存数量',
    status TINYINT DEFAULT 1 COMMENT '状态：1-正常，0-下架',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    INDEX idx_category_id (category_id),
    INDEX idx_status (status)
) COMMENT '产品表';

-- 分片表创建（TiDB 特有）
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    order_no VARCHAR(32) NOT NULL,
    total_amount DECIMAL(12,2) NOT NULL,
    status TINYINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user_id (user_id),
    INDEX idx_order_no (order_no)
) SHARD_ROW_ID_BITS = 4 PRE_SPLIT_REGIONS = 3;
```

#### 2.2.2 修改表结构

```sql
-- 添加列
ALTER TABLE users ADD COLUMN phone VARCHAR(20) COMMENT '手机号码';
ALTER TABLE users ADD COLUMN age INT COMMENT '年龄' AFTER email;

-- 修改列
ALTER TABLE users MODIFY COLUMN username VARCHAR(100) NOT NULL;
ALTER TABLE users CHANGE COLUMN phone mobile VARCHAR(20) COMMENT '手机号';

-- 删除列
ALTER TABLE users DROP COLUMN age;

-- 添加索引
ALTER TABLE users ADD INDEX idx_email (email);
ALTER TABLE users ADD UNIQUE INDEX idx_username (username);

-- 删除索引
ALTER TABLE users DROP INDEX idx_email;

-- 添加外键（TiDB 支持但建议谨慎使用）
ALTER TABLE products ADD CONSTRAINT fk_category
FOREIGN KEY (category_id) REFERENCES categories(id);
```

#### 2.2.3 查看表信息

```sql
-- 查看表结构
DESCRIBE users;
SHOW COLUMNS FROM users;

-- 查看创建语句
SHOW CREATE TABLE users;

-- 查看表索引
SHOW INDEX FROM users;

-- 查看表状态
SHOW TABLE STATUS LIKE 'users';

-- 查看表分区信息（TiDB 特有）
SHOW TABLE regions FROM users;
```

### 2.3 数据类型详解

#### 2.3.1 数值类型

```sql
-- 整数类型
CREATE TABLE number_types (
    tiny_int_col TINYINT,
    small_int_col SMALLINT,
    medium_int_col MEDIUMINT,
    int_col INT,
    big_int_col BIGINT,
    bool_col BOOLEAN
);

-- 浮点类型
CREATE TABLE float_types (
    float_col FLOAT(10,2),
    double_col DOUBLE(15,4),
    decimal_col DECIMAL(20,2)
);

-- 位类型
CREATE TABLE bit_types (
    bit_col BIT(8),
    bit_64_col BIT(64)
);
```

#### 2.3.2 字符串类型

```sql
-- 字符串类型
CREATE TABLE string_types (
    char_col CHAR(10),
    varchar_col VARCHAR(100),
    text_col TEXT,
    medium_text_col MEDIUMTEXT,
    long_text_col LONGTEXT,
    binary_col BINARY(10),
    varbinary_col VARBINARY(100),
    blob_col BLOB
);

-- 枚举和集合类型
CREATE TABLE enum_set_types (
    gender ENUM('male', 'female', 'other'),
    hobbies SET('reading', 'sports', 'music', 'travel')
);
```

#### 2.3.3 日期时间类型

```sql
-- 日期时间类型
CREATE TABLE datetime_types (
    date_col DATE,
    time_col TIME,
    datetime_col DATETIME,
    timestamp_col TIMESTAMP,
    year_col YEAR
);

-- 时间函数示例
SELECT
    CURDATE() as current_date,
    CURTIME() as current_time,
    NOW() as current_datetime,
    YEAR(NOW()) as current_year,
    MONTH(NOW()) as current_month,
    DAY(NOW()) as current_day;
```

## 3. 数据操作语言 (DML)

### 3.1 插入数据

#### 3.1.1 基本插入操作

```sql
-- 单行插入
INSERT INTO users (username, email, password_hash, phone)
VALUES ('zhangsan', 'zhangsan@example.com', 'hashed_password', '13800138000');

-- 多行插入
INSERT INTO users (username, email, password_hash, phone) VALUES
('lisi', 'lisi@example.com', 'hashed_password', '13800138001'),
('wangwu', 'wangwu@example.com', 'hashed_password', '13800138002'),
('zhaoliu', 'zhaoliu@example.com', 'hashed_password', '13800138003');

-- 插入时忽略重复数据
INSERT IGNORE INTO users (username, email, password_hash)
VALUES ('zhangsan', 'zhangsan2@example.com', 'hashed_password');

-- 插入时更新重复数据
INSERT INTO users (username, email, password_hash)
VALUES ('zhangsan', 'zhangsan3@example.com', 'new_password')
ON DUPLICATE KEY UPDATE password_hash = VALUES(password_hash);
```

#### 3.1.2 从其他表插入

```sql
-- 从查询结果插入
INSERT INTO user_backup (username, email, created_at)
SELECT username, email, created_at
FROM users
WHERE created_at >= '2024-01-01';

-- 插入默认值
INSERT INTO users (username, email, password_hash)
VALUES ('newuser', 'newuser@example.com', 'password');
-- 其他字段使用默认值
```

### 3.2 查询数据

#### 3.2.1 基本查询

```sql
-- 简单查询
SELECT * FROM users;
SELECT username, email FROM users;

-- 条件查询
SELECT * FROM users WHERE id = 1;
SELECT * FROM users WHERE email LIKE '%@example.com';
SELECT * FROM users WHERE created_at >= '2024-01-01';

-- 限制结果
SELECT * FROM users LIMIT 10;
SELECT * FROM users LIMIT 5, 10; -- 跳过前5条，取10条
```

#### 3.2.2 条件查询和逻辑运算

```sql
-- 比较运算符
SELECT * FROM users WHERE id > 10;
SELECT * FROM users WHERE id BETWEEN 10 AND 20;
SELECT * FROM users WHERE id IN (1, 3, 5, 7);

-- 逻辑运算符
SELECT * FROM users WHERE id > 10 AND email LIKE '%@example.com';
SELECT * FROM users WHERE status = 0 OR status = 1;
SELECT * FROM users WHERE NOT (status = 2);

-- NULL 值处理
SELECT * FROM users WHERE phone IS NULL;
SELECT * FROM users WHERE phone IS NOT NULL;
SELECT * FROM users WHERE COALESCE(phone, '') = '';
```

#### 3.2.3 排序和分组

```sql
-- 排序
SELECT * FROM users ORDER BY created_at DESC;
SELECT * FROM users ORDER BY username ASC, created_at DESC;

-- 分组
SELECT status, COUNT(*) as count FROM users GROUP BY status;
SELECT YEAR(created_at) as year, MONTH(created_at) as month, COUNT(*) as count
FROM users GROUP BY YEAR(created_at), MONTH(created_at);

-- 分组条件
SELECT status, COUNT(*) as count
FROM users
GROUP BY status
HAVING COUNT(*) > 10;
```

#### 3.2.4 连接查询

```sql
-- 内连接
SELECT u.username, o.order_no, o.total_amount
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- 左连接
SELECT u.username, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id;

-- 多表连接
SELECT u.username, p.name as product_name, oi.quantity
FROM users u
INNER JOIN orders o ON u.id = o.user_id
INNER JOIN order_items oi ON o.id = oi.order_id
INNER JOIN products p ON oi.product_id = p.id;

-- 自连接
SELECT e1.name as employee, e2.name as manager
FROM employees e1
LEFT JOIN employees e2 ON e1.manager_id = e2.id;
```

#### 3.2.5 子查询

```sql
-- IN 子查询
SELECT * FROM users WHERE id IN (SELECT user_id FROM orders WHERE total_amount > 1000);

-- EXISTS 子查询
SELECT * FROM users u WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.status = 1
);

-- FROM 子查询
SELECT * FROM (
    SELECT
        user_id,
        COUNT(*) as order_count,
        SUM(total_amount) as total_spent
    FROM orders
    GROUP BY user_id
) as user_stats
WHERE order_count > 5;
```

### 3.3 更新和删除数据

#### 3.3.1 更新数据

```sql
-- 基本更新
UPDATE users SET phone = '13900139000' WHERE id = 1;

-- 多字段更新
UPDATE users
SET email = 'new_email@example.com',
    phone = '13900139000'
WHERE id = 1;

-- 条件更新
UPDATE users SET status = 0 WHERE last_login < '2024-01-01';

-- 基于其他表更新
UPDATE users u
SET status = 1
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id AND o.created_at >= '2024-01-01'
);
```

#### 3.3.2 删除数据

```sql
-- 基本删除
DELETE FROM users WHERE id = 1;

-- 条件删除
DELETE FROM users WHERE created_at < '2023-01-01';

-- 基于其他表删除
DELETE FROM users WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = users.id
);

-- 清空表
TRUNCATE TABLE user_backup;
```

## 4. 高级查询功能

### 4.1 聚合函数

#### 4.1.1 基本聚合函数

```sql
-- 计数函数
SELECT COUNT(*) as total_count FROM users;
SELECT COUNT(DISTINCT email) as unique_emails FROM users;
SELECT COUNT(email) as non_null_emails FROM users;

-- 求和函数
SELECT SUM(total_amount) as total_sales FROM orders;
SELECT SUM(quantity * unit_price) as total_value FROM order_items;

-- 平均值函数
SELECT AVG(total_amount) as avg_order_value FROM orders;
SELECT AVG(price) as avg_product_price FROM products;

-- 最大最小值
SELECT MAX(created_at) as latest_order FROM orders;
SELECT MIN(price) as min_product_price FROM products;
SELECT MAX(price) - MIN(price) as price_range FROM products;
```

#### 4.1.2 分组聚合

```sql
-- 按状态分组统计
SELECT
    status,
    COUNT(*) as user_count,
    COUNT(CASE WHEN created_at >= '2024-01-01' THEN 1 END) as new_users
FROM users
GROUP BY status;

-- 按时间分组
SELECT
    DATE(created_at) as order_date,
    COUNT(*) as order_count,
    SUM(total_amount) as daily_sales
FROM orders
GROUP BY DATE(created_at)
ORDER BY order_date DESC;

-- 多级分组
SELECT
    YEAR(created_at) as year,
    MONTH(created_at) as month,
    DAY(created_at) as day,
    COUNT(*) as order_count
FROM orders
GROUP BY YEAR(created_at), MONTH(created_at), DAY(created_at)
ORDER BY year, month, day;
```

### 4.2 窗口函数

#### 4.2.1 基本窗口函数

```sql
-- ROW_NUMBER() - 行号
SELECT
    id,
    username,
    email,
    ROW_NUMBER() OVER (ORDER BY created_at DESC) as row_num
FROM users;

-- RANK() 和 DENSE_RANK() - 排名
SELECT
    user_id,
    total_amount,
    RANK() OVER (ORDER BY total_amount DESC) as rank,
    DENSE_RANK() OVER (ORDER BY total_amount DESC) as dense_rank
FROM (
    SELECT user_id, SUM(total_amount) as total_amount
    FROM orders
    GROUP BY user_id
) as user_totals;

-- 窗口函数分组
SELECT
    user_id,
    order_date,
    total_amount,
    SUM(total_amount) OVER (PARTITION BY user_id ORDER BY order_date) as running_total,
    AVG(total_amount) OVER (PARTITION BY user_id) as avg_order_amount
FROM orders;
```

#### 4.2.2 分析函数

```sql
-- LAG 和 LEAD 函数
SELECT
    order_date,
    total_amount,
    LAG(total_amount, 1) OVER (ORDER BY order_date) as prev_day_amount,
    LEAD(total_amount, 1) OVER (ORDER BY order_date) as next_day_amount
FROM orders;

-- 累计和移动平均
SELECT
    order_date,
    total_amount,
    SUM(total_amount) OVER (ORDER BY order_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) as cumulative_sum,
    AVG(total_amount) OVER (ORDER BY order_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) as moving_avg_3day
FROM orders;
```

### 4.3 公用表表达式 (CTE)

#### 4.3.1 基本 CTE

```sql
-- 简单 CTE
WITH user_stats AS (
    SELECT
        user_id,
        COUNT(*) as order_count,
        SUM(total_amount) as total_spent
    FROM orders
    GROUP BY user_id
)
SELECT
    u.username,
    us.order_count,
    us.total_spent,
    CASE
        WHEN us.order_count > 10 THEN 'VIP'
        WHEN us.order_count > 5 THEN 'Regular'
        ELSE 'New'
    END as user_level
FROM users u
LEFT JOIN user_stats us ON u.id = us.user_id;
```

#### 4.3.2 递归 CTE

```sql
-- 组织结构递归查询
WITH RECURSIVE org_hierarchy AS (
    -- 基础查询：顶级管理者
    SELECT id, name, manager_id, 1 as level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- 递归查询：下属员工
    SELECT e.id, e.name, e.manager_id, oh.level + 1
    FROM employees e
    JOIN org_hierarchy oh ON e.manager_id = oh.id
)
SELECT
    id,
    name,
    manager_id,
    level,
    REPEAT('  ', level - 1) || name as formatted_name
FROM org_hierarchy
ORDER BY level, name;
```

## 5. TiDB 特有功能

### 5.1 分布式特性

#### 5.1.1 分片和 Region 信息

```sql
-- 查看表分片信息
SHOW TABLE regions FROM users;

-- 查看 Region 状态
SHOW REGIONS WHERE db_name = 'my_database';

-- 查看数据分布
SELECT
    TABLE_NAME,
    REGION_COUNT,
    REGION_SIZE,
    APPROXIMATE_KEYS
FROM information_schema.TIDB_TABLES
WHERE TABLE_SCHEMA = 'my_database';
```

#### 5.1.2 全局唯一 ID

```sql
-- 使用序列生成全局唯一 ID
CREATE SEQUENCE user_id_seq START WITH 1000 INCREMENT BY 1;

-- 使用序列
INSERT INTO users (id, username, email)
VALUES (NEXTVAL(user_id_seq), 'newuser', 'newuser@example.com');

-- 查看序列值
SELECT CURRVAL(user_id_seq);
SELECT NEXTVAL(user_id_seq);
```

### 5.2 HTAP 功能

#### 5.2.1 TiFlash 副本管理

```sql
-- 为表创建 TiFlash 副本
ALTER TABLE orders SET TIFLASH REPLICA 1;

-- 查看 TiFlash 副本状态
SHOW TABLE tiflash_replica FROM my_database;

-- 删除 TiFlash 副本
ALTER TABLE orders SET TIFLASH REPLICA 0;
```

#### 5.2.2 HTAP 查询优化

```sql
-- 强制使用 TiFlash 进行查询
SELECT /*+ TIFLASH() */ region, SUM(quantity) as total_quantity
FROM sales_data
GROUP BY region;

-- 查看查询执行计划
EXPLAIN SELECT region, SUM(quantity) as total_quantity
FROM sales_data
GROUP BY region;

-- 查看详细执行计划
EXPLAIN ANALYZE SELECT region, SUM(quantity) as total_quantity
FROM sales_data
GROUP BY region;
```

### 5.3 事务管理

#### 5.3.1 事务控制

```sql
-- 基本事务
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE user_id = 2;

-- 检查余额
SELECT balance FROM accounts WHERE user_id IN (1, 2);

COMMIT;

-- 事务回滚
BEGIN;
UPDATE products SET stock = stock - 1 WHERE id = 1;
-- 发现问题，回滚
ROLLBACK;
```

#### 5.3.2 隔离级别

```sql
-- 设置事务隔离级别
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- 查看当前隔离级别
SELECT @@transaction_isolation;

-- 使用特定隔离级别的事务
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- 事务操作
COMMIT;
```

## 6. 性能优化技巧

### 6.1 索引优化

#### 6.1.1 索引创建策略

```sql
-- 复合索引（最左前缀原则）
CREATE INDEX idx_user_status_created ON users(status, created_at);

-- 函数索引
CREATE INDEX idx_email_lower ON users((LOWER(email)));

-- 部分索引
CREATE INDEX idx_active_users ON users(username) WHERE status = 1;

-- 删除未使用的索引
DROP INDEX idx_unused_index ON users;
```

#### 6.1.2 查询优化提示

```sql
-- 使用索引提示
SELECT /*+ USE_INDEX(users, idx_username) */ * FROM users WHERE username = 'zhangsan';

-- 强制使用特定执行计划
SELECT /*+ MERGE_JOIN(t1, t2) */ * FROM table1 t1 JOIN table2 t2 ON t1.id = t2.id;

-- 避免全表扫描
SELECT /*+ TIFLASH() */ * FROM large_table WHERE condition = 'value';
```

### 6.2 查询优化

#### 6.2.1 避免常见问题

```sql
-- 避免在 WHERE 子句中使用函数（无法使用索引）
-- 不推荐
SELECT * FROM users WHERE YEAR(created_at) = 2024;

-- 推荐
SELECT * FROM users WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';

-- 避免使用 OR 连接不同的索引字段
-- 不推荐
SELECT * FROM users WHERE username = 'zhangsan' OR email = 'zhangsan@example.com';

-- 推荐
SELECT * FROM users WHERE username = 'zhangsan'
UNION
SELECT * FROM users WHERE email = 'zhangsan@example.com';
```

#### 6.2.2 分页优化

```sql
-- 传统分页（大数据量时性能差）
SELECT * FROM large_table ORDER BY id LIMIT 10000, 10;

-- 基于游标的分页（推荐）
SELECT * FROM large_table WHERE id > 10000 ORDER BY id LIMIT 10;

-- 记忆分页位置
SELECT * FROM large_table WHERE id > last_seen_id ORDER BY id LIMIT 10;
```

## 7. 实用脚本和工具

### 7.1 数据迁移脚本

#### 7.1.1 数据导出脚本

```bash
#!/bin/bash

# TiDB 数据导出脚本
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backup/tidb"
DB_NAME="my_database"

# 创建备份目录
mkdir -p $BACKUP_DIR/$DATE

# 导出数据
mysqldump -h 127.0.0.1 -P 4000 -u root $DB_NAME > $BACKUP_DIR/$DATE/$DB_NAME.sql

# 压缩备份
gzip $BACKUP_DIR/$DATE/$DB_NAME.sql

# 清理 30 天前的备份
find $BACKUP_DIR -name "*.sql.gz" -mtime +30 -delete

echo "备份完成: $BACKUP_DIR/$DATE/$DB_NAME.sql.gz"
```

#### 7.1.2 性能监控查询

```sql
-- 慢查询监控
SELECT
    query_time,
    digest_text,
    count(*) as execution_count,
    avg(query_time) as avg_time,
    max(query_time) as max_time
FROM information_schema.CLUSTER_SLOW_QUERY
WHERE query_time > 1000000  -- 1秒
GROUP BY digest_text
ORDER BY avg_time DESC
LIMIT 10;

-- 热点 Region 监控
SELECT
    table_name,
    region_id,
    read_bytes,
    write_bytes,
    written_keys
FROM information_schema.TIDB_HOT_REGIONS
ORDER BY write_bytes DESC
LIMIT 10;
```

### 7.2 常用管理操作

#### 7.2.1 表维护操作

```sql
-- 分析表
ANALYZE TABLE users;

-- 优化表
OPTIMIZE TABLE users;

-- 检查表
CHECK TABLE users;

-- 修复表（如需要）
REPAIR TABLE users;
```

#### 7.2.2 用户权限管理

```sql
-- 创建用户
CREATE USER 'app_user'@'%' IDENTIFIED BY 'secure_password';

-- 授权
GRANT SELECT, INSERT, UPDATE, DELETE ON my_database.* TO 'app_user'@'%';
GRANT ALL PRIVILEGES ON my_database.* TO 'app_user'@'%';

-- 查看权限
SHOW GRANTS FOR 'app_user'@'%';

-- 撤销权限
REVOKE DELETE ON my_database.* FROM 'app_user'@'%';

-- 删除用户
DROP USER 'app_user'@'%';
```

## 8. 总结

本教程涵盖了 TiDB 的基本操作和 SQL 使用，包括：

- **基础连接和配置**
- **数据库和表管理**
- **数据增删改查**
- **高级查询功能**
- **TiDB 特有功能**
- **性能优化技巧**

掌握这些基本操作后，可以继续学习 TiDB 的分布式特性、HTAP 功能和高级运维知识。

---

**下一步学习**：学习 TiDB 的分布式特性详解。