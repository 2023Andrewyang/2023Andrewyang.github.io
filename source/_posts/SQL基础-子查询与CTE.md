---
title: SQL基础-子查询与CTE
date: 2026-02-20 14:30:00
tags:
  - SQL
  - 数据库
  - 学习笔记
categories:
    - 技术文章
---

继上一篇整理了 JOIN 和 UNION 之后，这篇笔记接着复习子查询（Subquery）和 CTE（公用表表达式）。子查询是 SQL 中非常灵活的工具，而 CTE 可以看作子查询的"升级版"，能让复杂查询的可读性大幅提升。两者经常搭配 JOIN、UNION 一起使用，是编写复杂查询的核心技能。

---

<!-- more -->

本文沿用[上一篇](/2026/02/20/SQL基础-多表连接与集合查询/)的示例表，这里再列一次建表语句方便独立阅读：

```sql
-- 部门表
CREATE TABLE department (
    id INT PRIMARY KEY,
    dept_name VARCHAR(50),
    location VARCHAR(50)
);

INSERT INTO department VALUES
(1, '技术部', '北京'),
(2, '销售部', '上海'),
(3, '人事部', '北京'),
(4, '财务部', '深圳');

-- 员工表
CREATE TABLE employee (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    dept_id INT,
    salary DECIMAL(10, 2),
    hire_date DATE,
    manager_id INT
);

INSERT INTO employee VALUES
(1,  '张三', 1,    15000, '2022-03-15', 6),
(2,  '李四', 1,    18000, '2020-07-01', 6),
(3,  '王五', 2,    12000, '2023-01-10', 7),
(4,  '赵六', 2,    9000,  '2023-06-20', 7),
(5,  '孙七', 3,    11000, '2021-09-05', NULL),
(6,  '周八', 1,    22000, '2019-04-12', NULL),
(7,  '吴九', 2,    15000, '2022-11-08', 6),
(8,  '郑十', 3,    13000, '2022-05-18', 5),
(9,  '刘一', NULL, 9500,  '2024-01-15', NULL),
(10, '陈二', NULL, 8000,  '2024-03-01', NULL);

-- 项目表
CREATE TABLE project (
    id INT PRIMARY KEY,
    project_name VARCHAR(100),
    start_date DATE,
    budget DECIMAL(12, 2)
);

INSERT INTO project VALUES
(1, '电商平台重构', '2024-01-01', 500000),
(2, '数据分析系统', '2024-03-15', 300000),
(3, 'APP 开发',     '2024-06-01', 200000),
(4, '内部工具升级', '2025-01-01', 100000);

-- 员工项目关联表
CREATE TABLE emp_project (
    emp_id INT,
    project_id INT,
    role VARCHAR(50),
    PRIMARY KEY (emp_id, project_id)
);

INSERT INTO emp_project VALUES
(1, 1, '开发'),
(2, 1, '架构师'),
(2, 2, '技术负责人'),
(3, 3, '销售对接'),
(6, 1, '项目经理'),
(6, 2, '技术顾问'),
(7, 3, '销售负责人'),
(8, 2, '人事协调');
```

---

## 一、子查询概述

**子查询**（Subquery）就是嵌套在另一个 SQL 语句内部的 `SELECT` 查询，通常用括号 `()` 包裹。子查询可以出现在 `WHERE`、`FROM`、`SELECT`、`HAVING` 等子句中。

```sql
-- 子查询的基本形式
SELECT * FROM employee
WHERE salary > (SELECT AVG(salary) FROM employee);
--              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
--              这就是一个子查询（嵌套在 WHERE 中）
```

子查询可以从两个维度来分类：

| 分类维度 | 类型 | 说明 |
|---------|------|------|
| **按返回结果** | 标量子查询 | 返回**单个值**（一行一列） |
| | 列子查询 | 返回**一列多行** |
| | 行子查询 | 返回**一行多列** |
| | 表子查询 | 返回**多行多列**（一张临时表） |
| **按关联性** | 非关联子查询 | 子查询**独立执行**，与外层无关 |
| | 关联子查询 | 子查询**依赖外层**查询的值 |

---

## 二、按返回结果分类

