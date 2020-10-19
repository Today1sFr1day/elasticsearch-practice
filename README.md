# Elasticsearch学习
参照《Elasticsearch核心技术与实战》
## 文档 Document
ES 是面向文档的，文档是可以被搜索的最小数据单元，被序列化成 JSON 文件保存。类似于 Mongo 中的 Document，区别在于 Mongo 是以 BSON 二进制格式存储**（BSON 其实是 JSON 的超集，包含了 JSON 没有的 date 和 byte 数组类型，例外是没有 JSON 的 number 通用数字类型）**
每个文档也有一个 Unique ID 即 _id，可以手动指定，也可以自动生成，这点同 Mongo 一样  
### 文档元数据
* _index 文档所属索引名
* _type 文档所属类型名
* _id 文档 Unique ID
* _source 文档的原始 JSON 格式数据
* _all 整合所有内容到该字段（***Deprecated***）
* _version 文档的版本信息
* _score 相关性打分

## 索引 Index
索引是文档的容器，是一类文档的集合  
* 逻辑空间：每个索引都有自己的 mappings 定义，用于定义包含的文档字段名及类型信息
* 物理空间：索引中的文档被分散在 Shard 上

索引的 Mappings 和 Settings
* Mappings：定义文档字段类型
* Settings：定义不同的数据分布

### Type
7.0 之前，一个 Index 可以设置多个 Types；6.0 开始，Type 已经被 Deprecated；7.0 开始，一个 Index 只能创建一个 Type(_doc)

## 传统关系型数据库与 Elasticsearch 区别
Elasticsearch 适用于全文检索，对搜索结果进行打分。RDBMS 适用于事务及表 Join 场景

| RDBMS  | Elasticsearch |
| ---    | ---           |
| Table  | Index(Type)   |
| Row    | Document      |
| Column | Field         |
| Schema | Mapping       |
| SQL    | DSL           |

## RESTful API
Elasticsearch 封装了一层 RESTful API，解决不同语言带来的调用差异性，直接通过 HTTP 协议来进行请求和响应  
```
GET index_name # 获取索引信息，包含 mappings 和 settings
GET index_name/_count # 获取索引文档统计信息
GET _cat/nodes # 所有 es 节点信息
GET _cat/shards # 所有 es 分片信息
GET _cat/indices # 所有索引简单信息，包含 health、status、index、uuid、primary、replica、docs.count、store.size...
```

## 分布式特性
Elasticsearch 的分布式架构好处同分布式系统一样，具备高可用性，部分节点宕机不影响集群服务的正常运行，支持水平扩展，可以把数据分散到集群所有节点当中保证数据不丢失。不同的集群根据名字来区分，默认是 "elasticsearch" ，可以通过配置文件修改，也可指定 `-E cluster.name=xxx`  

## 节点

一个节点就是一个 es 实例，可以在一台机器上运行多个实例来达到伪集群效果，但实际生产中不建议这么使用，一是因为受限于性能影响，二是为了达到真正高可用目的。每个节点都有自己的名字，可以通过配置文件修改，也可指定 `-E node.name=xxx` ，每个节点启动后都会被分配一个 UID ，保存在 data 目录下

每个节点启动后都默认是 Master eligible 节点，可以参与 Master 节点选举过程，可以指定 `node.master:false` 禁止，第一个 Master-eligible 节点启动后就会把自己选举为 Master 节点，每个节点保存了集群的状态，只有 Master 节点才可以修改集群的状态信息。集群的状态信息，维护了一个集群的所有节点信息、所有索引以及 mapping 和 setting 信息、分片的路由信息

### Data Node
可以保存数据的节点，只负责保存分片数据，这也是水平扩展的核心所在
### Coordinating Node
负责接收 REST ful 请求，将请求分发到合适的节点，最后将结果合并返回。每个节点都充当了 Coordinating Node 职责
### Hot & Warm Node
不同硬件配置的冷热节点，可以用来实现冷热架构，降低集群部署成本
### Machine Learning Node
负责跑机器学习的 Job ，用来做异常检测发出警告
### Tribe Node
5.3开始使用 Cross Cluster Search ，Tribe Node 连接到不同的 ES 集群，并且支持将这些集群当成一个单独集群处理，即合并多集群效果

### 配置节点类型
开发环境中一个节点可以承担多个角色，但实际生产环境中，一个节点应该只承担一个角色，不只为了单一职责更明确，也是为了性能的考究

default参数：
```
node.master = true
node.data = true
node.ingest = true
coordinating only 默认都是 true ，将其他全部置为 false
node.ml = true，需要 enable x-pack
```

## 分片

实际生产环境中的分片，不能过少也不宜过多，需要提前做好容量规划，主分片一旦规划好就不能再去修改，假设一开始设置 3 个主分片，后续不管数据量如何增加也最多只能分散到 3 个节点上
* 分片数设置过小会导致后续无法进行水平扩展，单个分片数据量过多也会导致数据重新分配带来的耗时，造成性能的损耗
* 分片数设置过大也会影响搜索结果的相关性打分，进而影响统计结果准确性，单个节点过多分片会导致资源浪费，性能也会受到影响。7.0 版本开始就将原本默认的 primary shard 从 5 改为 1，避免了 over sharding 问题

### Primary Shard
解决水平扩展问题，通过主分片可以将数据分布到集群的各个节点上。一个分片是一个运行的 Lucene 实例，主分片数可以在创建索引时指定，后续不允许修改，除非执行 Reindex 操作

### Replica Shard
解决高可用问题，副本分片其实就是主分片的备份，副本数可以动态调整，增加副本数，可以提高系统的高可用性

