---
title: SQL基础-条件查询与分组查询
date: 2026-02-20 00:10:00
tags:
  - SQL
  - 数据库
  - 学习笔记
categories:
    - 技术文章
---

最近复习 SQL 的时候，发现条件查询（WHERE）和分组查询（GROUP BY + HAVING）这两块内容很容易混淆，尤其是 `WHERE` 和 `HAVING` 的区别。再加上 `CASE WHEN` 在实际开发中几乎无处不在，所以把这些知识点整理到一起，方便对比学习。

---

<!-- more -->

本文使用以下示例表来演示所有查询： 

```sql
-- 员工表
CREATE TABLE employee (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    department VARCHAR(50),
    salary DECIMAL(10, 2),
    age INT,
    hire_date DATE
);

INSERT INTO employee VALUES
(1,  '张三', '技术部', 15000, 28, '2022-03-15'),
(2,  '李四', '技术部', 18000, 32, '2020-07-01'),
(3,  '王五', '销售部', 12000, 25, '2023-01-10'),
(4,  '赵六', '销售部', 9000,  23, '2023-06-20'),
(5,  '孙七', '人事部', 11000, 30, '2021-09-05'),
(6,  '周八', '技术部', 22000, 35, '2019-04-12'),
(7,  '吴九', '销售部', 15000, 29, '2022-11-08'),
(8,  '郑十', '人事部', 13000, 27, '2022-05-18'),
(9,  '刘一', '技术部', 9500,  22, '2024-01-15'),
(10, '陈二', '销售部', NULL,  26, '2023-08-01');
```

---

## 一、条件查询（WHERE）

`WHERE` 子句用于在查询时**过滤行**，只返回满足条件的记录。它在数据分组（`GROUP BY`）**之前**执行。

**基本语法**：

```sql
SELECT 列名 FROM 表名 WHERE 条件;
```

### 1. 比较运算符

| 运算符 | 说明 | 示例 |
|--------|------|------|
| `=` | 等于 | `WHERE age = 28` |
| `!=` 或 `<>` | 不等于 | `WHERE department != '技术部'` |
| `>` | 大于 | `WHERE salary > 15000` |
| `<` | 小于 | `WHERE age < 30` |
| `>=` | 大于等于 | `WHERE salary >= 12000` |
| `<=` | 小于等于 | `WHERE age <= 25` |

```sql
-- 查询工资大于 15000 的员工
SELECT name, salary FROM employee WHERE salary > 15000;
```

### 2. `BETWEEN...AND`

用于筛选某个**范围**内的值（包含两端边界值）。

```sql
-- 查询工资在 10000 到 18000 之间的员工（包含 10000 和 18000）
SELECT name, salary FROM employee
WHERE salary BETWEEN 10000 AND 18000;

-- 等价于
SELECT name, salary FROM employee
WHERE salary >= 10000 AND salary <= 18000;
```

> ⚠️ **注意**：`BETWEEN...AND` 是**闭区间**，即包含边界值。并且小值必须写在前面，`BETWEEN 18000 AND 10000` 查不到任何结果。

也可以用于日期范围：

```sql
-- 查询 2022 年入职的员工
SELECT name, hire_date FROM employee
WHERE hire_date BETWEEN '2022-01-01' AND '2022-12-31';
```

`NOT BETWEEN` 可以排除范围：

```sql
-- 查询工资不在 10000~15000 范围内的员工
SELECT name, salary FROM employee
WHERE salary NOT BETWEEN 10000 AND 15000;
```

### 3. `IN`

用于判断值是否在一个**指定的集合**中，替代多个 `OR` 条件。

```sql
-- 查询技术部和销售部的员工
SELECT name, department FROM employee
WHERE department IN ('技术部', '销售部');

-- 等价于
SELECT name, department FROM employee
WHERE department = '技术部' OR department = '销售部';
```

`NOT IN` 可以排除指定值：

```sql
-- 查询不是技术部和销售部的员工
SELECT name, department FROM employee
WHERE department NOT IN ('技术部', '销售部');
```

> ⚠️ **注意**：`IN` 列表中如果包含 `NULL`，`NOT IN` 的结果可能不符合预期。例如 `WHERE salary NOT IN (15000, NULL)` 会返回**空结果**，因为任何值与 `NULL` 的比较结果都是 `UNKNOWN`。这是一个非常容易踩的坑。

`IN` 还可以配合子查询使用：

