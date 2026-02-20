---
title: SQL基础-开窗函数
date: 2026-02-20 13:42:00
tags:
  - SQL
  - 数据库
  - 学习笔记
categories:
    - 技术文章
---

最近在复习 SQL 的开窗函数（Window Functions），这是 SQL 中非常强大的高级功能，能在**保留每一行原始数据**的同时进行分组计算，在数据分析场景中几乎无处不在。排名、累计求和、环比计算等需求都离不开它。内容不少，整理成一份速查手册方便日后回顾。

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
    hire_date DATE
);

INSERT INTO employee VALUES
(1, '张三', '技术部', 15000, '2022-03-15'),
(2, '李四', '技术部', 18000, '2020-07-01'),
(3, '王五', '销售部', 12000, '2023-01-10'),
(4, '赵六', '销售部', 9000,  '2023-06-20'),
(5, '孙七', '人事部', 11000, '2021-09-05'),
(6, '周八', '技术部', 22000, '2019-04-12'),
(7, '吴九', '销售部', 15000, '2022-11-08'),
(8, '郑十', '人事部', 13000, '2022-05-18');

-- 月度销售记录表
CREATE TABLE monthly_sales (
    id INT PRIMARY KEY AUTO_INCREMENT,
    salesperson VARCHAR(50),
    month VARCHAR(7),
    amount DECIMAL(10, 2)
);

INSERT INTO monthly_sales VALUES
(1,  '王五', '2025-01', 30000),
(2,  '赵六', '2025-01', 25000),
(3,  '吴九', '2025-01', 35000),
(4,  '王五', '2025-02', 28000),
(5,  '赵六', '2025-02', 32000),
(6,  '吴九', '2025-02', 40000),
(7,  '王五', '2025-03', 35000),
(8,  '赵六', '2025-03', 28000),
(9,  '吴九', '2025-03', 38000),
(10, '王五', '2025-04', 42000),
(11, '赵六', '2025-04', 30000),
(12, '吴九', '2025-04', 45000);
```

> 💡 **表说明**：
> - `employee` 表：8 名员工分布在 3 个部门，用于演示排名、分组计算等
> - `monthly_sales` 表：3 位销售人员 4 个月的业绩记录，用于演示累计求和、环比分析等

---

## 一、开窗函数概述

### 1. 什么是开窗函数

开窗函数（Window Functions）可以对结果集的一个子集（称为"**窗口**"）进行计算，**不需要 GROUP BY 就能做聚合计算，而且不会减少行数**。每一行都能"看到"自己所在窗口内的其他行，并基于它们进行计算。

> ⚠️ **版本要求**：开窗函数需要 **MySQL 8.0+** 才支持。

### 2. ⭐ 与 GROUP BY 聚合函数的区别

这是理解开窗函数最关键的一点：

| 对比维度 | GROUP BY + 聚合函数 | 开窗函数 |
|---------|-------------------|---------|
| **行数变化** | 多行合并为一行 | ✅ **保留所有原始行** |
| **原始数据** | 聚合后看不到明细 | ✅ 聚合结果和明细**并存** |
| **使用位置** | 需要 GROUP BY 子句 | 使用 OVER() 子句 |

```sql
-- GROUP BY：按部门求平均工资 → 只有 3 行
SELECT department, AVG(salary) AS avg_salary
FROM employee
GROUP BY department;
```

| department | avg_salary |
|------------|------------|
| 技术部 | 18333.33 |
| 销售部 | 12000.00 |
| 人事部 | 12000.00 |

```sql
-- 开窗函数：每个员工都保留，同时附上部门平均工资 → 仍然 8 行
SELECT
    name, department, salary,
    AVG(salary) OVER(PARTITION BY department) AS dept_avg
