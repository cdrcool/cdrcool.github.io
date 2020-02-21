---
title: Elasticsearch 映射
date: 2020-02-12 20:33:00
categories: Elasticsearch
---
Mapping 类似数据库中的 schema 的定义，作用如下：
* 定义索引中的字段的名称
* 定义字段的数据类型，如字符串、数字、布尔...
* 字段倒排索引的相关设置，如 Analyzed or Not Analyzed

## 字段数据类型
* 简单类型
    + Text / Keyword
    + Date
    + Integer
    + Floating
    + Boolean
    + IPv4 & IPv6

* 复杂类型
    + 对象类型
    + 嵌套类型
    
* 特殊类型
    + geo_point /geo_shape
    + percolator

* ~~数组类型~~
Elasticsearch 中不提供专门的数组类型。但是任何字段，都可以包含多个相同类型的数值。

## Dynamic Mapping
Dynamic Mapping，是指在写入文档前无需指定 Mapping，Elasticsearch 会自动根据文档信息推算出字段的类型。这种机制使得我们无需手动定义 Mappings。当然有时候会推算的不对，这时候就需要手动设置 Mapping 了。

类型的自动识别：
JSON 类型 | Elasticsearch 类型
:-: | :-:
字符串 | 日期格式 -> Date；数字 -> float || long（默认关闭）；string -> text & keyword
布尔值 | boolean
浮点数 | float
整数 | long
对象 | Object
数组 | 由第一个非空数值的类型所决定
空值 | 忽略

## 显示指定 Mapping
语法：
```
PUT index_name
{
    "mappings": {
        // define your mappings here
    }
}
```

建议：
为了减少输入的工作量，减少出错概率，可以依照以下步骤设置 Mapping：
1. 创建一个临时的 idex，写入一些样本数据;
2. 通过访问 Mapping API 获得该临时文件的动态 Mapping 定义；
3. 修改后用，使用该配置创建真实索引；
4. 删除临时索引

### 语法
#### index
控制当前字段是否被索引。默认为 true。如果设置成 false，该字段不可被搜索。

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
            "mobile": {
                "type": "text",
                "index": false
            }
        }        
    }
}
```

#### index_options
控制倒排索引记录的内容，有四种不同级别的配置：
* doc：记录 doc id
* freqs：记录 doc id 和 term frequencies
* positions：记录 doc id / term frequencies / term position，用于距离查询，默认设置
* offsets：记录 doc id / term frequencies / term position / character offsets，用于高亮显示

Text 类型默认记录级别为 positions，其它类型默认为 docs。记录内容越多，占用的存储空间越大。

#### null_value
当我们需要对 Null 值实现搜索时用到，只有 keyword 类型支持设置 null_value。

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
            "mobile": {
                "type": "text",
                "null_value": "NULL"
            }
        }        
    }
}
```

#### copy_to
copy_to 将字段的数值拷贝到目标字段，目标字段不出现在 _source 中。

```
PUT users
{
    "mappings": {
        "properties": {
            "firstname": {
                "type": "text",
                "copy_to": "fullName"
            },
            "lastname": {
                "type": "text",
                "copy_to": "fullName"
            }
        }        
    }
}

GET users/_search?q=fullName:(Ruan Yiming)
```

#### 多字段
可以给 Text 字段增加 keyword 字段以实现精确匹配（默认），还可以为字段的搜索和索引指定不同的 analyzer。

```
PUT products
{
    "mappings": {
        "properties": {
            "company": {
                "type": "text",
                "fields": {
                    "keyword": {
                        "type": "keyword",
                        "ignore_above": 256
                    }
                }
            },
            "comment": {
                "type": "text",
                "fields": {
                    "english_comment": {
                        "type": "text",
                        "analyzer": "english",
                        "search_analyzer": "english",
                    }
                }
            }
        }
    }
}
```

## 更新 Mapping
* 新增字段
    + dynamic 为 true：一旦有新增字段的文档写入，Mapping 也同时被更新。
    + dynamic 为 false：Mapping 不会被更新，新增字段的数据无法被索引，但是信息会出现在 _source 中
    + dynamic 为 strict：文档写入失败

* 已有字段：一旦已经有数据写入，就不再支持修改字段定义