# 第5章 Elasticsearch映射

## 5.1 映射定义

### 5.1.3 数据类型

#### 数组定义

```
PUT my_index_0501/_doc/1
{
  "media_array": [
    "新闻",
    "论坛",
    "博客",
    "电子报"
  ],
  "users_array": [
    {
      "name": "Mary",
      "age": 12
    },
    {
      "name": "John",
      "age": 10
    }
  ],
  "size_array": [
    0,
    50,
    100
  ]
}
```

es中，允许用户对单个文档设置多个不同的数据类型，以满足不同的查询需求

```
#### 多字段类型Multi_fields
PUT my_index_0502
{
  "mappings": {
    "properties": {
      "cont": {    //类型名字为cont，properties下面接的是名字
        "type": "text",
        "analyzer": "english",
        "fields": {
          "keyword": {
            "type": "keyword"
          },
          "stand": {
            "type": "text",
            "analyzer": "standard"
          }
        }
      }
    }
  }
}
```

```
#### Multi_fields 应用
PUT my_index_0503
{
  "mappings": {
    "properties": {
      "title": {   //类型名字为cont，properties下面接的是名字
        "type": "text",
        "analyzer": "ik_max_word",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      }
    }
  }
}
```

### 5.1.4 映射类型

#### 动态映射

es中是动态映射，这和mysql不一样的

动态映射的核心是在自动检测字段类型后添加新字段

哪些字段支持动态监测   bool,float,long.Object,Array,date,string,除此之外的其他的都不支持

```
#### 创建索引
PUT my_index_0504/_bulk
{"index":{"_id":1}}
{"cont":"Each document has metadata associated with it","visit_count":35,"publish_time":"2023-05-20T18:00:00"}
```

动态匹配的弊端是可能会出现字段类型不匹配

比如把日期应该是date类型，却写成了keyword类型

```
#### 字段匹配不正确
DELETE my_index_0505
PUT my_index_0505/_doc/1
{
  "create_date": "2020-12-26 12:00:00"
}
GET my_index_0505/_mapping
```

结局动态映射可能出现的问题

方法是先提前设置匹配规则

```
#### 提前设置匹配规则
DELETE my_index_0505
PUT my_index_0505
{
  "mappings": {
    "dynamic_date_formats": ["yyyy-MM-dd HH:mm:ss"]
  }
}
PUT my_index_0505/_doc/1
{
  "create_date": "2020-12-26 12:00:00"
}
GET my_index_0505/_mapping
```

#### 静态映射

```
#### 创建索引，指定dynamic:false
PUT my_index_0506
{
  "mappings": {
    "dynamic": false,   //动态映射设置为false则是静态映射
    "properties": {
      "user": {
        "properties": {
          "name": {
            "type": "text"
          },
          "social_networks": {
            "dynamic": true,
            "properties": {}
          }
        }
      }
    }
  }
}
```

```
#### 数据可以写入成功
PUT my_index_0506/_doc/1
{
  "cont": "Each document has metadata associated"
}

#### 检索不能找回数据，核心原因：cont是未映射字段
POST my_index_0506/_search
{
  "profile": true, 
  "query": {
    "match": {
      "cont": "document"
    }
  }
}

#### 可以返回结果
GET my_index_0506/_doc/1

#### Mapping中并没有cont
GET my_index_0506/_mapping
```





设置为strict后，是不允许写入未定义过的字段

```
#### dynamic设置为strict
DELETE my_index_0507
PUT my_index_0507
{
  "mappings": {
    "dynamic": "strict", 
    "properties": {
      "user": { 
        "properties": {
          "name": {
            "type": "text"
          },
          "social_networks": { 
            "dynamic": true,
            "properties": {}
          }
        }
      }
    }
  }
}

#### 数据写入失败，设置为strict后，是不允许写入未定义过的字段
PUT my_index_0507/_doc/1
{
  "cont": "Each document has metadata associated"
}
```

### 5.1.5 实战：Mapping创建后还可以更新吗

