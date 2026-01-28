**建表语句**：
```SQL
CREATE TABLE `employees` (
`emp_no` int(11) NOT NULL,
`birth_date` date NOT NULL,
`first_name` varchar(14) NOT NULL,
`last_name` varchar(16) NOT NULL,
`gender` char(1) NOT NULL,
`hire_date` date NOT NULL,
PRIMARY KEY (`emp_no`));
```
**找到最晚的入职日期**
```SQL
select max(hire_date) from employees;
```
**找到入职最晚的那个员工**
```SQL
select * 
from employees
where hire_date = (
select max(hire_date) from employees
);
```
**查找入职员工时间倒数第三的员工**
```SQL
select *
from employees
where hire_date = (
select hire_date 
from employees
order by hire_date desc
limit 2,1
);
```

**查找各部门领导当前的薪水，以及部门编号**：
```SQL
CREATE TABLE `dept_manager` (
`dept_no` char(4) NOT NULL comment '部门编号',
`emp_no` int(11) NOT NULL comment '员工编号',
`from_date` date NOT NULL,
`to_date` date NOT NULL,
PRIMARY KEY (`emp_no`,`dept_no`));

CREATE TABLE `salaries` (
`emp_no` int(11) NOT NULL comment '员工编号',
`salary` int(11) NOT NULL,
`from_date` date NOT NULL,
`to_date` date NOT NULL,
PRIMARY KEY (`emp_no`,`from_date`));
```

```SQL
select s.salary,d.dept_no
from salaries s
JOIN dept_manager d
ON d.`emp_no` = s.`emp_no`
where s.to_date='9999-01-01' and d.to_date = '9999-01-01';
```

**查找已经分配部门的员工的 last_name,first_name,还有部门dept_no:**
```SQL
CREATE TABLE `dept_emp` (
`emp_no` int(11) NOT NULL,
`dept_no` char(4) NOT NULL,
`from_date` date NOT NULL,
`to_date` date NOT NULL,
PRIMARY KEY (`emp_no`, `dept_no`));

CREATE TABLE `employees` (
`emp_no` int(11) NOT NULL,
`birth_date` date NOT NULL,
`first_name` varchar(14) NOT NULL,
`last_name` varchar(16) NOT NULL,
`gender` char(1) NOT NULL,
`hire_date` date NOT NULL,
PRIMARY KEY (`emp_no`));
```

```SQL
select e.last_name,e.first_name,d.dept_no
from dept_emp d
INNER JOIN employees e
ON d.`emp_no` = e.`emp_no`;
```
**查找所有员工的last_name,first_name和dept_no**
```SQL
select e.last_name,e.first_name,d.dept_no
from employees e
LEFT JOIN dept_emp
ON e.`emp_no` = d.`emp_no`;
```

**查询所有员工刚入职的薪水，给出salary，与emp_no,并且按照emp_no进行逆序一个员工可能有多个salaires记录**
```SQL
CREATE TABLE `employees` (
`emp_no` int(11) NOT NULL,
`birth_date` date NOT NULL,
`first_name` varchar(14) NOT NULL,
`last_name` varchar(16) NOT NULL,
`gender` char(1) NOT NULL,
`hire_date` date NOT NULL,
PRIMARY KEY (`emp_no`));

CREATE TABLE `salaries` (
  `emp_no` int(11) NOT NULL,
  `salary` int(11) NOT NULL,
  `from_date` date NOT NULL,
  `to_date` date NOT NULL,
  PRIMARY KEY (`emp_no`, `from_date`)
);
```

```SQL
select s.salar,e.emp_no
from employees e
LEFT JOIN salaries s
ON e.`emp_no` = s.`emp_no`
where e.hire_date = s.from_date
ORDER BY e.emp_no desc;
```

**查询薪水上涨次数超过15次的员工emp_no,以及上涨次数：**
```SQL
CREATE TABLE `salaries` (
  `emp_no` int(11) NOT NULL,
  `salary` int(11) NOT NULL,
  `from_date` date NOT NULL,
  `to_date` date NOT NULL,
  PRIMARY KEY (`emp_no`, `from_date`)
);

select emp_no,count(salary) t
from salaries
GROUP BY emp_no
HAVING count(salary) >= 15;
```