```sql
-- 查询工资高于平均值的部门有哪些员工
SELECT name, department FROM employee
WHERE department IN (
    SELECT department FROM employee
    GROUP BY department
    HAVING AVG(salary) > 13000
);
```

### 4. `LIKE`（模糊匹配）

用于字符串的**模式匹配**，支持两个通配符：

| 通配符 | 说明 | 示例 |
|--------|------|------|
| `%` | 匹配**任意数量**的字符（包括 0 个） | `'张%'` 匹配以「张」开头的任意字符串 |
| `_` | 匹配**恰好 1 个**字符 | `'张_'` 匹配「张」+ 任意一个字符 |

```sql
-- 查询姓「张」的员工
SELECT * FROM employee WHERE name LIKE '张%';

-- 查询名字恰好两个字且姓「张」的员工
SELECT * FROM employee WHERE name LIKE '张_';

-- 查询名字中包含「三」的员工
SELECT * FROM employee WHERE name LIKE '%三%';
```

`NOT LIKE` 可以排除匹配：

```sql
-- 查询名字不以「张」开头的员工
SELECT * FROM employee WHERE name NOT LIKE '张%';
```

> 💡 **补充**：`LIKE` 默认不区分大小写（取决于表的字符集排序规则 `COLLATION`）。如果需要区分大小写，可以使用 `LIKE BINARY`：
> ```sql
> SELECT * FROM employee WHERE name LIKE BINARY 'zhang%';
> ```

> ⚠️ **性能提示**：`LIKE '%xxx'`（前缀有 `%`）会导致**全表扫描**，无法利用索引。尽量使用 `LIKE 'xxx%'` 的形式。

### 5. `IS NULL` / `IS NOT NULL`

用于判断值是否为 `NULL`。

```sql
-- 查询工资为空的员工
SELECT name FROM employee WHERE salary IS NULL;

-- 查询工资不为空的员工
SELECT name, salary FROM employee WHERE salary IS NOT NULL;
```

> ⚠️ **注意**：不能用 `= NULL` 来判断空值！`WHERE salary = NULL` 永远返回空结果，因为 `NULL` 与任何值（包括 `NULL` 自身）的比较结果都是 `UNKNOWN`。必须使用 `IS NULL`。

### 6. 逻辑运算符与优先级

| 运算符 | 说明 |
|--------|------|
| `AND` | 逻辑与，两个条件**都满足** |
| `OR` | 逻辑或，**任一条件**满足 |
| `NOT` | 逻辑非，取反 |

**优先级（从高到低）**：`NOT` > `AND` > `OR`

```sql
-- 查询技术部工资大于 15000 的员工
SELECT * FROM employee
WHERE department = '技术部' AND salary > 15000;

-- 查询技术部或者工资大于 15000 的员工
SELECT * FROM employee
WHERE department = '技术部' OR salary > 15000;
```

> ⚠️ **易错点**：由于 `AND` 优先级高于 `OR`，混用时一定要注意加**括号**！

```sql
-- ❌ 错误理解：想查询「技术部或销售部」中工资大于 15000 的人
SELECT * FROM employee
WHERE department = '技术部' OR department = '销售部' AND salary > 15000;
-- 实际执行：department = '技术部' OR (department = '销售部' AND salary > 15000)
-- 技术部的所有人都会被选出来，不管工资多少！

-- ✅ 正确写法：用括号明确优先级
SELECT * FROM employee
WHERE (department = '技术部' OR department = '销售部') AND salary > 15000;
```

> 💡 **建议**：即使你清楚优先级，也推荐使用**括号**来让查询意图更加明确、可读性更好。

---

## 二、分组查询（GROUP BY + HAVING）

`GROUP BY` 用于将数据按照某些列**分组**，通常搭配**聚合函数**一起使用，对每组数据进行统计计算。

### 1. 聚合函数

在讲 `GROUP BY` 之前，先回顾一下常用的聚合函数：

| 函数 | 说明 | 注意事项 |
|------|------|----------|
| `COUNT(*)` | 统计行数（包含 NULL） | - |
| `COUNT(列名)` | 统计非 NULL 的行数 | 会忽略 NULL 值 |
| `SUM(列名)` | 求和 | 忽略 NULL |
| `AVG(列名)` | 求平均值 | 忽略 NULL（分母不含 NULL 行） |
| `MAX(列名)` | 求最大值 | - |
| `MIN(列名)` | 求最小值 | - |

