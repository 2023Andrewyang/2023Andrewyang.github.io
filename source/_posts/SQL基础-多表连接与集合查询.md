---
title: SQL基础-多表连接与集合查询
date: 2026-02-20 14:00:00
tags:
  - SQL
  - 数据库
  - 学习笔记
categories:
    - 技术文章
---

最近在复习 SQL 的多表查询部分，发现 JOIN 的种类不少，再加上 UNION 集合操作，内容虽然不难但容易混淆，尤其是各种外连接的区别、ON 和 WHERE 在外连接中的不同表现等。所以把这些知识点整理到一起，方便对比学习和日后速查。

---

<!-- more -->

本文使用以下示例表来演示所有查询：

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
(4, '财务部', '深圳');  -- 该部门没有任何员工

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
(9,  '刘一', NULL, 9500,  '2024-01-15', NULL),  -- 未分配部门
(10, '陈二', NULL, 8000,  '2024-03-01', NULL);  -- 未分配部门

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
(4, '内部工具升级', '2025-01-01', 100000);  -- 该项目没有分配人员

-- 员工项目关联表（多对多）
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

> 💡 **表关系说明**：
> - `employee.dept_id` → `department.id`（多对一）
> - `employee.manager_id` → `employee.id`（自引用，用于自连接）
> - `employee` ↔ `project` 通过 `emp_project` 关联（多对多）
> - 刻意设计了一些"不匹配"的数据：员工9、10 没有部门；部门4 没有员工；项目4 没有成员——方便演示各类 JOIN。

---

## 一、交叉连接（CROSS JOIN）

交叉连接会将两张表的每一行**两两组合**，生成**笛卡尔积**，结果行数 = 表1行数 × 表2行数。

**基本语法**：

```sql
-- 显式写法
SELECT * FROM 表1 CROSS JOIN 表2;

-- 隐式写法（逗号分隔，不加 WHERE 条件）
SELECT * FROM 表1, 表2;
```

```sql
-- 示例：生成所有"员工 × 项目"的组合
SELECT e.name, p.project_name
FROM employee e
CROSS JOIN project p;
-- 结果：10 × 4 = 40 行
```

> ⚠️ **注意**：交叉连接会产生**大量数据**（M × N 行），实际开发中很少直接使用。如果在 `FROM` 中用逗号分隔多张表却**忘记写 WHERE 条件**，就会意外产生笛卡尔积，这是一个常见的性能陷阱。

> 💡 **应用场景**：生成所有可能的组合（如商品 × 折扣活动）、填充维度表、配合 `WHERE` 过滤后使用。

---

## 二、⭐ 内连接（INNER JOIN）

内连接是最常用的连接方式，**只返回两张表中满足连接条件的匹配行**，不匹配的行会被丢弃。

**基本语法**：

```sql
SELECT 列名 FROM 表1
INNER JOIN 表2 ON 表1.列 = 表2.列;

-- INNER 可以省略，JOIN 默认就是 INNER JOIN
SELECT 列名 FROM 表1
JOIN 表2 ON 表1.列 = 表2.列;
```

```sql
-- 查询每个员工所属的部门名称
SELECT e.name, e.salary, d.dept_name
FROM employee e
INNER JOIN department d ON e.dept_id = d.id;
```

结果（8 行，刘一和陈二因 dept_id 为 NULL 被排除，财务部因没有员工也不出现）：

| name | salary | dept_name |
|------|--------|-----------|
| 张三 | 15000 | 技术部 |
| 李四 | 18000 | 技术部 |
| 王五 | 12000 | 销售部 |
| 赵六 | 9000  | 销售部 |
| 孙七 | 11000 | 人事部 |
| 周八 | 22000 | 技术部 |
| 吴九 | 15000 | 销售部 |
| 郑十 | 13000 | 人事部 |

> ⚠️ **注意**：内连接只保留**两边都能匹配**的行。未分配部门的员工（刘一、陈二）和没有员工的部门（财务部）都**不会**出现在结果中。

**隐式内连接**（旧写法）：

