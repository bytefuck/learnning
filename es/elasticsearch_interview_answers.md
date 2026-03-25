# Elasticsearch 面试问题及详细答案

## 一、基础概念

### 1. ES 的倒排索引是什么？与传统 B+ 树索引有何区别？

**倒排索引（Inverted Index）**：
- 结构：单词 -> 文档列表的映射
- 包含两部分：
  - Term Dictionary（词项字典）：存储所有词项，按字典序排序
  - Postings List（倒排列表）：存储每个词项出现的文档ID列表

**与传统 B+ 树索引的区别**：

| 特性 | 倒排索引 | B+ 树索引 |
|------|---------|----------|
| 存储结构 | 词项 -> 文档列表 | 键 -> 数据指针 |
| 适用场景 | 全文搜索 | 精确匹配、范围查询 |
| 查询方式 | 先查词项，再查文档 | 树遍历查找 |
| 写入性能 | 较高（追加写入） | 需要维护树结构 |
| 压缩 | 支持压缩（FST） | 一般不支持 |

**ES 的倒排索引优化**：
- 使用 FST（Finite State Transducer）压缩 Term Dictionary
- 支持跳跃表（Skip List）加速查找
- Postings List 使用 RoaringBitmap 压缩

---

### 2. ES 中 document、index、type、mapping、field 的关系是什么？

```
Cluster（集群）
  └── Node（节点）
       └── Index（索引）
            ├── Mapping（映射）
            │    └── Field（字段）定义
            └── Document（文档）集合
                 └── Field（字段）值
```

**关系说明**：

- **Index（索引）**：类似 MySQL 的数据库，是文档的集合
- **Type（类型）**：ES 6.x 开始废弃，ES 7.x+ 默认为 _doc，一个 index 只有一个 type
- **Document（文档）**：索引中的基本数据单元，类似 MySQL 的行，以 JSON 格式存储
- **Mapping（映射）**：定义文档结构和字段类型，类似数据库的 schema
- **Field（字段）**：文档的属性，类似 MySQL 的列

**示例**：
```json
{
  "_index": "products",
  "_id": "1",
  "_source": {
    "name": "iPhone 15",      // Field
    "price": 5999,            // Field
    "category": "手机"        // Field
  }
}
```

---

### 3. ES 与 Solr 的主要区别？为什么选择 ES？

| 特性 | Elasticsearch | Solr |
|------|--------------|------|
| 诞生时间 | 2010 | 2004 |
| 底层 | 基于 Lucene | 基于 Lucene |
| 分布式 | 原生支持 | 后期支持（SolrCloud）|
| 实时性 | 近实时（1秒） | 较实时 |
| REST API | 原生支持 | 后期支持 |
| JSON | 原生支持 | XML 为主 |
| 生态工具 | Kibana、Logstash、Beats | 相对较少 |
| 社区活跃度 | 非常高 | 高 |
| 学习曲线 | 较平缓 | 较陡峭 |

**选择 ES 的原因**：
1. **开箱即用**：分布式特性原生支持，无需复杂配置
2. **实时搜索**：默认 1 秒刷新，满足大多数实时场景
3. **生态完善**：ELK Stack（Elasticsearch + Logstash + Kibana）提供完整解决方案
4. **水平扩展**：添加节点即可扩展，自动负载均衡
5. **RESTful API**：使用 HTTP + JSON，易于集成
6. **聚合分析**：内置强大的聚合功能，支持复杂数据分析
7. **社区支持**：活跃的开源社区，文档丰富

---

## 二、架构原理

### 4. ES 集群中 master 节点、data 节点、coordinating 节点的职责分别是什么？

**Master-eligible 节点（候选主节点）**：
- 职责：
  - 管理集群状态（cluster state）
  - 创建/删除索引
  - 跟踪节点加入/离开
  - 分配分片到数据节点
- 配置：`node.master: true`（默认）
- 最佳实践：建议配置 3-5 个 master-eligible 节点

**Data 节点（数据节点）**：
- 职责：
  - 存储数据分片
  - 执行 CRUD、搜索、聚合操作
  - 维护倒排索引
- 配置：`node.data: true`（默认）
- 特点：承载主要的计算和存储压力

**Coordinating 节点（协调节点）**：
- 职责：
  - 接收客户端请求
  - 路由到正确的数据节点
  - 合并多个节点的结果返回客户端
- 配置：`node.master: false` + `node.data: false`
- 优势：可减轻 master 和 data 节点的压力

**Ingest 节点（预处理节点）**：
- 职责：在索引前对文档进行预处理
- 配置：`node.ingest: true`（默认）

**节点组合示例**：
```yaml
# Master 节点配置
node.master: true
node.data: false
node.ingest: false

# Data 节点配置
node.master: false
node.data: true
node.ingest: false

# Coordinating 节点配置
node.master: false
node.data: false
node.ingest: false
```

---

### 5. ES 的分片和副本机制是怎样的？为什么分片数不宜过多？

**分片（Shard）**：
- **Primary Shard（主分片）**：索引数据的原始分片，数量在创建索引时确定，不可修改
- **Replica Shard（副本分片）**：主分片的副本，用于：
  - 提高可用性（failover）
  - 提高读性能（负载均衡）
  - 支持水平扩展

**分片分配策略**：
- 每个索引可以有多个主分片
- 副本分片不能和对应的主分片在同一节点
- 分片均匀分布在集群节点上

**为什么分片数不宜过多**：

1. **内存开销**：
   - 每个分片都需要维护 Lucene 索引结构
   - 1GB 堆内存大约支持 20-30 个分片
   - 过多分片会消耗大量内存

2. **性能问题**：
   - 查询需要在多个分片上执行，增加协调开销
   - 聚合操作需要合并多个分片的结果
   - 分片过多会导致 "千分片问题"

3. **资源竞争**：
   - 每个分片都是独立的 Lucene 索引
   - 后台 merge、refresh 操作会争抢资源

**最佳实践**：
- 单个分片大小控制在 10-50GB
- 每个节点总分片数不超过 1000 个
- 主分片数 = 数据节点数 × 1-3 倍
- 对于日志类数据，可以使用 rollover 和 ILM 管理分片

**计算公式**：
```
目标分片数 = 总数据量 / 目标分片大小（通常 20-30GB）
```

---

### 6. ES 写入流程是怎样的？（refresh、flush、merge 各自作用）

**完整写入流程**：

```
1. 客户端发送请求
2. Coordinating 节点路由到主分片
3. 主分片写入
4. 主分片同步到副本分片
5. 主分片返回成功响应
6. 后台执行 refresh、flush、merge
```

**详细步骤**：

**阶段一：写入内存**
```
Document → In-memory Buffer（内存缓冲区）
        → Translog（事务日志，保证持久性）
```

**阶段二：Refresh（内存 -> 文件系统缓存）**
- **触发时机**：默认每 1 秒，或 `refresh_interval` 设置
- **操作**：
  - 将 In-memory Buffer 中的文档写入 Segment（段）
  - Segment 存储在文件系统缓存（OS Cache）
  - 打开 Segment，使其可搜索
- **特点**：
  - 数据写入 OS Cache，未落盘
  - 此时数据可被搜索（近实时）
  - 开销较小

**阶段三：Flush（文件系统缓存 -> 磁盘）**
- **触发时机**：默认每 30 分钟，或 translog 超过 512MB
- **操作**：
  - 将 OS Cache 中的 Segment 刷写到磁盘
  - 清空 translog
  - 更新 commit point
- **特点**：
  - 数据持久化到磁盘
  - 开销较大

**阶段四：Merge（段合并）**
- **触发时机**：后台自动执行
- **操作**：
  - 将多个小的 Segment 合并成大的 Segment
  - 删除标记为删除的文档（物理删除）
  - 更新倒排索引
- **特点**：
  - 减少 Segment 数量
  - 提高搜索性能
  - 消耗 CPU 和 IO

**配置优化**：
```json
PUT /my_index/_settings
{
  "index": {
    "refresh_interval": "30s",      // 增大刷新间隔，适合批量导入
    "number_of_replicas": 0,        // 临时减少副本，提高写入速度
    "translog": {
      "durability": "async",        // 异步写入 translog
      "sync_interval": "120s"
    }
  }
}
```

---

### 7. ES 的近实时搜索是如何实现的？

**近实时（Near Real-Time）**：
- 文档写入后，默认 1 秒内可被搜索
- 而非实时（Real-Time）即写入立即可见

**实现原理**：

1. **内存缓冲（In-memory Buffer）**：
   - 新文档先写入内存缓冲区
   - 同时写入 translog（保证不丢失）

2. **Refresh 操作**：
   - 默认每 1 秒执行 refresh
   - 将内存缓冲区的数据刷新到文件系统缓存（OS Cache）
   - 生成新的 Segment
   - 新 Segment 立即可被搜索

3. **文件系统缓存（OS Cache）**：
   - Segment 先存在于 OS Cache
   - 搜索时直接从 OS Cache 读取
   - 速度接近内存，远快于磁盘

**数据可见性流程**：
```
写入请求
    ↓
[内存缓冲区] → [Translog]
    ↓ （1秒后 Refresh）
[OS Cache - 新 Segment]
    ↓ （可被搜索）
[磁盘] （30分钟后 Flush）
```

**为什么不是实时？**
- 如果每次写入都立即刷新，会产生大量小 Segment
- 大量 Segment 会严重影响搜索性能
- 需要 merge 操作合并 Segment，带来开销

**调整实时性**：
```json
// 设置为 -1 表示不自动刷新，适合批量导入
PUT /my_index/_settings
{
  "index": {
    "refresh_interval": "-1"
  }
}

// 导入完成后恢复
PUT /my_index/_settings
{
  "index": {
    "refresh_interval": "1s"
  }
}

// 手动刷新（立即可见）
POST /my_index/_refresh
```

---

## 三、索引与搜索

### 8. text 和 keyword 类型的区别？何时使用？

