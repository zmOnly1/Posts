### ES学习大纲

https://github.com/geektime-geekbang/geektime-ELK

#### 1、Kibana展示

#### 2、IndexTemplate和DynamicTemplate

##### 1、自定义template

1. Index Templates - 帮助你设定Mappings和Settings，并按照一定的规则自动匹配到新创建的索引之上

   - 可以设定多个索引模板，这些设置会被merge在一起

   - 可以指定order的数值，控制merging的顺序，merging顺序从order低到高，创建索引时指定Settings和Mapping会覆盖模板

   - Demo

     ```
     PUT /_template/template_test
     {
         "index_patterns" : ["test*"],
         "order" : 1,
         "settings" : {
         	"number_of_shards": 1,
             "number_of_replicas" : 2
         },
         "mappings" : {
         	"date_detection": false,
         	"numeric_detection": true
         }
     }
     
     PUT testtemplate/_doc/1
     {
     	"someNumber":"1",
     	"someDate":"2019/01/01"
     }
     ```

##### 2、DynamicTemplate

1. Elasticsearch自动识别数据类型

   - 所有字符串类型都设定成keyword，或者关闭keyword字段

   - is开头的字段都设置成boolean

   - long_开头的都设置成long类型

   - Demo

     ```
     PUT my_index/_doc/1
     {
       "firstName":"Ruan",
       "isVIP":"true"
     }
     
     GET my_index/_mapping
     DELETE my_index
     PUT my_index
     {
       "mappings": {
         "dynamic_templates": [
                 {
             "strings_as_boolean": {
               "match_mapping_type":   "string",
               "match":"is*",
               "mapping": {
                 "type": "boolean"
               }
             }
           },
           {
             "strings_as_keywords": {
               "match_mapping_type":   "string",
               "mapping": {
                 "type": "keyword"
               }
             }
           }
         ]
       }
     }
     
     PUT my_index
     {
       "mappings": {
         "dynamic_templates": [
           {
             "full_name": {
               "path_match":   "name.*",
               "path_unmatch": "*.middle",
               "mapping": {
                 "type":       "text",
                 "copy_to":    "full_name"
               }
             }
           }
         ]
       }
     }
     
     PUT my_index/_doc/1
     {
       "name": {
         "first":  "John",
         "middle": "Winston",
         "last":   "Lennon"
       }
     }
     ```

     

#### 3、DynamicMapping和Mapping和字段类型

##### 1、DynamicMapping

1. Mapping类似数据库中的schema的定义

2. 字段类型

   - 简单类型：Text/Keyword、Date、Integer/Floating、Boolean、IPv4&IPv6
   - 复杂类型 - 对象和嵌套对象
   - 特殊类型：geo_point & geo_shape / percolator

3. Dynamic Mapping在写入文档时，如果索引不存在，自动创建索引，根据文档信息（基于JSON）推算字段类型，有时候会推算不对。

   - 根据JSON类型推断：

     | JSON类型 | Elasticsearch类型                                            |
     | -------- | ------------------------------------------------------------ |
     | 字符串   | 匹配日期格式，设置成Date<br/><br />配置数字设置为float或者long，该选项默认关闭<br/><br />设置为Text，并且增加keyword子字段 |
     | 布尔值   | boolean                                                      |
     | 浮点数   | float                                                        |
     | 整数     | long                                                         |
     | 对象     | Object                                                       |
     | 数组     | 由第一个非空数值的类型所决定                                 |
     | 空值     | 忽略                                                         |

4. 更改Mapping的字段类型

   1. 新增字段

      - Dynamic设为true时，一旦有新增字段，Mapping同时被更新
      - Dynamic设为false，Mapping不会被更新，新增字段的数据无法被索引，但是信息会出现在_source中
      - Dynamic设置成Strict，文档写入失败

   2. 对已有字段，一旦已经有数据写入，就不再支持修改字段定义，Lucene实现的倒排索引，一旦生成后，就不允许修改

   3. 如果希望改变字段类型，必须Reindex API，重建索引

      1. 原因如果修改了字段的数据类型，会导致已被索引的数据无法被搜索

      2. |               | "true" | "flse" | "strict" |
         | ------------- | ------ | ------ | -------- |
         | 文档可索引    | YES    | YES    | NO       |
         | 字段可索引    | YES    | NO     | NO       |
         | Mapping被更新 | YES    | NO     | NO       |

   4. ```
      PUT dynamic_mapping_test/_mapping
      {
      "dynamic": "strict"/true/false
      }
      ```

   5. Demo

      ```
      #查看 Dynamic
      GET mapping_test/_mapping
      
      #默认Mapping支持dynamic，写入的文档中加入新的字段
      PUT dynamic_mapping_test/_doc/1
      {
        "newField":"someValue"
      }
      
      #该字段可以被搜索，数据也在_source中出现
      POST dynamic_mapping_test/_search
      {
        "query":{
          "match":{
            "newField":"someValue"
          }
        }
      }
      
      #修改为dynamic false
      PUT dynamic_mapping_test/_mapping
      {
        "dynamic": false
      }
      
      #新增 anotherField
      PUT dynamic_mapping_test/_doc/10
      {
        "anotherField":"someValue"
      }
      
      #该字段不可以被搜索，因为dynamic已经被设置为false
      POST dynamic_mapping_test/_search
      {
        "query":{
          "match":{
            "anotherField":"someValue"
          }
        }
      }
      
      GET dynamic_mapping_test/_doc/10
      
      #修改为strict
      PUT dynamic_mapping_test/_mapping
      {
        "dynamic": "strict"
      }
      
      #写入数据出错，HTTP Code 400
      PUT dynamic_mapping_test/_doc/12
      {
        "lastField":"value"
      }
      
      DELETE dynamic_mapping_test
      
      ```

##### 2、自定义Mapping

1. 控制当前字段是否被索引，index:false，默认为true，如果设置成false，该字段不可被搜索

   ```
   DELETE users
   PUT users
   {
     "mappings": {
       "properties": {
         "firstName": {
           "type": "text",
           "copy_to": "fullName"
         },
         "lastName": {
           "type": "text",
           "copy_to": "fullName"
         },
         "mobile": {
           "type": "text",
           "index": false,
           "null_value": "NULL"
         },
         "bio": {
           "type": "text",
           "index_options": "offsets"
         }
       }
     }
   }
   ```

2. 四种不同级别的Index Options配置，可以控制倒排索引记录的内容

   - docs - 记录doc id
   - freqs - 记录doc id/ term frequencies
   - positions - 记录doc id/ term frequencies /term position
   - offsets - 记录doc id/ term frequencies /term position / character offsets

3. Text类型默认记录positions，其他默认为docs

4. 记录内容越多，占用存储空间越大

5. null_value

   - 需要对Null值实现搜索

   - 只有Keyword类型支持设定Null_value

   - ```
     GET users/_search
     {
       "query": {
         "match": {
           "mobile": "NULL"
         }
       }
     }
     ```

     

6. copy_to设置

   - copy_to的目标字段不出现在_source中
   - `GET _users/_search?q=fullName:(Ruan Yiming)`

7. Demo

   ```
   #设置 index 为 false
   DELETE users
   PUT users
   {
       "mappings" : {
         "properties" : {
           "firstName" : {
             "type" : "text"
           },
           "lastName" : {
             "type" : "text"
           },
           "mobile" : {
             "type" : "text",
             "index": false
           }
         }
       }
   }
   
   PUT users/_doc/1
   {
     "firstName":"Ruan",
     "lastName": "Yiming",
     "mobile": "12345678"
   }
   
   POST /users/_search
   {
     "query": {
       "match": {
         "mobile":"12345678"
       }
     }
   }
   
   #设定Null_value
   
   DELETE users
   PUT users
   {
       "mappings" : {
         "properties" : {
           "firstName" : {
             "type" : "text"
           },
           "lastName" : {
             "type" : "text"
           },
           "mobile" : {
             "type" : "keyword",
             "null_value": "NULL"
           }
   
         }
       }
   }
   
   PUT users/_doc/1
   {
     "firstName":"Ruan",
     "lastName": "Yiming",
     "mobile": null
   }
   
   PUT users/_doc/2
   {
     "firstName":"Ruan2",
     "lastName": "Yiming2"
   }
   
   GET users/_search
   {
     "query": {
       "match": {
         "mobile":"NULL"
       }
     }
   }
   
   #设置 Copy to
   DELETE users
   PUT users
   {
     "mappings": {
       "properties": {
         "firstName":{
           "type": "text",
           "copy_to": "fullName"
         },
         "lastName":{
           "type": "text",
           "copy_to": "fullName"
         }
       }
     }
   }
   PUT users/_doc/1
   {
     "firstName":"Ruan",
     "lastName": "Yiming"
   }
   
   GET users/_search?q=fullName:(Ruan Yiming)
   
   POST users/_search
   {
     "query": {
       "match": {
          "fullName":{
           "query": "Ruan Yiming",
           "operator": "and"
         }
       }
     }
   }
   
   #数组类型
   PUT users/_doc/1
   {
     "name":"onebird",
     "interests":"reading"
   }
   
   PUT users/_doc/1
   {
     "name":"twobirds",
     "interests":["reading","music"]
   }
   
   POST users/_search
   {
     "query": {
   		"match_all": {}
   	}
   }
   
   GET users/_mapping
   ```

##### 3、多字段类型

1. 增加一个子字段

   ```
   PUT products
   {
       "mappings" : {
         "properties" : {
           "company" : {
             "type" : "text",
             "fields": {
               "keyword":{
                 "type": "keyword",
                 "ignore_above": 256
               }
             }
           },
           "comment" : {
             "type" : "text",
             "fields": {
               "english_comment":{
                 "type": "text",
                 "analyzer": "english",
                 "search_analyzer": "english"
               }
             }
           }
         }
       }
   }
   ```

2. 分词：

   ```
   POST _analyze
   {
     "tokenizer": "standard",
     "char_filter": [
         {
           "type" : "mapping",
           "mappings" : [ "- => _"]
         }
       ],
     "text": "123-456, I-test! test-990 650-555-1234"
   }
   ```

3. settings设置自定义分词

   ```
   // 创建索引指定分词器
   PUT my_inx
   {
     "settings": {
       "analysis": {
         "analyzer": {
           "my_custom_analyzer": {
             "type": "custom",
             "char_filter": [
               "emoticons"
               ],
               "tokenizer": "punctuation",
               "filter": [
                 "lowercase",
                 "english_stop"
                 ]
           }
         },
         "tokenizer": {
           "punctuation": {
             "type": "pattern",
             "pattern": "[ .,!?]"
           }
         },
         "char_filter": {
           "emoticons": {
             "type": "mapping",
             "mappings": [
               ":) => _happy_",
               ":( => _sad_"
               ]
           }
         },
         "filter": {
           "english_stop": {
             "type": "stop",
             "stopwords": "_englist_"
           }
         }
       }
     }
     
   POST my_inx/_analyze
   {
     "analyzer": "my_custom_analyzer",
     "text": "I'm a :) persion, and you?"
   }
    
   ```

   

#### 4、搜索：Term查询、Phrase查询

##### 1、Elasticsearch的搜索，会分成两阶段进行

- 第一阶段 - Query
  - 用户发出搜索请求到ES节点。节点收到请求后，会以Coordinating节点身份，在6个主副分片中随机选择3个分片，发送查询请求 
  - 被选中的分片执行查询，进行排序。然后，每个分片都会返回From+Size个排序后的文档Id和排序值给Coordinating节点
- 第二阶段 - Fetch 
  - Coordinating Node会将Query阶段，从每个分片获取的排序后的文档Id列表，重新进行排序。选取From到From+Size个文档Id
  - 以multi get请求的方式，到相应的分片获取详细的文档数据 
- Query Then Fetch潜在的问题
  - 每个分片上 需要查的文档个数=from+size
  - 最终协调节点需要处理：number_of_shard*(from+size)
  - 深度分页
- Lucene Index
  - 在Lucene中，单个倒排所以文件被称为Segment。Segment是自包含的，不可变更。多个Segments汇总在一起，称为Lucene的Index，其对应的就是ES中的Shard
  - 当有新文档写入时，会生成新的Segment，查询时会同时查询所有Segments，并且对结果汇总。Lucene中有一个文件，用来记录所有Segments信息，叫做Commit Point
  - 删除的文档信息，保存在".del"文件中
- Refresh
  - Index Document =>Index Buffer, 写入Index Buffer一定后会写入Segment，
  - 将Index Buffer写入Segment的过程叫Refresh。Refresh不执行fsync操作
  - Refresh频率：默认1秒发生一次，可通过index.refresh_interval配置。Refresh后数据就可以被搜索到了
  - 如果系统有大量数据写入，就会产生很多Segment
  - Index Buffer被沾满时，会触发Refresh，默认值是JVM的10%
- Transaction Log
  - Index Document写入Index Buffer的同时会写入Transaction log来保障数据不丢失。Transaction Log默认落盘，每个分片有一个Tansaction Log
- Flush
  - 调用Refresh，Index Buffer清空并且Refresh
  - 调用fsync，将魂村中的Segments写入磁盘
  - 清空（删除）Transaction log
  - 默认30分钟调用一次
  - Transaction Log满（默认512MB）

- Merge
  - Segment很多，需要被定期合并，减少Segments/删除已删除的文档.del文件

##### 2、排序

- 排序时针对字段原始内容进行的，倒排索引无法发挥作用，需要用到正排索引。通过文档ID和字段快速得到字段原始内容

