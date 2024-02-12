## 4.1 索引的定义

### 4.1.2 索引定义实现

新建一个索引，名字为index_00001

```
PUT index_00001
```
#### 编辑一个索引，带有具体内容，主要包括settings,mappings和aliases

注意索引的数据是json格式

```json
PUT hamlet-1
{
  "settings": {
    "number_of_shards": 2,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "cont":{
        "type":"text",
        "analyzer": "ik_max_word",
        "fields": {
          "field":{
            "type":"keyword"
          }
        }
      }
    }
  },
  "aliases": {
    "hamlet": {}
  }
}
```


创建索引

```
PUT news_index
```





修改索引的元数据之   设置

```
PUT news_index/_settings
{
  "number_of_replicas": 3
}
```
```
PUT news_index/_settings
{
  "refresh_interval": "30s"
}
```
```
PUT news_index/_settings
{
  "max_result_window": 50000
}
```

### 4.2.1 新增/创建索引
```
PUT myindex
```

### 4.2.2 删除索引
```
DELETE myindex
```

### 4.2.3 修改索引（为已有的索引添加别名）
```
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "myindex",
        "alias": "myindex_alias"
      }
    }
  ]
}
```

### 4.2.4 查询索引
```
GET myindex
GET myindex/_search
```

### 4.3.3 别名的实现

创建索引的时候指定别名

```
PUT myindex
{
  "aliases": {
    "myindex_alias": {}
  },
  "settings": {
    "refresh_interval": "30s",
    "number_of_shards": 1,
    "number_of_replicas": 0
  }
}
```

```json
#多索引检索
POST visitor_logs_202301,visitor_logs_202302/_search

POST visitor_logs_*/_search

PUT visitor_logs_202301
PUT visitor_logs_202302
```

为已有的索引添加别名

创建的时候未指定别名，后面指定别名

```json
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "visitor_logs_202301",
        "alias": "visitor_logs"
      }
    },
    {
      "add": {
        "index": "visitor_logs_202302",
        "alias": "visitor_logs"
      }
    }
  ]
}
```

获取该索引下的数据信息

```
POST visitor_logs/_search
```



### 4.3.4 别名使用的常见问题

```json
#不能用别名批量插入数据，或报错illegal_argument_exception    reason": "no write index
POST visitor_logs/_bulk
{"index":{}}
{"title":"001"}


#除非创建别名的时候就指定
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "visitor_logs_202302",
        "alias": "visitor_logs",
        "is_write_index": true
      }
    }
  ]
}


#再指定别名批量写入数据就不会报错！
POST visitor_logs/_bulk
{"index":{}}
{"title":"001"}

#查看所有别名
GET _cat/aliases?v


```

## 4.4 索引模板

### 4.4.2 模板定义

#### 创建普通模板

索引模板在template里面包含索引的三个主要部分，也就是aliases，settings，mappings

```
PUT _index_template/template_1   //如果定义普通模板
{
  "index_patterns": [  //要匹配的所有的索引集合
    "te*",
    "bar*"
  ],
  "template": {  //template下面包含了三个主要部分aliases，settings，mappings
    "aliases": {
      "alias1": {}
    },
    "settings": {
      "number_of_shards": 1
    },
    "mappings": {
      "_source": {
        "enabled": false
      },
      "properties": {
        "host_name": {
          "type": "keyword"
        },
        "created_at": {
          "type": "date",
          "format": "EEE MMM dd HH:mm:ss Z yyyy"
        }
      }
    }
  }
}
```

组件模版

组件模板的核心在于将原来的普通模板的mappings,settings,alias等，以组件的方式进行隔离，以便最小化更新模板





#### 创建组件模板

//创建mapping，setting的组件模板

```
//创建mapping的组件模板(业务)

PUT _component_template/component_mapping_template  
{
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": {
          "type": "date"
        },
        "host_name": {
          "type": "keyword"
        },
        "created_at": {
          "type": "date",
          "format": "EEE MMM dd HH:mm:ss Z yyyy"
        }
      }
    }
  }
}


#创建settings的组件模板
PUT _component_template/component_settings_template
{
  "template": {
    "settings": {
      "number_of_shards": 3
    },
     "aliases": {
      "mydata": { }
    }
  }
}




#通过组件模板来创建索引模板
PUT _index_template/mydata_template
{
  "index_patterns": [
    "mydata*"  //匹配哪些索引
  ],
  "priority": 500,
  "composed_of": [
    "component_mapping_template",   //需要用到哪些组件模板，或者说由哪些索引模板组成
    "component_settings_template"
  ],
  "version": 1,
  "_meta": {
    "description": "my custom template"
  }
}

```


### 4.4.3 模板基础操作

```
PUT _index_template/template_1   //和上面最后一步#通过组件模板来创建索引模板一样，记得带json模板组成
DELETE _index_template/template_1   //删除索引模板

GET _index_template/template_1   //查看索引模板
```

### 4.4.4 动态模板实战

将mapping改成integer类型

```
PUT _index_template/sample_dynamic_template
{
  "index_patterns": [
    "sample*"
  ],
  "template": {
    "mappings": {
      "dynamic_templates": [
        {
          "handle_integers": {        //动态模板的名字,自定义
            "match_mapping_type": "long",
            "mapping": {
              "type": "integer"  //将mapping_type为long类型映射为integer类型
            }
          }
        },
        {
          "handle_date": {
            "match": "date_*",   //将date开头的字段映射为date类型
            "mapping": {
              "type": "date"
            }
          }
        }
      ]
    }
  }
}


##测试
DELETE sampleindex
PUT sampleindex/_doc/1  //创建的索引是以sample开头的，刚好匹配上面的模板
{
  "ivalue":123,
  "date_curtime":"1574494620000"
}

##查看是否修改成功
GET sampleindex/_mapping

```


