### 基本语法
```SQL
SELECT ...
FROM ...
WHERE  ...          -- 行级过滤（聚合前）
GROUP BY ...
HAVING ...          -- 组级过滤（聚合后）
ORDER BY ...
```
`HAVING` 用来**过滤分组后的结果**，也就是对 `GROUP BY` 产生的“每组汇总行”再做条件限制。它和 `WHERE` 的核心区别是：**`WHERE` 过滤明细行，`HAVING` 过滤分组行**。

**1.实例**
```SQL
SELECT department_id,MAX(salary)
FROM employees
GROUP BY department_id
HAVING MAX(salary)>10000;
```

**2.1实例**
```SQL
SELECT department_id,MAX(salary)
FROM employees
WHERE department_id IN (10,20,30,40)
GROUP BY department_id
HAVING MAX(salary)>10000;
```
**2.2实例**
```SQL
SELECT department_id,MAX(salary)
FROM employees
GROUP BY department_id
HAVING MAX(salary)>10000 AND department_id IN (10,20,30,40);
```
**!!!二者效果等价**