```sql
-- 不推荐的旧写法，但需要能认识
SELECT e.name, d.dept_name
FROM employee e, department d
WHERE e.dept_id = d.id;
```

> 💡 **建议**：始终使用 `JOIN ... ON` 的显式写法，将**连接条件**（ON）和**过滤条件**（WHERE）分开，可读性更好，也不容易漏写条件导致笛卡尔积。

**多表连接**：

```sql
-- 查询参与了项目的员工：姓名、部门、项目名、角色
SELECT e.name, d.dept_name, p.project_name, ep.role
FROM employee e
JOIN department d ON e.dept_id = d.id
JOIN emp_project ep ON e.id = ep.emp_id
JOIN project p ON ep.project_id = p.id;
```

| name | dept_name | project_name | role |
|------|-----------|-------------|------|
| 张三 | 技术部 | 电商平台重构 | 开发 |
| 李四 | 技术部 | 电商平台重构 | 架构师 |
| 李四 | 技术部 | 数据分析系统 | 技术负责人 |
| 王五 | 销售部 | APP 开发 | 销售对接 |
| 周八 | 技术部 | 电商平台重构 | 项目经理 |
| 周八 | 技术部 | 数据分析系统 | 技术顾问 |
| 吴九 | 销售部 | APP 开发 | 销售负责人 |
| 郑十 | 人事部 | 数据分析系统 | 人事协调 |

> 💡 **补充**：多表连接时，JOIN 的书写顺序通常不影响最终结果（优化器会自动选择最优执行顺序），但建议按**逻辑关系**书写，方便阅读和维护。

---

## 三、⭐ 外连接（OUTER JOIN）

外连接可以保留**某一侧**或**两侧**表中不匹配的行，不匹配的部分用 `NULL` 填充。

### 1. 左外连接（LEFT JOIN）

保留**左表**的所有行，右表无匹配时填充 NULL。

```sql
SELECT 列名 FROM 表1
LEFT [OUTER] JOIN 表2 ON 连接条件;
```

```sql
-- 查询所有员工及其部门（包括未分配部门的员工）
SELECT e.name, e.salary, d.dept_name
FROM employee e
LEFT JOIN department d ON e.dept_id = d.id;
```

结果（10 行，刘一和陈二的部门显示为 NULL）：

| name | salary | dept_name |
|------|--------|-----------|
| 张三 | 15000 | 技术部 |
| 李四 | 18000 | 技术部 |
| 王五 | 12000 | 销售部 |
| 赵六 | 9000  | 销售部 |
| 孙七 | 11000 | 人事部 |
| 周八 | 22000 | 技术部 |
| 吴九 | 15000 | 销售部 |
| 郑十 | 13000 | 人事部 |
| 刘一 | 9500  | NULL |
| 陈二 | 8000  | NULL |

**⭐ 经典用法：查找"没有匹配"的记录**：

```sql
-- 查找未分配部门的员工
SELECT e.name
FROM employee e
LEFT JOIN department d ON e.dept_id = d.id
WHERE d.id IS NULL;
-- 输出: 刘一、陈二
```

> 💡 **实用技巧**：`LEFT JOIN + WHERE 右表.主键 IS NULL` 是查找"左表有、右表无"记录的经典模式，在数据完整性检查中非常常用。

### 2. 右外连接（RIGHT JOIN）

保留**右表**的所有行，左表无匹配时填充 NULL。

```sql
SELECT 列名 FROM 表1
RIGHT [OUTER] JOIN 表2 ON 连接条件;
```

```sql
-- 查询所有部门及其员工（包括没有员工的部门）
SELECT d.dept_name, d.location, e.name
FROM employee e
RIGHT JOIN department d ON e.dept_id = d.id;
```

结果（9 行，财务部出现但没有员工）：

| dept_name | location | name |
|-----------|----------|------|
| 技术部 | 北京 | 张三 |
| 技术部 | 北京 | 李四 |
| 技术部 | 北京 | 周八 |
| 销售部 | 上海 | 王五 |
| 销售部 | 上海 | 赵六 |
| 销售部 | 上海 | 吴九 |
| 人事部 | 北京 | 孙七 |
| 人事部 | 北京 | 郑十 |
| 财务部 | 深圳 | NULL |