FROM employee;
```

| name | department | salary | dept_avg |
|------|-----------|--------|----------|
| 张三 | 技术部 | 15000 | 18333.33 |
| 李四 | 技术部 | 18000 | 18333.33 |
| 周八 | 技术部 | 22000 | 18333.33 |
| 王五 | 销售部 | 12000 | 12000.00 |
| 赵六 | 销售部 | 9000  | 12000.00 |
| 吴九 | 销售部 | 15000 | 12000.00 |
| 孙七 | 人事部 | 11000 | 12000.00 |
| 郑十 | 人事部 | 13000 | 12000.00 |

> ⭐ **核心区别**：GROUP BY 把多行"压缩"成一行，开窗函数在每行"旁边"附加聚合结果。

### 3. 基本语法

```sql
窗口函数() OVER (
    [PARTITION BY 分区列]    -- 可选：按什么分组
    [ORDER BY 排序列]        -- 可选：组内按什么排序
    [窗口范围]               -- 可选：计算范围（Frame）
)
```

**常见的开窗函数类型**：

| 类型 | 函数 | 说明 |
|------|------|------|
| 聚合函数 | `SUM`、`AVG`、`COUNT`、`MAX`、`MIN` | 配合 OVER() 使用 |
| 排名函数 | `ROW_NUMBER`、`RANK`、`DENSE_RANK`、`NTILE` | 为行编号或排名 |
| 偏移函数 | `LAG`、`LEAD` | 访问前/后行的数据 |
| 取值函数 | `FIRST_VALUE`、`LAST_VALUE`、`NTH_VALUE` | 取窗口中特定位置的值 |

---

## 二、PARTITION BY 与 ORDER BY

`PARTITION BY` 和 `ORDER BY` 是开窗函数中最重要的两个子句，它们决定了"窗口"的形状。

### 1. `PARTITION BY`（分区）

将数据按指定列分成多个**分区**，开窗函数在每个分区内**独立计算**。类似 GROUP BY 的分组，但不合并行。

```sql
-- 计算每个员工的工资占其部门总工资的百分比
SELECT
    name, department, salary,
    SUM(salary) OVER(PARTITION BY department) AS dept_total,
    ROUND(salary / SUM(salary) OVER(PARTITION BY department) * 100, 2) AS pct
FROM employee;
```

| name | department | salary | dept_total | pct |
|------|-----------|--------|------------|-----|
| 张三 | 技术部 | 15000 | 55000 | 27.27 |
| 李四 | 技术部 | 18000 | 55000 | 32.73 |
| 周八 | 技术部 | 22000 | 55000 | 40.00 |
| 王五 | 销售部 | 12000 | 36000 | 33.33 |
| 赵六 | 销售部 | 9000  | 36000 | 25.00 |
| 吴九 | 销售部 | 15000 | 36000 | 41.67 |
| 孙七 | 人事部 | 11000 | 24000 | 45.83 |
| 郑十 | 人事部 | 13000 | 24000 | 54.17 |

> 💡 **省略 PARTITION BY**：如果不写 `PARTITION BY`，整个结果集被视为**一个分区**。

```sql
-- 不分区：计算每个人工资占全公司总工资的百分比
SELECT name, salary,
    ROUND(salary / SUM(salary) OVER() * 100, 2) AS company_pct
FROM employee;
```

### 2. `ORDER BY`（排序）

指定分区内的**行排列顺序**，这对排名函数和累计计算至关重要。

```sql
-- 按月份排序，计算累计销售额
SELECT
    salesperson, month, amount,
    SUM(amount) OVER(PARTITION BY salesperson ORDER BY month) AS running_total
FROM monthly_sales;
```

| salesperson | month | amount | running_total |
|-------------|-------|--------|---------------|
| 吴九 | 2025-01 | 35000 | 35000 |
| 吴九 | 2025-02 | 40000 | 75000 |
| 吴九 | 2025-03 | 38000 | 113000 |
| 吴九 | 2025-04 | 45000 | 158000 |
| 王五 | 2025-01 | 30000 | 30000 |
| 王五 | 2025-02 | 28000 | 58000 |
| 王五 | 2025-03 | 35000 | 93000 |
| 王五 | 2025-04 | 42000 | 135000 |
| 赵六 | 2025-01 | 25000 | 25000 |
| 赵六 | 2025-02 | 32000 | 57000 |
| 赵六 | 2025-03 | 28000 | 85000 |
| 赵六 | 2025-04 | 30000 | 115000 |

> ⚠️ **重要**：在聚合开窗函数中加了 `ORDER BY` 后，默认的计算范围会变成**从分区开头到当前行**（即累计计算），而不是整个分区！这一点很容易忽略，后面"窗口范围"章节会详细说明。

### 3. 组合使用

```sql
-- PARTITION BY + ORDER BY：每个销售人员按月排序的累计业绩
SELECT
    salesperson, month, amount,
    SUM(amount) OVER(PARTITION BY salesperson ORDER BY month) AS person_running,
    SUM(amount) OVER(PARTITION BY salesperson) AS person_total
