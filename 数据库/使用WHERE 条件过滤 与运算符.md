**1.WHERE条件过滤**
```SQL
SELECT * FROM 表名
WHERE 条件；
```

**2.算数运算符**

### SQL 算术运算符表

| 运算符   | 名称   | 功能说明       | 示例                  | 示例结果      |
| ----- | ---- | ---------- | ------------------- | --------- |
| `+`   | 加法   | 两个数相加      | `SELECT 10 + 3;`    | `13`      |
| `-`   | 减法   | 左减右        | `SELECT 10 - 3;`    | `7`       |
| `*`   | 乘法   | 两个数相乘      | `SELECT 10 * 3;`    | `30`      |
| `/`   | 除法   | 左除右（可能为小数） | `SELECT 10 / 3;`    | `3.3333…` |
| `DIV` | 整数除法 | 取整除结果      | `SELECT 10 DIV 3;`  | `3`       |
| `%`   | 取模   | 取余数        | `SELECT 10 % 3;`    | `1`       |
| `MOD` | 取模   | 与 `%` 等价   | `SELECT MOD(10,3);` | `1`       |
常见使用：
```SQL
-- 计算总价
SELECT price * quantity AS total_price FROM orders;

-- 分页计算页码
SELECT id, (id DIV 10) AS page_no FROM items;

-- 奇偶判断
SELECT id FROM users WHERE id % 2 = 0;
```
在SQL中
```SQL
SELECT 100+'1' FROM DAUL;//结果是101，会讲字符串转换为数值（隐式转换）
//如果转换不成功，就当成0
SELECT 1='1',0='a' FROM DAUL;//结果是true，true

SELECT 100+NULL FROM DAUL;//结果是NULL
```

---
### 比较运算符

### SQL 比较运算符表

| 运算符                   | 名称        | 功能说明        | 示例                          | 示例结果         |
| --------------------- | --------- | ----------- | --------------------------- | ------------ |
| `=`                   | 等于        | 判断是否相等      | `age = 18`                  | true / false |
| `!=` 或 `<>`           | 不等于       | 判断是否不相等     | `age != 18`                 | true / false |
| `>`                   | 大于        | 左值是否大于右值    | `score > 90`                | true / false |
| `<`                   | 小于        | 左值是否小于右值    | `score < 60`                | true / false |
| `>=`                  | 大于等于      | 左值是否 ≥ 右值   | `price >= 100`              | true / false |
| `<=`                  | 小于等于      | 左值是否 ≤ 右值   | `count <= 10`               | true / false |
| `BETWEEN … AND …`     | 区间判断      | 是否在区间内（含端点） | `age BETWEEN 18 AND 30`     | true / false |
| `NOT BETWEEN … AND …` | 非区间判断     | 不在区间内       | `age NOT BETWEEN 18 AND 30` | true / false |
| `IN (…)`              | 集合判断      | 是否属于集合      | `id IN (1,3,5)`             | true / false |
| `NOT IN (…)`          | 非集合判断     | 不属于集合       | `id NOT IN (1,3,5)`         | true / false |
| `LIKE`                | 模糊匹配      | 使用通配符匹配     | `name LIKE 'A%'`            | true / false |
| `NOT LIKE`            | 非模糊匹配     | 不匹配         | `name NOT LIKE '%test%'`    | true / false |
| `IS NULL`             | 判空        | 是否为 NULL    | `email IS NULL`             | true / false |
| `IS NOT NULL`         | 非空        | 是否不为 NULL   | `email IS NOT NULL`         | true / false |
| `<=>`                 | NULL 安全等于 | NULL 也可比较   | `a <=> b`                   | true / false |
！！！NULL不能用 **`=`** 比较
```SQL
col = NULL      -- 永远为 NULL（不成立）
col IS NULL     -- 正确
SELECT NULL=NULL FROM DAUL;//结果依然是NULL
```

**IS NULL/IS NOT NULL/ISNULL(函数)**

```SQL
SELECT * FROM 表名
WHERE id IS NULL;//寻找id为NULL的行做匹配

SELECT * FROM 表名
WHERE id IS NOT NULL;//寻找id不为NULL的行做匹配

SELECT * FROM 表名
WHERE ISNULL(id);//和IS NULL作用一样，但这个是函数
```

**LEAST与GREATEST**

```SQL
LEAST(expr1, expr2, ...)
GREATEST(expr1, expr2, ...)
```
**作用概述

- **`LEAST(a, b, …)`**：返回参数中**最小值**
    
- **`GREATEST(a, b, …)`**：返回参数中**最大值**
    

📌 常用于：

- 取上下限
    
- 业务阈值裁剪（clamp）
    
- 多列/多值比较

**模糊查询**

基本语法：
```SQL
SELECT 列名 FROM 表名 WHERE 列名 LIKE pattern；
expr NOT LIKE pattern
```
### SQL `LIKE` 通配符表