| 特性 | text | keyword |
|------|------|---------|
| **用途** | 全文搜索 | 精确匹配、排序、聚合 |
| **分词** | 会分词 | 不会分词 |
| **存储** | 分词后的词项 | 完整字符串 |
| **查询方式** | match、query_string | term、terms、range |
| **排序/聚合** | 不建议 | 支持 |
| **大小写** | 通常转小写 | 保持原样 |

**text 类型工作流程**：
```
原始文本："Elasticsearch is powerful"
    ↓（分析器分词）
分词结果：["elasticsearch", "is", "powerful"]
    ↓（建立倒排索引）
索引：elasticsearch → doc_1
      is → doc_1
      powerful → doc_1
```

**keyword 类型工作流程**：
```
原始文本："iPhone 15 Pro"
    ↓（不分词，原样存储）
索引："iPhone 15 Pro" → doc_1
```

**使用场景**：

**text**：
- 商品标题、描述、文章内容
- 日志信息
- 用户评论
- 任何需要全文搜索的文本

**keyword**：
- ID、邮箱、状态码、标签
- 分类、品牌、性别
- 精确过滤、排序、聚合的字段

**多字段策略（Multi-fields）**：
```json
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",           // 用于全文搜索
        "analyzer": "ik_max_word",
        "fields": {
          "keyword": {            // 子字段，用于精确匹配
            "type": "keyword"
          }
        }
      }
    }
  }
}

// 使用方式
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "苹果手机" }},      // 全文搜索
        { "term": { "title.keyword": "苹果手机" }} // 精确匹配
      ]
    }
  }
}
```

**常见错误**：
```json
// 错误：用 term 查询 text 字段
GET /products/_search
{
  "query": {
    "term": { "title": "iPhone" }  // 可能匹配不到，因为 title 被分词了
  }
}

// 正确：text 字段用 match
GET /products/_search
{
  "query": {
    "match": { "title": "iPhone" }
  }
}
```

---

### 9. analyzer 的组成部分有哪些？自定义分词器的场景？

**Analyzer（分析器）组成**：

分析器 = Character Filters（字符过滤器）+ Tokenizer（分词器）+ Token Filters（词项过滤器）

**1. Character Filters（字符过滤器）** - 可选，0 或多个
- 在分词前对原始文本进行预处理
- **html_strip**：去除 HTML 标签
- **mapping**：字符映射替换
- **pattern_replace**：正则替换

**2. Tokenizer（分词器）** - 必选，1 个
- 将文本切分成词项（token）
- **standard**：标准分词器（按 Unicode 文本分割算法）
- **whitespace**：按空格分词
- **keyword**：不分词，整体作为 token
- **pattern**：正则分词
- **ik_max_word / ik_smart**：IK 分词器（中文）
- **jieba**：结巴分词（中文）

**3. Token Filters（词项过滤器）** - 可选，0 或多个
- 对分词后的词项进行处理
- **lowercase**：转小写
- **stop**：去除停用词
- **synonym**：同义词处理
- **stemmer**：词干提取

**标准分析器流程示例**：
```
原始文本："The 2 QUICK Brown-Foxes jumped!"

Character Filter：无变化

Tokenizer (standard)：
→ ["The", "2", "QUICK", "Brown", "Foxes", "jumped"]

Token Filters：
- lowercase: ["the", "2", "quick", "brown", "foxes", "jumped"]
- stop (可选): ["2", "quick", "brown", "foxes", "jumped"]

最终词项：["2", "quick", "brown", "foxes", "jumped"]
```

**自定义分词器场景**：

**场景一：中文分词**
```json
{
  "settings": {
    "analysis": {
      "analyzer": {
        "ik_custom": {
          "type": "custom",
          "tokenizer": "ik_max_word",
          "filter": ["lowercase", "synonym_filter"]
        }
      },
      "filter": {
        "synonym_filter": {
          "type": "synonym",
          "synonyms": ["手机,移动电话,iphone"]
        }
      }
    }
  }
}
```

**场景二：保护特殊字符**
```json
{
  "settings": {
    "analysis": {
      "analyzer": {
        "preserve_special": {
          "type": "custom",
          "char_filter": ["preserve_special_chars"],
          "tokenizer": "standard",
          "filter": ["lowercase"]
        }
      },
      "char_filter": {
        "preserve_special_chars": {
          "type": "mapping",
          "mappings": ["-=>_", "@=>_"]
        }
      }
    }
  }
}
```

**场景三：同义词扩展**
```json
{
  "settings": {
    "analysis": {
      "filter": {
        "synonym_graph": {
          "type": "synonym_graph",
          "synonyms": [
            "电脑,计算机,pc",
            "番茄,西红柿",
            "iphone,苹果手机"
          ]
        }
      }
    }
  }
}
```

**常用内置分析器**：
- **standard**：默认，标准分词
- **simple**：按非字母切分，转小写
- **whitespace**：按空格切分
- **keyword**：不分词
- **pattern**：正则分词
- **english**：英文专用，移除停用词、词干提取

---

### 10. match、term、terms、bool 查询的区别？

**1. Match Query（全文搜索）**
```json
GET /products/_search
{
  "query": {
    "match": {
      "title": "iPhone 手机"
    }
  }
}
```
- 用于 `text` 类型字段
- 会先对查询词分词
- 然后搜索每个词项
- 默认使用 OR 逻辑，可指定 operator
- **示例**：查询 "iPhone 手机" 会分词为 ["iphone", "手机"]，匹配包含任一或两者的文档

**2. Term Query（精确匹配）**
```json
GET /products/_search
{
  "query": {
    "term": {
      "status": "active"
    }
  }
}
```
- 用于 `keyword` 类型字段
- **不会**对查询词分词
- 精确匹配字段值
- 对 `text` 字段使用可能匹配不到（因为 text 被分词存储）

**3. Terms Query（多值精确匹配）**
```json
GET /products/_search
{
  "query": {
    "terms": {
      "status": ["active", "pending", "completed"]
    }
  }
}
```
- 类似 Term，但支持多个值
- 字段值匹配任一值即返回
- 相当于 SQL 的 `IN` 操作

**4. Bool Query（布尔组合查询）**
```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [                    // 必须满足（AND）
        { "match": { "title": "手机" } },
        { "range": { "price": { "lte": 5000 } } }
      ],
      "should": [                  // 应该满足（OR）
        { "term": { "brand": "apple" } },
        { "term": { "brand": "samsung" } }
      ],
      "must_not": [                // 必须不满足（NOT）
        { "term": { "status": "deleted" } }
      ],
      "filter": [                  // 过滤（不影响评分，可缓存）
        { "term": { "category": "手机" } }
      ]
    }
  }
}
```

**Bool Query 子句说明**：

| 子句 | 逻辑 | 影响评分 | 说明 |
|-----|------|---------|------|
| must | AND | 是 | 必须匹配 |
| should | OR | 是 | 应该匹配（至少满足 minimum_should_match）|
| must_not | NOT | 否 | 必须不匹配 |
| filter | AND | 否 | 必须匹配，不计算评分，可缓存 |

**对比总结**：

| 查询类型 | 适用字段 | 是否分词 | 使用场景 |
|---------|---------|---------|---------|
| match | text | 是 | 全文搜索 |
| term | keyword | 否 | 精确匹配 |
| terms | keyword | 否 | 多值精确匹配 |
| bool | 任意 | 视子查询 | 复杂组合条件 |

**常见陷阱**：
```json
// 错误：对 text 字段使用 term
GET /products/_search
{
  "query": {
    "term": { "title": "iPhone" }  // text 字段存储的是分词后的词项
  }
}
// 正确做法：
{ "match": { "title": "iPhone" } }
// 或
{ "term": { "title.keyword": "iPhone" } }
```

---

### 11. 什么是 relevance score？BM25 算法原理？

**Relevance Score（相关性评分）**：
- 表示文档与查询的匹配程度
- 分数越高，相关性越强
- ES 默认按 `_score` 降序排列结果

**评分计算过程**：
1. 计算每个查询词项的 TF-IDF 或 BM25 分数
2. 组合多个词项的分数
3. 应用 boost 等调整因子
4. 得出最终相关性分数

**TF-IDF vs BM25**：

| 特性 | TF-IDF | BM25 |
|-----|--------|------|
| 词频饱和 | 线性增长 | 非线性饱和 |
| 文档长度归一化 | 简单 | 更精细 |
| 参数调优 | 少 | k1, b 可调 |
| 现代应用 | 较少 | ES 默认 |

**BM25 算法原理**：

BM25（Best Match 25）是基于概率检索模型的评分算法。

**公式**：
```
score(D, Q) = Σ(IDF(q_i) * TF(q_i, D) * boost)

其中：
IDF(q_i) = log(1 + (N - n(q_i) + 0.5) / (n(q_i) + 0.5))
TF(q_i, D) = (f(q_i, D) * (k1 + 1)) / (f(q_i, D) + k1 * (1 - b + b * |D| / avgdl))
```

**参数说明**：
- **N**：文档总数
- **n(q_i)**：包含词项 q_i 的文档数
- **f(q_i, D)**：词项 q_i 在文档 D 中的出现次数（词频）
- **|D|**：文档 D 的长度
- **avgdl**：平均文档长度
- **k1**：控制词频饱和（默认 1.2，范围 0-3）
  - k1=0：忽略词频
  - k1 越大，词频影响越大
- **b**：控制文档长度归一化（默认 0.75，范围 0-1）
  - b=0：忽略文档长度
  - b=1：完全归一化

**BM25 特点**：
1. **词频饱和**：一个词出现 10 次和 100 次的分数差距不会线性增长
2. **文档长度归一化**：长文档不会因为是长文档而获得不公平优势
3. **参数可调**：可根据不同数据集调整 k1 和 b

**自定义相似度**：
```json
PUT /my_index
{
  "settings": {
    "index": {
      "similarity": {
        "my_bm25": {
          "type": "BM25",
          "k1": 1.5,
          "b": 0.6
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "similarity": "my_bm25"
      }
    }
  }
}
```