- Elasticsearch有两种实现方法：Fielddata、Doc Values(列式存储，对Text类型无效 )

- 关闭Doc Values，默认启用，可以通过Mapping设置关闭，

  - 增加索引的速度/减少磁盘空间，

  - 如果重新打开，需要重建索引

  - 什么需要关闭，明确不需要做排序及聚合分析

  - ```
    #关闭 keyword的 doc values
    PUT test_keyword
    PUT test_keyword/_mapping
    {
      "properties": {
        "user_name":{
          "type": "keyword",
          "doc_values":false
        }
      }
    }
    ```

    

- ```
  #单字段排序
  POST /kibana_sample_data_ecommerce/_search
  {
    "size": 5,
    "query": {
      "match_all": {
      }
    },
    "sort": [
      {"order_date": {"order": "desc"}}
    ]
  }
  
  #多字段排序
  POST /kibana_sample_data_ecommerce/_search
  {
    "size": 5,
    "query": {
      "match_all": {
  
      }
    },
    "sort": [
      {"order_date": {"order": "desc"}},
      {"_doc":{"order": "asc"}},
      {"_score":{ "order": "desc"}}
    ]
  }
  
  ```

##### Demo:

```
#单字段排序
POST /kibana_sample_data_ecommerce/_search
{
  "size": 5,
  "query": {
    "match_all": {

    }
  },
  "sort": [
    {"order_date": {"order": "desc"}}
  ]
}

#多字段排序
POST /kibana_sample_data_ecommerce/_search
{
  "size": 5,
  "query": {
    "match_all": {

    }
  },
  "sort": [
    {"order_date": {"order": "desc"}},
    {"_doc":{"order": "asc"}},
    {"_score":{ "order": "desc"}}
  ]
}

GET kibana_sample_data_ecommerce/_mapping

#对 text 字段进行排序。默认会报错，需打开fielddata
POST /kibana_sample_data_ecommerce/_search
{
  "size": 5,
  "query": {
    "match_all": {

    }
  },
  "sort": [
    {"customer_full_name": {"order": "desc"}}
  ]
}

#打开 text的 fielddata
PUT kibana_sample_data_ecommerce/_mapping
{
  "properties": {
    "customer_full_name" : {
          "type" : "text",
          "fielddata": true,
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        }
  }
}

#关闭 keyword的 doc values
PUT test_keyword
PUT test_keyword/_mapping
{
  "properties": {
    "user_name":{
      "type": "keyword",
      "doc_values":false
    }
  }
}

DELETE test_keyword

PUT test_text
PUT test_text/_mapping
{
  "properties": {
    "intro":{
      "type": "text",
      "doc_values":true
    }
  }
}

DELETE test_text


DELETE temp_users
PUT temp_users
PUT temp_users/_mapping
{
  "properties": {
    "name":{"type": "text","fielddata": true},
    "desc":{"type": "text","fielddata": true}
  }
}

Post temp_users/_doc
{"name":"Jack","desc":"Jack is a good boy!","age":10}

#打开fielddata 后，查看 docvalue_fields数据
POST  temp_users/_search
{
  "docvalue_fields": [
    "name","desc"
    ]
}

#查看整型字段的docvalues
POST  temp_users/_search
{
  "docvalue_fields": [
    "age"
    ]
}
```

##### 3、分页

- 当一个查询：From=990，Size=10
  - 会在每个分片上都获取1000个文档。然后通过Coringating Node聚合所有结果。最后再通过排序选取前1000个文档
  - 页数越深，占用内存越多。为了避免深度分页带来的内存开销。ES有一个设定，默认限定到10000个文档

- ```
  POST tmdb/_search
  {
    "from": 10000,
    "size": 1,
    "query": {
      "match_all": {
  
      }
    }
  }
  ```

- 避免深度分页，search_after：

  - 不支持指定页数（From）
  - 只能往下翻
  - 第一步搜索需要指定sort，并且保证值是唯一的（可以通过加入_id保证唯一性）
  - 假定Size是10，当查询990-1000，通过唯一排序值定位，将每次要处理的文档数控制在10，即从每个分片取回10条数据，再排序

  ```
  POST users/_search
  {
      "size": 1,
      "query": {
          "match_all": {}
      },
      "sort": [
          {"age": "desc"} ,
          {"_id": "asc"}    
      ]
  }
   
  POST users/_search
  {
      "size": 1,
      "query": {
          "match_all": {}
      },
      "search_after":
          [10,"ZQ0vYGsBrR8X3IP75QqX"],
      "sort": [
          {"age": "desc"} ,
          {"_id": "asc"}    
      ]
  }
  ```

- ###### Scroll API

- 创建一个快照，有新的数据写入以后，无法被查到

- 每次查询，输入上一次的scroll id

```
POST /users/_search?scroll=5m
{
    "size": 1,
    "query": {
        "match_all" : {
        }
    }
}


POST users/_doc
{"name":"user5","age":50}
POST /_search/scroll
{
    "scroll" : "1m",
    "scroll_id" : "DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAAWAWbWdoQXR2d3ZUd2kzSThwVTh4bVE0QQ=="
}
```

- Demo：

  ```
  POST tmdb/_search
  {
    "from": 10000,
    "size": 1,
    "query": {
      "match_all": {
  
      }
    }
  }
  
  #Scroll API
  DELETE users
  
  POST users/_doc
  {"name":"user1","age":10}
  
  POST users/_doc
  {"name":"user2","age":11}
  
  
  POST users/_doc
  {"name":"user2","age":12}
  
  POST users/_doc
  {"name":"user2","age":13}
  
  POST users/_count
  
  POST users/_search
  {
      "size": 1,
      "query": {
          "match_all": {}
      },
      "sort": [
          {"age": "desc"} ,
          {"_id": "asc"}    
      ]
  }
  
  POST users/_search
  {
      "size": 1,
      "query": {
          "match_all": {}
      },
      "search_after":
          [
            10,
            "ZQ0vYGsBrR8X3IP75QqX"],
      "sort": [
          {"age": "desc"} ,
          {"_id": "asc"}    
      ]
  }
  
  
  #Scroll API
  DELETE users
  POST users/_doc
  {"name":"user1","age":10}
  
  POST users/_doc
  {"name":"user2","age":20}
  
  POST users/_doc
  {"name":"user3","age":30}
  
  POST users/_doc
  {"name":"user4","age":40}
  
  POST /users/_search?scroll=5m
  {
      "size": 1,
      "query": {
          "match_all" : {
          }
      }
  }
  
  
  POST users/_doc
  {"name":"user5","age":50}
  POST /_search/scroll
  {
      "scroll" : "1m",
      "scroll_id" : "DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAAWAWbWdoQXR2d3ZUd2kzSThwVTh4bVE0QQ=="
  }
  ```

##### 4、Function Score Query

##### 5、Term Suggester

##### 6、查询分类

- 基于全文本查找Math Query、Match phrase Query、Query String Query
- Constant Score转为Filter避免算分，Filter可以利用缓存

```
#查看不同的analyzer的效果
#standard
GET _analyze
{
  "analyzer": "standard",
  "text": "2 running Quick brown-foxes leap over lazy dogs in the summer evening."
}

#ignore_unavailable=true，可以忽略尝试访问不存在的索引“404_idx”导致的报错
#查询movies分页
POST /movies,404_idx/_search?ignore_unavailable=true
{
  "profile": true,
	"query": {
		"match_all": {}
	}
}

POST /kibana_sample_data_ecommerce/_search
{
  "from":10,
  "size":20,
  "query":{
    "match_all": {}
  }
}


#对日期排序
POST kibana_sample_data_ecommerce/_search
{
  "sort":[{"order_date":"desc"}],
  "query":{
    "match_all": {}
  }

}

#source filtering
POST kibana_sample_data_ecommerce/_search
{
  "_source":["order_date"],
  "query":{
    "match_all": {}
  }
}


#脚本字段
GET kibana_sample_data_ecommerce/_search
{
  "script_fields": {
    "new_field": {
      "script": {
        "lang": "painless",
        "source": "doc['order_date'].value+'hello'"
      }
    }
  },
  "query": {
    "match_all": {}
  }
}


POST movies/_search
{
  "query": {
    "match": {
      "title": "last christmas"
    }
  }
}

POST movies/_search
{
  "query": {
    "match": {
      "title": {
        "query": "last christmas",
        "operator": "and"
      }
    }
  }
}

POST movies/_search
{
  "query": {
    "match_phrase": {
      "title":{
        "query": "one love"

      }
    }
  }
}

POST movies/_search
{
  "query": {
    "match_phrase": {
      "title":{
        "query": "one love",
        "slop": 1

      }
    }
  }
}
```

- Query Strin Query

  - ```
    PUT /users/_doc/1
    {
      "name":"Ruan Yiming",
      "about":"java, golang, node, swift, elasticsearch"
    }
    
    PUT /users/_doc/2
    {
      "name":"Li Yiming",
      "about":"Hadoop"
    }
    
    
    POST users/_search
    {
      "query": {
        "query_string": {
          "default_field": "name",
          "query": "Ruan AND Yiming"
        }
      }
    }
    
    
    POST users/_search
    {
      "query": {
        "query_string": {
          "fields":["name","about"],
          "query": "(Ruan AND Yiming) OR (Java AND Elasticsearch)"
        }
      }
    }
    
    
    #Simple Query 默认的operator是 Or
    POST users/_search
    {
      "query": {
        "simple_query_string": {
          "query": "Ruan AND Yiming",
          "fields": ["name"]
        }
      }
    }
    
    
    POST users/_search
    {
      "query": {
        "simple_query_string": {
          "query": "Ruan Yiming",
          "fields": ["name"],
          "default_operator": "AND"
        }
      }
    }
    
    
    GET /movies/_search
    {
    	"profile": true,
    	"query":{
    		"query_string":{
    			"default_field": "title",
    			"query": "Beafiful AND Mind"
    		}
    	}
    }
    
    
    # 多fields
    GET /movies/_search
    {
    	"profile": true,
    	"query":{
    		"query_string":{
    			"fields":[
    				"title",
    				"year"
    			],
    			"query": "2012"
    		}
    	}
    }
    
    
    
    GET /movies/_search
    {
    	"profile":true,
    	"query":{
    		"simple_query_string":{
    			"query":"Beautiful +mind",
    			"fields":["title"]
    		}
    	}
    }
    ```

- Term Query

  - ```
    DELETE products
    PUT products
    {
      "settings": {
        "number_of_shards": 1
      }
    }
    
    
    POST /products/_bulk
    { "index": { "_id": 1 }}
    { "productID" : "XHDK-A-1293-#fJ3","desc":"iPhone" }
    { "index": { "_id": 2 }}
    { "productID" : "KDKE-B-9947-#kL5","desc":"iPad" }
    { "index": { "_id": 3 }}
    { "productID" : "JODL-X-1937-#pV7","desc":"MBP" }
    
    GET /products
    
    POST /products/_search
    {
      "query": {
        "term": {
          "desc": {
            //"value": "iPhone"
            "value":"iphone"
          }
        }
      }
    }
    
    POST /products/_search
    {
      "query": {
        "term": {
          "desc.keyword": {
            //"value": "iPhone"
            //"value":"iphone"
          }
        }
      }
    }
    
    
    POST /products/_search
    {
      "query": {
        "term": {
          "productID": {
            "value": "XHDK-A-1293-#fJ3"
          }
        }
      }
    }
    
    POST /products/_search
    {
      //"explain": true,
      "query": {
        "term": {
          "productID.keyword": {
            "value": "XHDK-A-1293-#fJ3"
          }
        }
      }
    }
    POST /products/_search
    {
      "explain": true,
      "query": {
        "constant_score": {
          "filter": {
            "term": {
              "productID.keyword": "XHDK-A-1293-#fJ3"
            }
          }
    
        }
      }
    }
    
    
    #设置 position_increment_gap
    DELETE groups
    PUT groups
    {
      "mappings": {
        "properties": {
          "names":{
            "type": "text",
            "position_increment_gap": 0
          }
        }
      }
    }
    
    GET groups/_mapping
    
    POST groups/_doc
    {
      "names": [ "John Water", "Water Smith"]
    }
    
    POST groups/_search
    {
      "query": {
        "match_phrase": {
          "names": {
            "query": "Water Water",
            "slop": 100
          }
        }
      }
    }
    
    
    POST groups/_search
    {
      "query": {
        "match_phrase": {
          "names": "Water Smith"
        }
      }
    }
    ```

    

- 参考：

  - https://www.elastic.co/guide/en/elasticsearch/reference/7.0/search-uri-request.html
  - https://www.elastic.co/guide/en/elasticsearch/reference/7.0/search-search.html
  - https://www.elastic.co/guide/en/elasticsearch/reference/7.1/term-level-queries.html
  - https://www.elastic.co/guide/en/elasticsearch/reference/7.1/full-text-queries.html

##### 7、结构化查询

