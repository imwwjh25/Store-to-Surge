### 一、先理解背景：倒排索引的短板

ES 默认的倒排索引（Inverted Index）是为**全文检索**设计的：它记录 “词项 → 包含该词项的文档 ID 列表”，能快速回答 “哪些文档包含某个关键词”，但无法高效回答：

- “按某个字段对文档排序”
- “按某个字段统计聚合（比如按分类统计数量）”
- “在脚本中快速获取文档的字段值”

如果直接用倒排索引做这些操作，需要遍历大量数据，性能极低。而 DocValues 就是为解决这个问题而生的。

### 二、DocValues 的核心作用

#### 1. 核心定义

DocValues 是在**文档索引时**就预先构建的、以**文档 ID 为键，字段值为值**的列式存储结构（可以理解为：把每个字段的所有值单独存成一列，按文档 ID 排序）。

它的核心目标是：**为排序、聚合、脚本等操作提供高效的随机访问能力**。

#### 2. 具体作用 & 特性

表格







|        作用场景         |                   DocValues 如何优化                    |
| :---------------------: | :-----------------------------------------------------: |
|    字段排序（sort）     | 直接从 DocValues 中读取字段值进行排序，无需遍历倒排索引 |
| 聚合操作（aggregation） |    快速遍历字段的所有值，统计计数 / 求和 / 平均值等     |
|   脚本计算（script）    |          快速获取文档的字段值，用于自定义计算           |
| 字段值获取（fielddata） |        替代内存密集型的 fielddata（下文会对比）         |

#### 3. 关键特性

- **默认启用**：对于除 `text` 外的大多数字段（如 `keyword`、`numeric`、`date` 等），DocValues 是**默认开启**的（因为这些字段常用来排序 / 聚合）。
- **磁盘存储**：DocValues 存储在磁盘上（而非纯内存），但会利用操作系统的页缓存，兼顾内存效率和访问速度。
- **不可变**：文档索引后，DocValues 就固定了，更新文档会生成新的 DocValues 片段，保证查询性能。
- **text 类型默认禁用**：因为 `text` 字段分词后的值量大，启用 DocValues 会占用大量磁盘空间，且 `text` 字段通常不用于排序 / 聚合（如需则用 `.keyword` 子字段）。

### 三、代码示例：直观感受 DocValues 的使用

#### 1. 验证 DocValues 的启用状态

创建索引时，可显式指定字段是否启用 DocValues（以 `keyword` 为例，默认启用）：

json











```
# 创建索引，显式配置 DocValues
PUT /product_index
{
  "mappings": {
    "properties": {
      "product_name": { "type": "text" },  // text 类型默认禁用 DocValues
      "category": { 
        "type": "keyword",
        "doc_values": true  // 显式启用（默认值，可省略）
      },
      "price": { 
        "type": "double",
        "doc_values": true  // 数值类型默认启用
      }
    }
  }
}

# 插入测试数据
POST /product_index/_doc/1
{
  "product_name": "小米手机 14 Pro",
  "category": "手机",
  "price": 4999
}

POST /product_index/_doc/2
{
  "product_name": "华为 Mate 60 Pro",
  "category": "手机",
  "price": 5999
}
```

#### 2. 依赖 DocValues 的操作（排序 / 聚合）

json











```
# 1. 按 price 排序（依赖 DocValues）
GET /product_index/_search
{
  "sort": [ { "price": "asc" } ]  // 快速排序，底层用 DocValues
}

# 2. 按 category 聚合（依赖 DocValues）
GET /product_index/_search
{
  "size": 0,
  "aggs": {
    "category_count": {
      "terms": { "field": "category" }  // 快速聚合，底层用 DocValues
    }
  }
}
```

#### 3. 若禁用 DocValues，排序 / 聚合会报错

修改索引配置，禁用 `category` 的 DocValues：

json











```
PUT /product_index/_mapping
{
  "properties": {
    "category": { 
      "type": "keyword",
      "doc_values": false  // 禁用 DocValues
    }
  }
}

# 再次执行聚合，会报错：Fielddata is disabled on text fields by default...
GET /product_index/_search
{
  "aggs": {
    "category_count": {
      "terms": { "field": "category" }
    }
  }
}
```

### 四、DocValues vs Fielddata（补充对比）

很多新手会混淆这两个概念，简单说：

- **DocValues**：磁盘优先，索引时构建，默认启用（除 text），内存占用低，性能稳定。
- **Fielddata**：纯内存，查询时动态构建（从倒排索引加载），默认禁用（text 字段），内存占用高，易引发 OOM。

ES 推荐优先使用 DocValues，仅在万不得已时（如必须对 text 字段排序）才临时启用 Fielddata。

### 总结

1. **核心作用**：DocValues 是 ES 为排序、聚合、脚本等操作设计的列式存储结构，弥补了倒排索引在这些场景下的性能短板。
2. **使用特性**：默认启用（除 text 类型），存储在磁盘 + 页缓存，不可变，兼顾性能和内存效率。
3. **核心原则**：需要排序 / 聚合的字段（如 keyword、数值、日期）依赖 DocValues，text 字段优先用 `.keyword` 子字段而非启用 Fielddata。
