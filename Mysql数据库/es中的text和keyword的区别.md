### 一、核心区别（由浅入深）

#### 1. 最直观的差异：是否分词

- **text 类型**：会对字符串进行**分词处理**（比如把 "Elasticsearch 入门教程" 拆成 ["elasticsearch", "入门", "教程"]），拆分后再建立倒排索引，目的是支持**全文检索**。
- **keyword 类型**：不会分词，会把整个字符串作为一个完整的 “词项（term）” 存入索引，目的是支持**精确匹配、聚合、排序**。

#### 2. 具体特性对比






|        特性         |             text 类型             |         keyword 类型         |
| :-----------------: | :-------------------------------: | :--------------------------: |
|        分词         |  是（默认使用 standard 分词器）   |   否（整串作为单个 term）    |
|      全文检索       |    支持（匹配分词后的任意词）     | 不支持（只能匹配完整字符串） |
|      精确匹配       |   不支持（分词后无法匹配整串）    |             支持             |
| 聚合（aggregation） |    不支持（分词后聚合无意义）     |             支持             |
|    排序（sort）     |  不支持（分词后排序结果无意义）   |             支持             |
|    占用存储空间     | 相对较大（存储分词后的多个 term） | 相对较小（仅存储完整字符串） |

#### 3. 代码示例：直观理解使用场景

假设我们创建一个索引，包含 `title`（text 类型，用于全文检索）和 `category`（keyword 类型，用于分类聚合）：






```
# 1. 创建索引（指定字段类型）
PUT /article_index
{
  "mappings": {
    "properties": {
      "title": { "type": "text" },       // 文章标题：全文检索
      "category": { "type": "keyword" }  // 文章分类：精确匹配/聚合
    }
  }
}

# 2. 插入测试数据
POST /article_index/_doc/1
{
  "title": "Elasticsearch text 和 keyword 区别",
  "category": "ES 基础"
}

POST /article_index/_doc/2
{
  "title": "ES keyword 类型 聚合排序 示例",
  "category": "ES 基础"
}

# 3. 测试 text 类型的全文检索（能匹配分词后的任意词）
GET /article_index/_search
{
  "query": {
    "match": { "title": "keyword" }  // 能匹配到两条数据，因为标题都包含 "keyword"
  }
}

# 4. 测试 keyword 类型的精确匹配（必须完全一致）
GET /article_index/_search
{
  "query": {
    "term": { "category": "ES 基础" }  // 能匹配到两条数据
  }
}

# 5. 测试 keyword 类型的聚合（text 类型聚合会报错）
GET /article_index/_search
{
  "size": 0,  // 不返回原始数据，只返回聚合结果
  "aggs": {
    "category_count": {
      "terms": { "field": "category.keyword" }  // 按分类统计数量
    }
  }
}
```

#### 4. 常见使用场景

- **text 类型**：文章内容、商品描述、用户评论、日志详情等需要 “模糊检索 / 全文检索” 的场景。
- **keyword 类型**：商品分类、用户 ID、订单状态、地区编码、标签等需要 “精确匹配、排序、聚合” 的场景。

#### 5. 小技巧：多字段映射

如果一个字段既需要全文检索，又需要聚合 / 排序，可以同时定义 text 和 keyword 类型（ES 会自动为 text 字段生成一个 `.keyword` 的子字段）：









```
PUT /article_index
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",          // 用于全文检索
        "fields": {
          "keyword": {           // 用于聚合/排序
            "type": "keyword",
            "ignore_above": 256  // 超过256字符的字符串不索引（节省空间）
          }
        }
      }
    }
  }
}
```

### 总结

1. **核心差异**：`text` 分词（支持全文检索），`keyword` 不分词（支持精确匹配 / 聚合 / 排序）。
2. **使用场景**：`text` 用于全文检索的长文本，`keyword` 用于精确匹配 / 聚合的短字符串（如分类、ID）。
3. **灵活用法**：可通过多字段映射让一个字符串同时支持全文检索和聚合排序。
