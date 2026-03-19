## 所有成绩都大于 80 分” 学生名字的 SQL

```sql
SELECT s.student_name
FROM students s
JOIN scores sc ON s.student_id = sc.student_id  -- 关联两表
GROUP BY s.student_id, s.student_name          -- 按学生分组
HAVING MIN(sc.score) > 80;                     -- 组内最低分>80，即所有成绩>80
```


### 测试场景：查询所有成绩都大于 80 分的学生名字

#### 1. 测试目标

验证 SQL 查询是否能正确筛选出 “所有课程成绩均大于 80 分” 的学生名字，即该学生在所有已修课程中的分数均满足 > 80 分。

#### 2. 测试数据准备

假设存在两张表：

- `students`：存储学生基本信息（学生 ID、姓名）
- `scores`：存储学生成绩（学生 ID、课程 ID、分数）

表结构及测试数据如下：












```sql
-- 创建学生表
CREATE TABLE IF NOT EXISTS students (
    sid INT PRIMARY KEY,  -- 学生ID
    sname VARCHAR(50)     -- 学生姓名
);

-- 创建成绩表
CREATE TABLE IF NOT EXISTS scores (
    sid INT,              -- 学生ID
    cid INT,              -- 课程ID
    score INT,            -- 分数（0-100）
    PRIMARY KEY (sid, cid)
);

-- 插入学生数据
INSERT INTO students (sid, sname) VALUES
(1, '张三'),
(2, '李四'),
(3, '王五'),
(4, '赵六'),
(5, '孙七'),
(6, '周八');  -- 无成绩的学生

-- 插入成绩数据（覆盖多种场景）
INSERT INTO scores (sid, cid, score) VALUES
-- 场景1：张三（sid=1）所有成绩>80（90、85）→ 符合条件
(1, 1, 90),
(1, 2, 85),
-- 场景2：李四（sid=2）有一门成绩≤80（81、80）→ 不符合
(2, 1, 81),
(2, 3, 80),
-- 场景3：王五（sid=3）有一门成绩<80（95、79）→ 不符合
(3, 2, 95),
(3, 3, 79),
-- 场景4：赵六（sid=4）只有一门成绩且>80（88）→ 符合条件
(4, 1, 88),
-- 场景5：孙七（sid=5）所有成绩均为81（>80）→ 符合条件
(5, 2, 81),
(5, 3, 81),
(5, 4, 81);
```

#### 3. 预期结果

应查询出所有课程成绩均 > 80 分的学生，结果如下：

| sname |
| ----- |
| 张三  |
| 赵六  |
| 孙七  |

#### 4. 测试 SQL 语句

实现逻辑：排除有任何一门成绩≤80 分的学生，剩余学生即为 “所有成绩> 80 分” 的学生。







```sql
-- 方法1：使用NOT IN排除有不及格成绩的学生
SELECT sname
FROM students
WHERE sid NOT IN (
    SELECT DISTINCT sid
    FROM scores
    WHERE score <= 80  -- 找出有任何一门成绩≤80的学生ID
)
AND sid IN (SELECT DISTINCT sid FROM scores);  -- 排除无成绩的学生

-- 方法2：使用GROUP BY + HAVING（更高效）
SELECT s.sname
FROM students s
JOIN scores sc ON s.sid = sc.sid
GROUP BY s.sid, s.sname
HAVING MIN(sc.score) > 80;  -- 所有成绩的最小值>80，即全部>80
```

#### 5. 测试步骤

1. 执行建表语句创建`students`和`scores`表。
2. 插入测试数据。
3. 分别执行两种测试 SQL 语句。
4. 将查询结果与预期结果对比，验证是否一致。

#### 6. 验证点

- 是否包含 “所有成绩> 80 分” 的学生（张三、赵六、孙七）。
- 是否排除 “有一门成绩≤80 分” 的学生（李四、王五）。
- 是否排除 “无任何成绩” 的学生（周八）。
- 对于只有一门成绩且 > 80 分的学生（赵六）是否正确包含。
- 对于多门成绩均 > 80 分的学生（孙七）是否正确包含。

#### 7. 清理测试数据

测试完成后删除测试表：





```sql
DROP TABLE IF EXISTS scores;
DROP TABLE IF EXISTS students;
```

## ===================


在 `GROUP BY` 中加入 `s.name`（学生姓名），是由 SQL 的 **分组逻辑** 和 **列的可见性规则** 决定的，具体原因如下：

### 1. **SQL 分组的核心规则**

当使用 `GROUP BY` 时，`SELECT` 子句中只能出现两种类型的列：

- **分组列**（即 `GROUP BY` 后指定的列）；
- **聚合函数**（如 `MIN()`、`MAX()`、`AVG()` 等，对分组后的结果进行计算）。

如果 `SELECT` 中出现了 **非分组列且非聚合函数的列**，SQL 会报错（部分数据库如 MySQL 有宽松模式允许，但不符合标准 SQL）。

### 2. **为什么需要 `s.name` 加入 `GROUP BY`？**

在查询 “所有成绩都大于 80 分的学生名字” 时，我们的 SQL 通常是这样的：






```sql
SELECT s.sid, s.sname  -- 同时查询学生ID和姓名
FROM students s
JOIN scores sc ON s.sid = sc.sid
GROUP BY s.sid, s.sname  -- 必须同时包含sid和sname
HAVING MIN(sc.score) > 80;
```

原因是：

- `s.sid`（学生 ID）是学生的唯一标识，是分组的核心依据（确保按 “每个学生” 分组）。
- `s.sname`（学生姓名）虽然依赖于 `s.sid`（一个学生 ID 对应唯一姓名），但在 `SELECT` 中出现了 `s.sname`，而它不是聚合函数，因此必须加入 `GROUP BY` 中，否则违反 SQL 标准。

### 3. **如果不加入 `s.name` 会怎样？**

假设写成 `GROUP BY s.sid`（只按学生 ID 分组），而 `SELECT` 中包含 `s.sname`：





```sql
-- 错误示例（不符合标准SQL）
SELECT s.sid, s.sname
FROM students s
JOIN scores sc ON s.sid = sc.sid
GROUP BY s.sid  -- 缺少s.sname
HAVING MIN(sc.score) > 80;
```

此时会出现两种情况：

- **严格模式的数据库**（如 PostgreSQL、Oracle）：直接报错，提示 `s.sname` 不在 `GROUP BY` 中。
- **宽松模式的数据库**（如 MySQL 默认设置）：可能 “侥幸” 运行，但结果不可靠（数据库会随机选择一个姓名，虽然实际中一个 ID 对应一个姓名，但这是逻辑巧合，而非语法正确）。

### 4. **本质：确保分组列与查询列的一致性**

`GROUP BY` 的作用是将数据按指定列 “分组聚合”，分组后的数据集中，只有分组列的值是确定的（每个组唯一），其他列的值必须通过聚合函数计算。

由于 `s.sname` 与 `s.sid` 是 **一对一关系**（一个 ID 对应一个姓名），加入 `GROUP BY s.sname` 不会改变分组结果（不会把同一学生拆分成多个组），但能保证 SQL 语法合规。

### 总结

在 `GROUP BY` 中加入 `s.name`，是为了遵守 SQL 标准：**`SELECT` 中所有非聚合列必须出现在 `GROUP BY` 中**。即使姓名依赖于唯一 ID，语法上仍需显式包含，以确保查询的合法性和结果的可靠性。
