# SQL 索引使用与失效详解

> 本文是一份**系统、可落地的 SQL 索引使用指南**，适用于 MySQL / PostgreSQL / Oracle / SQL Server 等主流关系型数据库（个别实现细节略有差异）。

---

## 一、什么是索引（Index）

**索引 = 数据表的“目录”**，用于加速数据查找。

- 本质是一种 **有序数据结构**（B+Tree、Hash 等）
- 通过减少磁盘 I/O 和扫描行数来提升查询性能

> 没有索引的查询：**全表扫描（Full Table Scan）**
>
> 有索引的查询：**索引查找 → 回表（可能）**

---

## 二、什么时候“适合”创建索引

### 1️⃣ 经常出现在 `WHERE` 条件中的列

```sql
SELECT * FROM orders WHERE user_id = 123;
```

✅ 适合建立索引：
```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

原因：
- 过滤行数
- 缩小扫描范围

---

### 2️⃣ 经常用于 `JOIN` 的列

```sql
SELECT *
FROM orders o
JOIN users u ON o.user_id = u.id;
```

✅ 索引建议：
```sql
-- 主表
PRIMARY KEY (id)
-- 从表
INDEX (user_id)
```

---

### 3️⃣ 经常用于 `ORDER BY` 的列（尤其是大表）

```sql
SELECT * FROM logs ORDER BY create_time DESC LIMIT 20;
```

✅ 索引可避免 filesort：
```sql
CREATE INDEX idx_logs_time ON logs(create_time DESC);
```

---

### 4️⃣ 经常用于 `GROUP BY` 的列

```sql
SELECT status, COUNT(*) FROM orders GROUP BY status;
```

索引可减少分组排序开销。

---

### 5️⃣ 区分度（选择性）高的列

> 区分度 = 不重复值 / 总行数

| 字段 | 区分度 | 是否适合索引 |
|----|----|----|
| user_id | 高 | ✅ |
| email | 高 | ✅ |
| gender | 极低 | ❌ |
| status | 低 | ⚠️（视场景） |

---

## 三、什么时候“不适合”使用索引

### ❌ 1️⃣ 数据量很小的表

- 几百行以内
- 全表扫描更快

---

### ❌ 2️⃣ 更新非常频繁的列

```sql
UPDATE orders SET status = 'paid';
```

原因：
- 每次 UPDATE / INSERT / DELETE 都要维护索引
- 写入性能下降

---

### ❌ 3️⃣ 低区分度字段

```sql
WHERE gender = 'male'
```

如果 90% 都是 male：
- 索引几乎无过滤作用

---

### ❌ 4️⃣ 很少用于查询的列

- 不在 WHERE / JOIN / ORDER / GROUP 中

---

## 四、索引“失效”的常见场景（重点）

> **索引存在 ≠ 一定会被使用**

以下是面试 + 实战最常见的坑。

---

### ⚠️ 1️⃣ 在索引列上使用函数

```sql
SELECT * FROM users WHERE YEAR(create_time) = 2024;
```

❌ 索引失效

✅ 改写：
```sql
WHERE create_time >= '2024-01-01'
  AND create_time <  '2025-01-01'
```

---

### ⚠️ 2️⃣ 隐式类型转换

```sql
-- phone 是 varchar
SELECT * FROM users WHERE phone = 13800138000;
```

❌ 索引失效（发生类型转换）

✅ 正确：
```sql
WHERE phone = '13800138000';
```

---

### ⚠️ 3️⃣ 使用 `!=` 或 `<>`

```sql
WHERE status != 'deleted';
```

通常无法有效使用索引。

---

### ⚠️ 4️⃣ 使用 `OR`（部分情况）

```sql
WHERE user_id = 1 OR status = 'paid';
```

- 两列都无索引 ❌
- 只有一个有索引 ❌

✅ 可改写为 `UNION`

---

### ⚠️ 5️⃣ 模糊查询以 `%` 开头

```sql
WHERE name LIKE '%abc';
```

❌ 索引失效

```sql
WHERE name LIKE 'abc%';
```

✅ 可使用索引

---

### ⚠️ 6️⃣ 复合索引未遵守“最左前缀原则”

```sql
INDEX (a, b, c)
```

| 查询条件 | 是否用索引 |
|----|----|
| WHERE a=1 | ✅ |
| WHERE a=1 AND b=2 | ✅ |
| WHERE b=2 | ❌ |
| WHERE a=1 AND c=3 | ⚠️（部分） |

---

### ⚠️ 7️⃣ `IS NULL / IS NOT NULL`

- 不同数据库表现不同
- MySQL 8+ 可用，但选择性差时仍可能不用

---

### ⚠️ 8️⃣ 查询返回大量数据

```sql
SELECT * FROM orders WHERE status = 'paid';
```

如果返回行数占表的大比例：
- 优化器可能选择全表扫描

---

## 五、索引类型与使用场景

### 1️⃣ B+Tree（默认）

- 范围查询
- 排序
- 等值查询

适用：**90% 的场景**

---

### 2️⃣ Hash 索引

- 只支持等值
- 不支持范围

适用：
- Memory 表
- KV 精确匹配

---

### 3️⃣ 覆盖索引（Covering Index）

```sql
SELECT id, name FROM users WHERE email = 'a@b.com';
```

如果索引是 `(email, id, name)`：

✅ 不回表，性能极高

---

## 六、如何判断索引是否生效

### 使用 `EXPLAIN`

```sql
EXPLAIN SELECT * FROM users WHERE id = 1;
```

重点关注：
- `type`（最好是 ref / range / const）
- `key`（使用了哪个索引）
- `rows`（扫描行数）

---

## 七、实战索引设计原则（总结）

✅ 少而精（不是越多越好）  
✅ 为查询服务，而不是为字段服务  
✅ 写多读少 → 少索引  
✅ 读多写少 → 可多索引  
✅ 复合索引优于多个单列索引  

---

## 八、经典面试总结一句话

> **索引是用空间换时间，但不是所有查询都会走索引。**

---

如果你需要：
- 📌 **面试速记版**（一页 PDF）
- 📌 **MySQL 专属版本**（含 InnoDB 细节）
- 📌 **索引设计实战案例（订单表 / 日志表）**

告诉我你的使用场景，我可以继续细化。