### 1. 标量子查询（返回单个值）

子查询只返回**一行一列**（一个标量值），常用于比较运算符（`=`、`>`、`<` 等）的右侧。

```sql
-- 查询工资高于全公司平均工资的员工
SELECT name, salary
FROM employee
WHERE salary > (SELECT AVG(salary) FROM employee);
-- AVG(salary) = 13250，返回工资大于 13250 的员工
```

| name | salary |
|------|--------|
| 张三 | 15000 |
| 李四 | 18000 |
| 周八 | 22000 |
| 吴九 | 15000 |

**在 SELECT 中使用标量子查询**：

```sql
-- 查询每个员工的工资与公司平均工资的差值
SELECT
    name,
    salary,
    salary - (SELECT AVG(salary) FROM employee) AS diff_from_avg
FROM employee;
```

> ⚠️ **注意**：如果标量子查询返回了**多行**，SQL 会报错！确保子查询结果只有一行一列。

```sql
-- ❌ 错误：子查询返回多行，不能用 = 比较
SELECT name FROM employee
WHERE dept_id = (SELECT id FROM department WHERE location = '北京');
-- 北京有两个部门（技术部和人事部），子查询返回了 2 行，报错！

-- ✅ 正确：多行时应使用 IN
SELECT name FROM employee
WHERE dept_id IN (SELECT id FROM department WHERE location = '北京');
```

### 2. 列子查询（返回一列多行）

子查询返回**一列多行**，常配合 `IN`、`ANY`、`ALL`、`EXISTS` 使用。

```sql
-- 查询有员工的部门名称
SELECT dept_name
FROM department
WHERE id IN (SELECT DISTINCT dept_id FROM employee WHERE dept_id IS NOT NULL);
```

| dept_name |
|-----------|
| 技术部 |
| 销售部 |
| 人事部 |

```sql
-- 查询参与了项目的员工
SELECT name, salary
FROM employee
WHERE id IN (SELECT DISTINCT emp_id FROM emp_project);
```

| name | salary |
|------|--------|
| 张三 | 15000 |
| 李四 | 18000 |
| 王五 | 12000 |
| 周八 | 22000 |
| 吴九 | 15000 |
| 郑十 | 13000 |

### 3. 行子查询（返回一行多列）

子查询返回**一行多列**，可以与多个列同时比较。

```sql
-- 查询与张三在同一部门且工资相同的员工
SELECT name, dept_id, salary
FROM employee
WHERE (dept_id, salary) = (SELECT dept_id, salary FROM employee WHERE name = '张三');
-- 张三: dept_id=1, salary=15000
```

| name | dept_id | salary |
|------|---------|--------|
| 张三 | 1 | 15000 |

> 💡 **补充**：行子查询用的不多，但在需要同时匹配多个列时很有用。注意子查询必须恰好返回**一行**。

### 4. 表子查询（派生表 / Derived Table）

子查询返回**多行多列**，放在 `FROM` 子句中当作临时表使用，也叫**派生表**。

```sql
-- 查询每个部门工资最高的员工
SELECT e.name, e.salary, ds.dept_name, ds.max_salary
FROM employee e
JOIN (
    SELECT d.id, d.dept_name, MAX(e2.salary) AS max_salary
    FROM department d
    JOIN employee e2 ON d.id = e2.dept_id
    GROUP BY d.id, d.dept_name
) ds ON e.dept_id = ds.id AND e.salary = ds.max_salary;
```

| name | salary | dept_name | max_salary |
|------|--------|-----------|------------|
| 周八 | 22000 | 技术部 | 22000 |
| 吴九 | 15000 | 销售部 | 15000 |
| 郑十 | 13000 | 人事部 | 13000 |

> ⚠️ **注意**：MySQL 要求 `FROM` 中的子查询（派生表）**必须指定别名**，否则会报错。

```sql
-- ❌ 错误：派生表没有别名
SELECT * FROM (SELECT dept_id, AVG(salary) FROM employee GROUP BY dept_id);
-- 报错：Every derived table must have its own alias

-- ✅ 正确：加上别名
SELECT * FROM (SELECT dept_id, AVG(salary) AS avg_sal FROM employee GROUP BY dept_id) AS dept_avg;
```

