## 搜索匹配

1、搜索发帖日期为2017-01-01，或者帖子ID为XHDK-A-1293-#fJ3的帖子，同时要求帖子的发帖日期绝对不为2017-01-02

select *
from forum.article
where (post_date='2017-01-01' or article_id='XHDK-A-1293-#fJ3')
and post_date!='2017-01-02'

GET /forum/article/_search
{
  "query": {
​    "constant_score": {
​      "filter": {
​        "bool": {
​          "should": [
​            {"term": { "postDate": "2017-01-01" }},
​            {"term": {"articleID": "XHDK-A-1293-#fJ3"}}
​          ],
​          "must_not": {
​            "term": {
​              "postDate": "2017-01-02"
​            }
​          }
​        }
​      }
​    }
  }
}

must，should，must_not，filter：必须匹配，可以匹配其中任意一个即可，必须不匹配

2、搜索帖子ID为XHDK-A-1293-#fJ3，或者是帖子ID为JODL-X-1937-#pV7而且发帖日期为2017-01-01的帖子

select *
from forum.article
where article_id='XHDK-A-1293-#fJ3'
or (article_id='JODL-X-1937-#pV7' and post_date='2017-01-01')

GET /forum/article/_search 
{
  "query": {
​    "constant_score": {
​      "filter": {
​        "bool": {
​          "should": [
​            {
​              "term": {
​                "articleID": "XHDK-A-1293-#fJ3"
​              }
​            },
​            {
​              "bool": {
​                "must": [
​                  {
​                    "term":{
​                      "articleID": "JODL-X-1937-#pV7"
​                    }
​                  },
​                  {
​                    "term": {
​                      "postDate": "2017-01-01"
​                    }
​                  }
​                ]
​              }
​            }
​          ]
​        }
​      }
​    }
  }
}

3、梳理学到的知识点

（1）bool：must，must_not，should，组合多个过滤条件
（2）bool可以嵌套
（3）相当于SQL中的多个and条件：当你把搜索语法学好了以后，基本可以实现部分常用的sql语法对应的功能



## 9.基于boost的细粒度搜索条件权重控制

需求：搜索标题中包含java的帖子，同时呢，如果标题中包含hadoop或elasticsearch就优先搜索出来，同时呢，如果一个帖子包含java hadoop，一个帖子包含java elasticsearch，包含hadoop的帖子要比elasticsearch优先搜索出来

知识点，搜索条件的权重，boost，可以将某个搜索条件的权重加大，此时当匹配这个搜索条件和匹配另一个搜索条件的document，计算relevance score时，匹配权重更大的搜索条件的document，relevance score会更高，当然也就会优先被返回回来

默认情况下，搜索条件的权重都是一样的，都是1

```GET /forum/article/_search 
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "title": "blog"
          }
        }
      ],
      "should": [
        {
          "match": {
            "title": {
              "query": "java"
            }
          }
        },
        {
          "match": {
            "title": {
              "query": "hadoop"
            }
          }
        },
        {
          "match": {
            "title": {
              "query": "elasticsearch"
            }
          }
        },
        {
          "match": {
            "title": {
              "query": "spark",
              "boost": 5
            }
          }
        }
      ]
    }
  }
}
```

##  10. 在多shard场景下 relevance score 不准确

>  如果一个一个index 存储在多个shard上, 可能搜索结果不准确

当一个搜索title包含java的请求到shard上了来时, (这个shard存储了很多document, 其中有10的title包含java).

计算relevance score (TF/IDF):   

+  在一个document的title中, 字符'java' 出现的次数

+ 在所有的document的title中

+ 此document的title的长度



**默认情况下就在shard local本地计算IDF** ,字符'java' 出现的次数,(此时是10次). 如果此时在另外一个shad中只有一个document title包含 字符串'java', 计算shard local IDF 就会分数很高,相关度很高.   



**如何解决这个问题?**