FROM monthly_sales;
```

> 💡 **对比理解**：
> - 只有 `PARTITION BY`（无 ORDER BY）→ 整个分区的汇总值（每行相同）
> - `PARTITION BY` + `ORDER BY` → 分区内按顺序的**累计值**（逐行递增）

---

## 三、聚合开窗函数

常见的聚合函数 `SUM`、`AVG`、`COUNT`、`MAX`、`MIN` 都可以搭配 `OVER()` 使用。

### 1. `SUM() OVER()`

```sql
-- 1. 全局总和（每行都显示相同的总数）
SELECT salesperson, month, amount,
    SUM(amount) OVER() AS grand_total
FROM monthly_sales;
-- grand_total 每行都是 408000

-- 2. 分区总和
SELECT salesperson, month, amount,
    SUM(amount) OVER(PARTITION BY salesperson) AS person_total
FROM monthly_sales;

-- 3. 累计总和（加 ORDER BY 后变成逐行累加）
SELECT salesperson, month, amount,
    SUM(amount) OVER(PARTITION BY salesperson ORDER BY month) AS running_total
FROM monthly_sales;
```

### 2. `AVG() OVER()`

```sql
-- 每个员工的工资与部门平均工资的差值
SELECT
    name, department, salary,
    ROUND(AVG(salary) OVER(PARTITION BY department), 2) AS dept_avg,
    salary - ROUND(AVG(salary) OVER(PARTITION BY department), 2) AS diff
FROM employee;
```

| name | department | salary | dept_avg | diff |
|------|-----------|--------|----------|------|
| 张三 | 技术部 | 15000 | 18333.33 | -3333.33 |
| 李四 | 技术部 | 18000 | 18333.33 | -333.33 |
| 周八 | 技术部 | 22000 | 18333.33 | 3666.67 |
| 王五 | 销售部 | 12000 | 12000.00 | 0.00 |
| 赵六 | 销售部 | 9000  | 12000.00 | -3000.00 |
| 吴九 | 销售部 | 15000 | 12000.00 | 3000.00 |
| 孙七 | 人事部 | 11000 | 12000.00 | -1000.00 |
| 郑十 | 人事部 | 13000 | 12000.00 | 1000.00 |

### 3. `COUNT() / MAX() / MIN() OVER()`

```sql
-- 每个部门的人数、最高工资、最低工资
SELECT
    name, department, salary,
    COUNT(*) OVER(PARTITION BY department) AS dept_count,
    MAX(salary) OVER(PARTITION BY department) AS dept_max,
    MIN(salary) OVER(PARTITION BY department) AS dept_min
FROM employee;
```

---

## 四、窗口范围（Frame）详解

窗口范围（Frame）定义了当前行参与计算的"可见范围"。这是开窗函数中**最容易被忽略但非常重要**的概念。

### 1. 基本语法

```sql
ROWS BETWEEN 起点 AND 终点
```

**常用的起点/终点关键字**：

| 关键字 | 说明 |
|--------|------|
| `UNBOUNDED PRECEDING` | 分区的**第一行** |
| `N PRECEDING` | 当前行的**前 N 行** |
| `CURRENT ROW` | **当前行** |
| `N FOLLOWING` | 当前行的**后 N 行** |
| `UNBOUNDED FOLLOWING` | 分区的**最后一行** |

### 2. 默认窗口范围

> ⚠️ **易错点**：有无 `ORDER BY` 会导致默认窗口范围不同！

| 情况 | 默认窗口范围 | 效果 |
|------|------------|------|
| 无 ORDER BY | `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` | 整个分区 |
| 有 ORDER BY | `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | 分区开头到当前行 |

这就是为什么加了 `ORDER BY` 后，`SUM() OVER()` 会变成**累计求和**的原因。