**解释查询**：
```json
GET /products/_search
{
  "explain": true,
  "query": {
    "match": { "title": "手机" }
  }
}
```

---

### 12. 如何实现搜索高亮、suggester、聚合？

**1. 搜索高亮（Highlight）**

高亮显示匹配的文本片段。

```json
GET /articles/_search
{
  "query": {
    "match": { "content": "Elasticsearch" }
  },
  "highlight": {
    "fields": {
      "content": {
        "pre_tags": ["<em>"],
        "post_tags": ["</em>"],
        "fragment_size": 150,          // 片段大小
        "number_of_fragments": 3,      // 返回片段数
        "type": "unified"              // 高亮算法
      }
    }
  }
}
```

**高亮类型**：
- **unified**（默认）：使用 Lucene 的 Unified Highlighter，性能好
- **plain**：标准高亮器，精确但慢
- **fvh**：Fast Vector Highlighter，需要 term_vectors，适合大字段

**高亮配置示例**：
```json
{
  "mappings": {
    "properties": {
      "content": {
        "type": "text",
        "term_vector": "with_positions_offsets"  // 启用 FVH
      }
    }
  }
}
```

**2. Suggester（搜索建议）**

ES 提供四种 suggester：

**a) Term Suggester（词项建议）**
```json
POST /articles/_search
{
  "suggest": {
    "my-suggest": {
      "text": "elasricsearch",  // 拼写错误的词
      "term": {
        "field": "title"
      }
    }
  }
}
// 返回：elasticsearch
```
- 基于编辑距离（Levenshtein distance）
- 用于拼写纠错

**b) Phrase Suggester（短语建议）**
```json
POST /articles/_search
{
  "suggest": {
    "my-suggest": {
      "text": "elasric search",
      "phrase": {
        "field": "title"
      }
    }
  }
}
```
- 考虑词项之间的关系
- 更好的纠错效果

**c) Completion Suggester（自动完成）**
```json
// 定义字段
{
  "mappings": {
    "properties": {
      "suggest": {
        "type": "completion"
      }
    }
  }
}

// 使用
POST /products/_search
{
  "suggest": {
    "product-suggest": {
      "prefix": "ipho",
      "completion": {
        "field": "suggest",
        "size": 10
      }
    }
  }
}
```
- 使用 FST（Finite State Transducer）结构
- 性能极高，适合搜索框自动补全

**d) Context Suggester（上下文建议）**
```json
{
  "suggest": {
    "field": "suggest",
    "contexts": {
      "category": "phone"
    }
  }
}
```
- 基于特定上下文过滤建议

**3. 聚合（Aggregation）**

**a) Metrics Aggregations（指标聚合）**
```json
GET /sales/_search
{
  "size": 0,
  "aggs": {
    "avg_price": {
      "avg": { "field": "price" }
    },
    "max_price": {
      "max": { "field": "price" }
    },
    "stats_price": {
      "stats": { "field": "price" }  // 返回计数、最小、最大、平均、总和
    }
  }
}
```

**b) Bucket Aggregations（桶聚合）**
```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": {                      // 按类别分组
        "field": "category",
        "size": 10
      }
    },
    "by_price_range": {
      "range": {                      // 按价格区间分组
        "field": "price",
        "ranges": [
          { "to": 1000 },
          { "from": 1000, "to": 5000 },
          { "from": 5000 }
        ]
      }
    },
    "by_date": {
      "date_histogram": {             // 按日期分组
        "field": "created_at",
        "calendar_interval": "month"
      }
    }
  }
}
```

**c) Pipeline Aggregations（管道聚合）**
```json
GET /sales/_search
{
  "size": 0,
  "aggs": {
    "sales_per_month": {
      "date_histogram": {
        "field": "date",
        "calendar_interval": "month"
      },
      "aggs": {
        "total_sales": {
          "sum": { "field": "amount" }
        },
        "sales_diff": {
          "derivative": {             // 计算环比
            "buckets_path": "total_sales"
          }
        }
      }
    }
  }
}
```

**子聚合示例**：
```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category" },
      "aggs": {
        "avg_price": {
          "avg": { "field": "price" }
        },
        "by_brand": {
          "terms": { "field": "brand" }
        }
      }
    }
  }
}
```

---

## 四、性能优化

### 13. 如何避免深度分页问题？有哪些方案？

**深度分页问题**：

传统 `from + size` 方式在深度分页时性能急剧下降。

```json
// 问题：翻到第 10000 页，每页 100 条
GET /products/_search
{
  "from": 999900,  // 需要跳过 999900 条
  "size": 100
}
```

**性能问题**：
- 每个分片需要返回 `from + size` 条数据
- 协调节点需要排序和合并大量数据
- 内存消耗巨大
- ES 默认限制 `from + size <= 10000`

**解决方案**：

**方案一：Scroll API（快照遍历）**
```java
// 1. 获取 scroll_id（保持 1 分钟）
POST /products/_search?scroll=1m
{
  "size": 100,
  "query": { "match_all": {} }
}

// 2. 使用 scroll_id 继续获取
POST /_search/scroll
{
  "scroll": "1m",
  "scroll_id": "DXF1ZXJ5QW5kRmV0Y2gB..."
}
```
- 适用于导出大量数据
- 不适合实时场景（数据快照）
- 需要维护 scroll 上下文

**方案二：Search After（推荐）**
```json
// 第一页
GET /products/_search
{
  "size": 100,
  "sort": [
    { "price": "asc" },
    { "_id": "asc" }    // 必须加上唯一字段确保稳定排序
  ]
}

// 下一页（使用上一页最后一条数据）
GET /products/_search
{
  "size": 100,
  "sort": [
    { "price": "asc" },
    { "_id": "asc" }
  ],
  "search_after": [    // 上一页最后一个文档的排序值
    5999,
    "product_1000"
  ]
}
```
- 适用于实时翻页
- 不支持跳页，只能一页一页翻
- 性能好，无深度分页问题

**方案三：增加限制**
```json
// 设置最大 from + size
PUT /_cluster/settings
{
  "persistent": {
    "index.max_result_window": 50000
  }
}
```
- 简单但不根本解决问题
- 增加内存消耗

**方案对比**：

| 方案 | 适用场景 | 优点 | 缺点 |
|-----|---------|------|------|
| from + size | 浅分页 | 简单 | 深度分页性能差 |
| scroll | 数据导出 | 适合大量数据 | 非实时，占用资源 |
| search_after | 实时翻页 | 性能好，实时 | 不支持跳页 |

**最佳实践**：
1. 业务层面限制最大翻页深度（如最多 100 页）
2. 提供搜索过滤，减少结果集
3. 使用 search_after 替代传统分页
4. 导出数据使用 Scroll API

---

### 14. ES 写入性能调优有哪些手段？（bulk、refresh_interval、buffer 等）

**写入性能优化手段**：

**1. 使用 Bulk API**
```java
// 批量写入，减少网络开销
BulkRequest bulkRequest = new BulkRequest();
bulkRequest.add(new IndexRequest("index").source(data1));
bulkRequest.add(new IndexRequest("index").source(data2));
// ... 更多文档

client.bulk(bulkRequest);
```
- 单批次大小建议：5-15MB
- 文档数建议：1000-5000 条
- 避免超大 bulk 请求导致内存问题

**2. 调整 Refresh Interval**
```json
PUT /my_index/_settings
{
  "index": {
    "refresh_interval": "-1"      // 禁用自动刷新
  }
}

// 批量导入完成后恢复
PUT /my_index/_settings
{
  "index": {
    "refresh_interval": "1s"
  }
}
```
- 批量导入时禁用 refresh
- 导入完成后手动 refresh

**3. 调整 Translog 设置**
```json
PUT /my_index/_settings
{
  "index": {
    "translog": {
      "durability": "async",      // 异步写入（默认 request）
      "sync_interval": "120s"     // 同步间隔（默认 5s）
    }
  }
}
```
- `durability: request`：每次请求都 fsync（安全但慢）
- `durability: async`：异步 fsync（更快但有丢失风险）

**4. 减少副本数**
```json
PUT /my_index/_settings
{
  "index": {
    "number_of_replicas": 0       // 导入时设为 0
  }
}

// 导入完成后恢复
PUT /my_index/_settings
{
  "index": {
    "number_of_replicas": 1
  }
}
```
- 导入期间无需维护副本
- 导入完成后动态增加副本

**5. 使用 Ingest Node 预处理**
```json
// 定义 pipeline
PUT _ingest/pipeline/my_pipeline
{
  "processors": [
    {
      "script": {
        "source": "ctx.timestamp = new Date()"
      }
    }
  ]
}

// 使用 pipeline
POST /my_index/_doc?pipeline=my_pipeline
{
  "title": "test"
}
```
- 将计算密集操作 offload 到 ingest 节点

**6. 优化 Mapping**
```json
{
  "mappings": {
    "dynamic": "strict",          // 禁用动态 mapping
    "properties": {
      "title": {
        "type": "text",
        "index": true,
        "store": false            // 不存储原始值
      },
      "description": {
        "type": "text",
        "index": false             // 不索引（仅存储）
      }
    }
  }
}
```
- 禁用不必要的字段索引
- 使用合适的字段类型

**7. 调整 JVM 和系统参数**
```yaml
# elasticsearch.yml
# 增加索引缓冲区
indices.memory.index_buffer_size: 20%
indices.memory.min_index_buffer_size: 96mb

# 增加线程池大小
thread_pool.write.size: 8
thread_pool.write.queue_size: 1000
```

**8. 使用多线程写入**
```java
// 使用线程池并发写入
ExecutorService executor = Executors.newFixedThreadPool(8);

for (List<Document> batch : batches) {
    executor.submit(() -> {
        // 批量写入
        bulkIndex(batch);
    });
}
```
- 并发写入提升吞吐量
- 注意控制并发度，避免过载

**优化效果对比**：

| 优化项 | 性能提升 |
|-------|---------|
| Bulk API | 10-100 倍 |
| 禁用 refresh | 5-10 倍 |
| 异步 translog | 2-3 倍 |
| 减少副本 | 2-3 倍 |