```sql
-- 统计总人数
SELECT COUNT(*) AS total FROM employee;
-- 输出: 10

-- 统计有工资记录的人数（排除 NULL）
SELECT COUNT(salary) AS has_salary FROM employee;
-- 输出: 9（陈二的 salary 是 NULL，被排除）

-- 求平均工资
SELECT AVG(salary) AS avg_salary FROM employee;
-- 注意：AVG 会忽略 NULL，即除以 9 而不是 10
```

> ⚠️ **易错点**：`COUNT(*)` 和 `COUNT(列名)` 的区别——`COUNT(*)` 统计所有行，`COUNT(列名)` 只统计该列非 NULL 的行。`AVG` 也会忽略 NULL 行，这在有缺失数据时可能导致平均值偏高。

### 2. `GROUP BY` 基础语法

```sql
SELECT 分组列, 聚合函数 FROM 表名
WHERE 条件
GROUP BY 分组列;
```

```sql
-- 按部门统计人数和平均工资
SELECT
    department,
    COUNT(*) AS emp_count,
    ROUND(AVG(salary), 2) AS avg_salary
FROM employee
GROUP BY department;
```

结果：

| department | emp_count | avg_salary |
|------------|-----------|------------|
| 技术部 | 4 | 16125.00 |
| 销售部 | 4 | 12000.00 |
| 人事部 | 2 | 12000.00 |

> ⚠️ **重要规则**：`SELECT` 中出现的列，要么是 `GROUP BY` 的分组列，要么包裹在聚合函数中。否则在严格模式（`ONLY_FULL_GROUP_BY`）下会报错。

```sql
-- ❌ 错误：name 既不是分组列，也没用聚合函数包裹
SELECT name, department, COUNT(*) FROM employee GROUP BY department;

-- ✅ 正确
SELECT department, COUNT(*) FROM employee GROUP BY department;
```

多列分组：

```sql
-- 按部门和年龄段分组
SELECT
    department,
    CASE
        WHEN age < 25 THEN '25岁以下'
        WHEN age BETWEEN 25 AND 30 THEN '25-30岁'
        ELSE '30岁以上'
    END AS age_group,
    COUNT(*) AS emp_count
FROM employee
GROUP BY department, age_group;
```

### 3. `HAVING` 过滤分组

`HAVING` 用于对**分组后的结果**进行过滤，它是配合 `GROUP BY` 使用的。

```sql
SELECT 分组列, 聚合函数 FROM 表名
GROUP BY 分组列
HAVING 聚合条件;
```

```sql
-- 查询平均工资大于 12000 的部门
SELECT
    department,
    ROUND(AVG(salary), 2) AS avg_salary
FROM employee
GROUP BY department
HAVING AVG(salary) > 12000;
```

```sql
-- 查询员工人数大于等于 3 的部门
SELECT
    department,
    COUNT(*) AS emp_count
FROM employee
GROUP BY department
HAVING COUNT(*) >= 3;
```

### 4. ⭐ `WHERE` vs `HAVING` 对比（重点）

这是最容易混淆的地方，我们从多个维度进行对比：

| 对比维度 | WHERE | HAVING |
|----------|-------|--------|
| **作用对象** | 过滤**原始行** | 过滤**分组后**的结果 |
| **执行时机** | `GROUP BY` **之前** | `GROUP BY` **之后** |
| **能否使用聚合函数** | ❌ 不能 | ✅ 可以 |
| **能否使用列别名** | ❌ 不能（MySQL 中不行） | ✅ 可以（MySQL 支持） |
| **是否依赖 GROUP BY** | 不依赖，可以单独使用 | 通常需要配合 GROUP BY |

**SQL 执行顺序**：

```
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

> 💡 理解这个执行顺序是区分 `WHERE` 和 `HAVING` 的关键。`WHERE` 在分组前过滤，所以它无法使用聚合函数；`HAVING` 在分组后过滤，所以它可以用聚合函数作为条件。

**对比示例**：

```sql
-- 需求：查询「技术部和销售部」中，平均工资大于 12000 的部门

-- WHERE 负责分组前的过滤：筛选部门
-- HAVING 负责分组后的过滤：筛选平均工资
SELECT
    department,
    ROUND(AVG(salary), 2) AS avg_salary
FROM employee
WHERE department IN ('技术部', '销售部')   -- 先过滤行
GROUP BY department                         -- 再分组
HAVING AVG(salary) > 12000;                 -- 最后过滤分组
```

```sql
-- ❌ 错误：WHERE 中不能使用聚合函数
SELECT department, AVG(salary)
FROM employee
WHERE AVG(salary) > 12000   -- 报错！
GROUP BY department;

