---
title: Elasticsearch 分词与分词器
date: 2020-02-11 17:08:00
categories: Elasticsearch
---
分词（Analysis）是把全文本转换成一系列单词（term/token）的过程。
分词是通过分词器（Analyzer）来实现的。我们可以使用 ELasticsearch 内置的分词器，也可以按需定制化分词器。
**除了在数据写入时转换词条，匹配 Query 语句时也需要用相同的分词器对查询语句进行分析。**

## 分词器组成
分词器由三部分组成：
* Character Filters：针对原始文本处理，例如去除 html
* Tokenizer：按照规则切分为单词
* Token Filter：将切分的单词进行加工：小写、删除 stopwords、增加同义词..

![分词器处理流程示例](/images/elasticsearch/分词器处理流程示例.png)

## Elasticsearch 内置分词器
* Standard Analyzer：默认分词器，按词切分，小写处理
* Simple Analyzer：按照非字母切分（符号被过滤），小写处理
* Stop Analyzer：停用词过滤（the, a, is），小写处理
* Whitespace Analyzer：按照空格切分，不转小写
* Keyword Analyzer：正则表达式，默认 \W+（非字符分割）
* Language：提供了30多种常见语言的分词器
* Customer Analyzer：自定义分词器

## 中文分词
中文分词有它特定的一些难点：
* 中文句子，切分成一个一个词，而不是一个个字。（英文中，单词有自然的空格作为分割）
* 一句中文，在不同的上下文中，有不同的理解。（eg: 这个苹果，不大好吃/这个苹果，不大，好吃）

中文分词器：
* IK
* THULAC