```sql
-- 无 ORDER BY → 整个分区的总和（每行一样）
SELECT salesperson, month, amount,
    SUM(amount) OVER(PARTITION BY salesperson) AS total
FROM monthly_sales;
-- 王五: 135000, 135000, 135000, 135000

-- 有 ORDER BY → 累计求和（逐行递增）
SELECT salesperson, month, amount,
    SUM(amount) OVER(PARTITION BY salesperson ORDER BY month) AS running
FROM monthly_sales;
-- 王五: 30000, 58000, 93000, 135000
```

### 3. 常用窗口范围模式

```sql
-- 移动平均（前1行 + 当前行 + 后1行 = 3行滑动窗口）
SELECT
    salesperson, month, amount,
    ROUND(AVG(amount) OVER(
        PARTITION BY salesperson
        ORDER BY month
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ), 2) AS moving_avg_3
FROM monthly_sales
WHERE salesperson = '王五';
```

| salesperson | month | amount | moving_avg_3 |
|-------------|-------|--------|-------------|
| 王五 | 2025-01 | 30000 | 29000.00 |
| 王五 | 2025-02 | 28000 | 31000.00 |
| 王五 | 2025-03 | 35000 | 35000.00 |
| 王五 | 2025-04 | 42000 | 38500.00 |

> 💡 **解析**：
> - 2025-01：无前一行 → AVG(30000, 28000) = 29000
> - 2025-02：AVG(30000, 28000, 35000) = 31000
> - 2025-03：AVG(28000, 35000, 42000) = 35000
> - 2025-04：无后一行 → AVG(35000, 42000) = 38500

**其他常用模式**：

```sql
-- 从分区开头到当前行（默认，累计计算）
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW

-- 整个分区（不管 ORDER BY）
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING

-- 前 2 行到当前行（3 行窗口）
ROWS BETWEEN 2 PRECEDING AND CURRENT ROW

-- 当前行到后 2 行
ROWS BETWEEN CURRENT ROW AND 2 FOLLOWING
```

### 4. `ROWS` vs `RANGE`

> ⚠️ **注意**：`ROWS` 按**物理行数**计算，`RANGE` 按**值的范围**计算。当存在相同排序值时，两者结果可能不同。

```sql
-- ROWS：严格按行数，相同值的行也分开处理
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW

-- RANGE：相同排序值的行被视为同一组，一起计算
RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
```

> 💡 **建议**：大多数情况下使用 `ROWS` 更直观、可预测。`RANGE` 在处理相同排序值时有特殊行为，使用时要注意。

---

## 五、⭐ 排名函数

排名函数是开窗函数中使用频率最高的一类，面对"Top N"、"排名第几"等需求时必不可少。

### 1. `ROW_NUMBER()`

为分区内的每一行分配一个**唯一的连续序号**（1, 2, 3, ...），即使值相同，序号也不同。

```sql
SELECT
    name, department, salary,
    ROW_NUMBER() OVER(PARTITION BY department ORDER BY salary DESC) AS rn
FROM employee;
```

| name | department | salary | rn |
|------|-----------|--------|-----|
| 周八 | 技术部 | 22000 | 1 |
| 李四 | 技术部 | 18000 | 2 |
| 张三 | 技术部 | 15000 | 3 |
| 吴九 | 销售部 | 15000 | 1 |
| 王五 | 销售部 | 12000 | 2 |
| 赵六 | 销售部 | 9000  | 3 |
| 郑十 | 人事部 | 13000 | 1 |
| 孙七 | 人事部 | 11000 | 2 |

**⭐ 经典用法：取每组的 Top N**

```sql
-- 取每个部门工资最高的员工
SELECT * FROM (
    SELECT
        name, department, salary,
        ROW_NUMBER() OVER(PARTITION BY department ORDER BY salary DESC) AS rn
    FROM employee
) t
WHERE rn = 1;
```

| name | department | salary | rn |
|------|-----------|--------|-----|
| 周八 | 技术部 | 22000 | 1 |
| 吴九 | 销售部 | 15000 | 1 |
| 郑十 | 人事部 | 13000 | 1 |

> ⚠️ **注意**：`ROW_NUMBER()` 在值相同时的排序是**不确定的**（相同值的行谁排前面取决于数据库实现）。如果需要确定性排序，应在 `ORDER BY` 中加入额外的列（如主键）。

### 2. `RANK()`

相同值获得**相同排名**，但后续排名会**跳跃**（如 1, 2, 2, **4**）。