**写入流程优化总结**：
```
批量导入流程：
1. 创建索引，设置：refresh_interval: -1, number_of_replicas: 0
2. 调整 translog.durability: async
3. 使用 Bulk API 多线程批量写入
4. 导入完成后手动 refresh
5. 恢复 refresh_interval 和 number_of_replicas
```

---

### 15. 如何优化 ES 查询性能？

**查询性能优化手段**：

**1. 使用 Filter Context**
```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "手机" } }  // 参与评分
      ],
      "filter": [
        { "term": { "status": "active" }},  // 不参与评分，可缓存
        { "range": { "price": { "lte": 5000 } }}
      ]
    }
  }
}
```
- Filter 不计算评分，性能更好
- Filter 结果被缓存，重复查询更快

**2. 避免深度分页**
- 使用 search_after 替代 from + size
- 限制最大翻页深度
- 使用 Scroll API 导出数据

**3. 控制返回字段**
```json
GET /products/_search
{
  "_source": ["title", "price", "category"],  // 只返回需要的字段
  "query": { "match_all": {} }
}
```
- 使用 `_source` 过滤减少网络传输
- 或使用 `stored_fields`

**4. 使用 Routing**
```json
// 索引时指定 routing
POST /products/_doc?routing=category_1
{
  "title": "iPhone 15",
  "category": "手机"
}

// 查询时使用相同 routing
GET /products/_search?routing=category_1
{
  "query": {
    "match": { "title": "iPhone" }
  }
}
```
- 查询只搜索特定分片
- 减少需要查询的分片数

**5. 预加载（Eager Loading）**
```json
{
  "mappings": {
    "properties": {
      "category": {
        "type": "keyword",
        "eager_global_ordinals": true  // 预加载全局序数
      }
    }
  }
}
```
- 聚合字段启用 eager_global_ordinals
- 加速聚合查询

**6. 禁用字段的 Norms 和 Doc Values**
```json
{
  "mappings": {
    "properties": {
      "description": {
        "type": "text",
        "norms": false,      // 不参与评分时禁用
        "index_options": "freqs"  // 减少索引信息
      },
      "created_at": {
        "type": "date",
        "doc_values": false  // 不聚合排序时禁用
      }
    }
  }
}
```
- norms：用于评分归一化
- doc_values：用于聚合、排序

**7. 使用合适的数据类型**
```json
{
  "mappings": {
    "properties": {
      "status": { "type": "keyword" },  // 精确匹配用 keyword
      "price": { "type": "integer" },   // 整数用 integer
      "description": { "type": "text" } // 全文搜索用 text
    }
  }
}
```

**8. 优化查询语句**
```json
// 避免：wildcard 查询
{ "wildcard": { "title": "*手机*" } }  // 性能差

// 改用：match 或 prefix
{ "match": { "title": "手机" } }
{ "prefix": { "title": "手机" } }

// 避免：过多的 should 子句
// 使用 minimum_should_match 控制
{
  "bool": {
    "should": [...],
    "minimum_should_match": 2
  }
}
```

**9. 使用 Index Sorting**
```json
PUT /events
{
  "settings": {
    "index": {
      "sort.field": ["timestamp"],
      "sort.order": ["desc"]
    }
  }
}
```
- 数据按指定字段物理排序
- 加速范围查询和排序

**10. 监控慢查询**
```yaml
# elasticsearch.yml
index.search.slowlog.threshold.query.warn: 10s
index.search.slowlog.threshold.query.info: 5s
index.search.slowlog.threshold.fetch.warn: 1s
```

**性能优化检查清单**：
- [ ] 使用 filter 替代 must（不评分场景）
- [ ] 限制返回字段和结果数
- [ ] 避免深度分页
- [ ] 禁用不必要的 norms 和 doc_values
- [ ] 使用合适的分词器
- [ ] 监控慢查询日志
- [ ] 优化 JVM 堆内存（不超过 31GB）

---

### 16. 索引设计时需要考虑哪些因素？

**索引设计关键因素**：

**1. 分片策略**
```json
PUT /my_index
{
  "settings": {
    "number_of_shards": 3,      // 主分片数，创建后不可更改
    "number_of_replicas": 1     // 副本数，可动态调整
  }
}
```
- 主分片数 = 数据节点数 × 1-3
- 单个分片大小控制在 10-50GB
- 避免频繁刷盘的小分片

**2. Mapping 设计**
```json
{
  "mappings": {
    "dynamic": "strict",        // 严格模式，防止字段爆炸
    "_source": { "excludes": ["sensitive_data"] },
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "ik_max_word",
        "fields": {
          "keyword": { "type": "keyword" }  // 多字段映射
        }
      },
      "price": { "type": "integer" },
      "tags": { "type": "keyword" },        // 数组用 keyword
      "location": { "type": "geo_point" }   // 地理位置
    }
  }
}
```

**3. 字段设计原则**
- **text**：用于全文搜索
- **keyword**：用于过滤、排序、聚合
- **numeric**：选择合适的整数/浮点类型
- **date**：统一时间格式
- **nested**：处理嵌套对象
- **join**：处理父子关系（尽量少用）

**4. 索引生命周期管理（ILM）**
```json
PUT _ilm/policy/logs_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "50GB",
            "max_age": "30d"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "freeze": {}
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

**5. 索引模板（Index Template）**
```json
PUT _index_template/my_template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1
    },
    "mappings": {
      "properties": {
        "timestamp": { "type": "date" },
        "level": { "type": "keyword" },
        "message": { "type": "text" }
      }
    }
  }
}
```

**6. 别名（Alias）设计**
```json
POST /_aliases
{
  "actions": [
    { "add": { "index": "products_v1", "alias": "products", "is_write_index": true }}
  ]
}
```
- 使用别名进行索引切换（无停机重索引）
- 读操作使用 alias，写操作指定具体索引

**7. 数据模型设计**

**扁平化设计（推荐）**：
```json
{
  "order_id": "123",
  "user_id": "456",
  "products": [
    { "name": "iPhone", "price": 5999 },
    { "name": "AirPods", "price": 1299 }
  ]
}
```

**嵌套文档（Nested）**：
```json
{
  "order_id": "123",
  "products": {
    "type": "nested",
    "properties": {
      "name": { "type": "keyword" },
      "price": { "type": "integer" }
    }
  }
}
```

**父子关系（Join）**：
- 尽量避免使用
- 使用 application-side join 替代

**8. 索引命名规范**
```
<business>-<type>-<version>-<date>
示例：
- ecommerce-products-v1-2024.01
- logs-app-error-2024.01.15
```

**设计检查清单**：
- [ ] 预估数据量，确定分片数
- [ ] 定义明确的 Mapping，禁用动态 mapping
- [ ] 区分 text 和 keyword 使用场景
- [ ] 配置 ILM 自动管理索引生命周期
- [ ] 使用别名进行索引切换
- [ ] 配置索引模板统一设置
- [ ] 评估是否需要 nested 或 join
- [ ] 设置合理的 refresh_interval

---

## 五、运维与高可用

### 17. ES 集群脑裂问题如何解决？

**脑裂（Split Brain）问题**：
- 集群分裂成多个独立的子集群
- 每个子集群都认为自己是主集群
- 导致数据不一致和冲突

**产生原因**：
1. 网络分区导致节点间通信中断
2. Master 节点选举出现问题
3. 多个节点同时认为自己是 Master

**解决方案**：

**1. 配置 discovery.zen.minimum_master_nodes（ES 6.x 及以下）**
```yaml
# elasticsearch.yml
discovery.zen.minimum_master_nodes: 2
```
- 必须至少有 N/2 + 1 个 master-eligible 节点同意才能选举
- N 为 master-eligible 节点总数

**2. 配置 cluster.initial_master_nodes（ES 7.x+）**
```yaml
# elasticsearch.yml
cluster.name: my_cluster
node.name: node-1
node.master: true
cluster.initial_master_nodes:
  - node-1
  - node-2
  - node-3
discovery.seed_hosts:
  - 192.168.1.1:9300
  - 192.168.1.2:9300
  - 192.168.1.3:9300
```
- 首次启动时指定可成为 Master 的节点列表
- 避免错误的自动发现

**3. 增加 Master 节点数量**
- 建议配置 3-5 个 master-eligible 节点
- 奇数节点避免平局
- 至少 3 个以容忍 1 个节点故障

**4. 网络稳定性保障**
- 使用专用网络
- 配置防火墙规则允许节点间通信
- 避免跨数据中心部署 Master 节点

**5. 节点角色分离**
```yaml
# Master 节点配置
node.master: true
node.data: false
node.ingest: false

# Data 节点配置
node.master: false
node.data: true
node.ingest: false
```
- 专用 Master 节点不参与数据存储
- 减少 Master 节点的 GC 压力

**6. 监控和告警**
```json
GET /_cluster/health
{
  "cluster_name": "my_cluster",
  "status": "green",
  "number_of_nodes": 5,
  "number_of_data_nodes": 3,
  "active_primary_shards": 10,
  "active_shards": 20
}
```
- 监控 number_of_nodes 和 master 节点状态
- 设置告警：节点数异常、Master 切换频繁

**最佳实践**：
1. 专用 3 个节点作为 Master
2. 使用稳定的网络环境
3. 定期备份集群状态
4. 配置合适的 discovery 设置
5. 避免频繁的网络抖动

---

### 18. 如何处理集群 yellow/red 状态？

**集群状态说明**：

| 状态 | 含义 | 处理方式 |
|-----|------|---------|
| **green** | 所有分片正常分配 | 无需处理 |
| **yellow** | 所有主分片已分配，但副本未分配 | 检查节点和分片分配 |
| **red** | 存在未分配的主分片 | 紧急处理，可能数据丢失 |

**Yellow 状态处理**：

**原因分析**：
1. 节点离线或重启
2. 副本数大于数据节点数
3. 磁盘水位线触发（磁盘满了）
4. 分片分配延迟

**排查命令**：
```bash
# 查看集群健康
GET /_cluster/health