1. 生产环境下,数据量尽可能实现均匀分配

   数据量很大的话，其实一般情况下，在概率学的背景下，es都是在多个shard中均匀路由数据的，路由的时候根据_id，负载均衡
   比如说有10个document，title都包含java，一共有5个shard，那么在概率学的背景下，如果负载均衡的话，其实每个shard都应该有2个doc，title包含java
   如果说数据分布均匀的话，其实就没有刚才说的那个问题了



2. 测试环境下, 将索引的primary shard 设置为1 

如果说只有一个shard，那么当然，所有的document都在这个shard里面，就没有这个问题了



3. 测试环境下, 搜索附带 search_type=dfs_query_then_fetch 参数, 将local IDF 取出计算 globalIDF 

计算一个doc的相关度分数的时候，就会将所有shard对的local IDF计算一下，获取出来，在本地进行global IDF分数的计算，会将所有shard的doc作为上下文来进行计算，也能确保准确性。但是production生产环境下，不推荐这个参数，因为性能很差。



##  11. 基于dis_max 实现best filed 策略进行多字段搜索

### 1.  添加数据

**为帖子数据增加标题字段**

```python
def update_content_by_bulk_11_1():
    update_by_bulk_body = [
        {"update": {"_id": "1"}}
        {"doc": {"content": "i like to write best elasticsearch article"}},
        {"update": {"_id": "2"}},
        {"doc": {"content": "i think java is the best programming language"}},
        {"update": {"_id": "3"}},
        {"doc": {"content": "i am only an elasticsearch beginner"}},
        {"update": {"_id": "4"}},
        {"doc": {"content": "elasticsearch and hadoop are all very good solution, i am a beginner"}},
        {"update": {"_id": "5"}},
        {"doc": {"content": "spark is best big data solution based on scala ,an programming language similar to java"}},

    ]
    result = es.bulk(index=forum_index, doc_type=article_type, body=update_by_bulk_body)
    print(json.dumps(result, indent=4))
```



### 2. 搜索title或content中包含java或solution的帖子

```python
# 2. 搜索title或content中包含java或solution的帖子
def search_11_2():
    query_body = {
        query: {
            bool: {
                should: [
                    {
                        match: {"title": "java solution"},
                    },
                    {
                        match: {"content": "java solution"}
                    }
                ]
            }
        }
    }
    result = es.search(forum_index, article_type, body=query_body)
    print(json.dumps(result, indent=4))


search_11_2()
```

结果:

```json
/code/python/test_elasticsearch/venv/bin/python /code/python/test_elasticsearch/src/lean.py
{
    "took": 3,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 4,
        "max_score": 0.95348084,
        "hits": [
            {
                "_index": "forum",
                "_type": "article",
                "_id": "2",
                "_score": 0.95348084,
                "_source": {
                    "articleID": "KDKE-B-9947-#kL5",
                    "userID": 1,
                    "hidden": false,
                    "postDate": "2017-01-02",
                    "tag": [
                        "java"
                    ],
                    "tag_cnt": 1,
                    "view_cnt": 50,
                    "title": "this is java blog",
                    "content": "i think java is the best programming language"
                }
            },
            {
                "_index": "forum",
                "_type": "article",
                "_id": "4",
                "_score": 0.8092568,
                "_source": {
                    "articleID": "QQPX-R-3956-#aD8",
                    "userID": 2,
                    "hidden": false,
                    "postDate": "2017-01-02",
                    "tag": [
                        "java",
                        "elasticsearch"
                    ],
                    "tag_cnt": 2,
                    "view_cnt": 80,
                    "title": "this is java, elasticsearch, hadoop blog",
                    "content": "elasticsearch and hadoop are all very good solution, i am a beginner"
                }
            },
            {
                "_index": "forum",
                "_type": "article",
                "_id": "5",
                "_score": 0.5753642,
                "_source": {
                    "articleID": "DHJK-B-1395-#Ky5",
                    "userID": 3,
                    "hidden": false,
                    "postDate": "2017-03-01",
                    "tag": [
                        "elasticsearch"
                    ],
                    "tag_cnt": 1,
                    "view_cnt": 10,
                    "title": "this is spark blog",
                    "content": "spark is best big data solution based on scala ,an programming language similar to java"
                }
            },
            {
                "_index": "forum",
                "_type": "article",
                "_id": "1",
                "_score": 0.2876821,
                "_source": {
                    "articleID": "XHDK-A-1293-#fJ3",
                    "userID": 1,
                    "hidden": false,
                    "postDate": "2017-01-01",
                    "tag": [
                        "java",
                        "hadoop"
                    ],
                    "tag_cnt": 2,
                    "view_cnt": 30,
                    "title": "this is java and elasticsearch blog",
                    "content": "i like to write best elasticsearch article"
                }
            }
        ]
    }
}
```