## 文档 CRUD
| Operation | API                          | Remark                                                               |
| ---       | ---                          | ---                                                                  |
| Create    | POST index_name/_doc         | 不指定 ID，由 ES 自动生成                                             |
| Create    | PUT index_name/_create/100   | 手动指定 ID ，如果 ID 存在则创建失败                                   |
| Read      | GET index_name/_doc/100      |                                                                      |
| Update    | POST index_name/_update/100  | 文档必须存在，对相应字段做增量更新                                      |
| Delate    | DELETE index_name/_doc/100   |                                                                      |
| Index     | PUT index_name/_doc/100      | 如果 ID 不存在，则创建新的文档。否则会删除旧文档再创建新文档，版本号增加   |
| Bulk      | _bulk                        | 批量操作 API ，即使其中一条执行失败也不会影响其他的 API 执行，包含create/update/delete/doc/index等等
| Mget      | _mget or index_name/_mget    | 批量读取 API ，URI 中未指定 index_name 需要在 docs 中指定 _index       |
| Msearch   | index_name/_msearch          |

## Analyzer
分析器（Analyzer）用于进行字符串的分析处理工作，会先标记文本块为适用于倒排索引单独的词（term），然后标准化这些词为标准形式，从而提高这些词的查准率和查全率。包含了三部分工作：
1. 字符串（character filter）通过字符过滤器去除HTML标记，以及特殊字符转义&转and等
2. 分词器（tokenizer）将字符串根据拆分规则比如空格将单词分开
3. 标记过滤（token filters）修改单词，比如大写转小写，去掉停用词（in/the/and）
*Demo：2 running Quick brown-foxes leap over lazy dogs in the summer evening*
+ Standard Analyzer - 按照非字母切分（数字保留，符号被过滤），小写处理
+ Simple Analyzer – 按照非字母切分（数字，符号被过滤），小写处理
+ Stop Analyzer – 小写处理，数字和停用词过滤（the，a，is），小写处理
+ Whitespace Analyzer – 按照空格切分，其他不变
+ Keyword Analyzer – 不分词，直接将输入当作输出
+ Patter Analyzer – 正则表达式，默认 \W+ (非字符分隔)
+ Language – 提供了30多种常见语言的分词器，english复数转单数等等（**bug：lazy变成了lazi**）
+ icu_analyzer - 中文分词器，这个/苹果/不大/好吃

## Search API
### Information Retrieval 信息检索
  + True Positive 命中相关结果
  + False Positive 未命中相关结果
  + False Negative 命中无关结果
  + True Negative 未命中无关结果

1. Precision (查准率) - 尽可能返回较少无关文档 True Positive / 全部返回的结果 (True & False Positives)
2. Recall (查全率) - 尽可能返回较多相关文档 True Positive / 所有应该返回的结果 (True Positives & False Negatives)

### URI Search
```
GET index_name|index_na*|_all/_search?q=field_name:xxx

通过添加profile参数可以看到查询的type、description等信息
{
	"profile":"true"
}
# 指定title字段包含2012的，同q=title:2012，还可以排序分页sort=year:desc&from=0&size=10&timeout=1s，不指定df即为泛查询匹配所有字段
GET /movies/_search?q=2012&df=title

# 泛查询，指定title字段包含Beautiful OR 其他字段包含Mind，DisjunctionMaxQuery
GET /movies/_search?q=title:Beautiful Mind

# PhraseQuery，指定title字段包含Beautiful Mind且顺序一致
GET /movies/_search?q=title:"Beautiful Mind"

# BooleanQuery，title中包含了Beautiful OR Mind，用()进行分组
GET /movies/_search?q=title:(Beautiful Mind)

# 布尔操作符，AND同时包含，NOT不包含后边，
GET /movies/_search?q=title:(Beautiful AND Mind)

# +操作符，相当于title:beautiful +title:mind
GET /movies/_search?q=title:(Beautiful %2BMind)

# 区间查询，[]闭区间，{}开区间，算数符号表示时:>，多条件&&
GET /movies/_search?q=title:beautiful AND year:[2002 TO 2018%7D

# 通配符查询*，b开头的均命中，TermQuery只要title中有一个term是b开头均可
GET /movies/_search?q=title:b*

# 模糊匹配，运用于单词拼写错误情况
GET /movies/_search?q=title:beautifl~1

# title的term包含Lord和Rings就命中，不需要连续包含，区别于"Lord Rings"
GET /movies/_search?q=title:"Lord Rings"~2

```



### Request Body & Query DSL
```
POST index_name/_search
{
	"profile": true,
	"query": {
		"match_all": {}
	}
}

# 这里需要明白一点，就是query里的AND操作符仅仅代表两个Term都包含，Term就是具体field值分词后的结果集，不需要说顺序一致，只要这个field的terms都有就命中

# 查询name字段包含Ruan Yiming的，也可以使用操作符+号，即Ruan +Yiming
POST users/_search
{
  "query": {
    "query_string": {
      "default_field": "name",
      "query": "Ruan AND Yiming"
    }
  }
}

# 查询多列name、about包含Ruan Yiming 或者 Java Elasticsearch的
POST users/_search
{
  "query": {
    "query_string": {
      "fields":["name","about"],
      "query": "(Ruan AND Yiming) OR (Java AND Elasticsearch)"
    }
  }
}

# simple_query_string默认是OR操作符，query中不支持操作符，AND被拆成了and的term，即只要有name term是ruan OR and OR yiming即命中，如果要使用AND，就需要指定default_operator属性为AND，query为Ruan Yiming即可
POST users/_search
{
  "query": {
    "simple_query_string": {
      "query": "Ruan AND Yiming",
      "fields": ["name"]
    }
  }
}
```