---

## 三、⭐ 按关联性分类

### 1. 非关联子查询（独立子查询）

子查询可以**独立执行**，不依赖外层查询的任何值。执行顺序：**先执行子查询，再执行外层查询**。

前面大部分示例都是非关联子查询：

```sql
-- 非关联子查询：子查询独立执行，返回平均工资
SELECT name, salary FROM employee
WHERE salary > (SELECT AVG(salary) FROM employee);
```

### 2. ⭐ 关联子查询（相关子查询）

子查询**引用了外层查询的列**，每处理外层的一行，子查询就要重新执行一次。

```sql
-- 查询工资高于其所在部门平均工资的员工
SELECT name, salary, dept_id
FROM employee e1
WHERE salary > (
    SELECT AVG(salary)
    FROM employee e2
    WHERE e2.dept_id = e1.dept_id  -- 引用了外层的 e1.dept_id
);
```

执行逻辑：对于外层的**每一行**，子查询都会根据该行的 `dept_id` 重新计算对应部门的平均工资。

| name | salary | dept_id |
|------|--------|---------|
| 周八 | 22000 | 1 |
| 吴九 | 15000 | 2 |
| 郑十 | 13000 | 3 |

> 💡 **解析**：
> - 技术部平均：(15000 + 18000 + 22000) / 3 ≈ 18333 → 周八(22000) 高于平均
> - 销售部平均：(12000 + 9000 + 15000) / 3 = 12000 → 吴九(15000) 高于平均
> - 人事部平均：(11000 + 13000) / 2 = 12000 → 郑十(13000) 高于平均

> ⚠️ **性能提示**：关联子查询对外层每一行都要执行一次子查询，当数据量大时**性能较差**。很多关联子查询可以改写为 JOIN 来优化，后面会对比说明。

---

## 四、⭐ 子查询常用关键字

### 1. `IN` / `NOT IN`

判断值是否在子查询的结果集中。

```sql
-- 查询有项目的员工
SELECT name FROM employee
WHERE id IN (SELECT emp_id FROM emp_project);

-- 查询没有项目的员工
SELECT name FROM employee
WHERE id NOT IN (SELECT emp_id FROM emp_project);
```

> ⚠️ **NOT IN 的 NULL 陷阱**：如果子查询结果中包含 `NULL`，`NOT IN` 会返回**空结果**！因为任何值与 `NULL` 比较的结果都是 `UNKNOWN`。

```sql
-- ❌ 危险：如果子查询结果包含 NULL，NOT IN 返回空
SELECT name FROM employee
WHERE dept_id NOT IN (SELECT manager_id FROM employee);
-- manager_id 列有 NULL 值 → 整个查询返回空结果！

-- ✅ 安全写法：排除 NULL
SELECT name FROM employee
WHERE dept_id NOT IN (SELECT manager_id FROM employee WHERE manager_id IS NOT NULL);
```

### 2. ⭐ `EXISTS` / `NOT EXISTS`

判断子查询是否能返回**至少一行**数据（不关心具体值，只关心"有没有"）。

```sql
-- 查询参与了项目的员工
SELECT name FROM employee e
WHERE EXISTS (
    SELECT 1 FROM emp_project ep WHERE ep.emp_id = e.id
);

-- 查询没有参与任何项目的员工
SELECT name FROM employee e
WHERE NOT EXISTS (
    SELECT 1 FROM emp_project ep WHERE ep.emp_id = e.id
);
```

| NOT EXISTS 结果 |
|----------------|
| 赵六 |
| 孙七 |
| 刘一 |
| 陈二 |

> 💡 **补充**：`EXISTS` 中的 `SELECT 1` 只是占位符，写 `SELECT *` 或 `SELECT 42` 效果一样，因为 EXISTS 只判断有无结果行，不关心具体返回什么值。

### 3. `ANY` / `SOME` / `ALL`

配合比较运算符使用，与列子查询的结果进行批量比较。`SOME` 是 `ANY` 的同义词。