- ```
  #结构化搜索，精确匹配
  DELETE products
  POST /products/_bulk
  { "index": { "_id": 1 }}
  { "price" : 10,"avaliable":true,"date":"2018-01-01", "productID" : "XHDK-A-1293-#fJ3" }
  { "index": { "_id": 2 }}
  { "price" : 20,"avaliable":true,"date":"2019-01-01", "productID" : "KDKE-B-9947-#kL5" }
  { "index": { "_id": 3 }}
  { "price" : 30,"avaliable":true, "productID" : "JODL-X-1937-#pV7" }
  { "index": { "_id": 4 }}
  { "price" : 30,"avaliable":false, "productID" : "QQPX-R-3956-#aD8" }
  
  GET products/_mapping
  
  
  
  #对布尔值 match 查询，有算分
  POST products/_search
  {
    "profile": "true",
    "explain": true,
    "query": {
      "term": {
        "avaliable": true
      }
    }
  }
  
  
  
  #对布尔值，通过constant score 转成 filtering，没有算分
  POST products/_search
  {
    "profile": "true",
    "explain": true,
    "query": {
      "constant_score": {
        "filter": {
          "term": {
            "avaliable": true
          }
        }
      }
    }
  }
  
  
  #数字类型 Term
  POST products/_search
  {
    "profile": "true",
    "explain": true,
    "query": {
      "term": {
        "price": 30
      }
    }
  }
  
  #数字类型 terms
  POST products/_search
  {
    "query": {
      "constant_score": {
        "filter": {
          "terms": {
            "price": [
              "20",
              "30"
            ]
          }
        }
      }
    }
  }
  
  #数字 Range 查询
  GET products/_search
  {
      "query" : {
          "constant_score" : {
              "filter" : {
                  "range" : {
                      "price" : {
                          "gte" : 20,
                          "lte"  : 30
                      }
                  }
              }
          }
      }
  }
  
  
  # 日期 range
  POST products/_search
  {
      "query" : {
          "constant_score" : {
              "filter" : {
                  "range" : {
                      "date" : {
                        "gte" : "now-1y"
                      }
                  }
              }
          }
      }
  }
  
  
  
  #exists查询
  POST products/_search
  {
    "query": {
      "constant_score": {
        "filter": {
          "exists": {
            "field": "date"
          }
        }
      }
    }
  }
  
  #处理多值字段
  POST /movies/_bulk
  { "index": { "_id": 1 }}
  { "title" : "Father of the Bridge Part II","year":1995, "genre":"Comedy"}
  { "index": { "_id": 2 }}
  { "title" : "Dave","year":1993,"genre":["Comedy","Romance"] }
  
  
  #处理多值字段，term 查询是包含，而不是等于
  POST movies/_search
  {
    "query": {
      "constant_score": {
        "filter": {
          "term": {
            "genre.keyword": "Comedy"
          }
        }
      }
    }
  }
  
  
  #字符类型 terms
  POST products/_search
  {
    "query": {
      "constant_score": {
        "filter": {
          "terms": {
            "productID.keyword": [
              "QQPX-R-3956-#aD8",
              "JODL-X-1937-#pV7"
            ]
          }
        }
      }
    }
  }
  
  
  
  POST products/_search
  {
    "profile": "true",
    "explain": true,
    "query": {
      "match": {
        "price": 30
      }
    }
  }
  
  
  POST products/_search
  {
    "profile": "true",
    "explain": true,
    "query": {
      "term": {
        "date": "2019-01-01"
      }
    }
  }
  
  POST products/_search
  {
    "profile": "true",
    "explain": true,
    "query": {
      "match": {
        "date": "2019-01-01"
      }
    }
  }
  
  
  
  
  POST products/_search
  {
    "profile": "true",
    "explain": true,
    "query": {
      "constant_score": {
        "filter": {
          "term": {
            "productID.keyword": "XHDK-A-1293-#fJ3"
          }
        }
      }
    }
  }
  
  POST products/_search
  {
    "profile": "true",
    "explain": true,
    "query": {
      "term": {
        "productID.keyword": "XHDK-A-1293-#fJ3"
      }
    }
  }
  
  #对布尔数值
  POST products/_search
  {
    "query": {
      "constant_score": {
        "filter": {
          "term": {
            "avaliable": "false"
          }
        }
      }
    }
  }
  
  POST products/_search
  {
    "query": {
      "term": {
        "avaliable": {
          "value": "false"
        }
      }
    }
  }
  
  POST products/_search
  {
    "profile": "true",
    "explain": true,
    "query": {
      "term": {
        "price": {
          "value": "20"
        }
      }
    }
  }
  
  POST products/_search
  {
    "profile": "true",
    "explain": true,
    "query": {
      "match": {
        "price": "20"
      }
      }
    }
  }
  
  
  POST products/_search
  {
    "query": {
      "constant_score": {
        "filter": {
          "bool": {
            "must_not": {
              "exists": {
                "field": "date"
              }
            }
          }
        }
      }
    }
  }
  ```

- 日期

  - | y    | 年   |
    | ---- | ---- |
    | M    | 月   |
    | w    | 周   |
    | d    | 天   |
    | H/h  | 时   |
    | m    | 分   |
    | s    | 秒   |

- 总结：
  - 如果不需要算分，可以通过constant score，将查询转为Filtering
  - 范围查询和Date Math
  - 使用Exist查询处理非空Null值
  - Term查询时包含，不是完成相等。针对多值字段查询尤其注意

- 参考：
  - https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-exists-query.html 
  - https://www.elastic.co/guide/en/elasticsearch/reference/7.1/term-level-queries.html

##### 8、布尔查询

- | must     | 必须匹配。贡献算分                     |
  | -------- | -------------------------------------- |
  | should   | 选择性匹配，贡献算分                   |
  | must_not | Filter Context查询子句，必须不能匹配   |
  | filter   | Filter Context必须匹配，但是不贡献算分 |

  

- ```
  POST /products/_bulk
  { "index": { "_id": 1 }}
  { "price" : 10,"avaliable":true,"date":"2018-01-01", "productID" : "XHDK-A-1293-#fJ3" }
  { "index": { "_id": 2 }}
  { "price" : 20,"avaliable":true,"date":"2019-01-01", "productID" : "KDKE-B-9947-#kL5" }
  { "index": { "_id": 3 }}
  { "price" : 30,"avaliable":true, "productID" : "JODL-X-1937-#pV7" }
  { "index": { "_id": 4 }}
  { "price" : 30,"avaliable":false, "productID" : "QQPX-R-3956-#aD8" }
  
  
  
  #基本语法
  POST /products/_search
  {
    "query": {
      "bool" : {
        "must" : {
          "term" : { "price" : "30" }
        },
        "filter": {
          "term" : { "avaliable" : "true" }
        },
        "must_not" : {
          "range" : {
            "price" : { "lte" : 10 }
          }
        },
        "should" : [
          { "term" : { "productID.keyword" : "JODL-X-1937-#pV7" } },
          { "term" : { "productID.keyword" : "XHDK-A-1293-#fJ3" } }
        ],
        "minimum_should_match" :1
      }
    }
  }
  
  #改变数据模型，增加字段。解决数组包含而不是精确匹配的问题
  POST /newmovies/_bulk
  { "index": { "_id": 1 }}
  { "title" : "Father of the Bridge Part II","year":1995, "genre":"Comedy","genre_count":1 }
  { "index": { "_id": 2 }}
  { "title" : "Dave","year":1993,"genre":["Comedy","Romance"],"genre_count":2 }
  
  #must，有算分
  POST /newmovies/_search
  {
    "query": {
      "bool": {
        "must": [
          {"term": {"genre.keyword": {"value": "Comedy"}}},
          {"term": {"genre_count": {"value": 1}}}
  
        ]
      }
    }
  }
  
  #Filter。不参与算分，结果的score是0
  POST /newmovies/_search
  {
    "query": {
      "bool": {
        "filter": [
          {"term": {"genre.keyword": {"value": "Comedy"}}},
          {"term": {"genre_count": {"value": 1}}}
          ]
  
      }
    }
  }
  
  
  #Filtering Context
  POST _search
  {
    "query": {
      "bool" : {
  
        "filter": {
          "term" : { "avaliable" : "true" }
        },
        "must_not" : {
          "range" : {
            "price" : { "lte" : 10 }
          }
        }
      }
    }
  }
  
  
  #Query Context
  POST /products/_bulk
  { "index": { "_id": 1 }}
  { "price" : 10,"avaliable":true,"date":"2018-01-01", "productID" : "XHDK-A-1293-#fJ3" }
  { "index": { "_id": 2 }}
  { "price" : 20,"avaliable":true,"date":"2019-01-01", "productID" : "KDKE-B-9947-#kL5" }
  { "index": { "_id": 3 }}
  { "price" : 30,"avaliable":true, "productID" : "JODL-X-1937-#pV7" }
  { "index": { "_id": 4 }}
  { "price" : 30,"avaliable":false, "productID" : "QQPX-R-3956-#aD8" }
  
  
  POST /products/_search
  {
    "query": {
      "bool": {
        "should": [
          {
            "term": {
              "productID.keyword": {
                "value": "JODL-X-1937-#pV7"}}
          },
          {"term": {"avaliable": {"value": true}}
          }
        ]
      }
    }
  }
  
  
  #嵌套，实现了 should not 逻辑
  POST /products/_search
  {
    "query": {
      "bool": {
        "must": {
          "term": {
            "price": "30"
          }
        },
        "should": [
          {
            "bool": {
              "must_not": {
                "term": {
                  "avaliable": "false"
                }
              }
            }
          }
        ],
        "minimum_should_match": 1
      }
    }
  }
  
  
  #Controll the Precision
  POST _search
  {
    "query": {
      "bool" : {
        "must" : {
          "term" : { "price" : "30" }
        },
        "filter": {
          "term" : { "avaliable" : "true" }
        },
        "must_not" : {
          "range" : {
            "price" : { "lte" : 10 }
          }
        },
        "should" : [
          { "term" : { "productID.keyword" : "JODL-X-1937-#pV7" } },
          { "term" : { "productID.keyword" : "XHDK-A-1293-#fJ3" } }
        ],
        "minimum_should_match" :2
      }
    }
  }
  
  
  
  POST /animals/_search
  {
    "query": {
      "bool": {
        "should": [
          { "term": { "text": "brown" }},
          { "term": { "text": "red" }},
          { "term": { "text": "quick"   }},
          { "term": { "text": "dog"   }}
        ]
      }
    }
  }
  
  POST /animals/_search
  {
    "query": {
      "bool": {
        "should": [
          { "term": { "text": "quick" }},
          { "term": { "text": "dog"   }},
          {
            "bool":{
              "should":[
                 { "term": { "text": "brown" }},
                   { "term": { "text": "brown" }},
              ]
            }
  
          }
        ]
      }
    }
  }
  
  
  DELETE blogs
  POST /blogs/_bulk
  { "index": { "_id": 1 }}
  {"title":"Apple iPad", "content":"Apple iPad,Apple iPad" }
  { "index": { "_id": 2 }}
  {"title":"Apple iPad,Apple iPad", "content":"Apple iPad" }
  
  
  POST blogs/_search
  {
    "query": {
      "bool": {
        "should": [
          {"match": {
            "title": {
              "query": "apple,ipad",
              "boost": 1.1
            }
          }},
  
          {"match": {
            "content": {
              "query": "apple,ipad",
              "boost":
            }
          }}
        ]
      }
    }
  }
  
  DELETE news
  POST /news/_bulk
  { "index": { "_id": 1 }}
  { "content":"Apple Mac" }
  { "index": { "_id": 2 }}
  { "content":"Apple iPad" }
  { "index": { "_id": 3 }}
  { "content":"Apple employee like Apple Pie and Apple Juice" }
  
  
  POST news/_search
  {
    "query": {
      "bool": {
        "must": {
          "match":{"content":"apple"}
        }
      }
    }
  }
  
  POST news/_search
  {
    "query": {
      "bool": {
        "must": {
          "match":{"content":"apple"}
        },
        "must_not": {
          "match":{"content":"pie"}
        }
      }
    }
  }
  
  POST news/_search
  {
    "query": {
      "boosting": {
        "positive": {
          "match": {
            "content": "apple"
          }
        },
        "negative": {
          "match": {
            "content": "pie"
          }
        },
        "negative_boost": 0.5
      }
    }
  }
  ```

##### 9、单字符串多字段匹配：DisMax Query

- ```
  PUT /blogs/_doc/1
  {
      "title": "Quick brown rabbits",
      "body":  "Brown rabbits are commonly seen."
  }
  
  PUT /blogs/_doc/2
  {
      "title": "Keeping pets healthy",
      "body":  "My quick brown fox eats rabbits on a regular basis."
  }
  
  POST /blogs/_search
  {
      "query": {
          "bool": {
              "should": [
                  { "match": { "title": "Brown fox" }},
                  { "match": { "body":  "Brown fox" }}
              ]
          }
      }
  }
  
  POST blogs/_search
  {
      "query": {
          "dis_max": {
              "queries": [
                  { "match": { "title": "Quick pets" }},
                  { "match": { "body":  "Quick pets" }}
              ]
          }
      }
  }
  
  
  POST blogs/_search
  {
      "query": {
          "dis_max": {
              "queries": [
                  { "match": { "title": "Quick pets" }},
                  { "match": { "body":  "Quick pets" }}
              ],
              "tie_breaker": 0.2
          }
      }
  }
  ```

- 参考：
  - https://www.elastic.co/guide/en/elasticsearch/reference/7.1/query-dsl-dis-max-query.html

##### 10、单字符串多字段匹配：Mutil_match

