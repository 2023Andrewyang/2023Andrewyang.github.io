---
title: SQL基础-时间函数
date: 2026-02-19 23:12:01
tags:
  - SQL
  - 数据库
  - 学习笔记
categories:
    - 技术文章
---

最近在复习 SQL，发现时间相关的函数种类很多，在日常开发中也非常常用，所以整理了这份速查手册，方便日后快速查阅和复习。

---

<!-- more -->

## 一、获取当前时间

### 1. `NOW()`

返回当前的日期和时间（`YYYY-MM-DD HH:MM:SS`）。

```sql
SELECT NOW();
-- 输出: 2026-02-19 23:12:01
```

> ⚠️ **注意**：`NOW()` 在同一条 SQL 语句中多次调用，返回值**相同**（取语句开始执行的时间）。如果需要实时变化的时间，请使用 `SYSDATE()`。

### 2. `SYSDATE()`

返回函数**执行时刻**的日期和时间。

```sql
SELECT SYSDATE(), SLEEP(2), SYSDATE();
-- 第一个和第二个 SYSDATE() 结果会相差 2 秒
```

> ⚠️ **与 `NOW()` 的区别**：`NOW()` 取的是语句开始时间，`SYSDATE()` 取的是函数实际执行时间。在主从复制场景中，`SYSDATE()` 可能导致数据不一致，生产环境建议优先使用 `NOW()`。

### 3. `CURDATE()` / `CURRENT_DATE`

仅返回当前**日期**（`YYYY-MM-DD`）。

```sql
SELECT CURDATE();
-- 输出: 2026-02-19

SELECT CURRENT_DATE;
-- 输出: 2026-02-19（注意：CURRENT_DATE 不加括号）
```

### 4. `CURTIME()` / `CURRENT_TIME`

仅返回当前**时间**（`HH:MM:SS`）。

```sql
SELECT CURTIME();
-- 输出: 23:12:01

SELECT CURRENT_TIME;
-- 输出: 23:12:01
```

---

## 二、提取时间部分

### 1. `YEAR()` / `MONTH()` / `DAY()`

分别提取日期中的**年、月、日**。

```sql
SELECT YEAR('2026-02-19');   -- 输出: 2026
SELECT MONTH('2026-02-19');  -- 输出: 2
SELECT DAY('2026-02-19');    -- 输出: 19
```

> 💡 `DAY()` 等价于 `DAYOFMONTH()`。

### 2. `HOUR()` / `MINUTE()` / `SECOND()`

提取时间中的**时、分、秒**。

```sql
SELECT HOUR('23:12:01');    -- 输出: 23
SELECT MINUTE('23:12:01');  -- 输出: 12
SELECT SECOND('23:12:01');  -- 输出: 1
```

### 3. `EXTRACT(unit FROM date)`

通用的时间部分提取函数，`unit` 支持多种单位。

**常用 unit 值**：`YEAR`、`MONTH`、`DAY`、`HOUR`、`MINUTE`、`SECOND`、`QUARTER`、`WEEK`

```sql
SELECT EXTRACT(YEAR FROM '2026-02-19 23:12:01');    -- 输出: 2026
SELECT EXTRACT(MONTH FROM '2026-02-19 23:12:01');   -- 输出: 2
SELECT EXTRACT(HOUR FROM '2026-02-19 23:12:01');    -- 输出: 23
SELECT EXTRACT(QUARTER FROM '2026-02-19');           -- 输出: 1（第一季度）
```

> 💡 **补充**：`EXTRACT` 是 SQL 标准语法，跨数据库兼容性更好，值得优先掌握。

### 4. `DAYOFWEEK()` / `WEEKDAY()` / `DAYNAME()`

获取日期对应的**星期**信息。

```sql
-- DAYOFWEEK(): 1=周日, 2=周一, ..., 7=周六
SELECT DAYOFWEEK('2026-02-19');  -- 输出: 5（周四）

-- WEEKDAY(): 0=周一, 1=周二, ..., 6=周日
SELECT WEEKDAY('2026-02-19');    -- 输出: 3（周四）

-- DAYNAME(): 返回英文星期名
SELECT DAYNAME('2026-02-19');    -- 输出: Thursday
```