```sql
SELECT
    name, salary,
    RANK() OVER(ORDER BY salary DESC) AS rnk
FROM employee;
```

| name | salary | rnk |
|------|--------|-----|
| 周八 | 22000 | 1 |
| 李四 | 18000 | 2 |
| 张三 | 15000 | 3 |
| 吴九 | 15000 | 3 |
| 郑十 | 13000 | **5** |
| 王五 | 12000 | 6 |
| 孙七 | 11000 | 7 |
| 赵六 | 9000  | 8 |

> 注意：张三和吴九并列第 3，下一名直接跳到第 **5**（跳过了 4）。

### 3. `DENSE_RANK()`

相同值获得**相同排名**，后续排名**不跳跃**（如 1, 2, 2, **3**）。

```sql
SELECT
    name, salary,
    DENSE_RANK() OVER(ORDER BY salary DESC) AS drnk
FROM employee;
```

| name | salary | drnk |
|------|--------|------|
| 周八 | 22000 | 1 |
| 李四 | 18000 | 2 |
| 张三 | 15000 | 3 |
| 吴九 | 15000 | 3 |
| 郑十 | 13000 | **4** |
| 王五 | 12000 | 5 |
| 孙七 | 11000 | 6 |
| 赵六 | 9000  | 7 |

### 4. ⭐ `ROW_NUMBER` vs `RANK` vs `DENSE_RANK` 对比

将三个函数放在一起对比，差异一目了然：

```sql
SELECT
    name, salary,
    ROW_NUMBER() OVER(ORDER BY salary DESC) AS row_num,
    RANK()       OVER(ORDER BY salary DESC) AS rnk,
    DENSE_RANK() OVER(ORDER BY salary DESC) AS dense_rnk
FROM employee;
```

| name | salary | row_num | rnk | dense_rnk |
|------|--------|---------|-----|-----------|
| 周八 | 22000 | 1 | 1 | 1 |
| 李四 | 18000 | 2 | 2 | 2 |
| 张三 | 15000 | 3 | 3 | 3 |
| 吴九 | 15000 | 4 | 3 | 3 |
| 郑十 | 13000 | 5 | **5** | **4** |
| 王五 | 12000 | 6 | 6 | 5 |
| 孙七 | 11000 | 7 | 7 | 6 |
| 赵六 | 9000  | 8 | 8 | 7 |

| 函数 | 并列处理 | 排名跳跃 | 典型场景 |
|------|---------|---------|---------|
| `ROW_NUMBER` | ❌ 不并列（每行唯一） | 不涉及 | 分页、Top N、去重 |
| `RANK` | ✅ 并列 | ✅ 会跳跃 | 竞赛排名（第3名两人，下一个是第5名） |
| `DENSE_RANK` | ✅ 并列 | ❌ 不跳跃 | 连续排名（第3名两人，下一个是第4名） |

> 💡 **选择建议**：
> - 需要**唯一编号** → `ROW_NUMBER`（分页、去重取一）
> - 需要**真实名次**（如比赛排名）→ `RANK`
> - 需要**连续等级**（如工资等级）→ `DENSE_RANK`

### 5. `NTILE(n)`

将分区内的行**均分为 n 组**，返回组号（1 到 n）。常用于"前 25%"、"四分位"等场景。

```sql
-- 将员工按工资分成 4 组（四分位）
SELECT
    name, salary,
    NTILE(4) OVER(ORDER BY salary DESC) AS quartile
FROM employee;
```

| name | salary | quartile |
|------|--------|----------|
| 周八 | 22000 | 1 |
| 李四 | 18000 | 1 |
| 张三 | 15000 | 2 |
| 吴九 | 15000 | 2 |
| 郑十 | 13000 | 3 |
| 王五 | 12000 | 3 |
| 孙七 | 11000 | 4 |
| 赵六 | 9000  | 4 |

> 💡 8 行分 4 组，每组 2 行。如果不能整除，前面的组会多分 1 行（如 7 行分 3 组 → 3, 2, 2）。

---

## 六、⭐ 偏移函数与取值函数

### 1. `LAG(expr, offset, default)`

获取当前行**前面第 N 行**的值。

- `expr`：要获取的列
- `offset`：偏移量，默认 1（前一行）
- `default`：没有前一行时的默认值，默认 NULL