### 3.结果分析

> 期望的是doc5，结果是doc2,doc4排在了前面

计算每个document的relevance score：每个query的分数，乘以matched query数量，除以总query数量

**算doc4的分数**

{ "match": { "title": "java solution" }}，针对doc4，是有一个分数的
{ "match": { "content":  "java solution" }}，针对doc4，也是有一个分数的

所以是两个分数加起来，比如说，1.1 + 1.2 = 2.3
matched query数量 = 2
总query数量 = 2

2.3 * 2 / 2 = 2.3

**算doc5的分数**

{ "match": { "title": "java solution" }}，针对doc5，是没有分数的
{ "match": { "content":  "java solution" }}，针对doc5，是有一个分数的

所以说，只有一个query是有分数的，比如2.3
matched query数量 = 1
总query数量 = 2

2.3 * 1 / 2 = 1.15

doc5的分数 = 1.15 < doc4的分数 = 2.3



### 4. best fields 策略 dis_max

搜索到的结果应该是某一个field 中匹配到尽可能多的关键词, 不是尽可能多的field匹配到少的关键词排在前面

dis_max 语法 :  直接取多个query 中分数最高的那一个query即可. 

例如: 

{ "match": { "title": "java solution" }}，针对doc4，是有一个分数的，1.1
{ "match": { "content":  "java solution" }}，针对doc4，也是有一个分数的，1.2
取最大分数，1.2

{ "match": { "title": "java solution" }}，针对doc5，是没有分数的
{ "match": { "content":  "java solution" }}，针对doc5，是有一个分数的，2.3
取最大分数，2.3

然后doc4的分数 = 1.2 < doc5的分数 = 2.3，所以doc5就可以排在更前面的地方，符合我们的需要

代码示例 :

```python
def search_11_3():
    query_body = {
        query: {
            dis_max: {
                queries: [
                    {
                        match: {"title": "solution java"}
                    },
                    {
                        match: {"content": "solution java solution "}
                    }
                ]
            }
        }
    }
    result = es.search(forum_index, article_type, body=query_body)
    print(json.dumps(result, indent=4))
```



## 12. 基于 tie_breaker 参数优化dis_max搜索效果

### 1、搜索title或content中包含java beginner的帖子

```python
def search_12_1():
    query_body = {
        query: {
            dis_max: {
                queries: [
                    {
                        match: {"title": "beginner java"}
                    },
                    {
                        match: {"content": "beginner java"}
                    }
                ]
            }
        }
    }
    result = es.search(forum_index, article_type, body=query_body)
    print(json.dumps(result, indent=4))
```

可能出现的场景: 

1. doc1中title 中包含java, content 中不包含java beginner 任何一个关键词
2. doc2中title 中不包含任何关键字, content中包含 beginner 
3. doc3中title 中包含java, content中包含beginner 

最终搜索出来的结果是可能 doc1和doc2 排在doc3 前面 

dis_max 只是取**分数最高**的分数,  完全不考虑其他的query的分数. 可能导致匹配多个条件的doc分数较低. 

tie_breaker 可以将其他的query的分数也加入计算,  将其他的query的分数乘以tie_breaker,然后综合在一起计算. 