```sql
-- 查询工资大于销售部任意一人的员工（> ANY = 大于最小值）
SELECT name, salary FROM employee
WHERE salary > ANY (SELECT salary FROM employee WHERE dept_id = 2);
-- 销售部工资: 12000, 9000, 15000
-- > ANY 即 > 9000（最小值），返回所有工资 > 9000 的员工
```

```sql
-- 查询工资大于销售部所有人的员工（> ALL = 大于最大值）
SELECT name, salary FROM employee
WHERE salary > ALL (SELECT salary FROM employee WHERE dept_id = 2);
-- > ALL 即 > 15000（最大值）
```

| name | salary |
|------|--------|
| 李四 | 18000 |
| 周八 | 22000 |

**ANY / ALL 速查**：

| 表达式 | 等价于 | 含义 |
|--------|-------|------|
| `> ANY (子查询)` | `> MIN(子查询结果)` | 大于最小值 |
| `< ANY (子查询)` | `< MAX(子查询结果)` | 小于最大值 |
| `> ALL (子查询)` | `> MAX(子查询结果)` | 大于最大值 |
| `< ALL (子查询)` | `< MIN(子查询结果)` | 小于最小值 |
| `= ANY (子查询)` | `IN (子查询结果)` | 等于其中之一 |

### 4. ⭐ `IN` vs `EXISTS` 对比

这是最常被讨论的对比，两者在很多场景下可以互换，但有重要区别：

```sql
-- IN 写法
SELECT name FROM employee
WHERE dept_id IN (SELECT id FROM department WHERE location = '北京');

-- EXISTS 写法
SELECT name FROM employee e
WHERE EXISTS (
    SELECT 1 FROM department d
    WHERE d.id = e.dept_id AND d.location = '北京'
);
```

| 对比维度 | IN | EXISTS |
|---------|-----|--------|
| **执行方式** | 先执行子查询，生成结果集，再逐行比对 | 对外层每一行，执行子查询判断是否有结果 |
| **NULL 处理** | `NOT IN` 遇到 NULL 会返回空结果 | `NOT EXISTS` 正确处理 NULL ✅ |
| **适用场景** | 子查询结果集**小**时效率高 | 子查询结果集**大**、外层表**小**时效率高 |
| **可读性** | 较直观 | 稍复杂 |

> 💡 **实用建议**：
> - 简单场景优先用 `IN`，代码更直观
> - 涉及 `NOT IN` 且子查询可能有 `NULL` 时，**务必用 `NOT EXISTS`** 来避免陷阱
> - 大数据量场景，关注执行计划选择最优方案

```sql
-- ⭐ NOT IN vs NOT EXISTS 的 NULL 问题演示
-- ❌ NOT IN：如果子查询结果含 NULL，返回空
SELECT name FROM employee
WHERE dept_id NOT IN (SELECT dept_id FROM employee);
-- dept_id 列有 NULL（刘一、陈二）→ 结果为空！

-- ✅ NOT EXISTS：正确处理
SELECT e1.name FROM employee e1
WHERE NOT EXISTS (
    SELECT 1 FROM employee e2
    WHERE e2.dept_id = e1.dept_id AND e2.id != e1.id
);
```

---

## 五、子查询的使用位置

子查询几乎可以出现在 SQL 语句的任何位置，以下是常见的使用位置汇总：

### 1. WHERE 子句中（最常见）

```sql
-- 查询工资最高的员工
SELECT name, salary FROM employee
WHERE salary = (SELECT MAX(salary) FROM employee);
-- 输出: 周八, 22000
```

### 2. FROM 子句中（派生表）

```sql
-- 统计每个部门的平均工资，再筛选高于全公司平均的部门
SELECT dept_name, avg_salary
FROM (
    SELECT d.dept_name, AVG(e.salary) AS avg_salary
    FROM department d
    JOIN employee e ON d.id = e.dept_id
    GROUP BY d.dept_name
) AS dept_stats
WHERE avg_salary > (SELECT AVG(salary) FROM employee);
```

### 3. SELECT 子句中（标量子查询）

```sql
-- 为每个部门附上员工人数
SELECT
    d.dept_name,
    d.location,
    (SELECT COUNT(*) FROM employee e WHERE e.dept_id = d.id) AS emp_count
FROM department d;
```