-- ✅ 正确：聚合条件必须放在 HAVING 中
SELECT department, AVG(salary)
FROM employee
GROUP BY department
HAVING AVG(salary) > 12000;
```

> 💡 **简单记忆法**：
> - 能用 `WHERE` 的条件就尽量用 `WHERE`（先过滤可以减少分组的数据量，**性能更好**）
> - 涉及聚合函数的条件，只能用 `HAVING`

---

## 三、CASE WHEN 条件表达式

`CASE WHEN` 是 SQL 中的**条件表达式**，类似于编程语言中的 `if-else`，可以在查询中根据条件返回不同的值。它是 SQL 标准语法，所有数据库都支持。

### 1. 两种语法形式

**形式一：简单 CASE**（等值匹配）

```sql
CASE 表达式
    WHEN 值1 THEN 结果1
    WHEN 值2 THEN 结果2
    ...
    ELSE 默认结果
END
```

```sql
-- 将部门名转为英文
SELECT
    name,
    department,
    CASE department
        WHEN '技术部' THEN 'Tech'
        WHEN '销售部' THEN 'Sales'
        WHEN '人事部' THEN 'HR'
        ELSE 'Other'
    END AS dept_en
FROM employee;
```

**形式二：搜索 CASE**（条件判断，更灵活）

```sql
CASE
    WHEN 条件1 THEN 结果1
    WHEN 条件2 THEN 结果2
    ...
    ELSE 默认结果
END
```

```sql
-- 按工资划分等级
SELECT
    name,
    salary,
    CASE
        WHEN salary >= 20000 THEN '高薪'
        WHEN salary >= 15000 THEN '中高薪'
        WHEN salary >= 10000 THEN '中薪'
        WHEN salary IS NULL THEN '未知'
        ELSE '低薪'
    END AS salary_level
FROM employee;
```

> ⚠️ **注意事项**：
> - `CASE WHEN` 是**按顺序**匹配的，匹配到第一个满足的条件就停止，后面的不再判断。所以条件的顺序很重要。
> - `ELSE` 是可选的，如果省略 `ELSE` 且没有条件匹配，返回 `NULL`。
> - 别忘了最后的 **`END`** 关键字！这是最常见的语法错误。
> - 搜索 CASE 比简单 CASE 更强大，建议优先使用搜索 CASE。

### 2. 在 SELECT 中使用（数据分类/转换）

这是 `CASE WHEN` 最常见的用法，用来根据条件生成新的列。

```sql
-- 根据入职年份给员工打标签
SELECT
    name,
    hire_date,
    CASE
        WHEN DATEDIFF(CURDATE(), hire_date) >= 1095 THEN '老员工'
        WHEN DATEDIFF(CURDATE(), hire_date) >= 365  THEN '正式员工'
        ELSE '新员工'
    END AS emp_tag
FROM employee;
```

### 3. 在 WHERE 中使用（条件筛选）

```sql
-- 筛选出不同部门满足不同工资标准的员工
-- 技术部要求工资 >= 15000，其他部门要求 >= 10000
SELECT name, department, salary
FROM employee
WHERE salary >= CASE
    WHEN department = '技术部' THEN 15000
    ELSE 10000
END;
```

### 4. 在 ORDER BY 中使用（自定义排序）

```sql
-- 自定义部门排序：技术部排第一，销售部第二，其余最后
SELECT name, department, salary
FROM employee
ORDER BY
    CASE department
        WHEN '技术部' THEN 1
        WHEN '销售部' THEN 2
        WHEN '人事部' THEN 3
        ELSE 99
    END,
    salary DESC;
```

### 5. 在 GROUP BY 中使用（按条件分组）

```sql
-- 按工资区间分组统计人数
SELECT
    CASE
        WHEN salary >= 15000 THEN '高薪 (≥15000)'
        WHEN salary >= 10000 THEN '中薪 (10000-14999)'
        WHEN salary IS NOT NULL THEN '低薪 (<10000)'
        ELSE '未知'
    END AS salary_range,
    COUNT(*) AS emp_count,
    ROUND(AVG(salary), 2) AS avg_salary