加入tie_breaker 后除了最高的query参加计算, 其他的query也将参加计算 . 

 tie_breaker 值在0到1之间的小数  

```python
def search_12_2():
    query_body = {
        query: {
            dis_max: {
                queries: [
                    {
                        match: {"title": "beginner java"}
                    },
                    {
                        match: {"content": "beginner java"}
                    }
                ],
                "tie_breaker": 0.3
            }
        }
    }
    result = es.search(forum_index, article_type, body=query_body)
    print(json.dumps(result, indent=4))
```



##  13.基于mutil_match语法实现dis_max+tie_breaker

minimum_should_match， 去长尾，long tail
长尾，比如你搜索5个关键词，但是很多结果是只匹配1个关键词的，其实跟你想要的结果相差甚远，这些结果就是长尾
minimum_should_match，控制搜索结果的精准度，只有匹配一定数量的关键词的数据，才能返回

```python
def search_13_1():
    query_body = {
        query: {
            multi_match: {
                query: "java solution",
                fields: ["title^2", "content"],
                "type": "best_fields",
                minimum_should_match: "50%",
                tie_breaker : 0.3
            }
        }
    }
    result = es.search(forum_index, article_type, body=query_body)
    print(json.dumps(result, indent=4))
```



## 14. 基于multi_match+most fiels策略进行multi-field搜索

从best-field 策略切换成most-field策略 

+ best-filed策略 : 将某一个filed尽可能多的匹配关键词的doc优先返回
+ most-field策略: 将尽可能多匹配到filed的doc优先返回  

**most-field 策略实例:**

```python
def search_14_2():
    query_body = {
        query: {
            multi_match: {
                query: "learning courses",
                "type": most_fields,
                "fields": ["sub_title", "sub_title.std"]

            }
        }
    }
    result = es.search(forum_index, article_type, body=query_body)
    print(json.dumps(result, indent=4))
```

由于 用的是 English analyzer 分词器, 会将单词还原成最基本的形态 

learning --> lean

learned --> lean 

courses --> course

sub_title : learning coureses --> lean curse 

所以  'learned a lot of course' 排在了 'learning more courses' 的前面

**解决方案:**

给sub_title 添加一个field  std 使用 standard作为分词器, 使用两个field 一起搜. 

```
POST /forum/_mapping/article
{
  "properties": {
      "sub_title": { 
          "type":     "string",
          "analyzer": "english",
          "fields": {
              "std":   { 
                  "type":     "string",
                  "analyzer": "standard"
              }
          }
      }
  }
}
```



**most-field与best_fields的区别: **

1. best_fields，是对多个field进行搜索，挑选某个field匹配度最高的那个分数，同时在多个query最高分相同的情况下，在一定程度上考虑其他query的分数。简单来说，你对多个field进行搜索，就想搜索到某一个field尽可能包含更多关键字的数据

   优点：通过best_fields策略，以及综合考虑其他field，还有minimum_should_match支持，可以尽可能精准地将匹配的结果推送到最前面
   缺点：除了那些精准匹配的结果，其他差不多大的结果，排序结果不是太均匀，没有什么区分度了

   实际的例子：百度之类的搜索引擎，最匹配的到最前面，但是其他的就没什么区分度了

2. most_fields，综合多个field一起进行搜索，尽可能多地让所有field的query参与到总分数的计算中来，此时就会是个大杂烩，出现类似best_fields案例最开始的那个结果，结果不一定精准，某一个document的一个field包含更多的关键字，但是因为其他document有更多field匹配到了，所以排在了前面；所以需要建立类似sub_title.std这样的field，尽可能让某一个field精准匹配query string，贡献更高的分数，将更精准匹配的数据排到前面

   优点：将尽可能匹配更多field的结果推送到最前面，整个排序结果是比较均匀的
   缺点：可能那些精准匹配的结果，无法推送到最前面



## 15. 使用 most_fields策略进行 corss_field search的弊端

**corss-fields搜索: ** 一个唯一标识夸多个filed. 例如 人名,地址. 