# 查看未分配分片
GET /_cat/shards?v&h=index,shard,prirep,state,unassigned.reason

# 查看集群 allocation 解释
GET /_cluster/allocation/explain
```

**解决方案**：

```bash
# 1. 检查是否有节点离线
GET /_cat/nodes?v

# 2. 等待自动恢复（默认延迟 1 分钟）
PUT /_cluster/settings
{
  "transient": {
    "cluster.routing.allocation.enable": "all"
  }
}

# 3. 临时减少副本数
PUT /my_index/_settings
{
  "index": {
    "number_of_replicas": 0
  }
}

# 4. 检查磁盘水位线
GET /_cat/allocation?v

# 调整水位线（临时）
PUT /_cluster/settings
{
  "transient": {
    "cluster.routing.allocation.disk.watermark.low": "90%",
    "cluster.routing.allocation.disk.watermark.high": "95%",
    "cluster.routing.allocation.disk.watermark.flood_stage": "98%"
  }
}
```

**Red 状态处理**：

**原因分析**：
1. 数据节点全部离线
2. 分片损坏
3. 磁盘故障

**紧急处理**：

```bash
# 1. 检查未分配的分片
GET /_cat/indices?v&health=red

# 2. 尝试重新分配
POST /_cluster/reroute
{
  "commands": [
    {
      "allocate_empty_primary": {
        "index": "my_index",
        "shard": 0,
        "node": "node-1",
        "accept_data_loss": true
      }
    }
  ]
}

# 3. 如果数据无法恢复，删除损坏的索引（最后手段）
DELETE /corrupted_index
```

**预防措施**：
1. 配置合适的副本数（至少 1 个）
2. 监控磁盘空间
3. 使用 ILM 自动管理索引
4. 定期备份数据
5. 配置集群告警

---

### 19. ES 数据备份与恢复方案？

**方案一：Snapshot & Restore（推荐）**

**1. 配置仓库**
```bash
# 注册快照仓库（本地文件系统）
PUT /_snapshot/my_backup
{
  "type": "fs",
  "settings": {
    "location": "/mount/backups",
    "compress": true
  }
}

# 注册 S3 仓库
PUT /_snapshot/s3_backup
{
  "type": "s3",
  "settings": {
    "bucket": "my-es-backups",
    "region": "us-east-1",
    "compress": true
  }
}
```

**2. 创建快照**
```bash
# 创建全量快照
PUT /_snapshot/my_backup/snapshot_1

# 创建指定索引的快照
PUT /_snapshot/my_backup/snapshot_2
{
  "indices": "index_1,index_2",
  "ignore_unavailable": true,
  "include_global_state": false
}
```

**3. 查看和恢复快照**
```bash
# 查看快照列表
GET /_snapshot/my_backup/_all

# 查看快照详情
GET /_snapshot/my_backup/snapshot_1

# 恢复快照
POST /_snapshot/my_backup/snapshot_1/_restore
{
  "indices": "index_1,index_2",
  "ignore_unavailable": true,
  "include_global_state": false,
  "rename_pattern": "index_(.+)",
  "rename_replacement": "restored_index_$1"
}
```

**4. 自动化快照（SLM - Snapshot Lifecycle Management）**
```bash
PUT /_slm/policy/daily_snapshots
{
  "name": "<daily-snap-{now/d}>",
  "schedule": "0 30 1 * * ?",      // 每天 1:30 执行
  "repository": "my_backup",
  "config": {
    "indices": ["*"],
    "ignore_unavailable": true
  },
  "retention": {
    "expire_after": "30d",
    "min_count": 5,
    "max_count": 50
  }
}
```

**方案二：跨集群复制（CCR）**
```bash
# 配置远程集群
PUT /_cluster/settings
{
  "persistent": {
    "cluster.remote": {
      "leader_cluster": {
        "seeds": ["192.168.1.1:9300"]
      }
    }
  }
}

# 创建 follower 索引
PUT /follower_index/_ccr/follow
{
  "remote_cluster": "leader_cluster",
  "leader_index": "leader_index"
}
```

**方案三：导出导入工具**

**elasticdump**：
```bash
# 导出索引
elasticdump \
  --input=http://localhost:9200/my_index \
  --output=/backup/my_index.json \
  --type=data

# 导入索引
elasticdump \
  --input=/backup/my_index.json \
  --output=http://localhost:9200/my_index \
  --type=data
```

**logstash**：
```bash
# 使用 logstash 做数据迁移
input {
  elasticsearch {
    hosts => ["source_es:9200"]
    index => "my_index"
  }
}
output {
  elasticsearch {
    hosts => ["target_es:9200"]
    index => "my_index"
  }
}
```

**备份策略建议**：

| 备份类型 | 频率 | 保留时间 |
|---------|------|---------|
| 增量快照 | 每小时 | 1 天 |
| 全量快照 | 每天 | 7 天 |
| 全量快照 | 每周 | 30 天 |
| 全量快照 | 每月 | 1 年 |

**恢复流程**：
1. 评估数据丢失范围
2. 选择最近的可用快照
3. 在新集群或原集群恢复
4. 验证数据完整性
5. 切换应用流量

---

### 20. 如何监控 ES 集群健康状态？

**监控维度**：

**1. 集群层面**
```bash
# 集群健康
GET /_cluster/health

# 集群状态
GET /_cluster/state

# 集群统计
GET /_cluster/stats

# 集群设置
GET /_cluster/settings
```

**关键指标**：
- `status`: green/yellow/red
- `number_of_nodes`: 节点数
- `number_of_data_nodes`: 数据节点数
- `active_primary_shards`: 主分片数
- `active_shards`: 总分片数
- `unassigned_shards`: 未分配分片数
- `relocating_shards`: 正在迁移的分片数

**2. 节点层面**
```bash
# 节点信息
GET /_nodes/stats

# 热点线程
GET /_nodes/hot_threads

# 节点分配
GET /_cat/allocation?v
```

**关键指标**：
- JVM 堆内存使用（不超过 75%）
- CPU 使用率
- 磁盘使用率
- 网络流量
- 线程池队列大小

**3. 索引层面**
```bash
# 索引统计
GET /_stats

# 索引段信息
GET /_segments

# 索引恢复状态
GET /_recovery
```

**关键指标**：
- 索引大小
- 文档数
- 分片大小
- 段数量（越少越好）
- 合并统计

**4. 搜索性能**
```bash
# 索引级搜索统计
GET /my_index/_stats/search

# 慢查询日志
GET /_cluster/settings?include_defaults=true&filter_path=**.search
```

**关键指标**：
- 查询延迟
- 查询吞吐量
- 慢查询数量
- 缓存命中率

**监控工具**：

**1. Elasticsearch 自带监控（X-Pack）**
```bash
# 启用监控
PUT /_cluster/settings
{
  "persistent": {
    "xpack.monitoring.collection.enabled": true
  }
}
```

**2. 常用监控方案**
- **Kibana + Metricbeat**：官方方案
- **Prometheus + Grafana**：使用 elasticsearch_exporter
- **Zabbix**：使用自定义脚本
- **Cerebro**：Web UI 管理工具

**Prometheus + Grafana 配置示例**：

```yaml
# elasticsearch_exporter 启动
./elasticsearch_exporter --es.uri=http://localhost:9200

# Prometheus 配置
cat >> prometheus.yml << EOF
scrape_configs:
  - job_name: 'elasticsearch'
    static_configs:
      - targets: ['localhost:9114']
EOF
```

**告警规则示例**：

```yaml
# 集群状态告警
- alert: ElasticsearchClusterStatusRed
  expr: elasticsearch_cluster_health_status{color="red"} == 1
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "Elasticsearch cluster status is RED"

# JVM 堆内存告警
- alert: ElasticsearchHighJVMHeap
  expr: elasticsearch_jvm_memory_used_bytes{area="heap"} / elasticsearch_jvm_memory_max_bytes{area="heap"} > 0.8
  for: 5m
  labels:
    severity: warning

# 磁盘空间告警
- alert: ElasticsearchDiskHigh
  expr: elasticsearch_filesystem_data_used_percent > 85
  for: 5m
  labels:
    severity: warning
```

**监控检查清单**：
- [ ] 集群状态 green
- [ ] 无未分配分片
- [ ] JVM 堆内存 < 75%
- [ ] 磁盘使用率 < 85%
- [ ] 节点数符合预期
- [ ] 无慢查询增长
- [ ] 缓存命中率正常
- [ ] 线程池无堆积

---

## 六、实战场景

### 21. 如何设计一个电商商品搜索系统？

**系统架构**：

```
┌─────────────────────────────────────────────────────────────┐
│                          客户端                              │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│                     API Gateway                             │
└────────────────────┬────────────────────────────────────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
┌───────▼────┐ ┌────▼───┐ ┌──────▼──────┐
│   Query    │ │ Suggest │ │  Aggregation │
│  Service   │ │ Service │ │   Service   │
└──────┬─────┘ └────┬────┘ └──────┬──────┘
       │            │             │
       └────────────┼─────────────┘
                    │
┌───────────────────▼─────────────────────────────────────────┐
│                    Elasticsearch                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Index: products                                     │   │
│  │  - title: text + keyword                             │   │
│  │  - category: keyword                                 │   │
│  │  - brand: keyword                                    │   │
│  │  - price: integer                                    │   │
│  │  - attributes: nested                                │   │
│  │  - stock: integer                                    │   │
│  │  - sales: integer                                    │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Mapping 设计**：

