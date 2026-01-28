SELECT语句骨架：
```SQL
SELECT [DISTINCT] select_list
FROM table_source
[JOIN ... ON join_condition]
[WHERE row_condition]
[GROUP BY group_list]
[HAVING group_condition]
[ORDER BY order_list]
[LIMIT offset, count];
```


**1.伪表：**
```SQL
SELECT * FROM DUAL; //DUAL是伪表
```

**2.基本查询语句**
```SQL
SELECT 列名 FROM 表名；
//example
SELECT * FROM employee;

SELECT lastname,salary,employee_id from employee;
```
\*代表了表的所有列（属性）

**3.列的别名**

列的别名有3种形式：
```SQL
//1.空格表示
SELECT 列名 别名 FROM 表名；

SELECT employee_id emp_id,salary,department_id
FROM employee;


//2.空格加双引号
SELECT 列名 "别名" FROM 表名；

SELECT employee_id "emp_id",salary,department_id
FROM employee;

//3.显式符号AS(alias)
SELECT 列名 AS 别名 FROM 表名；

SELECT employee_id AS emp_id,salary*12 AS anual_sal,department_id AS dmp_id
FROM employee;
```


**4.向表中插入：**
```SQL
INSERT INTO emp
VALUES (1002,'TOM');
```


**5.去除重复行：**
```SQL
SELECT department_id;//这样会出现重复行

SELECT DISTINCT department_id;//正确的
```


**6.空值参与运算**
空值：NULL 不等于 0 ，‘’，‘null’

```SQL

SELECT salary*(1+commition)*12 AS "年工资"
FROM employee;//会有错误，年工资列有些值可能为NULL，如果commition有NULL值的话 

SELECT salary*(1+IFNULL(commition,0))*12 AS "年工资"
FROM employee；//正确的
```

**7.着重号（~ ）**
如果字段名和关键字重复的情况使用

```SQL
SELECT ORDER FROM employee;//错误的，ORDER是关键字

SELECT `order` FROM employee;//正确
```

**8.查询常数：**
常数自动匹配
```SQL
SELECT 列名,常数 FROM 表名；

SELECT name,123 FROM employee;
```

**9.显示表结构**

```SQL
DESCRIBE 表名；

//或者
DES 表名；
```