> 💡 **补充**：`RIGHT JOIN` 可以通过**交换表的顺序**改写为 `LEFT JOIN`，两者本质相同。实际开发中 **`LEFT JOIN` 更常用**，建议统一使用 LEFT JOIN，保持代码风格一致。

```sql
-- 上面的 RIGHT JOIN 等价于：
SELECT d.dept_name, d.location, e.name
FROM department d
LEFT JOIN employee e ON e.dept_id = d.id;
```

### 3. 全外连接（FULL OUTER JOIN）

保留**两侧**表的所有行，不匹配的部分用 NULL 填充。

> ⚠️ **注意**：**MySQL 不支持** `FULL OUTER JOIN` 语法！需要通过 `LEFT JOIN` + `UNION ALL` + `RIGHT JOIN` 来模拟。

```sql
-- MySQL 模拟全外连接
-- 第一部分：LEFT JOIN 获取所有员工（含无部门的）
SELECT e.name, d.dept_name
FROM employee e
LEFT JOIN department d ON e.dept_id = d.id

UNION ALL

-- 第二部分：RIGHT JOIN 获取"仅右表有"的记录（无员工的部门）
SELECT e.name, d.dept_name
FROM employee e
RIGHT JOIN department d ON e.dept_id = d.id
WHERE e.id IS NULL;
```

> 💡 **为什么用 `UNION ALL` 而不是 `UNION`？** 因为第一部分（LEFT JOIN）已经包含了所有左表行和匹配的右表行，第二部分只取右表独有的行（`WHERE e.id IS NULL`），两部分不会重复，用 `UNION ALL` 避免了不必要的去重排序，性能更好。

### 4. ⭐ 外连接中 ON 和 WHERE 的区别

这是一个**非常容易踩坑**的地方。在外连接中，条件放在 `ON` 和 `WHERE` 中的效果完全不同：

- **`ON` 中的条件**：用于判断**连接匹配**，不匹配的行仍然保留（填 NULL）
- **`WHERE` 中的条件**：在连接**之后**过滤，会剔除不满足条件的行（包括 NULL 行）

```sql
-- ❌ 错误理解：想要"所有员工 + 只显示技术部的部门名"
SELECT e.name, d.dept_name
FROM employee e
LEFT JOIN department d ON e.dept_id = d.id
WHERE d.dept_name = '技术部';
-- 实际结果：只有技术部的 3 个员工！LEFT JOIN 完全失去了意义
-- 因为 WHERE 在连接之后过滤，非技术部的行（包括 NULL）都被剔除了

-- ✅ 正确写法：把过滤条件放在 ON 中
SELECT e.name, d.dept_name
FROM employee e
LEFT JOIN department d ON e.dept_id = d.id AND d.dept_name = '技术部';
-- 结果：所有 10 个员工都出现，技术部员工显示部门名，其余显示 NULL
```

> ⚠️ **记忆口诀**：内连接中 ON 和 WHERE 效果一样；**外连接中，想保留行就放 ON，想剔除行就放 WHERE**。

---

## 四、自连接（SELF JOIN）

自连接是指**同一张表与自身进行连接**，通过不同的**别名**来区分两个"实例"。常用于处理表中存在层级关系或需要行与行之间比较的场景。

**基本语法**：

```sql
SELECT a.列, b.列
FROM 表 a
JOIN 表 b ON a.某列 = b.某列;
```

```sql
-- 查询每个员工及其直属上级的名字
SELECT
    e.name AS employee_name,
    m.name AS manager_name
FROM employee e
LEFT JOIN employee m ON e.manager_id = m.id;
```

结果：

| employee_name | manager_name |
|---------------|-------------|
| 张三 | 周八 |
| 李四 | 周八 |
| 王五 | 吴九 |
| 赵六 | 吴九 |
| 孙七 | NULL |
| 周八 | NULL |
| 吴九 | 周八 |
| 郑十 | 孙七 |
| 刘一 | NULL |
| 陈二 | NULL |

