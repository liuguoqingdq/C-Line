### 添加数据

**方式1:一条一条田间数据**
```SQL
//没有指明添加字段
INSERT INTO 表名
VALUES (字段数据);//需要按照表字段声明的顺序添加


//指明字段
INSERT INTO emp(id,hire,date,salary,`name`)
VALUES (emp(...)字段顺序的值)


//同时插入多条记录
INSERT INTO emp(id,hire,date,salary,`name`)
VALUES 
(emp(...)字段顺序的值)
(emp(...)字段顺序的值)
```

**方式2:将查询结果插入表中**
```SQL
INSERT INTO emp(id,NAME,salary,hire_date)
SELECT id,NAME,salary,hire_date
FROM employees
WHERE department_id IN (60,70 )
```

### 更新数据
```SQL
UPDATE emp
SET hire_date = CURDATE()
WHERE hire_date = 5;//一般要加WHERE
```

### 删除数据
```SQL
DELETE FROM emp
WHERE id=1;
```

### 计算列
```SQL
//算数计算
SELECT
  id,
  price,
  qty,
  price * qty AS amount,                 -- 金额
  (price * qty) * 0.08 AS tax,            -- 税
  (price * qty) * (1 - discount_rate) AS pay_amount
FROM orders;



//处理 NULL：用 COALESCE/IFNULL
SELECT
  id,
  price,
  COALESCE(qty, 0) AS qty2,
  price * COALESCE(qty, 0) AS amount
FROM orders;

```

**练习**
```SQL
//创建表
USE db;
create TABLE IF NOT EXISTS employees(
id INT(10);
first_name varchar(10);
last_name varchar(10);
userid varchar(10);
salary DOUBLE(10,2);
);

create TABLE IF NOT EXISTS users(
id INT(10);
userid varchar(10);
department_id INT;
);
```