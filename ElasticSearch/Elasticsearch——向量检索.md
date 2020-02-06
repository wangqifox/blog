---
title: Elasticsearch——向量检索
date: 2020/02/06 13:39:00
---

在[基于faiss的分布式特征向量搜索系统][1]一文中，介绍了我是如何借鉴`Elasticsearch`的框架来开发分布式向量检索系统的，它在底层利用了`faiss`强大的向量检索能力。从`7.2`版本开始，`Elasticsearch`提供了实验性的向量检索功能。

本文我们来介绍如何使用`Elasticsearch`来进行向量检索。

<!-- more -->

为了存储向量，需要将字段的数据类型设置为`dense_vector`，设置如下：

```
PUT index1024
{
  "mappings": {
    "properties": {
      "my_vector": {
        "type": "dense_vector",
        "dims": 1024
      },
      "my_text" : {
        "type" : "keyword"
      }
    }
  }
}
```

新建索引时设置`mappings`，`my_vector`字段数据类型指定为`dense_vector`，维度`dim`为向量的长度。注意向量维度不能超过`1024`。

新建索引之后插入数据：

```
PUT index1024/_doc/1
{
  "my_text" : "text1",
  "my_vector" : [0.5, 10, 6, ...]
}

PUT index1024/_doc/2
{
  "my_text" : "text2",
  "my_vector" : [-0.5, 10, 10, ...]
}
```

`Elasticsearch`提供了`cosineSimilarity`（余弦相似度）函数，使用`script_score`来查询向量的相似度：

```
{
  "query": {
    "script_score": {
      "query": {
        "match_all": {}
      },
      "script": {
        "source": "cosineSimilarity(params.queryVector, doc['my_vector'])+1.0",
        "params": {
          "queryVector": [0.5, 10, 6, ...]
        }
      }
    }
  }
}
```

注：`Elasticsearch`不允许分值为负，因此需要对余弦相似度加1。

`Elasticsearch`向量检索的限制：

1. 向量维度的限制，最大支持1024
2. 向量相似度的计算在文档查询完成之后，因此如果候选的文档非常多（比如`match_all`查询，需要遍历所有的文档），计算相似度需要的时间就非常大。我做过实验，500w数据(1024维)检索需要3分钟左右。如果需要检索大量的向量使用`faiss`会快的多。



































[1]: articles/Java/基于faiss的分布式特征向量搜索系统.html




> https://www.elastic.co/cn/blog/text-similarity-search-with-vectors-in-elasticsearch
> https://www.elastic.co/guide/en/elasticsearch/reference/7.3/query-dsl-script-score-query.html#vector-functions
> https://blog.mimacom.com/elastic-cosine-similarity-word-embeddings/