> ⚠️ **注意**：自连接**必须**使用别名（如 `e` 和 `m`），否则 SQL 无法区分是哪个"实例"的列。这里用 `LEFT JOIN` 是为了让没有上级的员工（如周八、孙七）也出现在结果中。

**更多自连接场景**：

```sql
-- 查找同一部门中工资更高的员工对
SELECT
    a.name AS high_salary_emp,
    b.name AS low_salary_emp,
    a.salary,
    a.dept_id
FROM employee a
JOIN employee b ON a.dept_id = b.dept_id
                AND a.salary > b.salary
                AND a.id != b.id;
```

> 💡 **自连接的应用场景**：
> - 查询**层级关系**（员工 → 上级、类别 → 父类别）
> - 同一表中**行与行之间的比较**（如找同部门工资更高的人）
> - **连续日期/序号**的记录比较（如找连续登录的用户）

---

## 五、JOIN 连接类型对比总结

| 连接类型 | 关键字 | 返回结果 | 典型场景 |
|---------|--------|---------|---------|
| 交叉连接 | `CROSS JOIN` | 所有行的笛卡尔积（M × N） | 生成组合、测试数据 |
| 内连接 | `[INNER] JOIN` | 仅两边都匹配的行 | 关联查询（**最常用**） |
| 左外连接 | `LEFT [OUTER] JOIN` | 左表全部 + 右表匹配 | 保留主表所有记录 |
| 右外连接 | `RIGHT [OUTER] JOIN` | 右表全部 + 左表匹配 | 同 LEFT JOIN（换方向） |
| 全外连接 | `FULL [OUTER] JOIN` | 两表全部行 | MySQL 需用 UNION 模拟 |
| 自连接 | 同表 `JOIN` 自身 + 别名 | 取决于使用的 JOIN 类型 | 层级关系、行间比较 |

> ⭐ **核心理解**：
> - `INNER JOIN` → 取**交集**（两边都有才返回）
> - `LEFT JOIN` → 左表**全集** + 右表匹配部分
> - `RIGHT JOIN` → 右表**全集** + 左表匹配部分
> - `FULL JOIN` → 两表**并集**（MySQL 不直接支持）

---

## 六、⭐ UNION 集合操作

`UNION` 用于将多个 `SELECT` 语句的结果**纵向合并**（上下拼接）。与 JOIN 的"横向拼接"不同，UNION 是把多个查询的结果**堆叠在一起**。

### 1. `UNION`（去重合并）

```sql
SELECT 列 FROM 表1 WHERE 条件1
UNION
SELECT 列 FROM 表2 WHERE 条件2;
```

```sql
-- 合并两个查询结果：高薪员工 和 技术部员工（有重叠）
SELECT name, salary FROM employee WHERE salary > 15000
UNION
SELECT name, salary FROM employee WHERE dept_id = 1;
```

结果（李四、周八同时满足两个条件，但 UNION 自动去重，只出现一次）：

| name | salary |
|------|--------|
| 李四 | 18000 |
| 周八 | 22000 |
| 张三 | 15000 |

### 2. `UNION ALL`（不去重合并）

```sql
SELECT 列 FROM 表1 WHERE 条件1
UNION ALL
SELECT 列 FROM 表2 WHERE 条件2;
```

```sql
-- 同样的查询，但不去重
SELECT name, salary FROM employee WHERE salary > 15000
UNION ALL
SELECT name, salary FROM employee WHERE dept_id = 1;
```

结果（李四、周八各出现 2 次）：

| name | salary |
|------|--------|
| 李四 | 18000 |
| 周八 | 22000 |
| 张三 | 15000 |
| 李四 | 18000 |
| 周八 | 22000 |

### 3. ⭐ `UNION` vs `UNION ALL` 对比

| 对比维度 | UNION | UNION ALL |
|---------|-------|-----------|
| **是否去重** | ✅ 自动去除重复行 | ❌ 保留所有行（含重复） |
| **性能** | 较慢（需要额外的排序去重操作） | 较快（直接拼接） |
| **使用场景** | 需要去重的合并 | 确定无重复，或不需要去重 |

