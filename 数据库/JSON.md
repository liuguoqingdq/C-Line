# MySQL JSON 类型速查表 (MySQL 5.7+)

## 一、JSON 类型是什么

JSON 是 **原生二进制存储的结构化类型**（不是 TEXT），具备以下特性：

- **自动校验**合法 JSON
    
- **二进制压缩存储**（非文本）
    
- 支持路径访问、函数操作
    
- 可配合 **生成列 + 索引** 做高效查询
    


```SQL
CREATE TABLE t (
  id BIGINT PRIMARY KEY,
  attrs JSON
);
```

---

## 二、JSON 写入方式（INSERT / UPDATE）

### 2.1 直接写 JSON 字面量

必须是合法 JSON，否则报错。


```SQL
INSERT INTO t VALUES (
  1,
  '{"ip":"1.1.1.1","port":80,"tags":["web","prod"]}'
);
```

### 2.2 使用 JSON 函数构造（推荐）

优点：避免手写 JSON 出错。


```SQL
INSERT INTO t VALUES (
  2,
  JSON_OBJECT(
    'ip', '1.1.1.1',
    'port', 80,
    'tags', JSON_ARRAY('web','prod')
  )
);
```

### 2.3 更新 JSON 的一部分


```SQL
UPDATE t
SET attrs = JSON_SET(attrs, '$.port', 443)
WHERE id = 1;
```

**常用更新函数：**

- `JSON_SET()`：设置/覆盖
    
- `JSON_INSERT()`：不存在才插入
    
- `JSON_REPLACE()`：存在才替换
    
- `JSON_REMOVE()`：删除键
    

---

## 三、JSON 读取与路径语法（最常用）

### 3.1 基本路径访问


```SQL
SELECT
  attrs->'$.ip'        AS ip_json,
  attrs->>'$.ip'       AS ip_text,
  attrs->>'$.port'     AS port
FROM t;
```

|**操作符**|**返回类型**|**说明**|
|---|---|---|
|**`->`**|JSON|包含引号（对象/数组）|
|**`->>`**|TEXT|**去引号**（值），最常用|

### 3.2 路径语法速记

|**路径**|**含义**|
|---|---|
|**`$`**|根|
|**`$.a`**|对象字段 a|
|**`$.a.b`**|嵌套字段|
|**`$.arr[0]`**|数组第 0 个|
|**`$.arr[*]`**|数组所有元素|

---

## 四、条件查询（WHERE 里怎么用）

### 4.1 普通等值判断


```SQL
SELECT * FROM t
WHERE attrs->>'$.ip' = '1.1.1.1';
```

### 4.2 判断是否包含（数组/对象）


```SQL
SELECT * FROM t
WHERE JSON_CONTAINS(attrs, '"web"', '$.tags');
```

### 4.3 判断键是否存在


```SQL
WHERE JSON_EXTRACT(attrs, '$.port') IS NOT NULL;
```

---

## 五、JSON 与索引（性能核心）

### 5.1 不能直接给 JSON 建普通索引

下面是**错误思路**：


```SQL
CREATE INDEX idx_json ON t(attrs); -- ❌ 不可行
```

### 5.2 正确做法：生成列 + 索引（必会）


```SQL
ALTER TABLE t
ADD COLUMN ip VARCHAR(15)
  GENERATED ALWAYS AS (attrs->>'$.ip') STORED,
ADD INDEX idx_ip (ip);
```

**之后查询：**


```SQL
SELECT * FROM t WHERE ip = '1.1.1.1';
```

- ✅ 能走索引
    
- ✅ 性能与普通列一致
    

---

## 六、JSON 常用函数速查

|**函数**|**作用**|
|---|---|
|`JSON_OBJECT()`|构造对象|
|`JSON_ARRAY()`|构造数组|
|`JSON_EXTRACT()`|提取|
|`JSON_SET()`|设置/覆盖|
|`JSON_INSERT()`|不存在才插|
|`JSON_REPLACE()`|存在才改|
|`JSON_REMOVE()`|删除|
|`JSON_CONTAINS()`|包含判断|
|`JSON_LENGTH()`|元素个数|
|`JSON_KEYS()`|键列表|
|`JSON_TYPE()`|类型|

---

## 七、JSON vs 关系型字段（如何选）

**适合 JSON 的场景：**

- 字段结构不稳定（可选属性多）
    
- 配置、扩展属性
    
- 事件/日志属性
    

**不适合 JSON 的场景：**

- 高频查询、排序、范围查询
    
- 强一致性约束（外键、唯一性）
    
- 明确结构、稳定字段（应拆表）
    

**经验法则：**

> “写多读少 + 结构松散” → JSON
> 
> “读多 + 条件复杂 + 需要索引” → 拆字段

---

## 八、旧版本与兼容性（你关心的点）

|**版本**|**情况**|
|---|---|
|**MySQL 5.6**|❌ 没有 JSON 类型（只能 TEXT）|
|**MySQL 5.7**|✅ 引入 JSON（功能已可用）|
|**MySQL 8.0**|✅ JSON 最成熟（函数、索引友好）|

如果你被迫用 5.6：

SQL

```
attrs TEXT CHECK (JSON_VALID(attrs))
```

_（仅“部分模拟”，无二进制存储与函数优化）_

---

## 九、一个“工程级”完整示例


```SQL
CREATE TABLE event_log (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  type VARCHAR(32) NOT NULL,
  attrs JSON NOT NULL,
  -- 提取核心字段用于索引
  ip VARCHAR(15)
    GENERATED ALWAYS AS (attrs->>'$.ip') STORED,
  created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  -- 建立索引
  INDEX idx_ip (ip)
);
```

---

## 十、最容易踩的 6 个坑

1. 把 JSON 当 TEXT 用（丢索引、慢）。
    
2. 在 WHERE 里对 JSON 路径做函数而不建生成列。
    
3. 用 JSON 存“核心高频字段”。
    
4. 忘了 `->>`（导致字符串比较失败）。
    
5. JSON 数字/字符串类型混用。
    
6. 在 MySQL 5.7 里期待 8.0 的高级行为。