**找出所有员工的薪资，并且去重，按照降序排序**
```SQL
select distinct salary
from salaries
where salaries.hire_date = '9999-01-01'
order by salary desc;

//或者
select salary
from salaries
where salaries.hire_date = '9999-01-01'
group by salary;
```

**获取所有非manager的员工emp_no**
```SQL
CREATE TABLE `dept_manager` (
`dept_no` char(4) NOT NULL comment '部门编号',
`emp_no` int(11) NOT NULL comment '员工编号',
`from_date` date NOT NULL,
`to_date` date NOT NULL,
PRIMARY KEY (`emp_no`,`dept_no`));

CREATE TABLE `employees` (
`emp_no` int(11) NOT NULL,
`birth_date` date NOT NULL,
`first_name` varchar(14) NOT NULL,
`last_name` varchar(16) NOT NULL,
`gender` char(1) NOT NULL,
`hire_date` date NOT NULL,
PRIMARY KEY (`emp_no`));
```

```SQL
select e.emp_no
from employees e
where NOT EXISTS (
select 1
from dept_manager d
where e.emp_no=d.emp_no
);
```


**获取所有部门中当前员工最高薪水，给出dept_no,emp_no,salary**
```SQL
CREATE TABLE `salaries` (
  `emp_no` int(11) NOT NULL,
  `salary` int(11) NOT NULL,
  `from_date` date NOT NULL,
  `to_date` date NOT NULL,
  PRIMARY KEY (`emp_no`, `from_date`)
);

CREATE TABLE `dept_emp` (
`emp_no` int(11) NOT NULL,
`dept_no` char(4) NOT NULL,
`from_date` date NOT NULL,
`to_date` date NOT NULL,
PRIMARY KEY (`emp_no`, `dept_no`));
```

```SQL
select d.dept_no,d.emp_no,max(s.salary) max_sal
from dept_emp d
left join salaries s
on d.emp_no = s.emp_no
where d.to_date = '9999-01-01' and s.to_date = '9999-01-01'
GROUP BY d.dept_no;
```



**从titles表获取title，并分组，每组数量大于等于2，给出分组与数量**
```SQL
CREATE TABLE IF NOT EXISTS "titles" (
 `emp_no` int(11) NOT NULL,
 `title` varchar(50) NOT NULL,
 `from_date` date NOT NULL,
 `to_date` date DEFAULT NULL); 
```

```SQL
select tile,count(title) t
from titles
group by title
having t >= 2;
```
**对重复的emp_no去重**
```SQL
select tile,count(distinct title) t
from titles
group by title
having t >= 2;
```


**定义数据表tb_emp5,并在tb_emp5表上创建外键约束**
```SQL
create table tb_dept1(
	id  INT primary key,
	name varchar(22) not null,
	location varchar(50)
);

create table tb_emp5(
	id INT primary key,
	name varchar(25),
	deptID INT,
	salary DECIMAL(10,2),
	constraint fk_dept_emp foreign key(deptID) references tb_dept1(id)
);
```

**修改表表名**
```SQL
alter table old_name rename to new_name;
```

**修改表中字段类型**
```SQL
alter table tb_dept1
modify name varchar(30);
```

**修改表中字段名**
```SQL
alter table tb_dept1
change location loc varchar(50);
```

**添加有完整性约束条件的字段**
```SQL
alter table tb_dept1
add column1 varchar(12) not null;
```

**在表中第一列添加一个字段**
```SQL
alter table tb_dept1
add column2 INT first;
```

**在指定列之后添加字段**
```SQL
alter table tb_dept1
add column3 int after name;
```

**删除字段**
```SQL
alter table tb_dept1
drop column1;
```

**修改字段为表的第一个字段**
```SQL
alter table tb_dept1
modify column2 varchar(25) first;
```

**修改字段到指定字段之后**
```SQL
alter table tb_dept1
modify column2 varchar(25) after location;
```


**统计出各title对应的当前是日期员工对应的平均工资**
```SQL
CREATE TABLE `salaries` (
`emp_no` int(11) NOT NULL,
`salary` int(11) NOT NULL,
`from_date` date NOT NULL,
`to_date` date NOT NULL,
PRIMARY KEY (`emp_no`, `from_date`));

CREATE TABLE IF NOT EXISTS "titles" (
`emp_no` int(11) NOT NULL,
`title` varchar(50) NOT NULL,
`from_date` date NOT NULL,
`to_date` date DEFAULT NULL);
```

