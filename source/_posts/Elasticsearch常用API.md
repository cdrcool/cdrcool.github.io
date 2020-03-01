---
title: Elasticsearch 常用 API
date: 2020-02-11 17:08:00
categories: Elasticsearch
tags:
    - Elasticsearch
---
## Document
### index
如果 id 不存在，创建新的文档。否则，先删除现有的文档，再创建新的文档，版本号加 1。

```
PUT my_index/_doc/1
{
    "user": "mike",
    "comment": "You konw, for search"
}
```

### create
* 自动创建 id

```
POST my_index/_doc
{
    "user": "mike",
    "comment": "You konw, for search"
}
```

* 指定 id（如果 id 已经存在，会失败）

```
PUT my_index/_create/1
{
    "user": "mike",
    "comment": "You konw, for search"
}
```

### update
文档必须已经存在，更新只会对相应字段做增量修改。

```
POST my_index/_update/1
{
    "doc" : {
        "user": "mike",
        "comment": "You konw, for search"
    }
}
```

### get
找到文档，返回 HTTP 200，否则返回 HTTP 404

```
GET my_index/_doc/1
```

### delete
删除文档。

```
DELETE my_index/_doc/1
```

### bulk
* 支持在一次 API 调用中，对不同的索引进行操作
* 支持 index、create、update、delete 四种操作类型
* 可以在 URI 中指定 index，也可以在请求的 payload 中进行
* 操作中单条操作失败，并不会影响其它操作
* 返回结果中包括了每一条操作执行的结果

```
POST _bulk
{"index": {"_index": "test", "_id": "1"}}
{"field1": "value1"}
{"delete": {"_index": "test", "_id": "2"}}
{"create": {"_index": "test2", "_id": "3"}}
{"field1": "value3"}
{"update": {"_index": "test", "_id": "1"}}
{"doc": {"field2": "value2"}}
```

### mget
批量操作，可以减少网路连接所产生的开销，提高性能。

```
GET _mget
{
    "docs": [
        {
            "_index": "user,
            "_id": 1
        },
        {
            "_index": "comment,
            "_id": 1
        }
    ]
}
```

### msearch
批量查询。

```
POST users/_msearch
{}
{"query": {"match_all" : {}}, "from": 0, size": 10}
{}
{"query": {"match_all" : {}}}
{"index": "twitter2"}
{"query": {"match_all" : {}}}
```

## Search
语法 | 范围
:-: | :-:
/_search | 集群上所有的索引
/index1/_search | index1
/index1,index2/_search | index1 和 index2
/index*/_search | 以 index 开头的索引

### URI Search
在 URL 中使用查询参数。

* q：指定查询语句，使用 Query String Syntax，KV 键值对
* df：默认字段，不指定时，会对所有字段进行查询
* sort：排序
* from/size：用于分页
* profile：查看查询是如何被执行的

eg:
```
GET /movies/_search?q=2012&df=title&sort=tear:desc&from=0&size=10&timeout=1s
{
    "profile": true
}
```

#### 指定字段 vs 范查询
```
q=title:2012 / q=2012
```

#### Term vs Phrase
```
Beautiful Mind => Beautiful OR Mind
"Beautiful Mind" => Beautiful AND Mind (Phrase Query)
```

#### 分组与引号
```
title: (Beautiful Mind) (Term Query)

title: "Beautiful Mind" (Phrase Query)
```

#### 布尔操作
AND / OR /NOT （必须大写）或者 && / || / !

```
title:(matrix NOT reloaded)
```

#### 分组
+ => must
- => must_not

```
title:(+matrix -reloaded)
```

#### 范围查询
[] => 闭区间
{} => 开区间

```
year:{2019 TO 2018]
year:[* TO 2018]
```

#### 算数符号

```
year:>2010
year:(>2010 && <= 2018)
year:(+>2010 +<= 2018)
```

* 通配符查询
? => 1 个字符
* 0 或多个字符

通配符查询效率低，占用内存大，不建议使用。特别是放在最前面。

```
title:mi?d
title:be*
```

#### 正则表达

```
title:[bt]oy
```

#### 模糊匹配与近似查询

```
title:befutifl~1
title:"lord rings"~2
```

### Request Body Search
使用 Elasticsearch 提供的，基于 JSON 格式的更加完备的 DSL。

eg:
```
GET kibana_sample_data_ecommerce/_search
{
    "query": {
        "match_all": {}
    }
}
```

#### 分页
from 从 0 开始，默认返回 10 个结果。获取靠后的翻页成本较高。

```
GET kibana_sample_data_ecommerce/_search
{
    "from": 10,
    ”size": 5,
    "query": {
        "match_all": {}
    }
}
```

#### 排序
最好在“数字型”与“日期型”字段上排序，因为对于多值类型或分析过的字段排序，系统会选一个值，无法得知该值。

```
GET kibana_sample_data_ecommerce/_search
{
    "sort": [{"order_date": "desc}],
    "from": 10,
    ”size": 5,
    "query": {
        "match_all": {}
    }
}
```

#### _source filtering
如果 _source 没有存储，那就只返回匹配的文档的元数据。_source 支持使用通配符，如 _source["name*","desc*"]。

```
GET kibana_sample_data_ecommerce/_search
{
    "_source": ["order_date", "category.keyword"],
    "from": 10,
    ”size": 5,
    "query": {
        "match_all": {}
    }
}
```