| dept_name | location | emp_count |
|-----------|----------|-----------|
| 技术部 | 北京 | 3 |
| 销售部 | 上海 | 3 |
| 人事部 | 北京 | 2 |
| 财务部 | 深圳 | 0 |

> 💡 **补充**：SELECT 中的子查询必须是**标量子查询**（返回单个值），否则报错。

### 4. HAVING 子句中

```sql
-- 查询平均工资高于全公司平均的部门
SELECT d.dept_name, AVG(e.salary) AS avg_salary
FROM department d
JOIN employee e ON d.id = e.dept_id
GROUP BY d.dept_name
HAVING AVG(e.salary) > (SELECT AVG(salary) FROM employee);
```

---

## 六、⭐ CTE 公用表表达式（WITH 语法）

**CTE**（Common Table Expression，公用表表达式）使用 `WITH` 关键字定义一个**临时命名的结果集**，可以在后续查询中像表一样引用它。CTE 本质上是子查询的"升级版"，**可读性更好、支持复用、还能递归**。

> ⚠️ **版本要求**：CTE 需要 **MySQL 8.0+** 才支持。

### 1. 基础 CTE 语法

```sql
WITH cte_name AS (
    SELECT ...
)
SELECT * FROM cte_name;
```

```sql
-- 用 CTE 查询工资高于部门平均工资的员工
WITH dept_avg AS (
    SELECT dept_id, AVG(salary) AS avg_salary
    FROM employee
    WHERE dept_id IS NOT NULL
    GROUP BY dept_id
)
SELECT e.name, e.salary, da.avg_salary
FROM employee e
JOIN dept_avg da ON e.dept_id = da.dept_id
WHERE e.salary > da.avg_salary;
```

| name | salary | avg_salary |
|------|--------|------------|
| 周八 | 22000 | 18333.33 |
| 吴九 | 15000 | 12000.00 |
| 郑十 | 13000 | 12000.00 |

> 💡 对比一下，同样的需求如果用子查询写：
> ```sql
> SELECT e.name, e.salary, da.avg_salary
> FROM employee e
> JOIN (
>     SELECT dept_id, AVG(salary) AS avg_salary
>     FROM employee WHERE dept_id IS NOT NULL
>     GROUP BY dept_id
> ) da ON e.dept_id = da.dept_id
> WHERE e.salary > da.avg_salary;
> ```
> CTE 把子查询提取到了前面并命名，主查询更简洁、意图更清晰。

### 2. 多个 CTE

多个 CTE 用**逗号**分隔，后面的 CTE 可以引用前面的 CTE。

```sql
WITH
    dept_stats AS (
        SELECT
            dept_id,
            COUNT(*) AS emp_count,
            AVG(salary) AS avg_salary
        FROM employee
        WHERE dept_id IS NOT NULL
        GROUP BY dept_id
    ),
    project_stats AS (
        SELECT
            emp_id,
            COUNT(*) AS project_count
        FROM emp_project
        GROUP BY emp_id
    )
SELECT
    e.name,
    d.dept_name,
    ROUND(ds.avg_salary, 2) AS dept_avg,
    COALESCE(ps.project_count, 0) AS projects
FROM employee e
JOIN department d ON e.dept_id = d.id
JOIN dept_stats ds ON e.dept_id = ds.dept_id
LEFT JOIN project_stats ps ON e.id = ps.emp_id;
```

| name | dept_name | dept_avg | projects |
|------|-----------|----------|----------|
| 张三 | 技术部 | 18333.33 | 1 |
| 李四 | 技术部 | 18333.33 | 2 |
| 王五 | 销售部 | 12000.00 | 1 |
| 赵六 | 销售部 | 12000.00 | 0 |
| 孙七 | 人事部 | 12000.00 | 0 |
| 周八 | 技术部 | 18333.33 | 2 |
| 吴九 | 销售部 | 12000.00 | 1 |
| 郑十 | 人事部 | 12000.00 | 1 |

> 💡 **补充**：多个 CTE 之间用逗号分隔，只需要写**一个 `WITH`** 关键字。每个 CTE 都可以引用在它之前定义的其他 CTE。