```SQL
select t.title,t.to_date,avg(s.salary) avg_sal
from titles t
JOIN salaries s
ON t.emp_no = s.emp_no
where t.to_date = '9999-01-01' and s.to_date = '9999-01-01'
group by t.title;
```

**查询薪资第二多的员工**
```SQL
CREATE TABLE `salaries` (
`emp_no` int(11) NOT NULL,
`salary` int(11) NOT NULL,
`from_date` date NOT NULL,
`to_date` date NOT NULL,
PRIMARY KEY (`emp_no`, `from_date`));
```

```SQL
select emp_no,salary
from salaries
where to_date = '9999-01-01' and salary = (
	select distinct salary
	from salaries
	order by salary 
	limit 1,1
);
```


**对所有员工的薪水，1-N排序,相同salary按照emp_no排序**
```SQL
SELECT s1.emp_no,
       s1.salary,
       (
         SELECT COUNT(DISTINCT s2.salary)
         FROM salaries s2
         WHERE s2.to_date = '9999-01-01'
           AND s2.salary > s1.salary
       ) + 1 AS ranking
FROM salaries s1
WHERE s1.to_date = '9999-01-01';
```
## 四、为什么要用 COUNT(DISTINCT salary)？

这是为了**实现“1–N 排名（RANK）”的数学定义**。

### 1–N 排名的本质公式

> **rank = 比它大的“不同值”的个数 + 1**

|salary|比它大的不同 salary|个数|rank|
|---|---|---|---|
|9000|无|0|1|
|8500|{9000}|1|2|
|8000|{9000, 8500}|2|3|

👉 这正是 `RANK()` 的行为。


**获取当前非manager员工的薪水，给出emp_no、dept_no以及salary**
```SQL
CREATE TABLE `salaries` (
`emp_no` int(11) NOT NULL,
`salary` int(11) NOT NULL,
`from_date` date NOT NULL,
`to_date` date NOT NULL,
PRIMARY KEY (`emp_no`, `from_date`));
CREATE TABLE `dept_emp` (
`emp_no` int(11) NOT NULL,
`dept_no` char(4) NOT NULL,
`from_date` date NOT NULL,
`to_date` date NOT NULL,
PRIMARY KEY (`emp_no`, `dept_no`));

CREATE TABLE `dept_manager` (
`dept_no` char(4) NOT NULL comment '部门编号',
`emp_no` int(11) NOT NULL comment '员工编号',
`from_date` date NOT NULL,
`to_date` date NOT NULL,
PRIMARY KEY (`emp_no`,`dept_no`));

CREATE TABLE `employees` (
`emp_no` int(11) NOT NULL,
`birth_date` date NOT NULL,
`first_name` varchar(14) NOT NULL,
`last_name` varchar(16) NOT NULL,
`gender` char(1) NOT NULL,
`hire_date` date NOT NULL,
PRIMARY KEY (`emp_no`));
```

```SQL
select e.emp_no,d.dept_no,s.salary
from employees e
join dept_emp d
on e.`emp_no` = d.`emp_no` and d.to_date = '9999-01-01'
join salaries s
on e.`emp_no` = s.`emp_no` and s.to_date = '9999-01-01'
where not exists (
	select 1
	from dept_manager m
	where to_date = '9999-01-01' and e.emp_no = m.emp_no
) and e.hire_date='9999-01-01';
```
not exists 只有当子查询“一行都查不出来”时，外层这一行才保留



**获取当前员工工资比其manager工资还高的，第一列emp_no,第二列manager_no，第三列emp_salary,第四列manager_salary**
```SQL
CREATE TABLE `dept_emp` (
`emp_no` int(11) NOT NULL,
`dept_no` char(4) NOT NULL,
`from_date` date NOT NULL,
`to_date` date NOT NULL,
PRIMARY KEY (`emp_no`, `dept_no`));

CREATE TABLE `dept_manager` (
`dept_no` char(4) NOT NULL,
`emp_no` int(11) NOT NULL,
`from_date` date NOT NULL,
`to_date` date NOT NULL,
PRIMARY KEY (`emp_no`, `dept_no`));

CREATE TABLE `salaries` (
`emp_no` int(11) NOT NULL,
`salary` int(11) NOT NULL,
`from_date` date NOT NULL,
`to_date` date NOT NULL,
PRIMARY KEY (`emp_no`, `from_date`));
```


