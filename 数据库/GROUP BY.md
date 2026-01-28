## 1) 最基本语法

```SQL
SELECT 分组列, 聚合函数(...)
FROM 表
WHERE 行级过滤条件
GROUP BY 分组列
HAVING 组级过滤条件
ORDER BY ...
```

**执行逻辑（非常重要）**：

1. `FROM` 取数
    
2. `WHERE` 先过滤明细行（行级）
    
3. `GROUP BY` 分组
    
4. `SELECT` 计算每组的聚合值
    
5. `HAVING` 过滤分组结果（组级）
    
6. `ORDER BY` 排序
    

---

## 2) 为什么需要 GROUP BY

你想“每个部门的人数/平均工资”这类问题时，必须分组。

### 示例：每个部门人数与平均工资

```SQL
SELECT dept_id,
       COUNT(*) AS cnt,
       AVG(salary) AS avg_salary
FROM employees
GROUP BY dept_id;
```

输出：每个 `dept_id` 一行。

---

## 3) GROUP BY 的核心规则

### 规则 A：`SELECT` 里出现的“非聚合列”必须出现在 `GROUP BY` 中

正确：

```SQL
SELECT dept_id, AVG(salary)
FROM employees
GROUP BY dept_id;
```

错误（在严格模式下会报错）：

```SQL
SELECT dept_id, last_name, AVG(salary)
FROM employees
GROUP BY dept_id;
```

原因：`last_name` 在一个部门里有很多个，不知道该显示哪一个。

> MySQL 若启用 `ONLY_FULL_GROUP_BY` 会严格报错；关闭时可能“侥幸能跑”，但非聚合列的值不确定，不建议依赖。

### 规则 B：可以按多个列分组（复合分组）

例如：按“部门 + 职位”分组统计人数：

```SQL
SELECT dept_id, job_id, COUNT(*) AS cnt
FROM employees
GROUP BY dept_id, job_id;
```

---

## 4) WHERE vs HAVING（你必须会用）

- `WHERE`：过滤明细行（聚合之前）
    
- `HAVING`：过滤分组后的结果（聚合之后），可以写聚合条件
    

### 示例：找“人数 ≥ 5”的部门

```SQL
SELECT dept_id, COUNT(*) AS cnt
FROM employees
GROUP BY dept_id
HAVING COUNT(*) >= 5;
```

如果写成 `WHERE COUNT(*) >= 5` 是不允许的。

---

## 5) 常见用法扩展

### 5.1 统计去重数量

```SQL
SELECT dept_id, COUNT(DISTINCT job_id) AS job_kinds
FROM employees
GROUP BY dept_id;
```

### 5.2 分组后排序/分页

```SQL
SELECT dept_id, AVG(salary) AS avg_salary
FROM employees
GROUP BY dept_id
ORDER BY avg_salary DESC
LIMIT 5;
```

---

## 6) MySQL 常见分组扩展：`WITH ROLLUP`

生成小计/总计（MySQL 支持）：

```SQL
SELECT dept_id, COUNT(*) AS cnt
FROM employees
GROUP BY dept_id WITH ROLLUP;
```

结果会额外多一行 `dept_id = NULL`，代表全表总计。


**1.实例**
```SQL
SELECT department_id,job_id,AVG(salary)
FROM employees
GROUP BY department_id,job_id;
```
**2.实例**
```SQL
SELECT job_id,department_id,AVG(salary)
FROM employees
GROUP BY job_id,department_id;
```
这两个都是一样的效果，区别只是谁前谁后的问题，因为分组条件是一样的
**3.实例**
```SQL
SELECT job_id,department_id,AVG(salary)
FROM employees
GROUP BY department_id;
```
会出现错误，因为一个department里有多个工种

**！！！ SELECT中出现的非函数字段一定要声明在GROUP BY中，相反，GROUP BY中声明的字段不一定要出现在SELECT中**

**！！！ GROUP BY出现在FROM、WHERE 后面，出现在ORDER BY、LIMIT前面**