## 一、 核心概念：什么是系统变量？

MySQL 服务器在运行时维护着许多配置信息，这些配置统称为 **系统变量（System Variables）**。它们决定了数据库的存储引擎、缓冲区大小、连接限制、字符集等核心行为。

为了灵活管理，MySQL 将这些变量划分为两个**作用域（Scope）**：

1. **GLOBAL（全局作用域）**：影响服务器的整体运行，对**所有**连接到该服务器的客户端生效。
    
2. **SESSION（会话作用域）**：仅影响**当前连接（当前会话）**。每个客户端连接后，都会拥有自己的一套会话变量。
    

---

## 二、 全局 vs. 会话：深度对比

|**特性**|**GLOBAL (全局变量)**|**SESSION (会话变量)**|
|---|---|---|
|**影响范围**|整个 MySQL 实例，所有用户。|仅限当前连接（当前窗口）。|
|**生命周期**|随服务器启动而存在，直到重启或手动修改。|随连接建立而产生，连接断开即销毁。|
|**初始值来源**|配置文件（my.cnf）或启动参数。|连接时，从当前的 GLOBAL 值中“继承”/复制。|
|**权限要求**|需要 `SUPER` 或 `SYSTEM_VARIABLES_ADMIN` 权限。|通常普通用户即可修改自己的会话变量。|
|**典型案例**|`max_connections` (最大连接数)|`sql_mode` (SQL模式)、`autocommit` (自动提交)|

---

## 三、 查看变量 (View)

查看变量时，如果不指定作用域，MySQL 通常默认显示 **SESSION** 变量（如果该变量存在会话级的话）。

### 1. 使用 `SHOW` 语句

```SQL  
-- 查看所有全局变量
SHOW GLOBAL VARIABLES;

-- 查看所有会话变量
SHOW SESSION VARIABLES; 
SHOW VARIABLES; -- 简写，默认就是 SESSION

-- 模糊查询特定的变量
SHOW GLOBAL VARIABLES LIKE 'char%';
```

### 2. 使用 `SELECT` 语句（更精准）


```SQL
-- 使用 @@ 符号
SELECT @@global.sql_mode;    -- 查看全局级
SELECT @@session.sql_mode;   -- 查看会话级
SELECT @@sql_mode;           -- 优先查找会话级，没有则找全局级
```

---

## 四、 修改变量 (Set)

这是最核心的操作部分，直接影响数据库的生产运行。

### 1. 修改会话变量（临时生效）

只对当前窗口有效，关闭窗口后修改丢失。


```SQL
SET SESSION auto_increment_increment = 2;
-- 或者简写
SET @@session.auto_increment_increment = 2;
SET auto_increment_increment = 2; 
```

### 2. 修改全局变量（运行时生效）

**注意：** 这种方式修改后，**已经存在的连接**（其他人的窗口）不会生效，只有**新建立的连接**才会看到新值。


```SQL
SET GLOBAL max_connections = 1000;
-- 或者
SET @@global.max_connections = 1000;
```

---

## 五、 持久化问题：重启后会失效吗？

这是新手最容易犯错的地方。

### 1. 传统方式 (MySQL 5.7 及以前)

使用 `SET GLOBAL` 修改的值只保存在**内存**中。如果 MySQL 服务重启，它会重新读取磁盘上的配置文件（`my.cnf` 或 `my.ini`），你之前的 `SET GLOBAL` 修改将全部丢失。

- **解决方法**：修改完内存后，必须手动打开 `my.cnf` 文件，把参数写进去。
    

### 2. 现代方式 (MySQL 8.0+)：`SET PERSIST`

MySQL 8.0 引入了持久化修改功能，**建议优先使用**：


```SQL
SET PERSIST max_connections = 1000;
```

- **原理**：MySQL 会在数据目录下生成一个 `mysqld-auto.json` 文件。重启时，MySQL 会读取这个文件来覆盖配置文件的默认值。这样你就不需要手动去服务器上改 `.cnf` 文件了。
    

---

## 六、 变量的继承关系（重要逻辑）

理解这个逻辑能帮你排查 90% 的配置不生效问题：

1. **启动时**：MySQL 读取 `my.cnf` 设置 **GLOBAL** 变量。
    
2. **连接时**：用户 A 连接 MySQL，MySQL 复制一份当前的 **GLOBAL** 变量，给用户 A 生成一份 **SESSION** 变量。
    
3. **修改时**：
    
    - 如果你修改了 **GLOBAL**，用户 A 的 **SESSION** 变量**不会变**。
        
    - 只有在用户 B **新连接**进来时，才会获取到修改后的新值。
        

---

## 七、 进阶：哪些变量只能是 GLOBAL？

并不是所有变量都有两个作用域。

- **仅 GLOBAL**：涉及物理资源或系统限制的。
    
    - `innodb_buffer_pool_size` (内存缓冲区)
        
    - `port` (端口号)
        
    - `datadir` (数据目录)
        
- **既有 GLOBAL 又有 SESSION**：涉及 SQL 执行习惯的。
    
    - `sql_mode`
        
    - `autocommit`
        
    - `character_set_client`
        

---

## 八、 总结：避坑指南

1. **不要在生产环境轻易 `SET GLOBAL`**：虽然不需要重启，但如果设置不当（如内存分少了），可能导致系统瞬间崩溃。
    
2. **修改了不生效？**：检查你改的是 `GLOBAL` 还是 `SESSION`。如果你改了 `GLOBAL`，请断开当前连接重新进一次。
    
3. **查询权限**：如果你没有 `SUPER` 权限，尝试修改 `GLOBAL` 变量会报错：`Variable 'xxx' is a GLOBAL variable and should be set with SET GLOBAL`。
    

---

### 使用
```SQL
//用户变量
SET @变量名 = '变量值';
SET @变量名 := '变量值'；

select @变量名 :=表达式 FROM 子句；
select 表达式 INTO @变量名 from 子句； 

SET @num1=1;
SET @num2 := 2;
SET @sum :=@num1+@num2;

select @sum;



//局部变量
#局部变量必须用declare声明，并且只能在begin与end之间声明
#declare声明的局部变量必须声明在begin首行的位置
//声明方式 ：
declare 变量名 变量类型 [default 值] 如果没有default默认值为NULL
delimiter //
create procedure 过程名()
begin
	declare emp_id INT;
	declare emp_name varchar(25);
	declare salary DECIMAL(10,2) default 2000;
	
	#赋值
	SET emp_id := 123;
	SET emp_name := 'olsen';
end //
delimiter ;
```