```json
PUT /products
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "ik_smart_synonym": {
          "type": "custom",
          "tokenizer": "ik_smart",
          "filter": ["synonym_filter"]
        }
      },
      "filter": {
        "synonym_filter": {
          "type": "synonym",
          "synonyms": [
            "iphone,苹果手机,爱疯",
            "华为,huawei,遥遥领先",
            "电脑,笔记本,计算机"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "id": { "type": "keyword" },
      "title": {
        "type": "text",
        "analyzer": "ik_smart_synonym",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "category": { "type": "keyword" },
      "category_path": { "type": "keyword" },
      "brand": { "type": "keyword" },
      "price": { "type": "integer" },
      "original_price": { "type": "integer" },
      "stock": { "type": "integer" },
      "sales": { "type": "integer" },
      "rating": { "type": "float" },
      "images": { "type": "keyword" },
      "description": { "type": "text", "analyzer": "ik_max_word" },
      "attributes": {
        "type": "nested",
        "properties": {
          "name": { "type": "keyword" },
          "value": { "type": "keyword" }
        }
      },
      "tags": { "type": "keyword" },
      "status": { "type": "keyword" },
      "created_at": { "type": "date" },
      "updated_at": { "type": "date" },
      "suggest": {
        "type": "completion",
        "analyzer": "ik_smart"
      }
    }
  }
}
```

**搜索功能实现**：

**1. 基础搜索**
```json
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "苹果手机",
      "fields": ["title^3", "description", "brand"],
      "type": "best_fields",
      "tie_breaker": 0.3
    }
  }
}
```

**2. 智能搜索（支持筛选+排序+高亮）**
```json
GET /products/_search
{
  "from": 0,
  "size": 20,
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "iPhone",
            "fields": ["title^3", "brand", "category"]
          }
        }
      ],
      "filter": [
        { "term": { "status": "active" } },
        { "term": { "category": "手机" } },
        { "terms": { "brand": ["Apple", "华为"] } },
        { "range": { "price": { "gte": 3000, "lte": 8000 } } },
        {
          "nested": {
            "path": "attributes",
            "query": {
              "bool": {
                "must": [
                  { "term": { "attributes.name": "颜色" } },
                  { "term": { "attributes.value": "黑色" } }
                ]
              }
            }
          }
        }
      ]
    }
  },
  "sort": [
    { "sales": "desc" },
    { "rating": "desc" },
    { "_score": "desc" }
  ],
  "highlight": {
    "fields": {
      "title": { "pre_tags": ["<em>"], "post_tags": ["</em>"] }
    }
  },
  "aggs": {
    "brands": {
      "terms": { "field": "brand", "size": 20 }
    },
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          { "to": 1000 },
          { "from": 1000, "to": 3000 },
          { "from": 3000, "to": 5000 },
          { "from": 5000 }
        ]
      }
    }
  }
}
```

**3. 搜索建议（Suggester）**
```json
POST /products/_search
{
  "suggest": {
    "product-suggest": {
      "prefix": "iph",
      "completion": {
        "field": "suggest",
        "fuzzy": { "fuzziness": 2 },
        "size": 10
      }
    },
    "text-suggest": {
      "text": "iphoe",
      "term": {
        "field": "title",
        "suggest_mode": "always"
      }
    }
  }
}
```

**4. 个性化排序**
```json
GET /products/_search
{
  "query": {
    "function_score": {
      "query": { "match": { "title": "手机" } },
      "functions": [
        {
          "field_value_factor": {
            "field": "sales",
            "factor": 1.2,
            "modifier": "log1p",
            "missing": 1
          }
        },
        {
          "field_value_factor": {
            "field": "rating",
            "factor": 1.5
          }
        },
        {
          "gauss": {
            "created_at": {
              "origin": "now",
              "scale": "7d",
              "offset": "1d",
              "decay": 0.5
            }
          }
        }
      ],
      "score_mode": "sum",
      "boost_mode": "multiply"
    }
  }
}
```

**搜索架构优化**：

1. **数据同步方案**：
   - 双写：应用同时写 MySQL 和 ES
   - Canal/Debezium：基于 Binlog 的 CDC 同步
   - 消息队列：异步同步保证最终一致

2. **缓存策略**：
   - 热门搜索词缓存
   - 聚合结果缓存
   - 搜索结果缓存（短时间）

3. **降级策略**：
   - ES 异常时返回 MySQL 搜索结果
   - 展示热门推荐商品
   - 限制搜索条件

---

### 22. ES 与 MySQL 数据同步方案有哪些？

**方案对比**：

| 方案 | 实时性 | 复杂度 | 适用场景 |
|-----|-------|-------|---------|
| 双写 | 高 | 低 | 小规模，简单场景 |
| Canal/Debezium | 高 | 中 | 大规模，复杂业务 |
| 消息队列 | 中 | 中 | 异步场景 |
| Logstash | 低 | 低 | 离线同步 |
| DataX/Seatunnel | 低 | 低 | 批量同步 |

**方案一：应用双写**

```java
@Service
public class ProductService {
    @Autowired
    private ProductRepository mysqlRepo;
    
    @Autowired
    private ElasticsearchRestTemplate esTemplate;
    
    public void saveProduct(Product product) {
        // 1. 写 MySQL
        mysqlRepo.save(product);
        
        // 2. 写 ES
        try {
            esTemplate.save(product);
        } catch (Exception e) {
            // 记录失败日志，后续补偿
            log.error("ES 写入失败", e);
        }
    }
}
```

**优点**：简单直观
**缺点**：数据不一致风险，需要补偿机制

**方案二：Canal（CDC）**

```java
// Canal Client 监听 MySQL binlog
@Component
public class CanalClient {
    @Autowired
    private ElasticsearchRestTemplate esTemplate;
    
    public void start() {
        CanalConnector connector = CanalConnectors.newSingleConnector(
            new InetSocketAddress("127.0.0.1", 11111), 
            "example", "", ""
        );
        
        connector.connect();
        connector.subscribe("products");
        
        while (running) {
            Message message = connector.getWithoutAck(100);
            for (Entry entry : message.getEntries()) {
                processEntry(entry);
            }
            connector.ack(message.getId());
        }
    }
    
    private void processEntry(Entry entry) {
        // 解析 binlog，同步到 ES
        if (entry.getEntryType() == EntryType.ROWDATA) {
            RowChange rowChange = RowChange.parseFrom(entry.getStoreValue());
            for (RowData rowData : rowChange.getRowDatasList()) {
                if (rowChange.getEventType() == EventType.INSERT) {
                    syncToES(rowData.getAfterColumnsList(), "index");
                } else if (rowChange.getEventType() == EventType.UPDATE) {
                    syncToES(rowData.getAfterColumnsList(), "update");
                } else if (rowChange.getEventType() == EventType.DELETE) {
                    deleteFromES(rowData.getBeforeColumnsList());
                }
            }
        }
    }
}
```

**优点**：
- 解耦业务代码和同步逻辑
- 实时性高
- 支持断点续传

**缺点**：
- 需要额外部署 Canal Server
- 增加系统复杂度

**方案三：Debezium（推荐）**

```yaml
# debezium-connector.json
{
  "name": "mysql-products-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "mysql",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "dbz",
    "database.server.id": "184054",
    "database.include.list": "inventory",
    "table.include.list": "inventory.products",
    "topic.prefix": "dbserver1",
    "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
    "schema.history.internal.kafka.topic": "schema-changes.inventory"
  }
}
```

```java
// Kafka Consumer 消费变更事件
@Component
public class ProductSyncConsumer {
    @Autowired
    private ElasticsearchRestTemplate esTemplate;
    
    @KafkaListener(topics = "dbserver1.inventory.products")
    public void handleChange(ConsumerRecord<String, String> record) {
        JsonNode event = objectMapper.readTree(record.value());
        String op = event.get("op").asText();
        
        switch (op) {
            case "c":  // create
            case "u":  // update
                Product product = extractProduct(event.get("after"));
                esTemplate.save(product);
                break;
            case "d":  // delete
                String id = extractId(event.get("before"));
                esTemplate.delete(id, Product.class);
                break;
        }
    }
}
```

**方案四：消息队列（MQ）**

```java
@Service
public class ProductService {
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    public void saveProduct(Product product) {
        // 1. 写 MySQL
        mysqlRepo.save(product);
        
        // 2. 发送消息到 MQ
        ProductEvent event = new ProductEvent("UPDATE", product);
        kafkaTemplate.send("product-sync", event);
    }
}

@Component
public class ProductSyncListener {
    @KafkaListener(topics = "product-sync")
    public void onMessage(ProductEvent event) {
        // 消费消息，同步到 ES
        esTemplate.save(event.getProduct());
    }
}
```

**方案五：Logstash（离线同步）**

```conf
# logstash.conf
input {
  jdbc {
    jdbc_connection_string => "jdbc:mysql://localhost:3306/products"
    jdbc_user => "root"
    jdbc_password => "password"
    jdbc_driver_library => "/path/to/mysql-connector-java.jar"
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    statement => "SELECT * FROM products WHERE updated_at > :sql_last_value"
    use_column_value => true
    tracking_column => "updated_at"
    schedule => "*/5 * * * *"
  }
}

filter {
  mutate {
    remove_field => ["@version", "@timestamp"]
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "products"
    document_id => "%{id}"
  }
}
```

**数据一致性保证**：

1. **最终一致性**：接受短暂不一致，异步补偿
2. **事务消息**：使用 RocketMQ 事务消息保证双写一致
3. **TCC 模式**：Try-Confirm-Cancel 补偿事务
4. **定期校验**：定时对比 MySQL 和 ES 数据差异

**监控和告警**：
- 同步延迟监控
- 数据不一致告警
- 同步失败重试机制
- 全量数据校验

---

### 23. 海量数据（亿级）场景下如何优化 ES？

**架构设计**：

```
┌─────────────────────────────────────────────────────────────┐
│                      数据接入层                              │
│                 Kafka / Pulsar                               │
└────────────────────┬────────────────────────────────────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
┌───────▼─────┐ ┌────▼───┐ ┌──────▼──────┐
│  Ingest     │ │ Data   │ │ Data        │
│  Nodes      │ │ Node 1 │ │ Node 2      │
└──────┬──────┘ └────┬───┘ └──────┬──────┘
       │             │            │
       └─────────────┼────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│                    存储层                                    │
│  Hot Nodes (SSD) → Warm Nodes (HDD) → Cold Nodes (Archive) │
└─────────────────────────────────────────────────────────────┘
```

**优化策略**：