```SQL
SELECT
  s.emp_no AS emp_no,
  d.emp_no AS manager_no,
  s.salary AS emp_salary,
  (
    SELECT ms.salary
    FROM salaries ms
    WHERE ms.emp_no = d.emp_no
      AND ms.to_date = '9999-01-01'
  ) AS manager_salary
FROM salaries s
JOIN dept_emp de
  ON de.emp_no = s.emp_no
 AND de.to_date = '9999-01-01'
JOIN dept_manager d
  ON d.dept_no = de.dept_no
 AND d.to_date = '9999-01-01'
WHERE s.to_date = '9999-01-01'
  AND s.salary > (
    SELECT ms.salary
    FROM salaries ms
    WHERE ms.emp_no = d.emp_no
      AND ms.to_date = '9999-01-01'
  );
```
**推荐写法**
```SQL
SELECT
  e.emp_no        AS emp_no,
  m.emp_no        AS manager_no,
  es.salary       AS emp_salary,
  ms.salary       AS manager_salary
FROM dept_emp e
JOIN dept_manager m
  ON e.dept_no = m.dept_no
 AND m.to_date = '9999-01-01'
JOIN salaries es
  ON es.emp_no = e.emp_no
 AND es.to_date = '9999-01-01'
JOIN salaries ms
  ON ms.emp_no = m.emp_no
 AND ms.to_date = '9999-01-01'
WHERE e.to_date = '9999-01-01'
  AND es.salary > ms.salary;
```



**汇总各个部门当前员工的title分配数目，给出dept_no,dept_name,title数目**
```SQL
CREATE TABLE `departments` (
  `dept_no` char(4) NOT NULL,
  `dept_name` varchar(40) NOT NULL,
  PRIMARY KEY (`dept_no`)
);

CREATE TABLE `dept_emp` (
  `emp_no` int(11) NOT NULL,
  `dept_no` char(4) NOT NULL,
  `from_date` date NOT NULL,
  `to_date` date NOT NULL,
  PRIMARY KEY (`emp_no`, `dept_no`)
);

CREATE TABLE IF NOT EXISTS "titles" (
  `emp_no` int(11) NOT NULL,
  `title` varchar(50) NOT NULL,
  `from_date` date NOT NULL,
  `to_date` date DEFAULT NULL -- 注意：这里使用了 DEFAULT NULL 而非 NOT NULL
);
```

```SQL
select d.dept_no,d.dept_name,count(t.title) as t
from departments d
join dept_emp de
on d.dept_no = de.dept_no
and de.to_date = '9999-01-01'
join titles t
on t.emp_no = de.emp_no
and t.to_date = '9999-01-01'
group by d.dept_no,d.dept_name;
```



**给出每个员工每年薪水涨幅超过5000的，给出emp_no,变更日期from_date，涨幅salary_growth，按照salary_growth逆序排序**
```SQL
CREATE TABLE `salaries` (
`emp_no` int(11) NOT NULL,
`salary` int(11) NOT NULL,
`from_date` date NOT NULL,
`to_date` date NOT NULL,
PRIMARY KEY (`emp_no`, `from_date`));
```

```SQL
SELECT
  s1.emp_no,
  s1.from_date,
  (s1.salary - s0.salary) AS salary_growth
FROM salaries s1
JOIN salaries s0
  ON s0.emp_no = s1.emp_no
 AND s0.from_date = (
   SELECT MAX(s2.from_date)
   FROM salaries s2
   WHERE s2.emp_no = s1.emp_no
     AND s2.from_date < s1.from_date
 )
WHERE (s1.salary - s0.salary) > 5000
ORDER BY salary_growth DESC;
```




**删除表的外键约束**
```SQL
create table tb_emp9(
id INT PRIMARY KEY,
name varchar(25),
deptID INT,
salary decimal(10,2),
constraint fk_emp_dept foreign key (deptID) references tb_dept1(id)
);
```

```SQL
alter table tb_emp9
drop foreign key fk_emp_dept;
```