> ⚠️ **易错点**：`DAYOFWEEK()` 和 `WEEKDAY()` 的编号方式不同！`DAYOFWEEK()` 以周日为 1 开始，`WEEKDAY()` 以周一为 0 开始。使用时一定注意区分。

### 5. `DAYOFYEAR()`

返回日期是当年的第几天（1-366）。

```sql
SELECT DAYOFYEAR('2026-02-19');  -- 输出: 50
```

### 6. `QUARTER()`

返回日期所属的**季度**（1-4）。

```sql
SELECT QUARTER('2026-02-19');  -- 输出: 1
SELECT QUARTER('2026-08-15');  -- 输出: 3
```

### 7. `WEEK()` / `WEEKOFYEAR()`

返回日期是当年的第几周。

```sql
SELECT WEEK('2026-02-19');        -- 输出: 7
SELECT WEEKOFYEAR('2026-02-19'); -- 输出: 8
```

> ⚠️ **注意**：`WEEK()` 的结果受 `default_week_format` 系统变量影响，可通过第二个参数指定模式。`WEEKOFYEAR()` 等价于 `WEEK(date, 3)`，遵循 ISO 标准。建议统一使用 `WEEKOFYEAR()` 或明确指定 `WEEK(date, mode)`，避免歧义。

---

## 三、时间运算（加减法）

### 1. `DATE_ADD(date, INTERVAL expr unit)` / `DATE_SUB(date, INTERVAL expr unit)`

对日期进行**加法/减法**运算，最常用的时间运算函数。

**常用 unit 值**：`SECOND`、`MINUTE`、`HOUR`、`DAY`、`WEEK`、`MONTH`、`QUARTER`、`YEAR`

```sql
-- 加 3 天
SELECT DATE_ADD('2026-02-19', INTERVAL 3 DAY);
-- 输出: 2026-02-22

-- 减 1 个月
SELECT DATE_SUB('2026-02-19', INTERVAL 1 MONTH);
-- 输出: 2026-01-19

-- 加 2 小时 30 分钟
SELECT DATE_ADD('2026-02-19 23:12:01', INTERVAL '2:30' HOUR_MINUTE);
-- 输出: 2026-02-20 01:42:01

-- 也可以用负数实现减法
SELECT DATE_ADD('2026-02-19', INTERVAL -7 DAY);
-- 输出: 2026-02-12
```

> 💡 **复合单位**：`HOUR_MINUTE`、`DAY_HOUR`、`YEAR_MONTH` 等可以一次操作多个部分。

> ⚠️ **月末溢出**：`DATE_ADD('2026-01-31', INTERVAL 1 MONTH)` 结果是 `2026-02-28`，MySQL 会自动调整到当月最后一天，而不是报错。

### 2. `ADDDATE()` / `SUBDATE()`

`DATE_ADD` / `DATE_SUB` 的同义函数。

```sql
-- 与 DATE_ADD 完全等价
SELECT ADDDATE('2026-02-19', INTERVAL 3 DAY);
-- 输出: 2026-02-22

-- 第二个参数为整数时，默认单位是 DAY
SELECT ADDDATE('2026-02-19', 3);
-- 输出: 2026-02-22
```

### 3. `DATEDIFF(date1, date2)`

计算两个日期之间相差的**天数**（`date1 - date2`）。

```sql
SELECT DATEDIFF('2026-02-19', '2026-01-01');
-- 输出: 49

SELECT DATEDIFF('2026-01-01', '2026-02-19');
-- 输出: -49
```

> ⚠️ **注意**：`DATEDIFF` 只比较日期部分，忽略时间部分。参数顺序影响正负号。

### 4. `TIMESTAMPDIFF(unit, datetime1, datetime2)`

计算两个日期/时间之间的差值，可指定返回单位（`datetime2 - datetime1`）。

```sql
-- 相差多少天
SELECT TIMESTAMPDIFF(DAY, '2026-01-01', '2026-02-19');
-- 输出: 49

-- 相差多少小时
SELECT TIMESTAMPDIFF(HOUR, '2026-02-19 00:00:00', '2026-02-19 23:12:01');
-- 输出: 23

-- 相差多少月
SELECT TIMESTAMPDIFF(MONTH, '2025-08-19', '2026-02-19');
-- 输出: 6

-- 相差多少年（常用于计算年龄）
SELECT TIMESTAMPDIFF(YEAR, '1995-06-15', CURDATE());
```

