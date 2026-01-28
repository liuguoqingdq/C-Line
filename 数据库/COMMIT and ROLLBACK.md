### 1) COMMIT：提交事务（把修改永久生效）

**作用**：把当前事务中执行的 `INSERT`/`UPDATE`/`DELETE` 等修改确认写入，对其他会话可见，并结束当前事务。

**典型流程**：


```SQL
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;  -- 两条 UPDATE 一起生效
```

**！！！ COMMIT一旦提交数据，就不可以回滚，数据就被永久保存到数据库中**

---

### 2) ROLLBACK：回滚事务（撤销未提交的修改）

**作用**：撤销当前事务中 **尚未提交** 的所有修改，使数据恢复到事务开始前的状态，并结束当前事务。


```SQL
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
ROLLBACK;  -- 两条 UPDATE 都被撤销
```

---

### 3) 关键前提与常见误区

#### A) 只对“事务性操作”有效

`ROLLBACK` 能撤销的是当前事务里对数据的更改（DML）。 对某些语句（典型是 DDL）在 MySQL 中常会隐式提交，导致你以为能回滚但回不去。

#### B) AUTOCOMMIT

MySQL 默认通常是 `autocommit=1`：每条 DML 语句执行完会自动提交。 因此想让 `ROLLBACK` 有意义，你必须：

- `START TRANSACTION;`（或 `BEGIN;`），或
    
- `SET autocommit = 0;`
    

#### C) 可见性

- 未 `COMMIT` 的修改一般只对当前会话可见（取决于隔离级别，但通常如此）
    
- `COMMIT` 后其他会话才能看到
    

---

### 4) SAVEPOINT：事务内“局部回滚”（很实用）

你可以在事务中打“保存点”，只回滚到某一步，而不是全部回滚。


```SQL
START TRANSACTION;
UPDATE t SET v = 1 WHERE id = 1;
SAVEPOINT s1;
UPDATE t SET v = 2 WHERE id = 2;
ROLLBACK TO s1;  -- 撤销第二条 UPDATE，但保留第一条
COMMIT;
```

---

### 5) 一句话对比

- **COMMIT**：确认本事务的所有修改，永久生效
    
- **ROLLBACK**：撤销本事务未提交的修改，恢复原状

**TRUNCATE TABLE 与DELETE FROM对比**
- **相同点：TRUNCATE TABLE与DELETE FROM 都可以删除表数据**
- **不同点：TRUNCATE TABLE一旦执行就不能够回滚，表数据全部清除，DELETE FROM 可以全部删除数据（DELETE FROM 不带WHERE） ，一旦清楚，可以回滚，当然也可以不能回滚**

**DDL是数据定义语言，一旦执行，是不可以回滚的**
**DML操作默认情况下，是不可以回滚的，但是如果在执行DML之前，执行SET autocommit = false  就可以回滚**