大多数只能reindex或者删除

小部分字段可以更新

```
PUT my_index_0508
{
  "mappings": {
    "properties": {
      "name": {
        "properties": {
          "first": {
            "type": "text"
          }
        }
      },
      "user_id": {
        "type": "keyword"
      }
    }
  }
}
```

```
#### 如下Mapping是可以更新成功的。
PUT my_index_0508/_mapping
{
  "properties": {
    "name": {
      "properties": {
        "first": {
          "type": "text",
          "fields": {
            "field": {
              "type": "keyword"
            }
          }
        },
        "last": {
          "type": "text"
        }
      }
    },
    "user_id": {
      "type": "keyword",
      "ignore_above": 100
    }
  }
}
```





## 5.2 Nested类型及应用

### 5.2.1 Nested类型定义

嵌套数据类型

相关的可以写在同一个文档中，比如文章和评论

简单来说，nested是升级版的object类型，它允许对象以彼此独立的方式进行索引

```
#### 数据构造
PUT my_index_0509/_bulk
{"index":{"_id":1}}
{"title":"Invest Money","body":"Please start investing money as soon...","tags":["money","invest"],"published_on":"18 Oct 2017","comments":[{"name":"William","age":34,"rating":8,"comment":"Nice article..","commented_on":"30 Nov 2017"},{"name":"John","age":38,"rating":9,"comment":"I started investing after reading this.","commented_on":"25 Nov 2017"},{"name":"Smith","age":33,"rating":7,"comment":"Very good post","commented_on":"20 Nov 2017"}]}
```

可以看到john的年纪是38，我们检索34岁的john试试

```
#### 执行检索
POST my_index_0509/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "comments.name": "John"
          }
        },
        {
          "match": {
            "comments.age": 34
          }
        }
      ]
    }
  }
}
```

竟然也有数据，这是为甚麽呢？

这是因为mapping的默认类型是object类型，会丢失关系



修改comments的类型为nested（嵌套类型）试试

```
PUT my_index_0510
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text"
      },
      "body": {
        "type": "text"
      },
      "tags": {
        "type": "keyword"
      },
      "published_on": {
        "type": "keyword"
      },
      "comments": {
        "type": "nested",
        "properties": {
          "name": {
            "type": "text"
          },
          "comment": {
            "type": "text"
          },
          "age": {
            "type": "short"
          },
          "rating": {
            "type": "short"
          },
          "commented_on": {
            "type": "text"
          }
        }
      }
    }
  }
}
```

导入构造的数据

```
PUT my_index_0510/_bulk
{"index":{"_id":1}}
{"title":"Invest Money","body":"Please start investing money as soon...","tags":["money","invest"],"published_on":"18 Oct 2017","comments":[{"name":"William","age":34,"rating":8,"comment":"Nice article..","commented_on":"30 Nov 2017"},{"name":"John","age":38,"rating":9,"comment":"I started investing after reading this.","commented_on":"25 Nov 2017"},{"name":"Smith","age":33,"rating":7,"comment":"Very good post","commented_on":"20 Nov 2017"}]}
```

这时候执行检索，由于数嵌套，会查不到不存在的数据

```
#### 执行检索，不会召回结果。
POST my_index_0510/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "nested": {
            "path": "comments",
            "query": {
              "bool": {
                "must": [
                  {
                    "match": {
                      "comments.name": "john"
                    }
                  },
                  {
                    "match": {
                      "comments.age": 34
                    }
                  }
                ]
              }
            }
          }
        }
      ]
    }
  }
}
```

改成正确的38岁的时候，可以正常查询到数据，这就是nested（嵌套类型）的作用



每个评论都在内部存储为单独的隐藏文档



### 5.2.2  Nested类型的操作（增删改查）

```
#### Nested 增
POST my_index_0510/_doc/2
{
  "title": "Hero",
  "body": "Hero test body...",
  "tags": [
    "Heros",
    "happy"
  ],
  "published_on": "6 Oct 2018",
  "comments": [
    {
      "name": "steve",
      "age": 24,
      "rating": 18,
      "comment": "Nice article..",
      "commented_on": "3 Nov 2018"
    }
  ]
}
```