**添加作者名:**

```python
def update_sub_author_name_by_bulk_15_1():
    update_by_bulk_body = [
        {"update": {"_id": "1"}},
        {"doc": {"author_first_name": "Peter", "author_last_name": "Smith"}},
        {"update": {"_id": "2"}},
        {"doc": {"author_first_name": "Smith", "author_last_name": "Williams"}},
        {"update": {"_id": "3"}},
        {"doc": {"author_first_name": "Jack", "author_last_name": "Ma"}},
        {"update": {"_id": "4"}},
        {"doc": {"author_first_name": "Robbin", "author_last_name": "Li"}},
        {"update": {"_id": "5"}},
        {"doc": {"author_first_name": "Tonny", "author_last_name": "Peter Smith"}},
    ]
    result = es.bulk(index=forum_index, doc_type=article_type, body=update_by_bulk_body)
    print(json.dumps(result, indent=4))
```

 **使用 most-filed 搜索人名:**

```json
def search_15_2():
    query_body = {
        query: {
            multi_match: {
                query: "Peter Smith",
                fields: ["author_first_name", "author_last_name"],
                "type": "most_fields"
            }
        }
    }
    result = es.search(forum_index, article_type, body=query_body)
    print(json.dumps(result, indent=4))
```

返回结果:



```
{
    "took": 2,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 3,
        "max_score": 0.6931472,
        "hits": [
            {
                "_index": "forum",
                "_type": "article",
                "_id": "2",
                "_score": 0.6931472,
                "_source": {
                    "articleID": "KDKE-B-9947-#kL5",
                    "userID": 1,
                    "hidden": false,
                    "postDate": "2017-01-02",
                    "tag": [
                        "java"
                    ],
                    "tag_cnt": 1,
                    "view_cnt": 50,
                    "title": "this is java blog",
                    "content": "i think java is the best programming language",
                    "sub_title": "learned a lot of course",
                    "author_first_name": "Smith",
                    "author_last_name": "Williams"
                }
            },
            {
                "_index": "forum",
                "_type": "article",
                "_id": "5",
                "_score": 0.5753642,
                "_source": {
                    "articleID": "DHJK-B-1395-#Ky5",
                    "userID": 3,
                    "hidden": false,
                    "postDate": "2017-03-01",
                    "tag": [
                        "elasticsearch"
                    ],
                    "tag_cnt": 1,
                    "view_cnt": 10,
                    "title": "this is spark blog",
                    "content": "spark is best big data solution based on scala ,an programming language similar to java",
                    "sub_title": "haha, hello world",
                    "author_first_name": "Tonny",
                    "author_last_name": "Peter Smith"
                }
            },
            {
                "_index": "forum",
                "_type": "article",
                "_id": "1",
                "_score": 0.5753642,
                "_source": {
                    "articleID": "XHDK-A-1293-#fJ3",
                    "userID": 1,
                    "hidden": false,
                    "postDate": "2017-01-01",
                    "tag": [
                        "java",
                        "hadoop"
                    ],
                    "tag_cnt": 2,
                    "view_cnt": 30,
                    "title": "this is java and elasticsearch blog",
                    "content": "i like to write best elasticsearch article",
                    "sub_title": "learning more courses",
                    "author_first_name": "Peter",
                    "author_last_name": "Smith"
                }
            }
        ]
    }
}
```



使用了most-fields 策略 导致最匹配的文档排在了最后. 



## 使用copy_to定制组合field解决corss-fields 搜索弊端



### 用copy_to，将多个field组合成一个field

添加字段mapping :

```
PUT /forum/_mapping/article
{
  "properties": {
      "new_author_first_name": {
          "type":     "string",
          "copy_to":  "new_author_full_name" 
      },
      "new_author_last_name": {
          "type":     "string",
          "copy_to":  "new_author_full_name" 
      },
      "new_author_full_name": {
          "type":     "string"
      }
  }
}
```

添加字段:





















 