- ```
  POST blogs/_search
  {
      "query": {
          "dis_max": {
              "queries": [
                  { "match": { "title": "Quick pets" }},
                  { "match": { "body":  "Quick pets" }}
              ],
              "tie_breaker": 0.2
          }
      }
  }
  
  POST blogs/_search
  {
    "query": {
      "multi_match": {
        "type": "best_fields",
        "query": "Quick pets",
        "fields": ["title","body"],
        "tie_breaker": 0.2,
        "minimum_should_match": "20%"
      }
    }
  }
  
  
  
  POST books/_search
  {
      "multi_match": {
          "query":  "Quick brown fox",
          "fields": "*_title"
      }
  }
  
  
  POST books/_search
  {
      "multi_match": {
          "query":  "Quick brown fox",
          "fields": [ "*_title", "chapter_title^2" ]
      }
  }
  
  
  
  DELETE /titles
  PUT /titles
  {
      "settings": { "number_of_shards": 1 },
      "mappings": {
          "my_type": {
              "properties": {
                  "title": {
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
      }
  }
  
  PUT /titles
  {
    "mappings": {
      "properties": {
        "title": {
          "type": "text",
          "analyzer": "english"
        }
      }
    }
  }
  
  POST titles/_bulk
  { "index": { "_id": 1 }}
  { "title": "My dog barks" }
  { "index": { "_id": 2 }}
  { "title": "I see a lot of barking dogs on the road " }
  
  
  GET titles/_search
  {
    "query": {
      "match": {
        "title": "barking dogs"
      }
    }
  }
  
  DELETE /titles
  PUT /titles
  {
    "mappings": {
      "properties": {
        "title": {
          "type": "text",
          "analyzer": "english",
          "fields": {"std": {"type": "text","analyzer": "standard"}}
        }
      }
    }
  }
  
  POST titles/_bulk
  { "index": { "_id": 1 }}
  { "title": "My dog barks" }
  { "index": { "_id": 2 }}
  { "title": "I see a lot of barking dogs on the road " }
  
  GET /titles/_search
  {
     "query": {
          "multi_match": {
              "query":  "barking dogs",
              "type":   "most_fields",
              "fields": [ "title", "title.std" ]
          }
      }
  }
  
  GET /titles/_search
  {
     "query": {
          "multi_match": {
              "query":  "barking dogs",
              "type":   "most_fields",#/cross_fields
              "fields": [ "title^10", "title.std" ]
          }
      }
  }
  ```

- 参考：
  - https://www.elastic.co/guide/en/elasticsearch/reference/7.1/query-dsl-dis-max-query.html

#### 5、SearchTemplate和IndexAlias查询

##### 1、解耦程序&搜索DSL

- Demo：https://www.elastic.co/guide/en/elasticsearch/reference/7.9/search-template.html

  ```
  POST _scripts/tmdb
  {
    "script": {
      "lang": "mustache",
      "source": {
        "_source": [
          "title","overview"
        ],
        "size": 20,
        "query": {
          "multi_match": {
            "query": "{{q}}",
            "fields": ["title","overview"]
          }
        }
      }
    }
  }
  DELETE _scripts/tmdb
  
  GET _scripts/tmdb
  
  POST tmdb/_search/template
  {
      "id":"tmdb",
      "params": {
          "q": "basketball with cartoon aliens"
      }
  }
  ```

##### 2、Index Alias实现零停机运维

1. Demo

   ```
   PUT movies-2019/_doc/1
   {
     "name":"the matrix",
     "rating":5
   }
   
   PUT movies-2019/_doc/2
   {
     "name":"Speed",
     "rating":3
   }
   
   POST _aliases
   {
     "actions": [
       {
         "add": {
           "index": "movies-2019",
           "alias": "movies-latest"
         }
       }
     ]
   }
   
   POST movies-latest/_search
   {
     "query": {
       "match_all": {}
     }
   }
   
   POST _aliases
   {
     "actions": [
       {
         "add": {
           "index": "movies-2019",
           "alias": "movies-lastest-highrate",
           "filter": {
             "range": {
               "rating": {
                 "gte": 4
               }
             }
           }
         }
       }
     ]
   }
   
   POST movies-lastest-highrate/_search
   {
     "query": {
       "match_all": {}
     }
   }
   ```

   

#### 6、UpdateByQuery & Reindex

一般在一下几种情况时，我们需要重建索引

- 索引的Mappings发生变更：字段类型更改，分词器及字典更新
- 索引的Settings发生变更：索引的主分片数发生改变
- 集群内，建群间需要做数据迁移

##### 1、Update By Query：在现有索引上重建

1. Demo:新增字段

   ```
   DELETE blogs
   # 写入文档
   PUT blogs/_doc/1
   {
     "content":"Hadoop is cool",
     "keyword":"hadoop"
   }
   
   # 查看 Mapping
   GET blogs/_mapping
   
   # 修改 Mapping，增加子字段，使用英文分词器
   PUT blogs/_mapping
   {
         "properties" : {
           "content" : {
             "type" : "text",
             "fields" : {
               "english" : {
                 "type" : "text",
                 "analyzer":"english"
               }
             }
           }
         }
       }
   
   
   # 写入文档
   PUT blogs/_doc/2
   {
     "content":"Elasticsearch rocks",
       "keyword":"elasticsearch"
   }
   
   # 查询新写入文档
   POST blogs/_search
   {
     "query": {
       "match": {
         "content.english": "Elasticsearch"
       }
     }
   
   }
   
   # 查询 Mapping 变更前写入的文档
   POST blogs/_search
   {
     "query": {
       "match": {
         "content.english": "Hadoop"
       }
     }
   }
   
   # Update所有文档
   POST blogs/_update_by_query
   {
   }
   
   # 查询之前写入的文档
   POST blogs/_search
   {
     "query": {
       "match": {
         "content.english": "Hadoop"
       }
     }
   }
   ```

##### 2、Reindex:修改字段类型，keyword的字段类型text改成keyword

- ES不允许在原有Mapping上对字段类型进行修改
- 只能创建新的索引，并且设定正确的字段类型，再重新导入数据

```
# 查询
GET blogs/_mapping
#尝试修改字段类型，将会报错
PUT blogs/_mapping
{
        "properties" : {
        "content" : {
          "type" : "text",
          "fields" : {
            "english" : {
              "type" : "text",
              "analyzer" : "english"
            }
          }
        },
        "keyword" : {
          "type" : "keyword"
        }
      }
}

DELETE blogs_fix
# 创建新的索引并且设定新的Mapping
PUT blogs_fix/
{
  "mappings": {
        "properties" : {
        "content" : {
          "type" : "text",
          "fields" : {
            "english" : {
              "type" : "text",
              "analyzer" : "english"
            }
          }
        },
        "keyword" : {
          "type" : "keyword"
        }
      }    
  }
}

# Reindx API
POST  _reindex
{
  "source": {
    "index": "blogs"
  },
  "dest": {
    "index": "blogs_fix"
  }
}

GET  blogs_fix/_doc/1

# 测试 Term Aggregation
POST blogs_fix/_search
{
  "size": 0,
  "aggs": {
    "blog_keyword": {
      "terms": {
        "field": "keyword",
        "size": 10
      }
    }
  }
}

# Reindx API，version Type Internal
POST  _reindex
{
  "source": {
    "index": "blogs"
  },
  "dest": {
    "index": "blogs_fix",
    "version_type": "internal"
  }
}

# 文档版本号增加
GET  blogs_fix/_doc/1

# Reindx API，version Type Internal
POST  _reindex
{
  "source": {
    "index": "blogs"
  },
  "dest": {
    "index": "blogs_fix",
    "version_type": "external"
  }
}

# Reindx API，version Type Internal
POST  _reindex
{
  "source": {
    "index": "blogs"
  },
  "dest": {
    "index": "blogs_fix",
    "version_type": "external"
  },
  "conflicts": "proceed"
}

#假如blogs_fix已经存在，会增量创建
# Reindx API，version Type Internal
POST  _reindex
{
  "source": {
    "index": "blogs"
  },
  "dest": {
    "index": "blogs_fix",
    "op_type": "create"
  }
}
```

##### 3、跨集群Reindex

- 需要修改elasticsearch.yml,并且重启节点

- ```
  reindex.remote.whitelist: "otherhost:9200,another:9200"
  ```

```
POST _reindex
{
  "source": {
    "remote": {
      "host": "http://otherhost:9200"
    },
    "index": "source",
    "size": 100,
    "query": {
      "match": {
        "test": "data"
      }
    }
  },
  "dest": {
    "index": "dest"
  }
}
```

##### 4、Reindex支持异步操作

```
POST _reindex?wait_for_completion=false
#返回Task Id
GET _tasks?detailed=true&actions=*reindex
```

##### 5、相关文档

- https://www.elastic.co/guide/en/elasticsearch/reference/7.1/docs-reindex.html
- https://www.elastic.co/guide/en/elasticsearch/reference/7.1/docs-update-by-query.html

#### 7、IngestPipeline & Painless

- Tag字段中，逗号分割的文本应该是数组，而不是一个字符串
- 需求：后期需要对Tags进行Aggregation

```
DELETE tech_blogs

#Blog数据，包含3个字段，tags用逗号间隔
PUT tech_blogs/_doc/1
{
  "title":"Introducing big data......",
  "tags":"hadoop,elasticsearch,spark",
  "content":"You konw, for big data"
}
```

##### 1、Ingest Node

- Elasticsearch5.0后，引入一种新的节点类型，具有预处理数据的能力，可拦截Index或BulkAPI的请求，对数据进行转换，并重新返回给Index或Bulk API
- 无需Logstash，就可以进行数据的预处理，例如：
  - 为某个字段设置默认值；重命名某个字段的字段名；对字段值进行Split操作
  - 支持设置Painless脚本，对数据进行更加复杂的加工

##### 2、 Pipeline & Processor

```
DELETE tech_blogs

#Blog数据，包含3个字段，tags用逗号间隔
PUT tech_blogs/_doc/1
{
  "title":"Introducing big data......",
  "tags":"hadoop,elasticsearch,spark",
  "content":"You konw, for big data"
}


# 测试split tags
POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "description": "to split blog tags",
    "processors": [
      {
        "split": {
          "field": "tags",
          "separator": ","
        }
      }
    ]
  },
  "docs": [
    {
      "_index": "index",
      "_id": "id",
      "_source": {
        "title": "Introducing big data......",
        "tags": "hadoop,elasticsearch,spark",
        "content": "You konw, for big data"
      }
    },
    {
      "_index": "index",
      "_id": "idxx",
      "_source": {
        "title": "Introducing cloud computering",
        "tags": "openstack,k8s",
        "content": "You konw, for cloud"
      }
    }
  ]
}


#同时为文档，增加一个字段。blog查看量
POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "description": "to split blog tags",
    "processors": [
      {
        "split": {
          "field": "tags",
          "separator": ","
        }
      },

      {
        "set":{
          "field": "views",
          "value": 0
        }
      }
    ]
  },

  "docs": [
    {
      "_index":"index",
      "_id":"id",
      "_source":{
        "title":"Introducing big data......",
  "tags":"hadoop,elasticsearch,spark",
  "content":"You konw, for big data"
      }
    },


    {
      "_index":"index",
      "_id":"idxx",
      "_source":{
        "title":"Introducing cloud computering",
  "tags":"openstack,k8s",
  "content":"You konw, for cloud"
      }
    }

    ]
}



# 为ES添加一个 Pipeline
PUT _ingest/pipeline/blog_pipeline
{
  "description": "a blog pipeline",
  "processors": [
      {
        "split": {
          "field": "tags",
          "separator": ","
        }
      },

      {
        "set":{
          "field": "views",
          "value": 0
        }
      }
    ]
}

#查看Pipleline
GET _ingest/pipeline/blog_pipeline


#测试pipeline
POST _ingest/pipeline/blog_pipeline/_simulate
{
  "docs": [
    {
      "_source": {
        "title": "Introducing cloud computering",
        "tags": "openstack,k8s",
        "content": "You konw, for cloud"
      }
    }
  ]
}

#不使用pipeline更新数据
PUT tech_blogs/_doc/1
{
  "title":"Introducing big data......",
  "tags":"hadoop,elasticsearch,spark",
  "content":"You konw, for big data"
}

#使用pipeline更新数据
PUT tech_blogs/_doc/2?pipeline=blog_pipeline
{
  "title": "Introducing cloud computering",
  "tags": "openstack,k8s",
  "content": "You konw, for cloud"
}


#查看两条数据，一条被处理，一条未被处理
POST tech_blogs/_search
{}

#update_by_query 会导致错误
POST tech_blogs/_update_by_query?pipeline=blog_pipeline
{
}

#增加update_by_query的条件
POST tech_blogs/_update_by_query?pipeline=blog_pipeline
{
    "query": {
        "bool": {
            "must_not": {
                "exists": {
                    "field": "views"
                }
            }
        }
    }
}

```

##### 3、内置Processors

- Split Process(字符串分割)、Remove/Rename Processor（移除一个/重命名字段）、Append（增加一个新的标签）、Convert（类型转换）、Date/Json、
- Fail Processor（一旦出现异常，该Pipeline指定的错误信息能返回给用户）、
- Foreach Processor（数组字段，数组每个元素都会使用相同的处理器）、
- Grok Processor（日志的日期格式切割）、
- Gsub/Join/Split(字符串替换、数据转字符串、字符串转数组)、
- Lowercase/Upcase
- Date Index Name Processor：将通过该处理器的文档，分配到制定时间格式的索引中

##### 4、Painless

- 通过Painless脚本访问字段