> 💡 **实用技巧**：计算年龄时推荐使用 `TIMESTAMPDIFF(YEAR, birth_date, CURDATE())`，比手动计算更准确。

> ⚠️ **参数顺序**：注意是 `datetime2 - datetime1`，与 `DATEDIFF` 的 `date1 - date2` 相反！

---

## 四、时间格式化与解析

### 1. `DATE_FORMAT(date, format)`

将日期按照指定格式输出为**字符串**，开发中使用频率极高。

**常用格式符**：

| 格式符 | 说明 | 示例 |
|--------|------|------|
| `%Y` | 四位年份 | 2026 |
| `%y` | 两位年份 | 26 |
| `%m` | 月份（补零） | 02 |
| `%c` | 月份（不补零） | 2 |
| `%d` | 日（补零） | 19 |
| `%e` | 日（不补零） | 19 |
| `%H` | 24小时制（补零） | 23 |
| `%h` / `%I` | 12小时制（补零） | 11 |
| `%i` | 分钟（补零） | 12 |
| `%s` | 秒（补零） | 01 |
| `%W` | 星期名（英文） | Thursday |
| `%w` | 星期（0=周日） | 4 |
| `%M` | 月份名（英文） | February |
| `%j` | 一年中的第几天 | 050 |
| `%p` | AM / PM | PM |
| `%T` | 24小时时间 | 23:12:01 |
| `%r` | 12小时时间 | 11:12:01 PM |

```sql
SELECT DATE_FORMAT('2026-02-19 23:12:01', '%Y-%m-%d');
-- 输出: 2026-02-19

SELECT DATE_FORMAT('2026-02-19 23:12:01', '%Y年%m月%d日');
-- 输出: 2026年02月19日

SELECT DATE_FORMAT('2026-02-19 23:12:01', '%Y/%m/%d %H:%i:%s');
-- 输出: 2026/02/19 23:12:01

SELECT DATE_FORMAT('2026-02-19 23:12:01', '%W, %M %d, %Y');
-- 输出: Thursday, February 19, 2026
```

> ⚠️ **易错点**：分钟是 `%i` 而不是 `%m`（`%m` 是月份）！这是非常常见的笔误。

### 2. `STR_TO_DATE(str, format)`

将**字符串**按照指定格式解析为日期，是 `DATE_FORMAT` 的逆操作。

```sql
SELECT STR_TO_DATE('2026-02-19', '%Y-%m-%d');
-- 输出: 2026-02-19

SELECT STR_TO_DATE('19/02/2026', '%d/%m/%Y');
-- 输出: 2026-02-19

SELECT STR_TO_DATE('February 19, 2026', '%M %d, %Y');
-- 输出: 2026-02-19
```

> 💡 **使用场景**：当数据源的日期格式不是 MySQL 标准格式时，用 `STR_TO_DATE` 进行转换后再存入数据库。

### 3. `TIME_FORMAT(time, format)`

与 `DATE_FORMAT` 类似，但专门用于格式化**时间**部分，只支持时间相关的格式符（`%H`, `%i`, `%s` 等）。

```sql
SELECT TIME_FORMAT('23:12:01', '%H时%i分%s秒');
-- 输出: 23时12分01秒
```

---

## 五、时间戳转换

### 1. `UNIX_TIMESTAMP()`

将日期转换为 **Unix 时间戳**（自 1970-01-01 00:00:00 UTC 以来的秒数）。

```sql
-- 获取当前时间戳
SELECT UNIX_TIMESTAMP();
-- 输出: 1771437121（示例）

-- 将指定日期转为时间戳
SELECT UNIX_TIMESTAMP('2026-02-19 23:12:01');
```

### 2. `FROM_UNIXTIME(unix_timestamp [, format])`

将 **Unix 时间戳**转换回日期时间，可选格式化。

```sql
SELECT FROM_UNIXTIME(1771437121);
-- 输出: 2026-02-19 23:12:01

-- 带格式化
SELECT FROM_UNIXTIME(1771437121, '%Y-%m-%d');
-- 输出: 2026-02-19
```