```
#### Nested 删
POST my_index_0510/_update/1
{
  "script": {
    "lang": "painless",
    "source": "ctx._source.comments.removeIf(it -> it.name == 'John');"
  }
}
```

```
#### 改
POST my_index_0510/_update/2
{
  "script": {
    "source": "for(e in ctx._source.comments){if (e.name == 'steve') {e.age = 25; e.comment= 'very very good article...';}}"
  }
}
```

```
#### 查
POST my_index_0510/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "nested": {
            "path": "comments",
            "query": {
              "bool": {
                "must": [
                  {
                    "match": {
                      "comments.name": "William"
                    }
                  },
                  {
                    "match": {
                      "comments.age": 34
                    }
                  }
                ]
              }
            }
          }
        }
      ]
    }
  }
}
```

```
#### 聚合，查找最小年龄的评论者的年纪
POST my_index_0510/_search
{
  "size": 0,
  "aggs": {
    "comm_aggs": {
      "nested": {
        "path": "comments"
      },
      "aggs": {
        "min_age": {
          "min": {
            "field": "comments.age"
          }
        }
      }
    }
  }
}
```

在处理业务索引时，使用嵌套对象对于数据的组织和查询非常重要，我们在查询前确认，数据是否为嵌套类型为nested，否则可能范围无效的结果，影响判断



## 5.3 Join类型及应用（类似mysql的多表关联，不推荐使用）

### 5.3.3 Join类型实战

```
#### Join定义
PUT my_index_0511
{
  "mappings": {
    "properties": {
      "my_join_field": {
        "type": "join",
        "relations": {
          "question": [ //父子结构，question是父
            "answer"
          ]
        }
      }
    }
  }
}
```

```
#### 写入父文档
POST my_index_0511/_doc/1
{
  "text": "This is a question",
  "my_join_field": "question" 
}

POST my_index_0511/_doc/2
{
  "text": "This is another question",
  "my_join_field": "question"
}
```

```
#### 写入子文档
PUT my_index_0511/_doc/3?routing=1&refresh      //路由值是强制的，父文件和子文件必须在相同的分片上建立索引
{
  "text": "This is an answer",
  "my_join_field": {
    "name": "answer",
    "parent": "1"   //指定与父文档的id为1
  }
}

PUT my_index_0511/_doc/4?routing=1&refresh
{
  "text": "This is another answer",
  "my_join_field": {
    "name": "answer",
    "parent": "1"
  }
}
```

