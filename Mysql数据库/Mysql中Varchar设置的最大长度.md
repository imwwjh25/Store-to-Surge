### VARCHAR 括号内数值（长度）的核心规则

MySQL 中 `VARCHAR(n)` 里的 `n` 表示「**最大字符数**」（不是字节数），其最大可设置值和 MySQL 版本、存储引擎（InnoDB/MyISAM）、字符集（UTF8/UTF8MB4）密切相关，核心结论先明确：

表格







| MySQL 版本 |    最大可设置字符数（n）     | 底层存储上限（字节） |           关键限制           |
| :--------: | :--------------------------: | :------------------: | :--------------------------: |
|  ≤ 5.0.3   |             255              |       255 字节       |         无行格式限制         |
|  ≥ 5.0.4   | 65535（理论）/ 65532（实际） |      65535 字节      | 受「行最大字节数 65535」限制 |

### 一、关键概念先理清

1. 字符数 vs 字节数

   ：

   - `n` 是「字符数」：比如 `VARCHAR(10)` 可以存 10 个汉字（UTF8 下每个汉字占 3 字节），或 10 个字母（1 字节 / 个）；
   - 底层存储按「字节」算：最终占用空间 = 实际字符的字节数 + 1~2 字节（长度标识）。

   

2. 行最大字节数限制

   ：

   

   MySQL 规定「单行所有字段的总字节数」不能超过 65535 字节（这是 VARCHAR 实际最大长度的核心约束）。

### 二、不同场景下的最大可设置值

#### 场景 1：MySQL ≥ 5.0.4 + 单字段（无其他字段）

- 纯字母 / 数字（latin1 字符集，1 字节 / 字符）

  ：

  

  可设置 

  ```
  VARCHAR(65532)
  ```

  （65535 字节 - 2 字节长度标识 - 1 字节 NULL 标记 = 65532）；

  

  若字段加 

  ```
  NOT NULL
  ```

  ，可设 

  ```
  VARCHAR(65533)
  ```

  （少 1 字节 NULL 标记）。

- UTF8 字符集（3 字节 / 字符）

  ：

  

  最大可设置 

  ```
  VARCHAR(21844)
  ```

  （65535 ÷ 3 ≈ 21845，扣掉长度标识后≈21844）；

- UTF8MB4 字符集（4 字节 / 字符）

  ：

  

  最大可设置 

  ```
  VARCHAR(16383)
  ```

  （65535 ÷ 4 ≈ 16383）。

#### 场景 2：表中有多个字段

单行总字节数不能超 65535，因此 `VARCHAR` 的最大可设置值会被其他字段「挤占」。

**举例**：









```
-- 错误案例：两个VARCHAR字段总字节数超65535
CREATE TABLE test_varchar (
  col1 VARCHAR(32768),  -- UTF8下，32768×3=98304字节（已超65535）
  col2 VARCHAR(10)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

执行会报错：







```
ERROR 1118 (42000): Row size too large. The maximum row size for the used table type, not counting BLOBs, is 65535. This includes storage overhead, check the manual. You have to change some columns to TEXT or BLOBs
```

#### 场景 3：MySQL ≤ 5.0.3

无论字符集是什么，`VARCHAR(n)` 的 `n` 最大只能设 255（字符数），超过会报错。

### 三、实操验证例子

#### 例子 1：UTF8 下设置最大可取值










```
-- UTF8 + NOT NULL，最大可设 VARCHAR(21844)
CREATE TABLE test_varchar_max (
  content VARCHAR(21844) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

-- 插入测试（存21844个汉字）
INSERT INTO test_varchar_max VALUES (REPEAT('测', 21844));

-- 查询验证
SELECT LENGTH(content) FROM test_varchar_max; -- 输出 65532（21844×3）
```

#### 例子 2：设置超过上限会报错








```
-- 尝试设置 VARCHAR(21845)（UTF8）
CREATE TABLE test_varchar_error (
  content VARCHAR(21845) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

报错：








```
ERROR 1118 (42000): Row size too large. The maximum row size for the used table type, not counting BLOBs, is 65535...
```

### 四、常见问题 & 避坑点

1. 为什么不建议设成最大 65535？

   - 单行字节数接近 65535 时，性能会下降（InnoDB 行存储效率降低）；
   - 若实际业务不需要这么长的字符串，设过大的值会让 MySQL 优化器误判，影响查询效率。

   

2. 超过设置的 n 会怎样？

   

   插入字符数超过 

   ```
   n
   ```

    时，MySQL 会直接报错（严格模式）或截断并警告（非严格模式）：



   

   

   ```
   -- 严格模式下插入超过长度的值
   INSERT INTO test_varchar_max VALUES (REPEAT('测', 21845));
   -- 报错：Data truncation: Data too long for column 'content' at row 1
   ```

   

3. VARCHAR 和 TEXT 的选择？

   

   若字符串长度经常超过 2000 字符，建议用 

   ```
   TEXT
   ```

   （不受行字节数 65535 限制），而非超大的 VARCHAR。

### 总结

1. **核心规则**：`VARCHAR(n)` 的 `n` 是最大字符数，MySQL ≥5.0.4 理论最大可设 65535（字符），但受「行总字节数 65535」和字符集限制；

2. 实际取值 ：

   - UTF8 下最大可设～21844 字符，UTF8MB4 下～16383 字符，latin1 下～65532 字符；
   - 表中有多个字段时，`n` 需按「单行总字节数 ≤65535」调整；

   

3. **最佳实践**：按实际业务需求设置 `n`（比如姓名设 `VARCHAR(20)`、地址设 `VARCHAR(100)`），不要盲目设最大值。