> 💡 **开发常用**：很多系统使用时间戳存储时间，`FROM_UNIXTIME` 是查询时转换显示的必备函数。

---

## 六、日期构造与转换

### 1. `DATE(expr)`

从日期时间表达式中**提取日期部分**。

```sql
SELECT DATE('2026-02-19 23:12:01');
-- 输出: 2026-02-19
```

### 2. `TIME(expr)`

从日期时间表达式中**提取时间部分**。

```sql
SELECT TIME('2026-02-19 23:12:01');
-- 输出: 23:12:01
```

### 3. `MAKEDATE(year, dayofyear)`

根据年份和当年的第几天**构造日期**。

```sql
SELECT MAKEDATE(2026, 50);
-- 输出: 2026-02-19

SELECT MAKEDATE(2026, 1);
-- 输出: 2026-01-01
```

### 4. `MAKETIME(hour, minute, second)`

根据时、分、秒**构造时间**。

```sql
SELECT MAKETIME(23, 12, 1);
-- 输出: 23:12:01
```

---

## 七、其他常用函数

### 1. `LAST_DAY(date)`

返回指定日期所在月份的**最后一天**。

```sql
SELECT LAST_DAY('2026-02-19');
-- 输出: 2026-02-28

SELECT LAST_DAY('2024-02-19');
-- 输出: 2024-02-29（闰年）
```

> 💡 **实用技巧**：结合使用可以计算当月天数：`DAY(LAST_DAY(date))`。

### 2. `DATE()` 截断技巧

获取某天/某月/某年的起始时间：

```sql
-- 当月第一天
SELECT DATE_FORMAT('2026-02-19', '%Y-%m-01');
-- 输出: 2026-02-01

-- 也可以用 DATE_SUB 配合 DAY
SELECT DATE_SUB('2026-02-19', INTERVAL DAY('2026-02-19') - 1 DAY);
-- 输出: 2026-02-01

-- 当年第一天
SELECT MAKEDATE(YEAR('2026-02-19'), 1);
-- 输出: 2026-01-01
```

### 3. `CONVERT_TZ(dt, from_tz, to_tz)`

将日期时间从一个时区**转换**到另一个时区。

```sql
SELECT CONVERT_TZ('2026-02-19 23:12:01', '+08:00', '+00:00');
-- 输出: 2026-02-19 15:12:01（从北京时间转为 UTC）

SELECT CONVERT_TZ('2026-02-19 23:12:01', '+08:00', '-05:00');
-- 输出: 2026-02-19 10:12:01（从北京时间转为纽约时间）
```

---

## 八、速查对照表

| 需求 | 函数 | 示例 |
|------|------|------|
| 当前日期时间 | `NOW()` | `2026-02-19 23:12:01` |
| 当前日期 | `CURDATE()` | `2026-02-19` |
| 当前时间 | `CURTIME()` | `23:12:01` |
| 提取年份 | `YEAR(date)` | `2026` |
| 提取月份 | `MONTH(date)` | `2` |
| 提取日 | `DAY(date)` | `19` |
| 提取时分秒 | `HOUR/MINUTE/SECOND` | `23 / 12 / 1` |
| 星期几 | `DAYOFWEEK(date)` | `5`（周四） |
| 第几季度 | `QUARTER(date)` | `1` |
| 日期加减 | `DATE_ADD/DATE_SUB` | 见上文 |
| 日期差（天） | `DATEDIFF(d1, d2)` | `49` |
| 日期差（任意单位） | `TIMESTAMPDIFF(unit, d1, d2)` | 见上文 |
| 格式化输出 | `DATE_FORMAT(date, fmt)` | `2026年02月19日` |
| 字符串转日期 | `STR_TO_DATE(str, fmt)` | `2026-02-19` |
| 日期转时间戳 | `UNIX_TIMESTAMP(date)` | `1771437121` |
| 时间戳转日期 | `FROM_UNIXTIME(ts)` | `2026-02-19 23:12:01` |
| 月末最后一天 | `LAST_DAY(date)` | `2026-02-28` |
| 时区转换 | `CONVERT_TZ(dt, from, to)` | 见上文 |

---

## 九、LeetCode 相关练习

<!-- 在此添加相关 LeetCode 题目链接 -->

> 🔗 后续待补充..