#### match

默认 OR 查询：
```
GET /comments/_doc/_search
{
    "query": {
        "match": {
            "comment": "Last Christmas"
        }
    }
}
```

AND 查询：
```
GET /comments/_doc/_search
{
    "query": {
        "match": {
            "comment": "Last Christmas",
            "operator": "AND"
        }
    }
}
```

#### match_phrase

```
GET /comments/_doc/_search
{
    "query": {
        "match_phrase": {
            "comment": "Song Last Chrismas",
            "slop": 1
        }
    }
}
```

#### query_string

单字段：
```
GET /users/_search
{
    "query": {
        "query_string": {
            "default_field": "name",
            "query": "Ruan AND Yiming"
        }
    }
}
```

多字段：
```
GET /users/_search
{
    "query": {
        "query_string": {
            "fields": ["name", "about"],
            "query": "(Ruan AND Yiming) OR (Java AND Elasticsearch)"
        }
    }
}
```

#### simple_query_string
类似 query_string，但是会忽略错误的语法，同时只支持部分查询语法；不支持 AND OR NOT，会当作字符串处理；Term 之间默认的关系是 OR，可以指定 Operator；支持部分逻辑：+ 替代 AND，| 替代 OR，- 替代 NOT。

```
GET /users/_search
{
    "query": {
        "simple_query_string": {
            "fields": ["name"],
            "query": "Ruan -Yiming",
            "default_operator": "AND"
        }
    }
}
```

## Aggregation
### Bucket

```
GET kibana_sample_data_flights/_search
{
    "size": 0,
    "aggs": {
        "flight_dest": {
            "terms": {
                "field": "DestCountry"
            }
        }
    }
}
```

### Metric

```
GET kibana_sample_data_flights/_search
{
    "size": 0,
    "aggs": {
        "flight_dest": {
            "terms": {
                "field": "DestCountry"
            },
            "aggs": {
                "average_price": {
                    "avg": {
                        "field": "AbgTicketPrice"
                    }
                },
                "max_price": {
                    "max": {
                        "field": "AbgTicketPrice"
                    }
                },
                "min_price": {
                    "min": {
                        "field": "AbgTicketPrice"
                    }
                }
            }
        }
    }
}
```

### Nested

```
GET kibana_sample_data_flights/_search
{
    "size": 0,
    "aggs": {
        "flight_dest": {
            "terms": {
                "field": "DestCountry"
            },
            "aggs": {
                "average_price": {
                    "avg": {
                        "field": "AbgTicketPrice"
                    }
                },
                "weather": {
                    "terms": {
                        "field": DestWeather
                    }
                }
            }
        }
    }
}
```


## Mapping
* 查看 Mapping
```
GET users/_mapping
```

* 设置 Mapping
```
PUT users
{
    "mappings": {
        "properties": {
            "firstname": {
                "type": "text"
            },
            "lastname": {
                "type": "text"
            },
        }        
    }
}
```

## Analyzer
### analyze
* 指定 Analyzer 进行测试

```
GET _analyze
{
    "analyzer": "standard",
    "text": "2 running Quik brown-foxes leap over lazy dogs in the summer evening."
}
```

* 指定索引的字段进行测试
```
POST books/_analyze
{
    "field": "title",
    "text": "Mastering Elasticsearch"
}
```

* 自定义分词器进行测试
```
POST _analyze
{
    "tokenizer": "standard",
    "filter": ["lowercase"],
    "text": "Mastering Elasticsearch"
}
```

## Template
### Index Template
Index Template 可以帮助我们设置 Mappings 和 Settings，并按照一定的规则，自动匹配到新创建的索引之上。
* 模板仅在一个索引被创建时，才会产生作用。修改模板不会影响已创建的索引。
* 可以设定多个索引模板，这些设置会被“merge”在一起。
* 可以指定“order”的数值，控制“merging”的过程。（由低到高）
* 应用创建索引时，用户所指定的 Settings 和 Mappings，会覆盖之前模板中的设定。

**设置 Template**
```
PUT /_template/template_test
{
    "index_patterns": ["test*"],
    "order": 1,
    "settings": {
        "number_of_shards": 1,
        "number_of_replicas": 2
    },
    "mappings": {
        "date_detection": false,
        "numeric_detection": true
    }
}
```

** 查看 Template：**
```
GET /_template/template_default

GET /_template/temp*
```

### Dynamic Template
根据 Elasticsearch 识别的数据类型，结合字段名称，来动态设定字段类型。
* Dynamic Template 是定义在某个索引的 Mapping 中
* Template 有一个名称
* 匹配规则是一个数组
* 为匹配到字段设置 Mapping

比如我们可以：
* 所有的字符串类型都设定为 keyword，或者关闭 keyword
* is 开头的字段都设置成 boolean
* long_开头的都设置成 long 类型

```
PUT my_text_index
{
    "mappings": {
        "dynamic_templates": [
            {
                "full_name": {
                    "path_match": "name.*",
                    "path_unmatch": "*.middle",
                    "mapping": {
                        "type": "text",
                        "copy_to": "full_name"
                    }
                }
            },
            {
                "string_as_boolean": {
                    "match_mapping_type": "string",
                    "match": "is*",
                    "mapping": {
                        "type": "boolean"
                    }
                }
            }
        ]
    }
}
```