**1. 索引分片策略**
```json
// 按时间分区，每天一个索引
PUT /logs-2024.01.15
{
  "settings": {
    "number_of_shards": 5,      // 根据数据量调整
    "number_of_replicas": 1,
    "refresh_interval": "30s"    // 批量场景增大间隔
  }
}

// 使用 Rollover 自动切分
POST /logs/_rollover
{
  "conditions": {
    "max_age": "1d",
    "max_size": "50GB",
    "max_docs": 100000000
  }
}
```

**2. 索引生命周期管理（ILM）**
```json
PUT _ilm/policy/hot_warm_cold_policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "50GB",
            "max_age": "1d",
            "max_docs": 100000000
          },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 },
          "allocate": {
            "require": { "box_type": "warm" }
          },
          "set_priority": { "priority": 50 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "allocate": {
            "require": { "box_type": "cold" }
          },
          "freeze": {},
          "set_priority": { "priority": 0 }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

**3. 写入优化**
```json
// 批量导入配置
PUT /my_index/_settings
{
  "index": {
    "refresh_interval": "-1",
    "number_of_replicas": 0,
    "translog": {
      "durability": "async",
      "sync_interval": "120s"
    }
  }
}
```

**4. 查询优化**
```json
// 使用 filter cache
GET /logs/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "message": "error" } }
      ],
      "filter": [
        {
          "range": {
            "@timestamp": {
              "gte": "now-7d/d",
              "lte": "now/d"
            }
          }
        }
      ]
    }
  }
}

// 按时间路由
GET /logs-2024.01.*/_search
{
  "query": { "match_all": {} }
}
```

**5. 冷热分离**

```yaml
# elasticsearch.yml
# Hot 节点
node.attr.box_type: hot

# Warm 节点
node.attr.box_type: warm

# Cold 节点
node.attr.box_type: cold
```

**6. 数据预处理**
```json
// 使用 Ingest Pipeline 减少字段
PUT _ingest/pipeline/cleanup_pipeline
{
  "processors": [
    {
      "script": {
        "source": "ctx.remove('unnecessary_field')"
      }
    },
    {
      "gsub": {
        "field": "message",
        "pattern": "\\s+",
        "replacement": " "
      }
    }
  ]
}
```

**7. 查询缓存优化**
```json
// 启用请求缓存
PUT /my_index/_settings
{
  "index.requests.cache.enable": true
}

// 查询时使用 cache
GET /my_index/_search?request_cache=true
{
  "query": { "match_all": {} }
}
```

**8. 聚合优化**
```json
// 预计算聚合（使用 Transforms）
PUT _transform/logs_by_hour
{
  "source": {
    "index": "logs-*"
  },
  "dest": { "index": "logs_hourly_summary" },
  "pivot": {
    "group_by": {
      "timestamp": {
        "date_histogram": {
          "field": "@timestamp",
          "calendar_interval": "1h"
        }
      },
      "level": { "terms": { "field": "level" } }
    },
    "aggregations": {
      "count": { "value_count": { "field": "_id" } },
      "avg_response_time": { "avg": { "field": "response_time" } }
    }
  }
}
```

**9. 分片优化**
```bash
# 避免过多小分片
GET /_cat/indices?v&s=store.size:desc

# 收缩索引（减少分片数）
POST /logs-2024.01.01/_shrink/shrunken-logs
{
  "settings": {
    "index.number_of_replicas": 1,
    "index.number_of_shards": 1
  }
}
```

**10. 监控和扩容**
```bash
# 监控分片分布
GET /_cat/allocation?v

# 监控节点负载
GET /_nodes/stats/os,process,jvm,fs

# 增加节点自动分片迁移
# 配置自动平衡
PUT /_cluster/settings
{
  "transient": {
    "cluster.routing.rebalance.enable": "all",
    "cluster.routing.allocation.allow_rebalance": "indices_all_active"
  }
}
```

**性能指标目标**：
- 写入：> 100k docs/s
- 查询延迟：P99 < 200ms
- 分片大小：10-50GB
- 节点分片数：< 1000
- JVM 堆使用率：< 75%

---

### 24. ES 在日志分析场景的应用？

**典型架构（ELK Stack）**：

```
App/Server → Filebeat/Logstash → Kafka → Logstash → Elasticsearch → Kibana
    │                                              │
    └───────────── 直接发送 ───────────────────────┘
```

**日志收集方案**：

**方案一：Filebeat + Logstash**
```yaml
# filebeat.yml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/app/*.log
  fields:
    service: my-app
    env: production
  multiline.pattern: '^\['
  multiline.negate: true
  multiline.match: after

output.logstash:
  hosts: ["logstash:5044"]
```

```conf
# logstash.conf
input {
  beats {
    port => 5044
  }
}

filter {
  grok {
    match => { 
      "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} %{GREEDYDATA:msg}" 
    }
  }
  
  date {
    match => ["timestamp", "ISO8601"]
    target => "@timestamp"
  }
  
  mutate {
    remove_field => ["timestamp", "beat", "input", "log", "offset"]
  }
}

output {
  elasticsearch {
    hosts => ["es:9200"]
    index => "app-logs-%{+YYYY.MM.dd}"
  }
}
```

**方案二：Fluentd/Fluent Bit**
```xml
# fluent.conf
<source>
  @type tail
  path /var/log/app/*.log
  pos_file /var/log/fluent/app.log.pos
  tag app.logs
  <parse>
    @type json
  </parse>
</source>

<match app.logs>
  @type elasticsearch
  host elasticsearch
  port 9200
  index_name app-logs
  logstash_format true
  logstash_prefix app-logs
</match>
```

**日志索引设计**：

```json
PUT _ilm/policy/logs_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "50GB",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "3d",
        "actions": {
          "shrink": { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "freeze": {}
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}

PUT _index_template/logs_template
{
  "index_patterns": ["app-logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 1,
      "index.lifecycle.name": "logs_policy",
      "index.lifecycle.rollover_alias": "app-logs"
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "level": { "type": "keyword" },
        "service": { "type": "keyword" },
        "host": { "type": "keyword" },
        "message": { "type": "text" },
        "trace_id": { "type": "keyword" },
        "span_id": { "type": "keyword" },
        "duration_ms": { "type": "float" },
        "error": { "type": "object" }
      }
    }
  }
}
```

**日志分析查询**：

**1. 错误日志统计**
```json
GET /app-logs-*/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "level": "ERROR" } },
        { "range": { "@timestamp": { "gte": "now-1h" } } }
      ]
    }
  },
  "aggs": {
    "errors_over_time": {
      "date_histogram": {
        "field": "@timestamp",
        "fixed_interval": "5m"
      }
    },
    "top_errors": {
      "terms": {
        "field": "message.keyword",
        "size": 10
      }
    },
    "by_service": {
      "terms": {
        "field": "service"
      }
    }
  }
}
```

**2. 链路追踪**
```json
GET /app-logs-*/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "trace_id": "abc-123-xyz" } }
      ]
    }
  },
  "sort": [
    { "@timestamp": "asc" }
  ]
}
```

**3. 性能分析**
```json
GET /app-logs-*/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "exists": { "field": "duration_ms" } }
      ]
    }
  },
  "aggs": {
    "latency_percentiles": {
      "percentiles": {
        "field": "duration_ms",
        "percents": [50, 95, 99, 99.9]
      }
    },
    "slow_requests": {
      "filter": {
        "range": { "duration_ms": { "gt": 1000 } }
      },
      "aggs": {
        "by_endpoint": {
          "terms": { "field": "endpoint" }
        }
      }
    }
  }
}
```

**日志分析场景优化**：

1. **索引切分**：按天或按服务切分索引
2. **字段优化**：
   - message 使用 `index: false`（仅存储不索引）
   - 时间戳使用 `doc_values: true`
   - ID 字段使用 `keyword`
3. **数据降采样**：
```json
PUT _transform/logs_rollup
{
  "source": { "index": "app-logs-*" },
  "dest": { "index": "app-logs-hourly" },
  "frequency": "1h",
  "pivot": {
    "group_by": {
      "timestamp": {
        "date_histogram": {
          "field": "@timestamp",
          "fixed_interval": "1h"
        }
      },
      "level": { "terms": { "field": "level" } },
      "service": { "terms": { "field": "service" } }
    },
    "aggregations": {
      "count": { "value_count": { "field": "_id" } },
      "avg_duration": { "avg": { "field": "duration_ms" } }
    }
  }
}
```

4. **冷热分离**：
   - 热节点（7 天内）：SSD，快速查询
   - 温节点（7-30 天）：HDD，聚合分析
   - 冷节点（30 天+）：归档存储

**告警配置**：
```json
PUT _watcher/watch/error_alert
{
  "trigger": {
    "schedule": { "interval": "5m" }
  },
  "input": {
    "search": {
      "request": {
        "indices": ["app-logs-*"],
        "body": {
          "size": 0,
          "query": {
            "bool": {
              "filter": [
                { "term": { "level": "ERROR" } },
                { "range": { "@timestamp": { "gte": "now-5m" } } }
              ]
            }
          }
        }
      }
    }
  },
  "condition": {
    "compare": { "ctx.payload.hits.total": { "gt": 100 } }
  },
  "actions": {
    "send_email": {
      "email": {
        "to": ["ops@company.com"],
        "subject": "Error Rate Alert",
        "body": "Error count: {{ctx.payload.hits.total}}"
      }
    }
  }
}
```

---

## 七、进阶问题

### 25. ES 的 GC 调优经验？

**ES 内存结构**：

```
JVM Heap (建议 31GB 以内)
├── Young Generation (新生代)
│   ├── Eden Space
│   ├── Survivor Space 0
│   └── Survivor Space 1
└── Old Generation (老年代)
```

**GC 类型**：
- **Young GC（Minor GC）**：清理新生代，速度快
- **Old GC（Major GC）**：清理老年代，影响大
- **Full GC**：清理整个堆，停顿时间长，需避免

**GC 问题现象**：
- 查询延迟增加
- 节点掉线
- Master 选举频繁
- 写入吞吐量下降

**GC 调优策略**：

**1. 选择合适的 GC 算法**
```bash
# ES 7.x+ 默认使用 G1GC
# jvm.options
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200