| 通配符                   | 含义      | 匹配长度     | 示例模式     | 可匹配示例              | 不匹配示例       |
| --------------------- | ------- | -------- | -------- | ------------------ | ----------- |
| `%`                   | 任意字符序列  | 0 个或多个字符 | `'ab%'`  | `ab`、`abc`、`abXYZ` | `a`、`xabc`  |
| `_`                   | 任意单个字符  | 恰好 1 个字符 | `'a_c'`  | `abc`、`a1c`、`a_c`  | `ac`、`abcc` |
| `\%`（配合 `ESCAPE '\'`） | 字面量 `%` | 1 个字符    | `'a\%b'` | `a%b`              | `ab`、`aXXb` |
| `\_`（配合 `ESCAPE '\'`） | 字面量 `_` | 1 个字符    | `'a\_b'` | `a_b`              | `ab`、`a1b`  |

说明：最后两行是“转义用法”。MySQL 常用 `ESCAPE '\'`（或默认反斜杠转义，视 SQL_MODE/实
现而定）。你也可以用别的转义符，例如 `ESCAPE '!'`，那就写成 `!%`、`!_`。

## 二、`$ ^ .` 在正则里的含义

REGEXP

| 符号     | 含义     | 作用说明                   | 示例        |
| ------ | ------ | ---------------------- | --------- |
| `^`    | 行首锚点   | 匹配字符串**开头**（以该字符后面的字符） | `'^abc'`  |
| `$`    | 行尾锚点   | 匹配字符串**结尾**（以该字符前面的字符） | `'abc$'`  |
| `.`    | 任意单个字符 | 匹配任意 1 个字符（不含换行）       | `'a.c'`   |
| [....] | 匹配任意字符 | 匹配[]内任意的字符             | '[a,b,c]' |
```SQL
SELECT id FROM employee
WHERE id LIKE '1%';  //ID 1开头

SELECT id FROM employee
WHERE id LIKE '_1%';//ID第二位是1

SELECT * FROM employee
WHERE id REGEXP '^12';//匹配以12开头的字符
```
### SQL 逻辑运算符表

| 运算符   | 名称            | 功能说明      | 示例                                | 说明       |
| ----- | ------------- | --------- | --------------------------------- | -------- |
| `AND` | 逻辑与           | 两个条件同时为真  | `age > 18 AND status = 'active'`  | 取条件交集    |
| `OR`  | 逻辑或           | 任一条件为真    | `role = 'admin' OR role = 'user'` | 取条件并集    |
| `NOT` | 逻辑非           | 对条件取反     | `NOT (status = 'disabled')`       | 取条件补集    |
| `XOR` | 异或（MySQL）     | 仅一边为真     | `a XOR b`                         | MySQL 扩展 |
| `&&`  | AND 别名（MySQL） | 等价于 `AND` | `a && b`                          | 不推荐      |
| `     |               | `         | OR 别名（MySQL）                      | 等价于 `OR` |
运算优先级；
```SQL
NOT  >  AND  >  OR
```
XOR：
```SQL
SELECT id, email, phone
FROM users
WHERE (email IS NOT NULL) XOR (phone IS NOT NULL)
ORDER BY id;
```
### 输出（结果集）

| id  | email   | phone |
| --- | ------- | ----- |
| 1   | a@x.com | NULL  |
| 2   | NULL    | 123   |
### SQL 位运算符表（MySQL）

|运算符|名称|功能说明|示例|示例结果|
|---|---|---|---|---|
|`&`|按位与|两数对应二进制位都为 1 才为 1|`SELECT 6 & 3;`|`2`（110 & 011 = 010）|
|`|`|按位或|对应位有 1 即为 1|`SELECT 6 \| 3;`|
|`^`|按位异或|对应位不同为 1|`SELECT 6 ^ 3;`|`5`（110 ^ 011 = 101）|
|`~`|按位取反|所有位翻转（受整数位宽影响）|`SELECT ~1;`|`-2`（二补码取反）|
|`<<`|左移|左移 n 位，低位补 0|`SELECT 3 << 2;`|`12`（0011→1100）|
|`>>`|右移|右移 n 位，高位补符号位（算术右移）|`SELECT 12 >> 2;`|`3`（1100→0011）|

---

### 补充说明（非常重要）

1. **位运算对象是整数**  
    对字符串会发生隐式转换（不推荐）。
    
2. `~` 的结果为什么是负数？  
    MySQL 整数使用二补码表示，`~x` 等价于 `-(x+1)`（在常见实现/位宽语义下），所以 `~1 = -2` 是常见结果。
    
3. NULL 参与位运算  
    只要有 `NULL`，结果就是 `NULL`：
    
```SQL
SELECT 1 & NULL;  -- NULL
```


4. 常见用途
    

- 权限位（bitmask）判断：
    
    ```SQL
    WHERE (perm & 4) != 0   -- 是否包含第 3 位权限
    ```
    
- 状态位组合/筛选