- | 上下文              | 语法                   |
  | ------------------- | ---------------------- |
  | Ingestion           | ctx.field_name         |
  | Update              | ctx._source.field_name |
  | Search& Aggregation | doc["field_name"]      |

- | 参数                         | 说明                           |
  | ---------------------------- | ------------------------------ |
  | script.cache.max_size        | 设置最大缓存数                 |
  | script.cache.expire          | 设置缓存超时                   |
  | script.max_compilations_rate | 默认5分钟最多75次编译（75/5m） |

- 编译开销比较大，Elasticsearch会将脚本编译后缓存在Cache中，Inline scripts和Stored Scripts都会被缓存，默认缓存100个脚本

- Demo

  ```
  #########Demo for Painless###############
  
  # 增加一个 Script Prcessor
  POST _ingest/pipeline/_simulate
  {
    "pipeline": {
      "description": "to split blog tags",
      "processors": [
        {
          "split": {
            "field": "tags",
            "separator": ","
          }
        },
        {
          "script": {
            "source": """
            if(ctx.containsKey("content")){
              ctx.content_length = ctx.content.length();
            }else{
              ctx.content_length=0;
            }
  
  
            """
          }
        },
  
        {
          "set":{
            "field": "views",
            "value": 0
          }
        }
      ]
    },
  
    "docs": [
      {
        "_index":"index",
        "_id":"id",
        "_source":{
          "title":"Introducing big data......",
    "tags":"hadoop,elasticsearch,spark",
    "content":"You konw, for big data"
        }
      },
  
  
      {
        "_index":"index",
        "_id":"idxx",
        "_source":{
          "title":"Introducing cloud computering",
    "tags":"openstack,k8s",
    "content":"You konw, for cloud"
        }
      }
  
      ]
  }
  
  
  DELETE tech_blogs
  PUT tech_blogs/_doc/1
  {
    "title":"Introducing big data......",
    "tags":"hadoop,elasticsearch,spark",
    "content":"You konw, for big data",
    "views":0
  }
  
  POST tech_blogs/_update/1
  {
    "script": {
      "source": "ctx._source.views += params.new_views",
      "params": {
        "new_views":100
      }
    }
  }
  
  # 查看views计数
  POST tech_blogs/_search
  {
  
  }
  
  #保存脚本在 Cluster State
  POST _scripts/update_views
  {
    "script":{
      "lang": "painless",
      "source": "ctx._source.views += params.new_views"
    }
  }
  
  POST tech_blogs/_update/1
  {
    "script": {
      "id": "update_views",
      "params": {
        "new_views":1000
      }
    }
  }
  
  
  GET tech_blogs/_search
  {
    "script_fields": {
      "rnd_views": {
        "script": {
          "lang": "painless",
          "source": """
            java.util.Random rnd = new Random();
            doc['views'].value+rnd.nextInt(1000);
          """
        }
      }
    },
    "query": {
      "match_all": {}
    }
  }
  ```

##### 5、参考：

- https://www.elastic.co/cn/blog/should-i-use-logstash-or-elasticsearch-ingest-nodes
- https://www.elastic.co/guide/en/elasticsearch/reference/7.1/ingest-apis.html
- https://www.elastic.co/guide/en/elasticsearch/reference/7.1/ingest-processors.html
- https://www.elastic.co/guide/en/elasticsearch/painless/7.1/painless-lang-spec.html
- https://www.elastic.co/guide/en/elasticsearch/painless/7.1/painless-api-reference.html

#### 7、数据建模

##### 1、对象关系

- 嵌套对象（Nested Object）
- 父子关联关系 （Parent/Child）

##### 2、Nested Object查询

```
#设置Mappings
PUT /blog
{
  "mappings": {
    "properties": {
      "content": {
        "type": "text"
      },
      "time": {
        "type": "date"
      },
      "user": {
        "properties": {
          "city": {
            "type": "text"
          },
          "userid": {
            "type": "long"
          },
          "username": {
            "type": "keyword"
          }
        }
      }
    }
  }
}
# 插入一条 Blog 信息
PUT blog/_doc/1
{
  "content":"I like Elasticsearch",
  "time":"2019-01-01T00:00:00",
  "user":{
    "userid":1,
    "username":"Jack",
    "city":"Shanghai"
  }
}
# 查询 Blog 信息
POST blog/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {"content": "Elasticsearch"}},
        {"match": {"user.username": "Jack"}}
      ]
    }
  }
}
```

##### 3、对象数组查询

```
# 电影的Mapping信息
PUT my_movies
{
      "mappings" : {
      "properties" : {
        "actors" : {
          "properties" : {
            "first_name" : {
              "type" : "keyword"
            },
            "last_name" : {
              "type" : "keyword"
            }
          }
        },
        "title" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        }
      }
    }
}
# 写入一条电影信息
POST my_movies/_doc/1
{
  "title":"Speed",
  "actors":[
    {
      "first_name":"Keanu",
      "last_name":"Reeves"
    },
    {
      "first_name":"Dennis",
      "last_name":"Hopper"
    }

  ]
}
# 查询电影信息，将会搜索出记录
POST my_movies/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {"actors.first_name": "Keanu"}},
        {"match": {"actors.last_name": "Hopper"}}
      ]
    }
  }
}
```

##### 4、为什么会搜到不需要的结果

- 存储时，内部对象的边界并没有考虑在内，JSON格式处理成扁平式键值对的结构

  - ```
    “title”:"Speed"
    "actors.first_name":["Keanu", "Dennies"]
    "actors.last_name":["Reeves", "Hopper "]
    ```

- 可以用Nested Data Type解决这个问题

  - Nested数据类型：允许对象数组中的对象被独立索引
  - 在内部，Nested文档会被保存在两个Lucene文档中，在查询时做Join处理

  ```
  DELETE my_movies
  # 创建 Nested 对象 Mapping
  PUT my_movies
  {
        "mappings" : {
        "properties" : {
          "actors" : {
            "type": "nested",
            "properties" : {
              "first_name" : {"type" : "keyword"},
              "last_name" : {"type" : "keyword"}
            }},
          "title" : {
            "type" : "text",
            "fields" : {"keyword":{"type":"keyword","ignore_above":256}}
          }
        }
      }
  }
  
  POST my_movies/_doc/1
  {
    "title":"Speed",
    "actors":[
      {
        "first_name":"Keanu",
        "last_name":"Reeves"
      },
  
      {
        "first_name":"Dennis",
        "last_name":"Hopper"
      }
  
    ]
  }
  
  # Nested 查询
  POST my_movies/_search
  {
    "query": {
      "bool": {
        "must": [
          {"match": {"title": "Speed"}},
          {
            "nested": {
              "path": "actors",
              "query": {
                "bool": {
                  "must": [
                    {"match": {
                      "actors.first_name": "Keanu"
                    }},
                    {"match": {
                      "actors.last_name": "Hopper"
                    }}
                  ]
                }
              }
            }
          }
        ]
      }
    }
  }
  
  # Nested Aggregation
  POST my_movies/_search
  {
    "size": 0,
    "aggs": {
      "actors": {
        "nested": {
          "path": "actors"
        },
        "aggs": {
          "actor_name": {
            "terms": {
              "field": "actors.first_name",
              "size": 10
            }
          }
        }
      }
    }
  }
  
  
  # 普通 aggregation不工作
  POST my_movies/_search
  {
    "size": 0,
    "aggs": {
      "NAME": {
        "terms": {
          "field": "actors.first_name",
          "size": 10
        }
      }
    }
  }
  ```

  

##### Fields和properties

```
PUT /blog_properties
{
  "mappings": {
    "properties": {
      "user": {
        "properties": {
          "city": {
            "type": "text"
          },
          "userid": {
            "type": "long"
          },
          "username": {
            "type": "keyword"
          }
        }
      }
    }
  }
}


PUT /blog_fields
{
  "mappings": {
    "properties": {
      "content": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      }
    }
  }
}
```

##### Demo

```
DELETE blog
# 设置blog的 Mapping
PUT /blog
{
  "mappings": {
    "properties": {
      "content": {
        "type": "text"
      },
      "time": {
        "type": "date"
      },
      "user": {
        "properties": {
          "city": {
            "type": "text"
          },
          "userid": {
            "type": "long"
          },
          "username": {
            "type": "keyword"
          }
        }
      }
    }
  }
}


# 插入一条 Blog 信息
PUT blog/_doc/1
{
  "content":"I like Elasticsearch",
  "time":"2019-01-01T00:00:00",
  "user":{
    "userid":1,
    "username":"Jack",
    "city":"Shanghai"
  }
}


# 查询 Blog 信息
POST blog/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {"content": "Elasticsearch"}},
        {"match": {"user.username": "Jack"}}
      ]
    }
  }
}


DELETE my_movies

# 电影的Mapping信息
PUT my_movies
{
      "mappings" : {
      "properties" : {
        "actors" : {
          "properties" : {
            "first_name" : {
              "type" : "keyword"
            },
            "last_name" : {
              "type" : "keyword"
            }
          }
        },
        "title" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        }
      }
    }
}


# 写入一条电影信息
POST my_movies/_doc/1
{
  "title":"Speed",
  "actors":[
    {
      "first_name":"Keanu",
      "last_name":"Reeves"
    },

    {
      "first_name":"Dennis",
      "last_name":"Hopper"
    }

  ]
}

# 查询电影信息
POST my_movies/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {"actors.first_name": "Keanu"}},
        {"match": {"actors.last_name": "Hopper"}}
      ]
    }
  }

}

DELETE my_movies
# 创建 Nested 对象 Mapping
PUT my_movies
{
      "mappings" : {
      "properties" : {
        "actors" : {
          "type": "nested",
          "properties" : {
            "first_name" : {"type" : "keyword"},
            "last_name" : {"type" : "keyword"}
          }},
        "title" : {
          "type" : "text",
          "fields" : {"keyword":{"type":"keyword","ignore_above":256}}
        }
      }
    }
}


POST my_movies/_doc/1
{
  "title":"Speed",
  "actors":[
    {
      "first_name":"Keanu",
      "last_name":"Reeves"
    },

    {
      "first_name":"Dennis",
      "last_name":"Hopper"
    }

  ]
}

# Nested 查询
POST my_movies/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {"title": "Speed"}},
        {
          "nested": {
            "path": "actors",
            "query": {
              "bool": {
                "must": [
                  {"match": {
                    "actors.first_name": "Keanu"
                  }},

                  {"match": {
                    "actors.last_name": "Hopper"
                  }}
                ]
              }
            }
          }
        }
      ]
    }
  }
}


# Nested Aggregation
POST my_movies/_search
{
  "size": 0,
  "aggs": {
    "actors": {
      "nested": {
        "path": "actors"
      },
      "aggs": {
        "actor_name": {
          "terms": {
            "field": "actors.first_name",
            "size": 10
          }
        }
      }
    }
  }
}


# 普通 aggregation不工作
POST my_movies/_search
{
  "size": 0,
  "aggs": {
    "NAME": {
      "terms": {
        "field": "actors.first_name",
        "size": 10
      }
    }
  }
}
```

##### 嵌套对象参考：

- https://www.elastic.co/guide/en/elasticsearch/reference/7.1/query-dsl-nested-query.html

##### 5、文档的父子关系

- 对象和Nested对象的局限性

  - 每次更新，需要重新索引整个对象
  - 比如blog和user的关系，user更新将会使得每个blog都需要更新
  - ES提供了类似关系型数据库中Join的实现，可以通过维护Parent/child的关系
    - 父文档和子文档是两个独立的文档
    - 更新父文档无需重新索引子文档，子文档被添加，更新或者删除也不会影响到父文档和其他子文档

- 定义父子关系的步骤

  - 设置索引的Mapping
  - 索引父文档
  - 索引子文档
  - 所需查询文档
  - ![image-20210709143938353](F:\note\ES学习大纲.assets\image-20210709143938353.png)
  - ![image-20210709144031055](F:\note\ES学习大纲.assets\image-20210709144031055.png)

  - ![image-20210709144215595](F:\note\ES学习大纲.assets\image-20210709144215595.png)

  - 根据父文档查询

    ```
    # Parent Id 查询
    POST my_blogs/_search
    {
      "query": {
        "parent_id": {
          "type": "comment",
          "id": "blog2"
        }
      }
    }
    
    # Has Child 查询,返回父文档
    POST my_blogs/_search
    {
      "query": {
        "has_child": {
          "type": "comment",
          "query" : {
                    "match": {
                        "username" : "Jack"
                    }
                }
        }
      }
    }
    
    # Has Parent 查询，返回相关的子文档
    POST my_blogs/_search
    {
      "query": {
        "has_parent": {
          "parent_type": "blog",
          "query" : {
                    "match": {
                        "title" : "Learning Hadoop"
                    }
                }
        }
      }
    }
    ```

  - 嵌套对象VS父子文档

    |          | Nested Object                        | Parent / Child                         |
    | -------- | ------------------------------------ | -------------------------------------- |
    | 优点     | 文档存储在一起，读取性能高           | 父子文档可以独立更新                   |
    | 缺点     | 更新嵌套的子文档时，需要更新整个文档 | 需要额外的内存维护关系。读取性能相对差 |
    | 使用场景 | 子文档偶尔更新，以查询为主           | 子文档更新频繁                         |

    