FROM employee
GROUP BY salary_range;
```

### 6. ⭐ 搭配聚合函数（条件统计）

这是 `CASE WHEN` 最实用也最强大的用法之一——在一次查询中按不同条件分别统计。

```sql
-- 一次查询统计各部门的工资分布
SELECT
    department,
    COUNT(*) AS total,
    SUM(CASE WHEN salary >= 15000 THEN 1 ELSE 0 END) AS high_salary_count,
    SUM(CASE WHEN salary < 15000 THEN 1 ELSE 0 END) AS low_salary_count,
    ROUND(AVG(CASE WHEN salary >= 15000 THEN salary END), 2) AS avg_high_salary
FROM employee
GROUP BY department;
```

> 💡 **补充**：`SUM(CASE WHEN ... THEN 1 ELSE 0 END)` 和 `COUNT(CASE WHEN ... THEN 1 END)` 效果一样，都是条件计数。区别是 `COUNT` 版本不需要 `ELSE 0`（因为 `COUNT` 不统计 NULL）。

### 7. ⭐ 行转列（经典应用）

`CASE WHEN` + 聚合函数实现**行转列**，这是非常经典的用法。

假设有一张成绩表：

```sql
CREATE TABLE score (
    student VARCHAR(20),
    subject VARCHAR(20),
    grade INT
);

INSERT INTO score VALUES
('张三', '语文', 85), ('张三', '数学', 92), ('张三', '英语', 78),
('李四', '语文', 90), ('李四', '数学', 88), ('李四', '英语', 95),
('王五', '语文', 75), ('王五', '数学', 60), ('王五', '英语', 82);
```

**行转列**：将每个科目从行变成列

```sql
SELECT
    student,
    MAX(CASE WHEN subject = '语文' THEN grade END) AS '语文',
    MAX(CASE WHEN subject = '数学' THEN grade END) AS '数学',
    MAX(CASE WHEN subject = '英语' THEN grade END) AS '英语'
FROM score
GROUP BY student;
```

结果：

| student | 语文 | 数学 | 英语 |
|---------|------|------|------|
| 张三 | 85 | 92 | 78 |
| 李四 | 90 | 88 | 95 |
| 王五 | 75 | 60 | 82 |

> 💡 这里用 `MAX` 是因为每个学生每科只有一条记录，`MAX` 用来"去掉" `NULL`（没匹配到的科目返回 NULL，`MAX` 会忽略 NULL 取出有效值）。用 `SUM` 或 `MIN` 也可以。

### 8. CASE WHEN vs IF()

MySQL 还提供了 `IF()` 函数，在**只有两个分支**的情况下可以替代简单的 CASE WHEN：

```sql
-- IF(条件, 为真的值, 为假的值)
SELECT
    name,
    IF(salary >= 15000, '高薪', '普通') AS salary_tag
FROM employee;

-- 等价的 CASE WHEN
SELECT
    name,
    CASE WHEN salary >= 15000 THEN '高薪' ELSE '普通' END AS salary_tag
FROM employee;
```

| 对比 | CASE WHEN | IF() |
|------|-----------|------|
| 分支数量 | 支持**多分支** | 仅支持 **2 个分支** |
| SQL 标准 | ✅ 标准语法 | ❌ MySQL 特有 |
| 可读性 | 多分支时更清晰 | 简单场景更简洁 |
| 推荐场景 | 通用场景 | 简单的二选一 |

> 💡 **建议**：简单的真/假判断用 `IF()`，多条件分支用 `CASE WHEN`。如果考虑跨数据库兼容，统一使用 `CASE WHEN`。

---

## 四、综合对比速查表

| 关键字 | 作用 | 执行时机 | 能否用聚合函数 |
|--------|------|----------|---------------|
| `WHERE` | 过滤**行** | `GROUP BY` 之前 | ❌ |
| `GROUP BY` | **分组**数据 | `WHERE` 之后 | 搭配聚合函数使用 |
| `HAVING` | 过滤**分组** | `GROUP BY` 之后 | ✅ |
| `CASE WHEN` | **条件表达式** | 取决于所在位置 | 可以搭配使用 |

**完整的 SQL 执行顺序**：

```
FROM        -- 确定数据来源
  → WHERE   -- 过滤行（不能用聚合函数）
  → GROUP BY -- 分组
  → HAVING  -- 过滤分组（可以用聚合函数）
  → SELECT  -- 选择列、计算表达式
  → ORDER BY -- 排序
  → LIMIT   -- 限制返回行数
```

---

## 五、LeetCode 相关练习

<!-- 在此添加相关 LeetCode 题目链接 -->

> 🔗 后续待补充..