> 💡 **性能建议**：如果确定两个查询的结果**不会重复**（比如从不同的表查询），或者**允许重复**，优先使用 `UNION ALL`，性能更好。

### 4. UNION 使用注意事项

> ⚠️ **注意事项**：
> - 所有 `SELECT` 的**列数必须相同**，且对应列的数据类型要兼容
> - 结果集的**列名取自第一个** `SELECT`
> - `ORDER BY` 只能放在**最后**，对**整个合并结果**排序
> - 单独的 `SELECT` 中使用 `ORDER BY` 需要配合 `LIMIT` 才有意义（否则优化器可能忽略）

```sql
-- ❌ 错误：两个 SELECT 列数不一致
SELECT name, salary FROM employee WHERE dept_id = 1
UNION
SELECT name FROM employee WHERE dept_id = 2;
-- 报错！列数不匹配

-- ✅ 正确：列数一致
SELECT name, salary FROM employee WHERE dept_id = 1
UNION
SELECT name, salary FROM employee WHERE dept_id = 2;
```

```sql
-- ORDER BY 放在最后，对合并结果统一排序
SELECT name, salary FROM employee WHERE dept_id = 1
UNION
SELECT name, salary FROM employee WHERE dept_id = 2
ORDER BY salary DESC;
```

```sql
-- 各 SELECT 中使用 ORDER BY 需配合 LIMIT（取各部门工资前2名再合并）
(SELECT name, salary FROM employee WHERE dept_id = 1 ORDER BY salary DESC LIMIT 2)
UNION ALL
(SELECT name, salary FROM employee WHERE dept_id = 2 ORDER BY salary DESC LIMIT 2);
```

> 💡 **补充**：MySQL 8.0.31+ 开始支持 `INTERSECT`（交集）和 `EXCEPT`（差集）操作：
> ```sql
> -- 同时满足两个条件的员工（交集）
> SELECT name FROM employee WHERE salary > 15000
> INTERSECT
> SELECT name FROM employee WHERE dept_id = 1;
> -- 输出: 李四、周八
>
> -- 在技术部但工资不超过 15000 的员工（差集）
> SELECT name FROM employee WHERE dept_id = 1
> EXCEPT
> SELECT name FROM employee WHERE salary > 15000;
> -- 输出: 张三
> ```

---

## 七、JOIN vs UNION 对比

| 对比维度 | JOIN | UNION |
|---------|------|-------|
| **合并方向** | 横向拼接（增加列） | 纵向拼接（增加行） |
| **表关系** | 通过关联条件连接不同表的列 | 将多个查询结果上下堆叠 |
| **列要求** | 各表列数可以不同 | 各 SELECT 列数必须相同 |
| **典型场景** | 关联查询不同表的字段 | 合并同结构的查询结果 |

---

## 八、综合速查表

| 需求 | 语法 | 要点 |
|------|------|------|
| 笛卡尔积 | `CROSS JOIN` | M × N 行，慎用 |
| 关联查询（仅匹配行） | `[INNER] JOIN ... ON` | 最常用的连接方式 |
| 保留左表全部 | `LEFT JOIN ... ON` | 右表无匹配填 NULL |
| 保留右表全部 | `RIGHT JOIN ... ON` | 可改写为 LEFT JOIN |
| 保留两边全部 | `LEFT JOIN` + `UNION ALL` + `RIGHT JOIN` | MySQL 不支持 FULL JOIN |
| 自身关联 | 同表 `JOIN` + 别名 | 层级关系 / 行间比较 |
| 纵向合并（去重） | `UNION` | 列数和类型须一致 |
| 纵向合并（不去重） | `UNION ALL` | 性能更好，优先考虑 |
| 查"左有右无" | `LEFT JOIN ... WHERE 右表.pk IS NULL` | 数据完整性检查 |
| 外连接保留行的过滤 | 条件放 `ON` 中 | 放 `WHERE` 会剔除 NULL 行 |

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