- Demo

  ```
  DELETE my_blogs
  
  # 设定 Parent/Child Mapping
  PUT my_blogs
  {
    "settings": {
      "number_of_shards": 2
    },
    "mappings": {
      "properties": {
        "blog_comments_relation": {
          "type": "join",
          "relations": {
            "blog": "comment"
          }
        },
        "content": {
          "type": "text"
        },
        "title": {
          "type": "keyword"
        }
      }
    }
  }
  
  
  #索引父文档
  PUT my_blogs/_doc/blog1
  {
    "title":"Learning Elasticsearch",
    "content":"learning ELK @ geektime",
    "blog_comments_relation":{
      "name":"blog"
    }
  }
  
  #索引父文档
  PUT my_blogs/_doc/blog2
  {
    "title":"Learning Hadoop",
    "content":"learning Hadoop",
      "blog_comments_relation":{
      "name":"blog"
    }
  }
  
  
  #索引子文档
  PUT my_blogs/_doc/comment1?routing=blog1
  {
    "comment":"I am learning ELK",
    "username":"Jack",
    "blog_comments_relation":{
      "name":"comment",
      "parent":"blog1"
    }
  }
  
  #索引子文档
  PUT my_blogs/_doc/comment2?routing=blog2
  {
    "comment":"I like Hadoop!!!!!",
    "username":"Jack",
    "blog_comments_relation":{
      "name":"comment",
      "parent":"blog2"
    }
  }
  
  #索引子文档
  PUT my_blogs/_doc/comment3?routing=blog2
  {
    "comment":"Hello Hadoop",
    "username":"Bob",
    "blog_comments_relation":{
      "name":"comment",
      "parent":"blog2"
    }
  }
  
  # 查询所有文档
  POST my_blogs/_search
  {
  
  }
  
  
  #根据父文档ID查看
  GET my_blogs/_doc/blog2
  
  # Parent Id 查询
  POST my_blogs/_search
  {
    "query": {
      "parent_id": {
        "type": "comment",
        "id": "blog2"
      }
    }
  }
  
  # Has Child 查询,返回父文档
  POST my_blogs/_search
  {
    "query": {
      "has_child": {
        "type": "comment",
        "query" : {
                  "match": {
                      "username" : "Jack"
                  }
              }
      }
    }
  }
  
  
  # Has Parent 查询，返回相关的子文档
  POST my_blogs/_search
  {
    "query": {
      "has_parent": {
        "parent_type": "blog",
        "query" : {
                  "match": {
                      "title" : "Learning Hadoop"
                  }
              }
      }
    }
  }
  
  
  
  #通过ID ，访问子文档
  GET my_blogs/_doc/comment3
  #通过ID和routing ，访问子文档
  GET my_blogs/_doc/comment3?routing=blog2
  
  #更新子文档
  PUT my_blogs/_doc/comment3?routing=blog2
  {
      "comment": "Hello Hadoop??",
      "blog_comments_relation": {
        "name": "comment",
        "parent": "blog2"
      }
  }
  ```

  

##### 父子文档参考：

- https://www.elastic.co/guide/en/elasticsearch/reference/7.1/query-dsl-has-child-query.html
- https://www.elastic.co/guide/en/elasticsearch/reference/7.1/query-dsl-has-parent-query.html
- https://www.elastic.co/guide/en/elasticsearch/reference/7.1/query-dsl-parent-id-query.html
- https://www.elastic.co/guide/en/elasticsearch/reference/7.1/query-dsl-parent-id-query.html

##### 6、数据建模

- 字段类型：Text VS Keyword

  - Text用于全文本字段，文本会被Analyzer分词，默认不支持聚合分析及排序。需要设置fielddata为true
  - Keyword用于ID，枚举及不需要分词的文本。如电话号码，email地址，手机号码，邮政编码，性别等，适用于Filter，Sorting和Aggregation
  - 设置多字段类型
    - 默认会为文本类型设置成text，并且设置一个keyword的子字段

- 检索

  - 如不需要检索，排序和聚合分析，Enable设置成false
  - 如不需要检索，Index设置成false
  - 对需要检索的字段，可以通过下面配置设定存储粒度，Index_options/Norms

- 聚合及排序

  - 不需要排序或者聚合分析功能，Doc_values/ fielddata设置成false
  - 更新频繁，聚合查询频繁的keyword类型的字段，推荐将eager_global_ordinals设置成true  

- 额外存储

  - store设置成true，可以存储该字段的原始内容

  - 一般结合_source的enabled为false时候使用

  - disable _source:节约磁盘；适用于指标性数据

    - 一般建议考虑增加压缩比
    - 无法看到_source字段，无法做reindex，无法做update

  - ```
    "_source": {"enabled": false},
    "author" : {"type" : "keyword","store": true},
    
    #查询结果中，Source不包含数据
    POST books/_search
    {}
    
    #搜索，通过store 字段显示数据，同时高亮显示 conent的内容
    POST books/_search
    {
      "stored_fields": ["title","author","public_date"],
      "query": {
        "match": {
          "content": "searching"
        }
      },
    
      "highlight": {
        "fields": {
          "content":{}
        }
      }
    
    }
    ```

  - Demo

    ```
    ###### Data Modeling Example
    
    # Index 一本书的信息
    PUT books/_doc/1
    {
      "title":"Mastering ElasticSearch 5.0",
      "description":"Master the searching, indexing, and aggregation features in ElasticSearch Improve users’ search experience with Elasticsearch’s functionalities and develop your own Elasticsearch plugins",
      "author":"Bharvi Dixit",
      "public_date":"2017",
      "cover_url":"https://images-na.ssl-images-amazon.com/images/I/51OeaMFxcML.jpg"
    }
    
    #查询自动创建的Mapping
    GET books/_mapping
    
    DELETE books
    
    #优化字段类型
    PUT books
    {
          "mappings" : {
          "properties" : {
            "author" : {"type" : "keyword"},
            "cover_url" : {"type" : "keyword","index": false},
            "description" : {"type" : "text"},
            "public_date" : {"type" : "date"},
            "title" : {
              "type" : "text",
              "fields" : {
                "keyword" : {
                  "type" : "keyword",
                  "ignore_above" : 100
                }
              }
            }
          }
        }
    }
    
    #Cover URL index 设置成false，无法对该字段进行搜索
    POST books/_search
    {
      "query": {
        "term": {
          "cover_url": {
            "value": "https://images-na.ssl-images-amazon.com/images/I/51OeaMFxcML.jpg"
          }
        }
      }
    }
    
    #Cover URL index 设置成false，依然支持聚合分析
    POST books/_search
    {
      "aggs": {
        "cover": {
          "terms": {
            "field": "cover_url",
            "size": 10
          }
        }
      }
    }
    
    
    DELETE books
    #新增 Content字段。数据量很大。选择将Source 关闭
    PUT books
    {
          "mappings" : {
          "_source": {"enabled": false},
          "properties" : {
            "author" : {"type" : "keyword","store": true},
            "cover_url" : {"type" : "keyword","index": false,"store": true},
            "description" : {"type" : "text","store": true},
             "content" : {"type" : "text","store": true},
            "public_date" : {"type" : "date","store": true},
            "title" : {
              "type" : "text",
              "fields" : {
                "keyword" : {
                  "type" : "keyword",
                  "ignore_above" : 100
                }
              },
              "store": true
            }
          }
        }
    }
    
    
    # Index 一本书的信息,包含Content
    PUT books/_doc/1
    {
      "title":"Mastering ElasticSearch 5.0",
      "description":"Master the searching, indexing, and aggregation features in ElasticSearch Improve users’ search experience with Elasticsearch’s functionalities and develop your own Elasticsearch plugins",
      "content":"The content of the book......Indexing data, aggregation, searching.    something else. something in the way............",
      "author":"Bharvi Dixit",
      "public_date":"2017",
      "cover_url":"https://images-na.ssl-images-amazon.com/images/I/51OeaMFxcML.jpg"
    }
    
    #查询结果中，Source不包含数据
    POST books/_search
    {}
    
    #搜索，通过store 字段显示数据，同时高亮显示 conent的内容
    POST books/_search
    {
      "stored_fields": ["title","author","public_date"],
      "query": {
        "match": {
          "content": "searching"
        }
      },
    
      "highlight": {
        "fields": {
          "content":{}
        }
      }
    
    }
    ```

##### Mapping字段相关配置参考：

- Enable - 设置成false，仅做存储，不支持搜索和聚合分析（数据保存在_source中）
- Index - 是否构倒排索引，设置成false，无法被搜索，但还是支持aggregation，并出现在_source中
- norms - 如果字段用来过滤和聚合分析，可以关闭，节约存储
- doc_values - 是否启用doc_values，用于排序和聚合分析
- field_data - 如果要对text类型启用排序和聚合分析，fielddata需要设置成true
- store - 默认不存才，数据默认存储在_source
- coerce - 默认开启，是否开启数据类型的自动转换（列如：字符串转数字）
- multifields - 多字段特性
- dynamic - true/false/strict控制Mapping的自动更新

- https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-params.html

##### 7、最佳实践

###### 1、使用嵌套字段使得字段不会不断增加

- ###### Kibana目前暂不支持nested类型和parent/child类型

- 一个例子：Cookie Service的数据，cookie中的字段会不断增加

  ```
  ##索引数据，dynamic mapping 会不断加入新增字段
  PUT cookie_service/_doc/1
  {
   "url":"www.google.com",
   "cookies":{
     "username":"tom",
     "age":32
   }
  }
  
  PUT cookie_service/_doc/2
  {
   "url":"www.amazon.com",
   "cookies":{
     "login":"2019-01-01",
     "email":"xyz@abc.com"
   }
  }
  
  DELETE cookie_service
  #使用 Nested 对象，增加key/value
  PUT cookie_service
  {
    "mappings": {
      "properties": {
        "cookies": {
          "type": "nested",
          "properties": {
            "name": {
              "type": "keyword"
            },
            "dateValue": {
              "type": "date"
            },
            "keywordValue": {
              "type": "keyword"
            },
            "IntValue": {
              "type": "integer"
            }
          }
        },
        "url": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword",
              "ignore_above": 256
            }
          }
        }
      }
    }
  }
  
  
  ##写入数据，使用key和合适类型的value字段
  PUT cookie_service/_doc/1
  {
   "url":"www.google.com",
   "cookies":[
      {
        "name":"username",
        "keywordValue":"tom"
      },
      {
         "name":"age",
        "intValue":32
  
      }
  
     ]
   }
  
  
  PUT cookie_service/_doc/2
  {
   "url":"www.amazon.com",
   "cookies":[
      {
        "name":"login",
        "dateValue":"2019-01-01"
      },
      {
         "name":"email",
        "IntValue":32
  
      }
  
     ]
   }
  
  
  # Nested 查询，通过bool查询进行过滤
  POST cookie_service/_search
  {
    "query": {
      "nested": {
        "path": "cookies",
        "query": {
          "bool": {
            "filter": [
              {
              "term": {
                "cookies.name": "age"
              }},
              {
                "range":{
                  "cookies.intValue":{
                    "gte":30
                  }
                }
              }
            ]
          }
        }
      }
    }
  }
  
  
  
  # 在Mapping中加入元信息，便于管理
  PUT softwares/
  {
    "mappings": {
      "_meta": {
        "software_version_mapping": "1.0"
      }
    }
  }
  
  GET softwares/_mapping
  PUT softwares/_doc/1
  {
    "software_version":"7.1.0"
  }
  
  DELETE softwares
  # 优化,使用inner object
  PUT softwares/
  {
    "mappings": {
      "_meta": {
        "software_version_mapping": "1.1"
      },
      "properties": {
        "version": {
          "properties": {
            "display_name": {
              "type": "keyword"
            },
            "hot_fix": {
              "type": "byte"
            },
            "marjor": {
              "type": "byte"
            },
            "minor": {
              "type": "byte"
            }
          }
        }
      }
    }
  }
  
  
  #通过 Inner Object 写入多个文档
  PUT softwares/_doc/1
  {
    "version":{
    "display_name":"7.1.0",
    "marjor":7,
    "minor":1,
    "hot_fix":0  
    }
  
  }
  
  PUT softwares/_doc/2
  {
    "version":{
    "display_name":"7.2.0",
    "marjor":7,
    "minor":2,
    "hot_fix":0  
    }
  }
  
  PUT softwares/_doc/3
  {
    "version":{
    "display_name":"7.2.1",
    "marjor":7,
    "minor":2,
    "hot_fix":1  
    }
  }
  
  
  # 通过 bool 查询，
  POST softwares/_search
  {
    "query": {
      "bool": {
        "filter": [
          {
            "match":{
              "version.marjor":7
            }
          },
          {
            "match":{
              "version.minor":2
            }
          }
  
        ]
      }
    }
  }
  
  
  
  
  # Not Null 解决聚合的问题
  DELETE ratings
  PUT ratings
  {
    "mappings": {
        "properties": {
          "rating": {
            "type": "float",
            "null_value": 1.0
          }
        }
      }
  }
  
  
  PUT ratings/_doc/1
  {
   "rating":5
  }
  PUT ratings/_doc/2
  {
   "rating":null
  }
  
  
  POST ratings/_search
  POST ratings/_search
  {
    "size": 0,
    "aggs": {
      "avg": {
        "avg": {
          "field": "rating"
        }
      }
    }
  }
  
  POST ratings/_search
  {
    "query": {
      "term": {
        "rating": {
          "value": 1
        }
      }
    }
  }
  ```

###### 2、避免正则查询

- 正则，通配符查询，前缀查询属于Term查询，但是性能不够好