**删除没有被关联的表**
```SQL
drop table if exists tb_emp,tb_dept;
```

**删除关联表**
```SQL
alter table tb_emp9
drop foreign key fk_emp_dept;

drop table if exists tb_emp9;
```

**创建表tmp3     YEAR**
```SQL
create table tmp3(y YEAR);
```
**插入数据**
```SQL
insert into tmp3
values (2010),
		('2010');//结果都一样（YEAR类型的限制是1901-2155）
```


**创建表tmp4    TIME HH:MM:SS**
```SQL
create table tmp4(t TIME);
```

```SQL
insert into tmp4
values ('10:05:05');
```


**创建表tmp5 DATE  YYYY-MM-DD类型**
```SQL
create table tmp5(d DATE);
```
**插入数据**
```SQL
insert into tmp5
values ('1998-08-08');

insert into tmp5
values (current_date()),
		(NOW());//current_date()只返回当前日期值，不包括时间部分；NOW()函数返回日期和时间值，保存到数据库时，只保留日期部分
```

**创建表6 DATETIME YYYY-MM-DD-HH-MM-SS 存储时需要8个字节**
```SQL
create table tmp6(dt DATETIME);
```
**插入值**
```SQL
insert into tmp6
values ('1998-08-08 10:10:10')

insert into tmp6
values (NOW());
```

**创建表tmp7 TIMESTAMP YYYY-MM-DD HH-MM-SS 固定19个字符的显示宽度，存储需要4个字节**
```SQL
create table tmp7(ts TIMESTAMP);
```
**插入数据**
```SQL
insert into tmp7
values ('1998-10-10 10:10:10');
```

**创建表tmp8**
```SQL
create table tmp8(ch char(4), vch varchar(4));//ch存储4字节，vch存储5字节
```
**插入值**
```SQL
insert into tmp8
values ('ab  ','ab  ');

//查询
select *
from tmp8;

//结果
ch 'ab'  vch 'ab  ' ch不保留空格，vch保留空格，ch是固定长度的，如果保存不满，保存时空格填充
```

**创建表13，定义binary和varbinary类型**
```SQL
create table tmp13(b binary(3),vb varbinary(3));//binary固定长度不满用/0填充
```
**插入值**
```SQL
insert into tmp13
values (5,5);

//查询长度
select length(b) ,length (vb) from tmp13;
结果 b长度为3，vb为1；
```


**在表book的year_publication字段上建立普通**
```SQL
create table book(
bookid INT NOT NULL,
bookname varchar(255) not null,
authors varchar(255) not null,
info varchar(255) null,
comment varchar(255) null,
year_publication YEAR NOT NULL,
INDEX(year_publication)
);
```

**创建唯一索引**
```SQL
create table t1(
id INT NOT NULL,
name CHAR(30) NOT NULL,
UNIQUE INDEX Uniqid(id)
);
```

**创建单列索引**
```SQL
create table t2(
id INT NOT NULL,
name CHAR(50) NULL,
INDEX Singleidx(name(20))
);
```

**创建组合索引**
```SQL
create table t3(
id INT NOT NULL,
name CHAR(30) NOT NULL,
age INT NOT NULL,
info VARCHAR(255),
INDEX MultiIdx(id,name,age)
);
```

**创建全文索引 全文索引只有文本类型才能创建，并且只有MyISAM引擎才有**
```SQL
create table t4(
id INT NOT NULL,
name CHAR(30) NOT NULL,
age INT NOT NULL,
info varchar(255),
FULLTEXT INDEX fulltextid(info)
)engine=MyISAM;
```

**使用alter table语句创建索引**
```SQL
alter table book
ADD INDEX bkNAMEIDx(bookname(30));
```

**使用alter table 创建唯一索引**
```SQL
alter table book
add unique index uniqueidx(bookId);
```

**建立组合索引**
```SQL
alter table book
add index bkauandinfo(authors(30),info(30));
```

**使用create语句创建组合索引**
```SQL
create index BkcmtIdx ON book(comment(50));

create unique index uniqidx ON book(bookId);
```

**删除索引**
```SQL
alter table book
drop index UniqidIDx;
```

**使用drop index语句删除索引**
```SQL
drop index index_name on table_name;
```