### 3. ⭐ 递归 CTE（WITH RECURSIVE）

递归 CTE 是 CTE 最强大的功能，可以实现**层级遍历**、**树形结构查询**等子查询做不到的事情。

**基本结构**：

```sql
WITH RECURSIVE cte_name AS (
    -- 锚点成员（Anchor）：递归的起点
    SELECT ...

    UNION ALL

    -- 递归成员（Recursive）：引用自身，逐层展开
    SELECT ...
    FROM cte_name
    JOIN ...
)
SELECT * FROM cte_name;
```

```sql
-- 从周八开始，递归查询所有下属（多层级）
WITH RECURSIVE subordinates AS (
    -- 锚点：从周八开始
    SELECT id, name, manager_id, 1 AS level
    FROM employee
    WHERE name = '周八'

    UNION ALL

    -- 递归：查找每个人的下属
    SELECT e.id, e.name, e.manager_id, s.level + 1
    FROM employee e
    INNER JOIN subordinates s ON e.manager_id = s.id
)
SELECT id, name, manager_id, level FROM subordinates;
```

结果：

| id | name | manager_id | level |
|----|------|-----------|-------|
| 6  | 周八 | NULL | 1 |
| 1  | 张三 | 6 | 2 |
| 2  | 李四 | 6 | 2 |
| 7  | 吴九 | 6 | 2 |
| 3  | 王五 | 7 | 3 |
| 4  | 赵六 | 7 | 3 |

> 💡 **执行过程**：
> 1. **第 1 轮**（锚点）：找到周八（level=1）
> 2. **第 2 轮**（递归）：找 manager_id=6 的员工 → 张三、李四、吴九（level=2）
> 3. **第 3 轮**（递归）：找 manager_id 为 1、2、7 的员工 → 王五、赵六（level=3，manager_id=7）
> 4. **第 4 轮**：找 manager_id 为 3、4 的员工 → 无 → 递归终止

> ⚠️ **注意事项**：
> - 递归 CTE **必须**包含 `UNION ALL`（或 `UNION`）连接锚点和递归成员
> - 递归成员**必须引用 CTE 自身**，并且有终止条件（否则无限递归）
> - MySQL 默认递归深度限制为 **1000 次**，可通过 `SET cte_max_recursion_depth = N` 调整
> - 递归 CTE 中推荐使用 `UNION ALL`（更高效），除非确实需要去重才用 `UNION`

**更多递归场景——生成连续日期序列**：

```sql
-- 生成 2026-02-01 到 2026-02-10 的日期序列
WITH RECURSIVE date_series AS (
    SELECT DATE('2026-02-01') AS dt
    UNION ALL
    SELECT DATE_ADD(dt, INTERVAL 1 DAY)
    FROM date_series
    WHERE dt < '2026-02-10'
)
SELECT dt FROM date_series;
```

> 💡 **实用场景**：生成日期序列后，可以 LEFT JOIN 业务表，统计**每天**的数据（包括没有数据的日子显示为 0），这在报表开发中非常常用。

### 4. ⭐ CTE vs 子查询 对比

| 对比维度 | CTE（WITH） | 子查询 |
|---------|------------|--------|
| **可读性** | ✅ 好，命名清晰，主查询简洁 | 嵌套深时较差 |
| **可复用** | ✅ 同一语句中可**多次引用** | ❌ 每次都要重写 |
| **递归支持** | ✅ 支持 `WITH RECURSIVE` | ❌ 不支持 |
| **MySQL 版本** | 8.0+ | 所有版本 |
| **性能** | 基本等同（优化器可能自动物化） | 基本等同 |
| **作用范围** | 仅在当前语句中有效 | 仅在所在位置有效 |

> 💡 **选择建议**：
> - 简单的一次性嵌套 → **子查询**即可
> - 需要复用、嵌套层次深、或需要递归 → 用 **CTE**
> - 需要兼容 MySQL 5.x → 只能用**子查询**

