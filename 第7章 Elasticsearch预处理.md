预处理也是以空间换时间



一般我们在写入数据或者从第三方数据源（mysql，oracle,hbase,spark等）导入数据，都是批量导入es的，也就是原始数据长啥样，索引完还长啥样



数据处理大概三个步骤：

1）数据清洗

2）数据集成:多个数据源统一数据存储平台

3）数据转换



预处理管道一般放在ingest节点

原始文档---》》预处理管道---》》索引





常见的预处理器

...





## 7.5 预处理实现

```
PUT _ingest/pipeline/my-pipeline-id
{
  "version": 1,
  "processors": [ ... ]
}
```

my-pipeline-id代表预处理管道的名称，集群唯一

processors每个预处理过程，可以之地昂多个预处理管道

version版本，非必须



## 7.6 预处理实战案例

### 7.6.1 字符串切分预处理案例
```
####定义索引
PUT my_index_0701
{
  "mappings": {
    "properties": {
      "mid": {
        "type": "keyword"
      }
    }
  }
}

####批量写入样例数据
POST my_index_0701/_bulk
{"index":{"_id":1}}
{"mid":"C12345"}
{"index":{"_id":2}}
{"mid":"C12456"}
{"index":{"_id":3}}
{"mid":"C31268"}



####预处理，提取前两个字符
PUT _ingest/pipeline/split_mid
{
  "processors": [
    {
      "script": {
        "lang": "painless",
        "source": "ctx.mid_prefix = ctx.mid.substring(0,2)"
      }
    }
  ]
}

####借助预处理执行更新操作
POST my_index_0701/_update_by_query?pipeline=split_mid
{
  "query": {
    "match_all": {}
  }
}
####执行检索，验证是否成功
GET my_index_0701/_search
```

### 7.6.2 字符串转JSON格式案例

```
#### 创建索引并写入数据，采用默认分词
POST my_index_0702/_doc/1
{
    "headers":{
        "userInfo":[
            "{  \"password\": \"test\",\n  \"username\": \"zy\"}"
        ]
    }
}

####查看Mapping，以便后续对比
GET my_index_0702/_mapping

            "userInfo": {
              "type": "text",
              "fields": {
                "keyword": {
                  "type": "keyword",

####创建json预处理器
PUT _ingest/pipeline/json_builder
{
  "processors": [
    {
      "json": {
        "field": "headers.userInfo",   //要处理的字段是headers.userInfo
        "target_field": "headers.userjson"  //目标
      }
    }
  ]
}

####批量更新操作
POST my_index_0702/_update_by_query?pipeline=json_builder   //用上面的已经创建好的json预处理器

####再次查看Mapping，和前面对比发现不同，已经多了个userjson
GET my_index_0702/_mapping

GET my_index_0702/_search


            "userInfo": [
              """{  "password": "test",
  "username": "zy"}"""
            ],
            "userjson": [
              {
                "password": "test",
                "username": "zy"
              }

```

### 7.6.3 list列表操作案例

如果想在一个list后面的每个值都加一个字符，

比如将{"tag":["a","b","c"]}变成{"tag":["a2",,b2","c2"]},应该如何实现

```
####创建索引并指定Mapping字段
PUT my_index_0703
{
  "mappings": {
    "properties": {
      "tag": {
        "type": "keyword"
      }
    }
  }
}


##创建数据
####批量写入数据
POST my_index_0703/_bulk
{"index":{"_id":1}}
{"tag":["a","b","c"]}



####预处理脚本实现添加字符，核心原理是借助script处理器循环遍历数组，实现每个字段的再填充
PUT _ingest/pipeline/add_builder
{
  "processors": [
    {
      "script": {
        "lang": "painless",
        "source": """
for (int i=0; i < ctx.tag.length;i++) {
      ctx.tag[i]=ctx.tag[i]+"2";
    } 
"""
      }
    }
  ]
}



####实现更新操作，引用上面的pipeline
POST my_index_0703/_update_by_query?pipeline=add_builder

####执行检索
POST my_index_0703/_search

#看到已经加进去了
POST my_index_0703/_update_by_query?pipeline=add_builder

```

### 7.6.4 enrich案例（不常用）

```
PUT /_enrich/policy/data-policy
{
  "match": {
    "indices": "index_test_b",
    "match_field": "field_a",
    "enrich_fields": [
      "author",
      "publisher"
    ]
  }
}
```

```
#### 1)创建索引
PUT my_index_0704
{
  "mappings": {
    "properties": {
      "field_a": {
        "type": "keyword"
      },
      "title": {
        "type": "keyword"
      },
      "publish_time": {
        "type": "date"
      }
    }
  }
}
####批量写入数据
POST my_index_0704/_bulk
{"index":{"_id":1}}
{"field_a":"aaa","title":"elasticsearch in action","publish_time":"2017-07-01T00:00:00"}
 
#### 创建索引 
PUT my_index_0705
{
  "mappings": {
    "properties": {
      "field_a": {
        "type": "keyword"
      },
      "author": {
        "type": "keyword"
      },
      "publisher": {
        "type": "keyword"
      }
    }
  }
}
####批量写入数据，和my_index_0704存在相同字段“field_a”。
POST my_index_0705/_bulk
{"index":{"_id":1}}
{"field_a":"aaa","author":"jerry","publisher":"Tsinghua"}
```

```
#### 2）创建data-policy。
DELETE _enrich/policy/data-policy
PUT /_enrich/policy/data-policy
{
  "match": {
    "indices": "index_test_b",
    "match_field": "field_a",
    "enrich_fields": ["author","publisher"]
  }
}
```
```
####3）执行data-policy。
POST /_enrich/policy/data-policy/_execute
```

```
####4）创建pipeline。
DELETE /_ingest/pipeline/data_lookup
PUT /_ingest/pipeline/data_lookup
{
  "processors": [
    {
      "enrich": {
        "policy_name": "data-policy",
        "field": "field_a",
        "target_field": "field_from_bindex",
        "max_matches": "1"
      }
    },
    {
      "append": {
        "field": "author",
        "value": "{{field_from_bindex.author}}"
      }
    },
    {
      "append": {
        "field": "publisher",
        "value": "{{field_from_bindex.publisher}}"
      }
    },
    {
      "remove": {
        "field": "field_from_bindex"
      }
    }
  ]
}
```

```
#### 5）建立reindex索引。
DELETE my_index_0706
POST _reindex
{
  "source": {
    "index": "my_index_0704"
  },
  "dest": {
    "index": "my_index_0706",
    "pipeline": "data_lookup"
  }
}
```
```
#### 6)检索结果。
POST my_index_0706/_search

```

### 7.6.5 预处理实战常见问题
```
####  1）创建索引环节指定pipeline，如下。
PUT my_index_0707
{
  "settings": {
    "index.default_pipeline": "split_mid"
  }
}

####  2）创建模板环节指定pipeline。
PUT _index_template/template_0708
{
  "index_patterns": [
    "myindex*"
  ],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "index.default_pipeline": "split_mid"
    }
  }
}

#### 3）reindex环节添加pipeline。
POST _reindex
{
  "source": {
    "index": "my_index_0701"
  },
  "dest": {
    "index": "my_index_0709",
    "pipeline": "split_mid"
  }
}

#### 4）update环节指定pipeline。
POST my_index_0702/_update_by_query?pipeline=json_builder
```