- 特别是将通配符放在开头，会导致性能的灾难

  - 案例：文档中某个字段包含了Elasticsearch的版本信息，例如version: "7.1.0"

  - 搜索所有是bug fix的版本？每个主要版本号所关联的文档

  - 解决方案：将字符串转换为对象

    ```
    # 优化,使用inner object
    PUT softwares/
    {
      "mappings": {
        "_meta": {
          "software_version_mapping": "1.1"
        },
        "properties": {
          "version": {
            "properties": {
              "display_name": {
                "type": "keyword"
              },
              "hot_fix": {
                "type": "byte"
              },
              "marjor": {
                "type": "byte"
              },
              "minor": {
                "type": "byte"
              }
            }
          }
        }
      }
    }
    
    #通过 Inner Object 写入多个文档
    PUT softwares/_doc/1
    {
      "version":{
      "display_name":"7.1.0",
      "marjor":7,
      "minor":1,
      "hot_fix":0  
      }
    
    }
    
    #通过 Inner Object 写入多个文档
    PUT softwares/_doc/1
    {
      "version":{
      "display_name":"7.1.0",
      "marjor":7,
      "minor":1,
      "hot_fix":0  
      }
    
    }
    
    PUT softwares/_doc/2
    {
      "version":{
      "display_name":"7.2.0",
      "marjor":7,
      "minor":2,
      "hot_fix":0  
      }
    }
    
    PUT softwares/_doc/3
    {
      "version":{
      "display_name":"7.2.1",
      "marjor":7,
      "minor":2,
      "hot_fix":1  
      }
    }
    
    
    # 通过 bool 查询，
    POST softwares/_search
    {
      "query": {
        "bool": {
          "filter": [
            {
              "match":{
                "version.marjor":7
              }
            },
            {
              "match":{
                "version.minor":2
              }
            }
    
          ]
        }
      }
    }
    ```

###### 3、 避免控制引起聚合不准，null_value

```
# Not Null 解决聚合的问题
DELETE ratings
PUT ratings
{
  "mappings": {
      "properties": {
        "rating": {
          "type": "float",
          "null_value": 1.0
        }
      }
    }
}


PUT ratings/_doc/1
{
 "rating":5
}
PUT ratings/_doc/2
{
 "rating":null
}


POST ratings/_search
POST ratings/_search
{
  "size": 0,
  "aggs": {
    "avg": {
      "avg": {
        "field": "rating"
      }
    }
  }
}

POST ratings/_search
{
  "query": {
    "term": {
      "rating": {
        "value": 1
      }
    }
  }
}
```



###### 参考：

- https://www.elastic.co/guide/en/elasticsearch/reference/7.1/general-recommendations.html
- https://www.elastic.co/guide/en/elasticsearch/reference/7.1/tune-for-disk-usage.html
- https://www.elastic.co/guide/en/elasticsearch/reference/7.1/tune-for-search-speed.html

###### 4、为索引的Mapping加入Meta信息

- Mappings设置时一个迭代的过程

  - 加入新的字段很容易（必要时需要update_by_query）

  - 更新删除字段不允许（需要Reindex重建数据）

  - 最好能对Mappings加入Meta信息，更好的精心版本管理

  - 可以考虑将Mapping文件上传git进行管理

  - ```
    PUT softwares/
    {
      "mappings":{
        "_meta":{
          "software_version_mapping":"1.0"
        }
      }
    
    }
    ```

    

#### 8、生命周期Shrink与RolloverAPI

##### 1、索引管理

- Open/Close Index：索引关闭后无法进行读写，但是索引数据不会被删除
- Shrink Index：可以将索引的主分片数收缩到较小的值
- Split Index：可以扩大主分片个数
- Rollover Index：类似Log4j记录日志的方式，索引尺寸或者时间超过一定值后，创建新的
- Rollup Index: 对数据进行处理后，重新写入，较少数据量

##### 2、Shrink API

- 源分片数必须时目标分片数的倍数。如果源分片数是素数，目标分片数只能为 1
- 如果文件系统支持硬链接，会将Segments硬链接到目标索引，所以性能好
- 完成后，可以删除源索引
- 要求：
  - 分片必须只读，（    "index.blocks.write": true）
  - 所有的分片必须在同一个节点上（"index.routing.allocation.include.box_type":"hot"）
  - 集群健康状态为Green

##### 3、Split API，跟Shrink相反的操作

##### 4、Rollover API

- 当满足一系列条件，Rollover API支持将一个Alias指向一个新的索引
  - 存活的时间、最大文档数、最大的文件尺寸
- 一般需要和Index Lifecycle Management 结合使用
  - 只有调用RolloverAPI时，才会去做相应的检测。ES并不会自动去监控这些索引
- 会保留旧的Index
  - "index.blocks.write": true

##### Demo

```
# 打开关闭索引
DELETE test
#查看索引是否存在
HEAD test

PUT test/_doc/1
{
  "key":"value"
}

#关闭索引
POST /test/_close
#索引存在
HEAD test
# 无法查询
POST test/_count

#打开索引
POST /test/_open
POST test/_search
{
  "query": {
    "match_all": {}
  }
}
POST test/_count


# 在一个 hot-warm-cold的集群上进行测试
GET _cat/nodes
GET _cat/nodeattrs

DELETE my_source_index
DELETE my_target_index
PUT my_source_index
{
 "settings": {
   "number_of_shards": 4,
   "number_of_replicas": 0
 }
}

PUT my_source_index/_doc/1
{
  "key":"value"
}

GET _cat/shards/my_source_index

# 分片数3，会失败
POST my_source_index/_shrink/my_target_index
{
  "settings": {
    "index.number_of_replicas": 0,
    "index.number_of_shards": 3,
    "index.codec": "best_compression"
  },
  "aliases": {
    "my_search_indices": {}
  }
}



# 报错，因为没有置成 readonly
POST my_source_index/_shrink/my_target_index
{
  "settings": {
    "index.number_of_replicas": 0,
    "index.number_of_shards": 2,
    "index.codec": "best_compression"
  },
  "aliases": {
    "my_search_indices": {}
  }
}

#将 my_source_index 设置为只读
PUT /my_source_index/_settings
{
  "settings": {
    "index.blocks.write": true
  }
}

# 报错，必须都在一个节点
POST my_source_index/_shrink/my_target_index
{
  "settings": {
    "index.number_of_replicas": 0,
    "index.number_of_shards": 2,
    "index.codec": "best_compression"
  },
  "aliases": {
    "my_search_indices": {}
  }
}

DELETE my_source_index
## 确保分片都在 hot
PUT my_source_index
{
 "settings": {
   "number_of_shards": 4,
   "number_of_replicas": 0,
   "index.routing.allocation.include.box_type":"hot"
 }
}

PUT my_source_index/_doc/1
{
  "key":"value"
}

GET _cat/shards/my_source_index

#设置为只读
PUT /my_source_index/_settings
{
  "settings": {
    "index.blocks.write": true
  }
}


POST my_source_index/_shrink/my_target_index
{
  "settings": {
    "index.number_of_replicas": 0,
    "index.number_of_shards": 2,
    "index.codec": "best_compression"
  },
  "aliases": {
    "my_search_indices": {}
  }
}


GET _cat/shards/my_target_index

# My target_index状态为也只读
PUT my_target_index/_doc/1
{
  "key":"value"
}



# Split Index
DELETE my_source_index
DELETE my_target_index

PUT my_source_index
{
 "settings": {
   "number_of_shards": 4,
   "number_of_replicas": 0
 }
}

PUT my_source_index/_doc/1
{
  "key":"value"
}

GET _cat/shards/my_source_index

# 必须是倍数
POST my_source_index/_split/my_target
{
  "settings": {
    "index.number_of_shards": 10
  }
}

# 必须是只读
POST my_source_index/_split/my_target
{
  "settings": {
    "index.number_of_shards": 8
  }
}


#设置为只读
PUT /my_source_index/_settings
{
  "settings": {
    "index.blocks.write": true
  }
}


POST my_source_index/_split/my_target_index
{
  "settings": {
    "index.number_of_shards": 8,
    "index.number_of_replicas":0
  }
}

GET _cat/shards/my_target_index



# write block
PUT my_target_index/_doc/1
{
  "key":"value"
}



#Rollover API
DELETE nginx-logs*
# 不设定 is_write_true
# 名字符合命名规范
PUT /nginx-logs-000001
{
  "aliases": {
    "nginx_logs_write": {}
  }
}

# 多次写入文档
POST nginx_logs_write/_doc
{
  "log":"something"
}


POST /nginx_logs_write/_rollover
{
  "conditions": {
    "max_age":   "1d",
    "max_docs":  5,
    "max_size":  "5gb"
  }
}

GET /nginx_logs_write/_count
# 查看 Alias信息
GET /nginx_logs_write


DELETE apache-logs*


# 设置 is_write_index
PUT apache-logs1
{
  "aliases": {
    "apache_logs": {
      "is_write_index":true
    }
  }
}
POST apache_logs/_count

POST apache_logs/_doc
{
  "key":"value"
}

# 需要指定 target 的名字
POST /apache_logs/_rollover/apache-logs8xxxx
{
  "conditions": {
    "max_age":   "1d",
    "max_docs":  1,
    "max_size":  "5gb"
  }
}


# 查看 Alias信息
GET /apache_logs
```

##### 参考：

- https://www.elastic.co/guide/en/elasticsearch/reference/7.1/indices-shrink-index.html
- https://www.elastic.co/guide/en/elasticsearch/reference/7.1/indices-rollover-index.html

##### 5、索引生命周期常见阶段

- Hot=>Warm=>Cold=>Delete
- Hot：索引还存在着大量的读写操作
- Warm: 索引不存在写操作，还有被查询的需要
- Cold: 数据不存在写操作，读操作也不多
- Delete:s索引不再需要，可以被安全删除
- Elasticsearch Curator
  - Elastic官方推出的工具，基于python的命令行工具，配置Actions，每个动作可以顺序执行，Filters支持多种条件
  - https://www.elastic.co/guide/en/elasticsearch/client/curator/current/index.html
- ![image-20210709110447563](F:\note\ES学习大纲.assets\image-20210709110447563.png)