```sql
-- 计算每个销售人员的月度环比增长
SELECT
    salesperson, month, amount,
    LAG(amount) OVER(PARTITION BY salesperson ORDER BY month) AS prev_amount,
    amount - LAG(amount) OVER(PARTITION BY salesperson ORDER BY month) AS growth
FROM monthly_sales;
```

| salesperson | month | amount | prev_amount | growth |
|-------------|-------|--------|-------------|--------|
| 王五 | 2025-01 | 30000 | NULL | NULL |
| 王五 | 2025-02 | 28000 | 30000 | -2000 |
| 王五 | 2025-03 | 35000 | 28000 | 7000 |
| 王五 | 2025-04 | 42000 | 35000 | 7000 |
| 赵六 | 2025-01 | 25000 | NULL | NULL |
| 赵六 | 2025-02 | 32000 | 25000 | 7000 |
| 赵六 | 2025-03 | 28000 | 32000 | -4000 |
| 赵六 | 2025-04 | 30000 | 28000 | 2000 |
| 吴九 | 2025-01 | 35000 | NULL | NULL |
| 吴九 | 2025-02 | 40000 | 35000 | 5000 |
| 吴九 | 2025-03 | 38000 | 40000 | -2000 |
| 吴九 | 2025-04 | 45000 | 38000 | 7000 |

```sql
-- 指定偏移量和默认值
SELECT salesperson, month, amount,
    LAG(amount, 2, 0) OVER(PARTITION BY salesperson ORDER BY month) AS two_months_ago
FROM monthly_sales;
-- 王五 2025-01: 0（无前2行，用默认值0）
-- 王五 2025-02: 0
-- 王五 2025-03: 30000（前2行是2025-01）
```

### 2. `LEAD(expr, offset, default)`

获取当前行**后面第 N 行**的值，参数与 `LAG` 相同，方向相反。

```sql
-- 查看每个销售人员下个月的业绩
SELECT
    salesperson, month, amount,
    LEAD(amount) OVER(PARTITION BY salesperson ORDER BY month) AS next_amount
FROM monthly_sales;
```

| salesperson | month | amount | next_amount |
|-------------|-------|--------|-------------|
| 王五 | 2025-01 | 30000 | 28000 |
| 王五 | 2025-02 | 28000 | 35000 |
| 王五 | 2025-03 | 35000 | 42000 |
| 王五 | 2025-04 | 42000 | NULL |

**⭐ 实用场景：计算环比增长率**

```sql
SELECT
    salesperson, month, amount,
    LAG(amount) OVER(PARTITION BY salesperson ORDER BY month) AS prev,
    CASE
        WHEN LAG(amount) OVER(PARTITION BY salesperson ORDER BY month) IS NULL THEN NULL
        ELSE ROUND(
            (amount - LAG(amount) OVER(PARTITION BY salesperson ORDER BY month))
            / LAG(amount) OVER(PARTITION BY salesperson ORDER BY month) * 100, 2
        )
    END AS growth_rate_pct
FROM monthly_sales;
```

### 3. `FIRST_VALUE()` / `LAST_VALUE()`

获取窗口中**第一行** / **最后一行**的值。

```sql
-- 每个部门工资最高的人的工资（第一行 = 最大值）
SELECT
    name, department, salary,
    FIRST_VALUE(salary) OVER(
        PARTITION BY department ORDER BY salary DESC
    ) AS dept_max_salary
FROM employee;
```

> ⚠️ **LAST_VALUE 的陷阱**：`LAST_VALUE` 搭配 `ORDER BY` 使用时，默认窗口范围是"到当前行为止"，所以 `LAST_VALUE` 实际返回的是**当前行的值**，而不是分区最后一行！

```sql
-- ❌ 结果不符合预期：每行的 last_val 都是自己
SELECT
    name, department, salary,
    LAST_VALUE(salary) OVER(
        PARTITION BY department ORDER BY salary DESC
    ) AS last_val
FROM employee;
-- 技术部: 周八→22000, 李四→18000, 张三→15000（每行返回自己的值）

-- ✅ 正确写法：显式指定窗口范围为整个分区
SELECT
    name, department, salary,
    LAST_VALUE(salary) OVER(
        PARTITION BY department ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS dept_min_salary
FROM employee;
-- 技术部所有行都返回 15000（分区内工资最低值）
```