```
####Join 检索
POST my_index_0511/_search
{
  "query": {
    "match_all": {}
  }
}
````

```
#### 通过父文档查询子文档
POST my_index_0511/_search
{
  "query": {
    "has_parent": {
      "parent_type": "question",
      "query": {
        "match": {
          "text": "This is"
        }
      }
    }
  }
}
```

```
#### 通过子文档查询父文档
POST my_index_0511/_search
{
  "query": {
    "has_child": {
      "type": "answer",
      "query": {
        "match": {
          "text": "This is question"
        }
      }
    }
  }
}
```

```
#### Join 聚合操作
POST my_index_05611/_search
{
  "query": {
    "parent_id": { 
      "type": "answer",
      "id": "1"
    }
  },
  "aggs": {
    "parents": {
      "terms": {
        "field": "my_join_field#question", 
        "size": 10
      }
    }
  }
}
```

### 5.3.4 Join一对多实战

```
#### 一对多Join类型索引定义
PUT my_index_0512
{
  "mappings": {
    "properties": {
      "my_join_field": {
        "type": "join",
        "relations": {
          "question": [
            "answer",
            "comment"
          ]
        }
      }
    }
  }
}
```

```
#### 多对多Join类型索引定义
PUT my_index_0513
{
  "mappings": {
    "properties": {
      "my_join_field": {
        "type": "join",
        "relations": {
          "question": [
            "answer",
            "comment"
          ],
          "answer": "vote"
        }
      }
    }
  }
}
```

```
#### 孙子文档导入数据
PUT my_index_0513/_doc/3?routing=1&refresh
{
  "text": "This is a vote",
  "my_join_field": {
    "name": "vote",
    "parent": "2" 
  }
}
```

### 5.3.5 小结

```
#### 创建1个索引包含2个Join类型。
PUT my_index_0514
{
  "mappings": {
    "properties": {
      "my_join_field": {
        "type": "join",
        "relations": {
          "question": [
            "answer"
          ]
        }
      },
      "my_join_field_02": {
        "type": "join",
        "relations": {
          "question_02": [
            "answer_02"
          ]
        }
      }
    }
  }
}
```

## 5.4 Flattened类型及应用（解决字段膨胀问题）

将dynamic设置为false或者strict不是普适的解决方案

true过于松散，strict过于严格，false也不太行



flattened类型的作用是将整个json对象及其nestes字段索引为单个关键字（keyword类型），以减少字段总数

### 5.4.1 Elasticsarch字段膨胀问题

```
#### 误操作，将检索语句写成插入语句。
PUT my_index_0515/_doc/1
{
  "query": {
    "bool": {
      "must": [
        {
          "nested": {
            "path": "comments",
            "query": {
              "bool": {
                "must": [
                  {
                    "match": {
                      "comments.name": "William"
                    }
                  },
                  {
                    "match": {
                      "comments.age": 34
                    }
                  }
                ]
              }
            }
          }
        }
      ]
    }
  }
}
```

### 5.4.3 Flattened类型解决的根本问题（设置上限）
```
PUT my_index_0516
{
  "settings": {
    "index.mapping.total_fields.limit": 2000
  }
}
```

### 5.4.4 Flattened类型实战解读

```
PUT my_index_0517
{
  "mappings": {
    "properties": {
      "host": {   //将host字段设置为flattened
        "type": "flattened"
      }
    }
  }
}
```

基于flattened插入数据

```
#### 写入数据
PUT my_index_0517/_doc/1
{
  "message": "[5592:1:0309/123054.737712:ERROR:child_process_sandbox_support_impl_linux.cc.",
  "fileset": {
    "name": "syslog"
  },
  "process": {
    "name": "org.gnome.Shell.desktop",
    "pid": 3383
  },
  "@timestamp": "2025-03-09T18:00:54.000+05:30",
  "host": {   //hots字段已经被设置成了"flattened"
    "hostname": "bionic",  
    "name": "bionic"
  }
}
```

```
#### 更新Flattened字段，添加数据
POST my_index_0517/_update/1
{
  "doc": {
    "host": {
      "osVersion": "Bionic Beaver",
      "osArchitecture": "x86_64"
    }
  }
}

#更新后查看映射结构，会发现没有字段扩增也不会出现mapping爆炸
GET my_index_0517

        "host": {
          "type": "flattened"
        },
```

下面两种都会召回数据

```
#### 精准匹配term检索
POST my_index_0517/_search
{
  "query": {
    "term": {
      "host": "Bionic Beaver"
    }
  }
}

POST my_index_0617/_search
{
  "query": {
    "term": {
      "host.osVersion": "Bionic Beaver"
    }
  }
}
```



下面区分大小写

下面这个不会返回数据，原因是es未对该字段进行分词，会区分大小写

```
#### match全文类型检索
POST my_index_0517/_search
{
  "query": {
    "match": {
      "host.osVersion": "bionic beaver"
    }
  }
}
 
