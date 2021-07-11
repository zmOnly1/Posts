### ES搜索

#### 排序

##### 单字段排序

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
  ```

##### 多字段排序

- ```
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

##### 查看docvalue_fields数据

- ```
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

#### 分页

##### 深度分页，占用内存

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

##### 避免深度分页，search_after

- ```
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

##### Scroll API，快照数据，不实时

- ```
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

#### 搜索

##### 分词

- ```
  #standard
  GET _analyze
  {
    "analyzer": "standard",
    "text": "2 running Quick brown-foxes leap over lazy dogs in the summer evening."
  }
  
  POST books/_analyze
  {
    "field":"title",
    "text":"Mastering Elasticsearch"
  }
  
  POST /_analyze
  {
    "tokenizer":"standard",
    "filter":["lowercase"],
    "text":"Mastering Elasticsearch"
  }
  ```

##### SearchAPI

- | 语法                   | 范围              |
  | ---------------------- | ----------------- |
  | /_search               | 集群上所有的索引  |
  | /index1/_search        | index1            |
  | /index1,index2/_search | index1和index2    |
  | /index*/_search        | 以index开头的索引 |

- ```
  #URI Query
  GET kibana_sample_data_ecommerce/_search?q=customer_first_name:Eddie
  GET kibana*/_search?q=customer_first_name:Eddie
  GET /_all/_search?q=customer_first_name:Eddie
  
  
  #REQUEST Body
  POST kibana_sample_data_ecommerce/_search
  {
  	"profile": true,
  	"query": {
  		"match_all": {}
  	}
  }
  ```

##### URISearch

- q指定查询语句，使用Query String Syntax
- df默认字段，不指定时，会对所有字段进行查询
- Sort排序/from和size用于分页
- Profile可以查看查询时如何被执行的

- ```
  #基本查询
  GET /movies/_search?q=2012&df=title&sort=year:desc&from=0&size=10&timeout=1s
  
  #带profile，term query
  GET /movies/_search?q=2012&df=title
  {
  	"profile":"true"
  }
  
  #泛查询，正对_all,所有字段,Dis Max Query
  GET /movies/_search?q=2012
  {
  	"profile":"true"
  }
  
  #指定字段,TermQuery
  GET /movies/_search?q=title:2012&sort=year:desc&from=0&size=10&timeout=1s
  {
  	"profile":"true"
  }
  
  # 查找美丽心灵, Mind为泛查询, title:Beautiful=>TermQuery
  #title:Beautiful Mind等效于title:Beautiful OR Mind，
  #Dis Max Query,  所有字段都查询Mind
  GET /movies/_search?q=title:Beautiful Mind
  {
  	"profile":"true"
  }
  
  # 泛查询
  GET /movies/_search?q=title:2012
  {
  	"profile":"true"
  }
  
  #使用引号，Phrase查询，Phrase Query，“Beautiful Mind”等效于Beautiful AND Mind
  GET /movies/_search?q=title:"Beautiful Mind"
  {
  	"profile":"true"
  }
  
  #分组，Bool查询
  #BolleanQuery, title:beautiful title:mind
  #title中包括beautiful或者title包含mind才能被查到
  GET /movies/_search?q=title:(Beautiful Mind)
  {
  	"profile":"true"
  }
  
  
  #布尔操作符
  # 查找美丽心灵
  #BooleanQuery, +title:beautiful +title:mind
  #title里面包含beautiful同时title包含mind
  GET /movies/_search?q=title:(Beautiful AND Mind)
  {
  	"profile":"true"
  }
  
  # 查找美丽心灵
  #BooleanQuery title:beautiful -title:mind
  #title包含beautiful但是不包含mind
  GET /movies/_search?q=title:(Beautiful NOT Mind)
  {
  	"profile":"true"
  }
  
  #分组
  #+表示must
  #-表示must_not
  #title:(+matrix -reloaded)
  
  # 查找美丽心灵
  #%2B=>+
  #BooleanQuery, title:beautiful +title:mind
  GET /movies/_search?q=title:(Beautiful %2BMind)
  {
  	"profile":"true"
  }
  
  #范围查询,[]闭区间，{}开区间
  #year:{2019 TO 2018}
  #year:{* TO 2018}
  #year:>2010
  #year:(>2010 && <=2018)
  #year:(+>2010 +<=2018)
  
  #范围查询 ,区间写法
  GET /movies/_search?q=title:beautiful AND year:[2002 TO 2018%7D
  {
  	"profile":"true"
  }
  
  #通配符查询
  #?代表1个字符，*代表0或多个字符
  #title:mi?d
  #title:be*
  #正则
  #title:[bt]oy
  #模糊匹配与近似查询
  #title:befutifl~1
  #title:"lord rings"~2
  
  #通配符查询
  #title里只有有一个term以b开头
  GET /movies/_search?q=title:b*
  {
  	"profile":"true"
  }
  
  #模糊匹配&近似度匹配
  GET /movies/_search?q=title:beautifl~1
  {
  	"profile":"true"
  }
  #Load of the Rings
  GET /movies/_search?q=title:"Lord Rings"~2
  {
  	"profile":"true"
  }
  ```

- 参考：
  - https://www.elastic.co/guide/en/elasticsearch/reference/7.0/search-uri-request.html
  - https://www.elastic.co/guide/en/elasticsearch/reference/7.0/search-search.html

##### 搜索不存在的索引

- ```
  #ignore_unavailable=true，可以忽略尝试访问不存在的索引“404_idx”导致的报错
  #查询movies分页
  POST /movies,404_idx/_search?ignore_unavailable=true
  {
    "profile": true,
  	"query": {
  		"match_all": {}
  	}
  }
  ```

##### 返回需要的字段

- ```
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
  ```

##### Match Query

- ```
  #or的关系，title:last or title:christmas
  POST movies/_search
  {
    "query": {
      "match": {
        "title": "last christmas"
      }
    }
  }
  #and的关系，title包含last christmas
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
  ```

##### Match phrase Query

- ```
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
  #One I Love
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

##### Query string &Simple Query String

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
  
  #name必须有ruan 和 yiming
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
  #会把AND当成字符串处理
  POST users/_search
  {
    "query": {
      "simple_query_string": {
        "query": "Ruan AND Yiming",
        "fields": ["name"]
      }
    }
  }
  
  #name必须同时包含ruan和yiming，"default_operator": "AND"
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

##### Term Query

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
    "profile": "true",
    "explain": true,
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
  ```

##### Constan_score query

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

  - 日期：

  - | y    | 年   |
    | ---- | ---- |
    | M    | 月   |
    | w    | 周   |
    | d    | 天   |
    | H/h  | 时   |
    | m    | 分   |
    | s    | 秒   |

##### 布尔查询

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

##### 单字符多字段匹配：Dis max Query

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

  

##### 单字符多字段匹配：Mutil match

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

  

#### 参考文档

- https://www.elastic.co/guide/en/elasticsearch/reference/7.0/search-uri-request.html
- https://www.elastic.co/guide/en/elasticsearch/reference/7.0/search-search.html
- https://www.elastic.co/guide/en/elasticsearch/reference/7.1/term-level-queries.html
- https://www.elastic.co/guide/en/elasticsearch/reference/7.1/full-text-queries.html
- https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-exists-query.html 
- https://www.elastic.co/guide/en/elasticsearch/reference/7.1/term-level-queries.html
- https://www.elastic.co/guide/en/elasticsearch/reference/7.1/query-dsl-dis-max-query.html
- https://www.elastic.co/guide/en/elasticsearch/reference/7.1/query-dsl-dis-max-query.html