> 💡 **建议**：使用 `LAST_VALUE` 时**必须显式指定窗口范围**，否则很容易得到错误结果。或者直接用 `FIRST_VALUE` 配合反向排序来替代。

---

## 七、⚠️ 开窗函数不能在 WHERE 中使用

这是一个**非常重要的限制**：开窗函数的结果**不能直接**在 `WHERE`、`GROUP BY`、`HAVING` 中使用。

**原因**：SQL 的执行顺序决定了这一点：

```
FROM → WHERE → GROUP BY → HAVING → SELECT（开窗函数在此计算） → ORDER BY → LIMIT
```

`WHERE` 在 `SELECT` **之前**执行，而开窗函数是在 `SELECT` 阶段才计算的，所以 `WHERE` 根本"看不到"开窗函数产生的列。

```sql
-- ❌ 错误：WHERE 中不能直接引用开窗函数
SELECT
    name, salary,
    RANK() OVER(ORDER BY salary DESC) AS rnk
FROM employee
WHERE rnk <= 3;
-- 报错！rnk 在 WHERE 执行时还不存在

-- ✅ 正确写法1：包装成子查询，在外层 WHERE 过滤
SELECT * FROM (
    SELECT
        name, salary,
        RANK() OVER(ORDER BY salary DESC) AS rnk
    FROM employee
) t
WHERE rnk <= 3;

-- ✅ 正确写法2：使用 CTE（可读性更好）
WITH ranked AS (
    SELECT
        name, salary,
        RANK() OVER(ORDER BY salary DESC) AS rnk
    FROM employee
)
SELECT * FROM ranked WHERE rnk <= 3;
```

| name | salary | rnk |
|------|--------|-----|
| 周八 | 22000 | 1 |
| 李四 | 18000 | 2 |
| 张三 | 15000 | 3 |
| 吴九 | 15000 | 3 |

> 💡 **实用建议**：需要对开窗函数的结果做过滤时，**套一层子查询或 CTE** 是标准做法。推荐用 CTE（`WITH` 语法），可读性更好。这也是前一篇文章中 CTE 的典型应用场景之一。

> ⭐ **记忆口诀**：开窗函数只能出现在 `SELECT` 和 `ORDER BY` 中，不能出现在 `WHERE`、`GROUP BY`、`HAVING` 中。要过滤？先套一层！

---

## 八、综合速查表

| 需求 | 函数 / 写法 | 要点 |
|------|------------|------|
| 分区内汇总（保留明细） | `SUM/AVG/COUNT() OVER(PARTITION BY ...)` | 不加 ORDER BY = 整个分区 |
| 累计求和 | `SUM() OVER(PARTITION BY ... ORDER BY ...)` | 加 ORDER BY 默认累计 |
| 移动平均 / 滑动窗口 | `AVG() OVER(... ROWS BETWEEN N PRECEDING AND N FOLLOWING)` | 手动指定 Frame |
| 唯一行号 | `ROW_NUMBER() OVER(ORDER BY ...)` | 不并列，常用于 Top N |
| 排名（跳跃） | `RANK() OVER(ORDER BY ...)` | 并列后跳号 |
| 排名（不跳跃） | `DENSE_RANK() OVER(ORDER BY ...)` | 并列后连续 |
| 均分为 N 组 | `NTILE(N) OVER(ORDER BY ...)` | 四分位、百分位 |
| 前一行的值 | `LAG(col, 1) OVER(ORDER BY ...)` | 环比计算 |
| 后一行的值 | `LEAD(col, 1) OVER(ORDER BY ...)` | 预测 / 对比 |
| 窗口内第一个值 | `FIRST_VALUE(col) OVER(...)` | 分区最大/最小 |
| 窗口内最后一个值 | `LAST_VALUE(col) OVER(... ROWS BETWEEN ... AND UNBOUNDED FOLLOWING)` | ⚠️ 必须显式指定 Frame |

---

## 九、LeetCode 相关练习

<!-- 在此添加相关 LeetCode 题目链接 -->

> 🔗 后续待补充..

---

## 参考资料

> 📚 本文内容参考以下资料整理：
> - [编程导航 - 第 9 章：开窗函数](https://www.codefather.cn/course/1952312567451275265/section/1952312568130752514)