```sql
-- 同一个需求的三种写法对比

-- 写法1：子查询嵌套（可读性差）
SELECT name, salary FROM employee
WHERE dept_id IN (
    SELECT dept_id FROM (
        SELECT dept_id, AVG(salary) AS avg_sal
        FROM employee
        WHERE dept_id IS NOT NULL
        GROUP BY dept_id
    ) t
    WHERE avg_sal > 13000
);

-- 写法2：CTE（可读性好）
WITH dept_avg AS (
    SELECT dept_id, AVG(salary) AS avg_sal
    FROM employee
    WHERE dept_id IS NOT NULL
    GROUP BY dept_id
),
high_avg_dept AS (
    SELECT dept_id FROM dept_avg WHERE avg_sal > 13000
)
SELECT name, salary
FROM employee
WHERE dept_id IN (SELECT dept_id FROM high_avg_dept);
```

---

## 七、子查询 vs JOIN 改写

很多子查询可以改写为 JOIN，通常 JOIN 的**性能更好**（优化器更容易优化），也**更易读**。

```sql
-- 子查询写法：查询参与了项目的员工
SELECT name FROM employee
WHERE id IN (SELECT emp_id FROM emp_project);

-- JOIN 改写：效果一样，但通常性能更好
SELECT DISTINCT e.name
FROM employee e
JOIN emp_project ep ON e.id = ep.emp_id;
```

```sql
-- 子查询写法：查询没有参与项目的员工
SELECT name FROM employee
WHERE id NOT IN (SELECT emp_id FROM emp_project);

-- JOIN 改写：用 LEFT JOIN + IS NULL
SELECT e.name
FROM employee e
LEFT JOIN emp_project ep ON e.id = ep.emp_id
WHERE ep.emp_id IS NULL;
```

> 💡 **改写建议**：
> - `IN (子查询)` → 可改写为 `JOIN`（注意去重）
> - `NOT IN (子查询)` → 可改写为 `LEFT JOIN ... WHERE IS NULL`（更安全，避免 NULL 问题）
> - `EXISTS (关联子查询)` → 可改写为 `JOIN`
> - 不是所有子查询都适合改写，**关联子查询中对分组计算的引用**有时用子查询更自然

---

## 八、综合速查表

| 需求 | 推荐语法 | 要点 |
|------|---------|------|
| 与单个值比较 | 标量子查询 + `=`/`>`/`<` | 子查询必须返回一行一列 |
| 判断是否在集合中 | `IN (子查询)` | 简单直观 |
| 判断集合中是否有匹配 | `EXISTS (关联子查询)` | NOT EXISTS 更安全（无 NULL 陷阱） |
| 大于/小于集合中的某个值 | `> ANY` / `< ANY` | ANY = 任意一个满足即可 |
| 大于/小于集合中的所有值 | `> ALL` / `< ALL` | ALL = 全部满足 |
| 把子查询当临时表 | `FROM (子查询) AS 别名` | 派生表必须有别名 |
| 提高复杂查询可读性 | `WITH ... AS (...)` | CTE，MySQL 8.0+ |
| 同一查询中复用结果集 | `WITH` 多个 CTE | 逗号分隔，只写一个 WITH |
| 层级遍历 / 树形结构 | `WITH RECURSIVE` | 递归 CTE，需有终止条件 |
| 生成序列（日期/数字） | `WITH RECURSIVE` | 配合 UNION ALL |
| 查"有匹配的记录" | `IN` 或 `EXISTS` | 小结果集用 IN，大结果集用 EXISTS |
| 查"无匹配的记录" | `NOT EXISTS` 或 `LEFT JOIN + IS NULL` | 避免 NOT IN 的 NULL 陷阱 |

---

## 九、LeetCode 相关练习

<!-- 在此添加相关 LeetCode 题目链接 -->

> 🔗 后续待补充..

---

## 参考资料

> 📚 本文内容参考以下资料整理：
> - [编程导航 - 第 6 章：多表查询](https://www.codefather.cn/course/1952312567451275265/section/1952312567975563265)
> - [编程导航 - 第 7 章：合并查询](https://www.codefather.cn/course/1952312567451275265/section/1952312568025894913)
> - [编程导航 - 第 8 章：子查询](https://www.codefather.cn/course/1952312567451275265/section/1952312568080420866)