POST my_index_0517/_search
{
  "query": {
    "match": {
      "host.osVersion": "Beaver"
    }
  }
}
```

### 5.4.5flattened的几个缺陷

1.无法涉及运算

2.不支持高亮

3.



## 5.6 内部数据结构解读

### 5.6.1倒排索引的特点

在索引时创建，

序列化到磁盘，

全文搜索快，

不适合做排序，

默认开启







### 5.6.3 doc_values正排索引

需要做排序的时候用正排索引，text字段不支持正排索引，但是可以用fielddata解决

```
PUT my_index_0618
{
  "mappings": {
    "properties": {
      "title": {
        "type": "keyword",
        "doc_values": false
      }
    }
  }
}
```

### 5.6.4 fielddata

（默认禁用，因为很昂贵）

text字段不支持正排索引，但是可以用fielddata解决

只在text字段支持

```
PUT my_index_0619
{
  "mappings": {
    "properties": {
      "body":{
        "type":"text",
        "analyzer": "standard",
        "fielddata": true   //指定
      }
    }
  }
}



POST my_index_0619/_bulk   //批量写入数据
{"index":{"_id":1}}
{"body":"The quick brown fox jumped over the lazy dog"}
{"index":{"_id":2}}
{"body":"Quick brown foxes leap over lazy dogs in summer"}



GET my_index_0619/_search
{
  "size": 0,
  "query": {
    "match": {
      "body": "brown"
    }
  },
  "aggs": {
    "popular_terms": {
      "terms": {
        "field": "body"
      }
    }
  }
}
```

### 5.6.5 _source字段解读
```
PUT my_index_0620
{
  "mappings": {
    "_source": {
      "enabled": false
    }
  }
}
```

### 5.6.6 store字段解读

```
#### store使用举例
PUT my_index_0621
{
  "mappings": {
    "_source": {
      "enabled": false
    },
    "properties": {
      "title": {
        "type": "text",
        "store": true
      },
      "date": {
        "type": "date",
        "store": true
      },
      "content": {
        "type": "text"
      }
    }
  }
}
PUT my_index_0621/_doc/1
{
  "title":   "Some short title",
  "date":    "2021-01-01",
  "content": "A very long content field..."
}
#### 不能召回数据
GET my_index_0621/_search

#### 可以召回数据
GET my_index_0621/_search
{
  "stored_fields": [ "title", "date" ] 
}
```

## 5.7 详解null value

```
#### 创建索引
PUT  my_index_0622
{
  "mappings": {
    "properties": {
      "status_code": {
        "type": "keyword"
      },
      "title": {
        "type": "text"
      }
    }
  }
}


#### 批量写入数据
PUT  my_index_0622/_bulk
{"index":{"_id":1}}
{"status_code":null,"title":"just test"}
{"index":{"_id":2}}
{"status_code":"","title":"just test"}
{"index":{"_id":3}}
{"status_code":[],"title":"just test"}

#### 执行检索
POST  my_index_0622/_search
{
  "query": {
    "term": {
      "status_code": null
    }
  }
}
```

### 5.7.1 null_value的含义

```
#### 创建索引
PUT my_index_0523
{
  "mappings": {
    "properties": {
      "status_code": {
        "type":       "keyword",
        "null_value": "NULL"
      }
    }
  }
}

#### 批量写入数据
PUT my_index_0523/_bulk
{"index":{"_id":1}}
{"status_code":null}
{"index":{"_id":2}}
{"status_code":[]}
{"index":{"_id":3}}
{"status_code":"NULL"}

#### 执行检索
POST my_index_0523/_search
{
  "query": {
    "term": {
      "status_code": "NULL"
    }
  }
}
```

### 5.7.2 null_value使用的注意事项

```
PUT my_index_0524
{
  "mappings": {
    "properties": {
      "status_code": {
        "type": "keyword"
      },
      "title": {
        "type": "long",
        "null_value": "NULL"
      }
    }
  }
}
```

### 5.7.3 支持null_value的核心字段

```
PUT my_index_0525
{
  "mappings": {
    "properties": {
      "status_code": {
        "type": "keyword"
      },
      "title": {
        "type": "text",
        "null_value": "NULL"
      }
    }
  }
}
```

```
PUT my_index_0526
{
  "mappings": {
    "properties": {
      "status_code": {
        "type": "keyword"
      },
      "title": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "null_value": "NULL"
          }
        }
      }
    }
  }
}
```