- 创建一个life cycle policy

  - [Stack Management](http://localhost:5601/app/management/)/Index Lifecycle Policies

- Demo

  ```
  # 运行三个节点，分片 将box_type设置成 hot，warm和cold
  # 具体参考 github下，docker-hot-warm-cold 下的docker-compose 文件
  
  DELETE *
  
  # 设置 1秒刷新1次，生产环境10分种刷新一次
  PUT _cluster/settings
  {
    "persistent": {
      "indices.lifecycle.poll_interval":"1s"
    }
  }
  
  # 设置 Policy
  PUT /_ilm/policy/log_ilm_policy
  {
    "policy": {
      "phases": {
        "hot": {
          "actions": {
            "rollover": {
              "max_docs": 5
            }
          }
        },
        "warm": {
          "min_age": "10s",
          "actions": {
            "allocate": {
              "include": {
                "box_type": "warm"
              }
            }
          }
        },
        "cold": {
          "min_age": "15s",
          "actions": {
            "allocate": {
              "include": {
                "box_type": "cold"
              }
            }
          }
        },
        "delete": {
          "min_age": "20s",
          "actions": {
            "delete": {}
          }
        }
      }
    }
  }
  
  
  
  # 设置索引模版
  PUT /_template/log_ilm_template
  {
    "index_patterns" : [
        "ilm_index-*"
      ],
      "settings" : {
        "index" : {
          "lifecycle" : {
            "name" : "log_ilm_policy",
            "rollover_alias" : "ilm_alias"
          },
          "routing" : {
            "allocation" : {
              "include" : {
                "box_type" : "hot"
              }
            }
          },
          "number_of_shards" : "1",
          "number_of_replicas" : "0"
        }
      },
      "mappings" : { },
      "aliases" : { }
  }
  
  
  
  #创建索引
  PUT ilm_index-000001
  {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0,
      "index.lifecycle.name": "log_ilm_policy",
      "index.lifecycle.rollover_alias": "ilm_alias",
      "index.routing.allocation.include.box_type":"hot"
    },
    "aliases": {
      "ilm_alias": {
        "is_write_index": true
      }
    }
  }
  
  # 对 Alias写入文档
  POST  ilm_alias/_doc
  {
    "dfd":"dfdsf"
  }
  ```

  

#### 9、Hot、Warm数据

##### 1、两类数据节点，不同的硬件配置

- Hot节点（通常使用SSD）：频繁查询的数据，比较新的数据
- Warm节点（通常使用HDD）：查询比较少的数据，比较旧的数据

##### 2、Hot 节点

- 用于数据的写入：
  - Indexing对CPU和IO都有很高的要求。所以需要使用高配置的机器
  - 存储新能要好。建议使用SSD

##### 3、Warm节点

- 用于保存制度的索引，比较旧的数据，通常使用大容量的磁盘

##### 4、配置Hot&Warm Architecture

- 使用Shard Filtering：

  - 标记节点：（Tagging）
  - 配置索引到Hot Node
  - 配置索引到Warm节点

- 标记节点

  - 通过“node.attr”来标记一个节点

    ```
    GET _cat/nodeattrs?V
    #命令行
    -E node.attr.my_node_type=hot
    -E node.attr.my_node_type=warm
    
    #修改elasticsearch.yml
    ```

  - 创建索引的时候，指定将其创建在节点上

    ```
    # 配置到 Hot节点
    PUT logs-2019-06-27
    {
      "settings":{
        "number_of_shards":2,
        "number_of_replicas":0,
        "index.routing.allocation.require.my_node_type":"hot"
      }
    }
    ```

  - 旧数据移动到Warm节点

    - index.routing.allocation是一个索引级的dynamic setting，可以通过API在后期进行设定

      - Curator/Index life Cycle Management Tool

      ```
      GET _cat/shards?v
      
      # 配置到 warm 节点
      PUT PUT logs-2019-06-27/_settings
      {  
        "index.routing.allocation.require.my_node_type":"warm"
      }
      ```

  - Rack Awareness:将主分片和副本分片发送到不同的数据中兴

    - ``-E node.attr.my_rack_id=rack2``

    - ```
      PUT _cluster/settings
      {
        "persistent": {
          "cluster.routing.allocation.awareness.attributes": "my_rack_id"
        }
      }
      ```

  - Forced Awareness：必须将主副分片在不同的数据中心，否则不能分配分片

    - ```
      # 标记一个 rack 1
      bin/elasticsearch  -E node.name=node1 -E cluster.name=my_cluster -E path.data=node1_data -E node.attr.my_rack_id=rack1
      
      # 标记一个 rack 1，数据不能插入，只能标记rack 2
      bin/elasticsearch  -E node.name=node2 -E cluster.name=my_cluster -E path.data=node2_data -E node.attr.my_rack_id=rack2
      
      PUT _cluster/settings
      {
        "persistent": {
          "cluster.routing.allocation.awareness.attributes": "my_rack_id",
          "cluster.routing.allocation.awareness.force.my_rack_id.values": "rack1,rack2"
        }
      }
      ```

  - Shard Filtering

    - node.attr - 标记节点

    - index.routing。allocation - 分配索引到节点

      - | 设置                                    | 分配索引到节点，节点的属性规则 |
        | --------------------------------------- | ------------------------------ |
        | index.routing.allocation.include.{attr} | 至少包含一个值                 |
        | index.routing.allocation.exclude.{attr} | 不能包含任何一个值             |
        | index.routing.allocation.require.{attr} | 所有值都需要包含               |

      

  - Demo

    ```
    #启动hot节点
    bin/elasticsearch -E node.name=hostnode -E cluster.name=my_cluster -E path.data=hot_data -E node.attr.my_node_type=hot
    
    #启动warm节点
    bin/elasticsearch -E node.name=warmnode -E cluster.name=my_cluster -E path.data=warm_data -E node.attr.my_node_type=warm
    
    http://localhost:9200/_cat/nodeattrs
    
    # 查看节点
    GET /_cat/nodeattrs?v
    
    # 配置到 Hot节点
    PUT logs-2019-06-27
    {
      "settings":{
        "number_of_shards":2,
        "number_of_replicas":0,
        "index.routing.allocation.require.my_node_type":"hot"
      }
    }
    
    PUT my_index1/_doc/1
    {
      "key":"value"
    }
    
    GET _cat/shards?v
    
    # 配置到 warm 节点
    PUT PUT logs-2019-06-27/_settings
    {  
      "index.routing.allocation.require.my_node_type":"warm"
    }
    
    
    # 标记一个 rack 1
    bin/elasticsearch  -E node.name=node1 -E cluster.name=my_cluster -E path.data=node1_data -E node.attr.my_rack_id=rack1
    
    # 标记一个 rack 2
    bin/elasticsearch  -E node.name=node2 -E cluster.name=my_cluster -E path.data=node2_data -E node.attr.my_rack_id=rack2
    
    PUT _cluster/settings
    {
      "persistent": {
        "cluster.routing.allocation.awareness.attributes": "my_rack_id"
      }
    }
    
    PUT my_index1
    {
      "settings":{
        "number_of_shards":2,
        "number_of_replicas":1
      }
    }
    
    PUT my_index1/_doc/1
    {
      "key":"value"
    }
    
    GET _cat/shards?v
    DELETE my_index1/_doc/1
    
    # Fore awareness
    # 标记一个 rack 1
    bin/elasticsearch  -E node.name=node1 -E cluster.name=my_cluster -E path.data=node1_data -E node.attr.my_rack_id=rack1
    
    # 标记一个 rack 2
    bin/elasticsearch  -E node.name=node2 -E cluster.name=my_cluster -E path.data=node2_data -E node.attr.my_rack_id=rack1
    
    PUT _cluster/settings
    {
      "persistent": {
        "cluster.routing.allocation.awareness.attributes": "my_rack_id",
        "cluster.routing.allocation.awareness.force.my_rack_id.values": "rack1,rack2"
      }
    }
    GET _cluster/settings
    
    # 集群黄色
    GET _cluster/health
    
    # 副本无法分配
    GET _cat/shards?v
    
    
    GET _cluster/allocation/explain?pretty
    ```

##### 5、参考：

- https://www.elastic.co/cn/blog/sizing-hot-warm-architectures-for-logging-and-metrics-in-the-elasticsearch-service-on-elastic-cloud
- https://www.elastic.co/cn/blog/deploying-a-hot-warm-logging-cluster-on-the-elasticsearch-service

#### 10、节点类型部署

##### 1、不同角色的节点

- Master eligible 、 Data、Ingest、Coordinating、Maching Learning

- 一个节点在默认情况下同时扮演：master eligibale、data node、ingest node

  - | 节点类型          | 配置参数    | 默认值                      |
    | ----------------- | ----------- | --------------------------- |
    | master eligible   | node.master | true                        |
    | data              | node.data   | true                        |
    | ingest            | node.ingest | true                        |
    | coordinating only | 无          | 设置上面三个参数全部为false |
    | machine learning  | node.ml     | true（需要enable x-pack）   |

- 单一职责，一个节点承担一个角色
  - Master节点
    - node.master:true、node.ingest：false、node.data: false
  - Data节点
    - node.master:false、node.ingest：false、node.data: true
  - Ingest节点
    - node.master:false、node.ingest：true、node.data: false
  - Coordinate节点
    - node.master:false、node.ingest：false、node.data: false
- 单一角色：职责分离的好处
  - **Dedicated master eligible nodes：负责集群状态（cluster state）的管理**
    - 使用低配置的CPU，RAM和磁盘
  - **Dedicated data nodes: 负责数据存储及处理客户端请求**
    - 使用高配置的CPU，RAM和磁盘
  - **Dedicated ingest nodes：负责数据处理**
    - 使用高配置的CPU；中等RAM；低配置的磁盘
  - **Dedicate Coordinating only node（Client Node）**
    - 使用中等好高配置CPU;  使用中等或高配置Ram； 低配置磁盘
    - **扮演Load Balancers。降低Master和Data Nodes的负载**
    - 负责搜索结果的Gather/Reduce
    - 有时候无法预知客户端会发送怎么样的请求
      - 大量占用内存的聚合操作，一个深度聚合可能会引发OOM
- Dedicate Master Node的好处
  - 从高可用&避免脑裂的角度出发
    - 一般生成环境配置3台
    - 一个集群只有1台活跃的主节点
      - 负责分片管理，索引创建，集群管理等操作
  - 如果和数据节点或者Coordinate节点混合部署
    - 数据节点相对有比较大的内存占用
    - Coordinate节点有时候可能会有开销很高的查询，导致OOM
    - 这些都有可能影响Master节点，导致集群的不稳定
- 部署方案，读写分离：
- ![image-20210708211552927](F:\note\ES学习大纲.assets\image-20210708211552927.png)

- 在集群中部署Kibana
- ![image-20210708211704105](F:\note\ES学习大纲.assets\image-20210708211704105.png)

- 异地多活的部署方案：Cross data center replication
- ![image-20210708211812509](F:\note\ES学习大纲.assets\image-20210708211812509.png)

#### 11、ES安全

#### 12、分片设计与管理

- 从存储的物理角度看
  - 日志类应用，单个分片不要大于50GB
  - 搜索应用，单个分片不要超过20GB
- 为什么要控制分片存储大小
  - 提供Update的性能
  - Merge时，减少所需的资源
  - 丢失节点后，具备更快的恢复速度、便于分片的集群内Rebalancing

#### 13、容量规划

- 写入时间序列的数据：基于Date Math的方式

- 2019-08-01T00:00:00

- | <logs-{now/d}>        | logs-2019.08.01                   |
  | --------------------- | --------------------------------- |
  | <logs-{now{YYYY.MM}}> | logs-2019.08                      |
  | <logs-{now/w}>        | logs-2019.07.29(这周最开始的时间) |

- ```
  # POST /<logs-{now/d}/_search
  POST /%3Clogs-%7Bnow%2Fd%7D%3E/_search
  
  # POST /<logs-{now/w}/_search
  POST /%3Clogs-%7Bnow%2Fw%7D%3E/_search
  ```

- 创建索引，每天/每周/每月，在索引的名字中增加时间信息

  ```
  POST _aliases
  {
    "actions": [
      {
        "add": {
          "index": "logs_2019-06-27",
          "alias": "logs_write"
        }
      },
      {
        "remove": {
          "index": "logs_2019-06-26",
          "alias": "logs_write"
        }
      }
    ]
  }
  ```

#### 14、监控

- X-Pach提供了免费集群监控的功能

  - Xpack.monitoring.collection.interval默认设置10秒

  - 建议搭建dedicated集群用于ES集群的监控

    - | 配置项                               | 含义                                             |
      | ------------------------------------ | ------------------------------------------------ |
      | xpack.monitoring.collection.indices  | 默认监控所有索引。支持配置索引，列表（逗号间隔） |
      | xpack.monitoring.collection.interval | 手机数据的时间间隔，默认10秒                     |
      | xpack.monitoring.history.duration    | 数据保留的时间，默认7天                          |

    - 参考：https://www.elastic.co/guide/en/x-pack/current/monitoring-settings.html

- Watcher for Alerting

  - 需要Gold账户
  - 一个Watcher由5个部分组成
    - Trigger：多久被触发一次
    - Input：查询条件（在所有日志索引中查看“ERROR”相关）
    - Condition：查询是否满足条件（例如：大于1000条返回）
    - Actions：执行相关操作（例如：发邮件）

#### 15、故障转移

- 当3个master elligible时，设置discovery.zen.minimum_master_nodes为2，即可避免脑裂 

- 跨集群搜索

  ```
  //启动3个集群
  
  bin/elasticsearch -E node.name=cluster0node -E cluster.name=cluster0 -E path.data=cluster0_data -E discovery.type=single-node -E http.port=9200 -E transport.port=9300
  bin/elasticsearch -E node.name=cluster1node -E cluster.name=cluster1 -E path.data=cluster1_data -E discovery.type=single-node -E http.port=9201 -E transport.port=9301
  bin/elasticsearch -E node.name=cluster2node -E cluster.name=cluster2 -E path.data=cluster2_data -E discovery.type=single-node -E http.port=9202 -E transport.port=9302
  
  
  //在每个集群上设置动态的设置
  PUT _cluster/settings
  {
    "persistent": {
      "cluster": {
        "remote": {
          "cluster0": {
            "seeds": [
              "127.0.0.1:9300"
            ],
            "transport.ping_schedule": "30s"
          },
          "cluster1": {
            "seeds": [
              "127.0.0.1:9301"
            ],
            "transport.compress": true,
            "skip_unavailable": true
          },
          "cluster2": {
            "seeds": [
              "127.0.0.1:9302"
            ]
          }
        }
      }
    }
  }
  
  #cURL
  curl -XPUT "http://localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
  {"persistent":{"cluster":{"remote":{"cluster0":{"seeds":["127.0.0.1:9300"],"transport.ping_schedule":"30s"},"cluster1":{"seeds":["127.0.0.1:9301"],"transport.compress":true,"skip_unavailable":true},"cluster2":{"seeds":["127.0.0.1:9302"]}}}}}'
  
  curl -XPUT "http://localhost:9201/_cluster/settings" -H 'Content-Type: application/json' -d'
  {"persistent":{"cluster":{"remote":{"cluster0":{"seeds":["127.0.0.1:9300"],"transport.ping_schedule":"30s"},"cluster1":{"seeds":["127.0.0.1:9301"],"transport.compress":true,"skip_unavailable":true},"cluster2":{"seeds":["127.0.0.1:9302"]}}}}}'
  
  curl -XPUT "http://localhost:9202/_cluster/settings" -H 'Content-Type: application/json' -d'
  {"persistent":{"cluster":{"remote":{"cluster0":{"seeds":["127.0.0.1:9300"],"transport.ping_schedule":"30s"},"cluster1":{"seeds":["127.0.0.1:9301"],"transport.compress":true,"skip_unavailable":true},"cluster2":{"seeds":["127.0.0.1:9302"]}}}}}'
  
  
  #创建测试数据
  curl -XPOST "http://localhost:9200/users/_doc" -H 'Content-Type: application/json' -d'
  {"name":"user1","age":10}'
  
  curl -XPOST "http://localhost:9201/users/_doc" -H 'Content-Type: application/json' -d'
  {"name":"user2","age":20}'
  
  curl -XPOST "http://localhost:9202/users/_doc" -H 'Content-Type: application/json' -d'
  {"name":"user3","age":30}'
  
  
  #查询
  GET /users,cluster1:users,cluster2:users/_search
  {
    "query": {
      "range": {
        "age": {
          "gte": 20,
          "lte": 40
        }
      }
    }
  }
  ```

  - 在Kibana中使用Cross data search https://kelonsoftware.com/cross-cluster-search-kibana/