# 对于大内存可考虑 ZGC（JDK 11+）
-XX:+UseZGC
```

**2. 配置合理的堆内存**
```bash
# jvm.options
-Xms31g
-Xmx31g
```
- 设置 Xms = Xmx 避免动态调整
- 不超过 31GB（32GB 以下压缩指针有效）
- 物理内存的 50%，留出 OS Cache

**3. 新生代配置**
```bash
# G1GC 新生代大小自动调整
# 手动设置（不推荐）
-XX:NewRatio=2
```

**4. 监控 GC 情况**
```bash
# 启用 GC 日志
# jvm.options
-Xlog:gc*:file=/var/log/elasticsearch/gc.log::filecount=32,filesize=64m

# 使用 jstat
jstat -gcutil <pid> 1000

# 使用 VisualVM 或 JConsole
```

**5. 减少内存压力**

```yaml
# elasticsearch.yml
# 减少字段缓存
indices.fielddata.cache.size: 30%

# 减少请求缓存
indices.requests.cache.size: 1%

# 限制查询内存
search.default_search_timeout: 30s
indices.query.bool.max_clause_count: 1024
```

**6. 优化查询**
- 避免深度聚合
- 使用 filter context
- 限制返回字段数
- 使用 terminate_after 限制扫描文档数

**7. 数据结构优化**
```json
// 禁用不必要的字段缓存
{
  "mappings": {
    "properties": {
      "large_text": {
        "type": "text",
        "fielddata": false  // 禁用 text 字段的 fielddata
      }
    }
  }
}
```

**GC 日志分析**：

```bash
# 使用 gcviewer 或 gceasy.io 分析
# 关键指标：
# - GC 频率
# - GC 停顿时间
# - 内存分配速率
# - 晋升率（Promotion Rate）
```

**调优检查清单**：
- [ ] 堆内存配置在 50% 物理内存，不超过 31GB
- [ ] Xms = Xmx
- [ ] 使用 G1GC 或 ZGC
- [ ] 监控 GC 停顿时间（目标 < 200ms）
- [ ] 避免 Full GC
- [ ] 优化大聚合查询
- [ ] 检查 fielddata 使用情况
- [ ] 监控节点堆内存使用率

---

### 26. 索引生命周期管理（ILM）如何配置？

**ILM 概念**：
- **Phase（阶段）**：hot、warm、cold、frozen、delete
- **Action（动作）**：rollover、shrink、forcemerge、allocate、freeze、delete
- **Policy（策略）**：定义索引在不同阶段的动作

**完整配置示例**：

**1. 创建 ILM Policy**
```json
PUT _ilm/policy/timeseries_policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_primary_shard_size": "50GB",
            "max_age": "1d",
            "max_docs": 100000000,
            "min_primary_shard_size": "1GB",
            "min_age": "1h",
            "min_docs": 1000000
          },
          "set_priority": {
            "priority": 100
          }
        }
      },
      "warm": {
        "min_age": "2d",
        "actions": {
          "set_priority": {
            "priority": 50
          },
          "shrink": {
            "number_of_shards": 1
          },
          "forcemerge": {
            "max_num_segments": 1
          },
          "allocate": {
            "require": {
              "data": "warm"
            },
            "exclude": {},
            "include": {}
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "set_priority": {
            "priority": 0
          },
          "freeze": {},
          "allocate": {
            "require": {
              "data": "cold"
            }
          }
        }
      },
      "frozen": {
        "min_age": "60d",
        "actions": {
          "frozen": {}
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

**2. 创建 Index Template**
```json
PUT _index_template/timeseries_template
{
  "index_patterns": ["logs-*", "metrics-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1,
      "index.lifecycle.name": "timeseries_policy",
      "index.lifecycle.rollover_alias": "logs",
      "index.routing.allocation.include.data": "hot"
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "message": { "type": "text" }
      }
    }
  },
  "composed_of": [],
  "priority": 500,
  "version": 1
}
```

**3. 创建初始索引**
```json
PUT logs-000001
{
  "aliases": {
    "logs": {
      "is_write_index": true
    }
  }
}
```

**4. 查看 ILM 状态**
```bash
# 查看索引 ILM 状态
GET logs-*/_ilm/explain

# 查看 Policy 详情
GET _ilm/policy/timeseries_policy

# 手动触发 rollover
POST logs/_rollover

# 重试失败的步骤
POST logs-000001/_ilm/retry
```

**常用 ILM Actions**：

| Action | 说明 | 适用阶段 |
|-------|------|---------|
| rollover | 滚动创建新索引 | hot |
| shrink | 收缩分片数 | warm |
| forcemerge | 合并段 | warm |
| allocate | 迁移到指定节点 | warm, cold |
| freeze | 冻结索引（只读） | cold |
| set_priority | 设置恢复优先级 | hot, warm, cold |
| delete | 删除索引 | delete |
| searchable_snapshot | 可搜索快照 | cold, frozen |

**ILM 执行流程**：

```
写入数据 → logs (别名) → logs-000001
                ↓
         触发 rollover 条件
                ↓
         创建 logs-000002
                ↓
     logs-000001 进入 warm 阶段（2天后）
                ↓
     shrink + forcemerge + 迁移到 warm 节点
                ↓
     30天后进入 cold 阶段
                ↓
     freeze + 迁移到 cold 节点
                ↓
     90天后删除
```

**注意事项**：
1. rollover 需要索引有别名且是写入索引
2. 时间计算从索引创建时间开始（不是文档时间）
3. 策略修改只影响新索引
4. 手动执行 rollover 不会重置 min_age 计时

---

### 27. ES 7.x/8.x 相比早期版本有哪些重要变化？

**ES 7.x 主要变化**：

**1. 移除 Type（重大变化）**
```json
# ES 6.x
PUT /my_index/my_type/1
{ "title": "document" }

# ES 7.x+
PUT /my_index/_doc/1
{ "title": "document" }
```
- 7.0 开始每个索引只能有一个 type（默认为 `_doc`）
- 8.0 开始移除 type 参数

**2. 默认主分片数改为 1**
```json
# ES 6.x 默认 5 个主分片
# ES 7.x+ 默认 1 个主分片
PUT /my_index
{
  "settings": {
    "number_of_shards": 1  // 默认
  }
}
```
- Oversharding 问题得到缓解

**3. 集群发现机制变更**
```yaml
# ES 6.x
discovery.zen.minimum_master_nodes: 2

# ES 7.x+
cluster.initial_master_nodes:
  - node-1
  - node-2
  - node-3
discovery.seed_hosts:
  - node-1:9300
  - node-2:9300
```
- Zen Discovery 被 Coordinator 取代
- 更安全，避免脑裂

**4. 评分算法默认改为 BM25**
- 之前默认 TF-IDF
- 7.0 开始默认 BM25

**5. 引入 ILM（Index Lifecycle Management）**
- 内置索引生命周期管理
- 替代 Curator 外部工具

**ES 8.x 主要变化**：

**1. 安全功能默认开启**
```yaml
# 8.0 开始默认启用安全
xpack.security.enabled: true
```
- 默认启用 TLS/SSL
- 默认启用认证

**2. KNN 搜索（向量搜索）**
```json
PUT /my_index
{
  "mappings": {
    "properties": {
      "vector": {
        "type": "dense_vector",
        "dims": 128,
        "index": true,
        "similarity": "cosine"
      }
    }
  }
}

GET /my_index/_knn_search
{
  "knn": {
    "field": "vector",
    "query_vector": [0.1, 0.2, ...],
    "k": 10,
    "num_candidates": 100
  }
}
```
- 内置向量相似度搜索
- 支持 approximate nearest neighbor

**3. 新的 Java API Client**
```java
// ES 7.x Java High Level REST Client（已废弃）

// ES 8.x Java API Client（推荐）
ElasticsearchClient client = new ElasticsearchClient(
    RestClient.builder(new HttpHost("localhost", 9200))
);

SearchResponse<Product> response = client.search(s -> s
    .index("products")
    .query(q -> q
        .match(m -> m
            .field("title")
            .query("手机")
        )
    ),
    Product.class
);
```

**4. 移除 Transport Client**
- 7.0 开始废弃
- 8.0 完全移除
- 使用 Java API Client 或 REST Client

**5. 网络层改进**
- 移除 Transport Protocol
- 完全基于 HTTP
- 内部通信也使用 HTTP

**6. 存储改进**
- 默认使用 _id 作为 routing
- 减少路由查询开销

**7. 性能优化**
- 更快的聚合
- 更高效的存储
- 更好的内存管理

**版本升级建议**：

| 当前版本 | 升级路径 |
|---------|---------|
| 6.x | 6.x → 7.x（最后版本）→ 8.x |
| 7.0-7.16 | 升级到 7.17 → 8.x |
| 7.17+ | 直接升级到 8.x |

**升级前检查清单**：
- [ ] 查看官方 Breaking Changes
- [ ] 使用 Upgrade Assistant 检查兼容性
- [ ] 备份数据
- [ ] 测试环境验证
- [ ] 准备好回滚方案

---

## 总结

### 面试重点

**必问问题**：
1. 倒排索引原理
2. 分片和副本机制
3. 写入流程（refresh/flush/merge）
4. text vs keyword 区别
5. match vs term 区别
6. 深度分页解决方案
7. 写入/查询优化手段

**进阶问题**：
1. 集群架构设计
2. 数据同步方案
3. ILM 配置
4. 性能调优实战
5. 故障排查

**加分项**：
1. 源码阅读经验
2. 大规模集群运维经验
3. 业务场景优化案例
4. 版本升级经验

### 学习路径

1. **基础**：倒排索引、分词、Mapping
2. **进阶**：分布式原理、写入流程、查询DSL
3. **实战**：性能优化、集群运维、业务场景
4. **深入**：源码阅读、JVM 调优、架构设计
