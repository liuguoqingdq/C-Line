**1.数据库三大范式是什么？**
- 答：
	- **第一范式**：数据表的每一列都是不可分割的原子项
	- **第二范式：** 第一范式需要在第一范式的基础之上，确保每一列都与主键相关，而不能只与主键的一部分相关
	- **第三范式**：在第二范式的基础之上，非主属性不能依赖于其他非主属性。

-----
**2.MySQL是怎么连接查询的？**
- 答：
	- SQL有一下几种连接：
		- **内连接(inner join)**：返回两个表中匹配关系的行。
		- **左外连接**:返回匹配的行，但是会保留左表中所有的行,未匹配的右表的行会为NULL。
		- **右外连接**:返回匹配的行，但是会保留右表中所有的行,未匹配的左表的行会为NULL。
		- **全外连接**: 全连接返回所有行，但是MySQL不支持全外连接，需要用union all实现。

![Pasted image 20251226215003.png](app://f9427eb390b0a278f946c9800a26fe165645/Users/Zhuanz/Documents/Obsidian%20Vault/Cpp-Note/Pasted%20image%2020251226215003.png?1766757003991)

------
**3.MySQL如何避免插入重复数据？**
- 答：
	- 使用unique约束
	- 在有unique约束的情况下，可以使用**INSERT ... ON DUPLICATE KEY UPDATE** 语句，如果新的记录，中有属性的值（被unique约束）则触发更新，否则触发插入。
	```SQL
	INSERT INTO user (id, email, name)
VALUES (1, 'a@test.com', 'Tom')
ON DUPLICATE KEY UPDATE name = 'Jerry';
	```
	触发更新name会变成Jerry
	- 使用insert ignore，当插入重复时，会忽略错误

-----
**4.char与varchar的区别是什么？**
- 答
	- **char**：是固定长度的字符类型（0-255），定义时需要固定长度，内容不够会在后面补充空格，适合存储长度固定的数据类型，状态码等，对短字符串效率高。
	- **varchar**：是变长类型，需要额外的空间存储数据长度，（<=255用一个字节，大于255用两个字节），varchar最大65535字节。varchar适合保存变长类型数据，如备注，节省空间。

------
**5.varchar(M)括号中的M代表字节还是字符？**
- 答：M代表字符数，而不是字节数，实际字节数根据实际使用字符集确定，如：
	- 使用ASCII码字符集，每个字符代表一个字节，那么10个字符就用10个字节，外加1个字节存长度，那么实际占用空间11字节。
	- 使用utf-8字符集，每个字符占4字节，实际大小根据实际字符确定。

------
**6.int(1)与int(10)有什么不同？**
- 答：在存储空间上，int(1)与int(10)都是占用4字节（32位）的空间大小，1与10不同在于显示长度不同，在于特定场景下控制数据的显示格式。
	- 唯一作用场景，在字段被zerofill修饰时，会对不满足显示宽度的字符，使用前导0填充，如字段使用int(4) zerofill，存5会显示005，存1234显示1234.

-----
**7.TEXT类型可以无限大吗？**
- 答：不可以。
	- TEXT类型大小限制为：65535bytes～64kb
	- MEDIUMTEXT类型大小限制为：1677215kb～64mb
	- LONGTEXT类型大小限制位：4,294,967,295 bytes ~4Gb

-----
**8.IP地址是如何存储的？**
- 答：
	- IP地址198.127.1.1 可以当成15位的字符串，使用varchar(15)来存储：
		- 优点：直观易懂，不需要额外的操作。
		- 缺点：占用空间比较大，操作性能不如整型。
	- 也可以用整型存储：
		- 优点：占用空间小，操作性能相较于字符串要高。
		- 缺点：需要额外的转换操作。

----
**9.外键约束**
- 答：外键约束的作用是维护表与表之间的完整性和一致性的约束。有外键约束的表要收到被引用键的约束。
	- MySQL 外键在高并发、分布式、微服务场景下弊大于利，生产环境中通常用“逻辑外键”替代。
	- **逻辑外键**：数据库层面不建立 FOREIGN KEY 约束，只在字段语义和业务逻辑上表示表之间的关联关系，由应用层来保证数据一致性。

-----
**10.IN与EXISTS关键字**
- 答：
	- **IN关键字**；用于检查左侧的结果是否存在与右侧的结果集中，如果存在返回TRUE，不存在返回FALSE。
	- **EXISTS关键字**：用于检查子查询是否能够至少有一个返回结果，有则TRUE，没有则FALSE。
	- 性能上，EXISTS往往优于IN。IN适合子结果集确定，且比较小不变的情况，EXISTS适合大结果集。
	- exist：
	- ```SQL
	  SELECT u.*
FROM user u
WHERE EXISTS (
  SELECT 1
  FROM orders o
  WHERE o.user_id = u.id
);
	  ```

----
**11.MySQL中的一些基本函数**

## 一、字符串函数

|函数|功能说明|示例|
|---|---|---|
|`CONCAT()`|拼接字符串|`CONCAT('a','b') → 'ab'`|
|`CONCAT_WS()`|带分隔符拼接|`CONCAT_WS('-', 'a','b') → 'a-b'`|
|`LENGTH()`|字节长度|`LENGTH('你好') → 6`|
|`CHAR_LENGTH()`|字符长度|`CHAR_LENGTH('你好') → 2`|
|`SUBSTRING()`|截取字符串|`SUBSTRING('abc',1,2) → 'ab'`|
|`LEFT()` / `RIGHT()`|左 / 右截取|`LEFT('abc',2) → 'ab'`|
|`UPPER()` / `LOWER()`|大小写转换|`UPPER('ab') → 'AB'`|
|`TRIM()`|去首尾空格|`TRIM(' a ') → 'a'`|
|`REPLACE()`|替换字符|`REPLACE('a-b','-','_')`|

---

## 二、数值函数

|函数|功能说明|示例|
|---|---|---|
|`ABS()`|绝对值|`ABS(-5) → 5`|
|`ROUND()`|四舍五入|`ROUND(1.56,1) → 1.6`|
|`CEILING()`|向上取整|`CEILING(1.2) → 2`|
|`FLOOR()`|向下取整|`FLOOR(1.8) → 1`|
|`MOD()`|取余|`MOD(5,2) → 1`|
|`RAND()`|随机数|`RAND()`|

---

## 三、日期 / 时间函数（面试必考）

|函数|功能说明|示例|
|---|---|---|
|`NOW()`|当前日期时间|`2026-01-11 10:00:00`|
|`CURDATE()`|当前日期|`2026-01-11`|
|`CURTIME()`|当前时间|`10:00:00`|
|`DATE()`|提取日期|`DATE(NOW())`|
|`YEAR()` / `MONTH()`|年 / 月|`YEAR(NOW())`|
|`DATEDIFF()`|日期差（天）|`DATEDIFF('2026-01-11','2026-01-01')`|
|`DATE_ADD()`|日期加|`DATE_ADD(NOW(), INTERVAL 1 DAY)`|
|`DATE_SUB()`|日期减|`DATE_SUB(NOW(), INTERVAL 1 MONTH)`|

---

## 四、聚合函数（分组统计核心）

|函数|功能说明|
|---|---|
|`COUNT()`|统计行数|
|`SUM()`|求和|
|`AVG()`|平均值|
|`MAX()`|最大值|
|`MIN()`|最小值|

📌 常和 `GROUP BY` 一起使用

---

## 五、条件 / 逻辑函数（高频）

|函数|功能说明|示例|
|---|---|---|
|`IF()`|条件判断|`IF(score>=60,'及格','不及格')`|
|`IFNULL()`|NULL 处理|`IFNULL(age,0)`|
|`NULLIF()`|相等则 NULL|`NULLIF(a,b)`|
|`CASE WHEN`|多条件判断|`CASE WHEN a>1 THEN 'A' END`|

---

## 六、类型转换函数

|函数|功能说明|示例|
|---|---|---|
|`CAST()`|类型转换|`CAST('123' AS INT)`|
|`CONVERT()`|类型转换|`CONVERT('123', SIGNED)`|

---

## 七、系统 / 信息函数

|函数|功能说明|
|---|---|
|`VERSION()`|MySQL 版本|
|`DATABASE()`|当前数据库|
|`USER()`|当前用户|
|`CONNECTION_ID()`|连接 ID|

---

## 八、NULL 相关（非常重要）

| 场景      | 正确写法                      |
| ------- | ------------------------- |
| 判断 NULL | `IS NULL` / `IS NOT NULL` |
| NULL 比较 | ❌ `= NULL`（错误）            |
| NULL 处理 | `IFNULL()`                |

-----
**12.SQL的查询语句执行顺序是怎样的？**
- 答：所有的查询语句都是从FROM开始执行，在执行过程中，每个步骤都会生成一个虚拟表，这个虚拟表将作为下一个执行步骤的输入，最后一个步骤产生的虚拟表即为输出结果。
- 执行顺序：FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT。所以where不能够使用select中定义的别名，因为where在select之前。

----
**13.给学生表、课程成绩表，求不存在01课程但存在02课程的学生的成绩**
可以使用SQL的子查询和`LEFT JOIN`或者`EXISTS`关键字来实现，这里我将展示两种不同的方法来完成这个查询。

假设我们有以下两张表：

1. `Student` 表，其中包含学生的`sid`（学生编号）和其他相关信息。
2. `Score` 表，其中包含`sid`（学生编号），`cid`（课程编号）和`score`（分数）。

**解法1.用NOT EXISTS**
```SQL
select s.name,sc.socre,sc.cid
from Student s
JOIN Score sc ON s.sid=sc.sid and sc.cid=='02'
where NOT EXISTS(
	select 1 from Score
	where Score.cid='01' and Student.sid=Score.sid
);
```

**解法2.用join过滤**
```SQL
select s.name,sc2.socre,sc2.cid
from Student s
LEFT JOIN Score sc1 ON s.sid=sc1.sid AND sc1.cid='01' /*用左外连接保留全部的学生信息，得到虚拟表，里面有全部学生记录，没有01课程成绩的为NULL*/
LEFT JOIN Score sc2 ON s.sid=sc2.sid AND sc2.cid='02'
/*再用一个左外连接，保留所有学生记录，匹配02课程，没有成绩的为NULL*/
where sc1.cid IS NULL AND sc2.cid IS NOT NULL;//去掉01的，保留02
```

----
**14.给定一个学生表 student_score（stu_id，subject_id，score），查询总分排名在5-10名的学生id及对应的总分**
```SQL
select stu_id,SUM(score) AS total_sc
from student_score
group by stu_id
order by total_sc desc
limit 4,6;
```

8.0写法
```SQL
select stu_id,total_sc
from (
	select 
	stu_id,
	sum(score) as total_sc,
	DENSE_RANK() over (order by taotal_sc desc) AS rk
	from student_score
	group by stu_id
) t
where rk between 5 and 10;
```

----
**15.如何用 MySQL 实现一个可重入的锁？**
```SQL
CREATE TABLE `lock_table` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    //该字段用于存储锁的名称，作为锁的唯一标识符。
    `lock_name` VARCHAR(255) NOT NULL, 
    // holder_thread该字段存储当前持有锁的线程的名称，用于标识哪个线程持有该锁。
    `holder_thread` VARCHAR(255),   
    // reentry_count 该字段存储锁的重入次数，用于实现锁的可重入性
    `reentry_count` INT DEFAULT 0
);
```
加锁的实现逻辑

1. 开启事务
    
2. 执行 SQL SELECT holder_thread, reentry_count FROM lock_table WHERE lock_name =? FOR UPDATE，查询是否存在该记录：
    
    - 如果记录不存在，则直接加锁，执行 INSERT INTO lock_table (lock_name, holder_thread, reentry_count) VALUES (?,?, 1)
        
    - 如果记录存在，且持有者是同一个线程，则可冲入，增加重入次数，执行 UPDATE lock_table SET reentry_count = reentry_count + 1 WHERE lock_name =?
        
3. 提交事务
    

解锁的逻辑：

1. 开启事务
    
2. 执行 SQL SELECT holder_thread, reentry_count FROM lock_table WHERE lock_name =? FOR UPDATE，查询是否存在该记录：
    
    - 如果记录存在，且持有者是同一个线程，且可重入数大于 1 ，则减少重入次数 UPDATE lock_table SET reentry_count = reentry_count - 1 WHERE lock_name =?
        
    - 如果记录存在，且持有者是同一个线程，且可重入数小于等于 0 ，则完全释放锁，DELETE FROM lock_table WHERE lock_name =?
        
3. 提交事务

-----
