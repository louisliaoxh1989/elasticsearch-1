
## 概述 

与传统数据库对比：

	关系数据库      ⇒ 数据库(database) ⇒ 表(table)    ⇒ 行(row)         ⇒ 列(column)
	Elasticsearch  ⇒ 索引(index)      ⇒ 类型(type)   ⇒ 文档(document)  ⇒ 字段(field)

一个 Elasticsearch 集群可以包含多个`索引（数据库）`，也就是说其中包含了很多`类型（表）`。这些类型中包含了很多的`文档（行）`，然后每个文档中又包含了很多的`字段（列）`。



## 索引（index）

### 一、新建索引

1. 新建一个普通的索引：

		curl -XPUT http://localhost:9200/index

2. 创建只有一个主分片，没有复制分片的索引：

		curl -XPUT 'localhost:9200/index' -d '
		{
		    "settings": {
		        "number_of_shards" :   1,
		        "number_of_replicas" : 0
		    }
		}'
		
	1. number_of_shards：定义一个索引的主分片个数，默认是 `5`。这个配置在索引创建后不能修改。
	2. number_of_replicas：每个主分片的复制分片个数，默认是 `1`。这个配置可以随时在活跃的索引上修改。

	用 update-index-settings API 动态修改复制分片个数：

		curl -XPUT 'localhost:9200/index/_settings' -d '
		{
		    "number_of_replicas": 1
		}'
		
3. 新建一个索引的同时设置分析器和类型映射的语法格式：

		curl -XPUT 'localhost:9200/index' -d '
		{
		    "settings": { ... any settings ... },
		    "mappings": {
		        "type_one": { ... any mappings ... },
		        "type_two": { ... any mappings ... },
		        ...
		    }
		}'
	
	settings是索引的配置，mappings是索引里面类型的映射。

### 二、删除索引

1. 删除指定索引
	
		curl -XDELETE 'localhost:9200/index'

2. 删除多个索引：

		curl -XDELETE 'localhost:9200/index_one,index_tow'
		curl -XDELETE 'localhost:9200/index_*'

3. 删除全部索引

		curl -XDELETE 'localhost:9200/_all'

## 类型（type）

### 一、新建类型

1. 在建索引的同时指定user和blogpost两种类型：

		curl -XPUT 'http://localhost:9200/index' -d '
		{
		    "mappings":{
		        "user":{
		            "_all":{ "enabled":false },
		            "properties":{
		                "title":{ "type":"string" },
		                "name":{ "type":"string" },
		                "age":{ "type":"integer" }
		            }
		        },
		        "blogpost":{
		            "properties":{
		                "title":{ "type":"string" },
		                "body":{ "type":"string" },
		                "user_id":{ "type":"string", "index":"not_analyzed" },
		                "created":{ "type":"date", "format":"strict_date_optional_time||epoch_millis"}
		            }
		        }
		    }
		}'

2. 建完索引之后再建立并配置类型：

		curl -XPOST http://localhost:9200/index/type/_mapping -d'
		{
		    "type": {
		            "_all": {
		            "analyzer": "ik_max_word",
		            "search_analyzer": "ik_max_word",
		            "term_vector": "no",
		            "store": "false"
		        },
		        "properties": {
		            "content": {
		                "type": "string",
		                "store": "no",
		                "term_vector": "with_positions_offsets",
		                "analyzer": "ik_max_word",
		                "search_analyzer": "ik_max_word",
		                "include_in_all": "true",
		                "boost": 8
		            }
		        }
		    }
		}'
	
3. 那如果没有指定类型呢？那就用默认mapping。例如当我们执行以下命令：

		curl -XPUT http://localhost:9200/index/type/1 -d '
		{
			"name":"zach", 
			"description": "A Pretty cool guy."
		}'

	ES能非常聪明的识别出"name"和"description"字段的类型是string， 并创建以下的mapping。

		mappings: {
		    type: {
		        properties: {
		            description: { type: string }
		            name: { type: string }
		        }
		    }
		}
	
ES的mapping非常类似于静态语言中的数据类型：声明一个变量为int类型的变量， 以后这个变量都只能存储int类型的数据。同样的一个number类型的mapping字段只能存储number类型的数据。同语言的数据类型相比，mapping还有一些其他的含义，mapping不仅告诉ES一个field中是什么类型的值， 它还告诉ES如何索引数据以及数据是否能被搜索到。当你的查询没有返回相应的数据，很有可能mapping有问题。

### 二、删除类型

类型不能被删除，只能在一个新的索引里面重新建立没有该类型的索引，如果仅仅是需要删除改类型下所有文档，可以使用delete-by-query插件。

### 三、更新类型

一般情况下，类型不能够被更新，但可以添加新的属性或者为已存在的字段添加多字段。例如：新建一个index索引，把在tweet类型的message字段设置为string类型。

	curl -XPUT "http://localhost:9200/index" -d '
	{
		"mappings": {
			"tweet": {
				"properties": {
					"message": {
						"type": "string"
					}
				}
			}
		}
	}'
	
然后为index索引添加一个新的类型user，并设置first_name字段:

	curl -XPUT "http://localhost:9200/index/_mapping/user" -d '
	{
		"properties": {
			"first_name": {
				"type": "string"
			}
		}
	}
	
为user类型增加last_name字段：

	curl -XPUT "http://localhost:9200/index/_mapping/user" -d '
	{
		"properties": {
			"last_name": {
				"type": "string"
			}
		}
	}

如果字段本身是一个对象，也可以为改对象添加字段，例如先创建一个拥有user类型的索引：

	curl -XPUT "http://localhost:9200/index' -d '
	{
		"mappings": {
			"user": {
				"properties": {
					"name": {
						"properties": {
							"first": {
								"type": "string"
							}
						}
					},
					"user_id": {
						"type": "string",
						"index": "not_analyzed"
					}
				}
			}
		}
	}'

这里name字段本身就是一个对象，可以name字段增加一个last字段：

	curl -XPUT "http://localhost:9200/index/_mapping/user' -d '
	{
		"properties": {
			"name": {
				"properties": {
					"last": { 
						"type": "string"
					}
				}
			}
		}
	}'


### 四、查询类型

1. 查询指定索引下指定类型

		curl -XGET 'http://localhost:9200/index/_mapping/type'

2. 查询指定索引下多个类型

		curl -XGET 'http://localhost:9200/index/_mapping/tweet,book'

3. 查询多个索引下多个类型

		curl -XGET 'http://localhost:9200/_all/_mapping/tweet,book'

4. 查询所有类型

		curl -XGET 'http://localhost:9200/_mapping'
	
	
## 文档（document）


### 一、新建文档

	curl -XPOST 'http://localhost:9200/megacorp/employee' -d '
	{
		"first_name" : "John",
		"last_name" : "Smith",
		"age" : 25,
		"about" : "I love to go rock climbing",
		"interests" : [ "sports", "music" ]
	}'

elasticsearch会自动帮我们生成一个自动增加的id或者uuid，注意这里的POST不能改成PUT，如果把POST改成PUT那么就会报错：
	
	No handler found for uri [/megacorp/employee] and method [PUT]

如果要新建一个指定id的项目的话，可以使用PUT：

	curl -XPUT 'http://localhost:9200/megacorp/employee/1' -d '
	{
		"first_name" : "Bruce",
		"last_name" : "Lee",
		"age" : 30,
		"about" : "movie star, kongfu",
		"interests" : [ "movie", "sport" ]
	}'

如果id没有被使用，那么就是用给定的id进行创建，如果id已被占用的话，会覆盖原来的数据！所以不能使用这种方式进行指定id的创建，用下面的命令替代：

	curl -XPUT 'http://localhost:9200/megacorp/employee/1/_create' -d '
	{
		"first_name" : "Bruce",
		"last_name" : "Lee",
		"age" : 30,
		"about" : "movie star, kongfu",
		"interests" : [ "movie", "sport" ]
	}'

（_create可以换成?op_type=create）如果查到id已经存在，那么会返回error，http状态码是409。注意最后跟在id后面的_create不能少，如果没带这个的话就会覆盖原来已经存在的！（我发现把上面的PUT换成POST也是可以）


### 二、删除文档
	
删除单个文档：
	
	curl -XDELETE 'http://localhost:9200/megacorp/employee/1'
	
删除某类型下所有文档要用delete-by-query插件

### 三、更新文档

PUT会基于id修改数据，如果这个ID已经存在数据，那么原数据将会被覆盖，如果要局部修改则需要：

	curl -XPOST 'http://localhost:9200/megacorp/employee/1/_update' -d '
	{
		"doc":{
				"age" :         28,
	    		"about":        "I like shopping",
	    		"interests":  [ "watch movie" ]
		}
	}'

注意这里的POST不能换成PUT，否则也会报类似的错误。

	No handler found for uri [/megacorp/employee/1/_update] and method [PUT]%

还要注意，doc里面的内容是被更新的内容，如果有这个字段就更新，如果没有这个字段就添加。

### 四、查询文档

访问elasticsearch文档的模式：

	curl -X<REST Verb> <Node>:<Port>/<Index>/<Type>/<ID>

1. 查询指定文档：
	
		curl -XGET 'localhost:9200/megacorp/employee/1?pretty'
	
	返回结果：

		{
		  "_index" : "megacorp",
		  "_type" : "employee",
		  "_id" : "1",
		  "_version" : 8,
		  "found" : true,
		  "_source" : {
		    "first_name" : "Bruce",
		    "last_name" : "Lee",
		    "age" : 28,
		    "about" : "I like shopping",
		    "interests" : [ "watch movie" ]
		  }
		}
	
	\_index是索引，_type是类型，_id是文档编号，_version是修改次数，found是是否找到，_source是数据详情。
	
2. 查询(索引)(类型)下所有文档：
	
		curl -XGET 'localhost:9200/_search?pretty'
		curl -XGET 'localhost:9200/megacorp/_search?pretty'
		curl -XGET 'localhost:9200/megacorp/employee/_search?pretty'
	
	返回结果：
	
		{	
			"took" : 98,
			  "timed_out" : false,
			  "_shards" : {
			    "total" : 16,
			    "successful" : 16,
			    "failed" : 0
			  },
			  "hits" : {
			    "total" : 30,
			    "max_score" : 1.0,
			    "hits" : [ {
			      "_index" : ".kibana",
			      "_type" : "index-pattern",
			      "_id" : "cn.elixiao.search",
			      "_score" : 1.0,
			      "_source" : {
			      ...
			      },
			      ...
		  }
	
	took是整个搜索花费的毫秒数，time_out是查询是否超时，_shards是参与查询的分片总数(total)中有多少是成功的(successful)或失败的(failed），hits中total是匹配到的文档总数，max_score是给文档做相关性评分的最大值，hits数组是前10条数据。



## 搜索

### 简易搜索（将所有参数放在查询字符串里面）

查询megacorp索引里面employee类型中last_name字段包含Smith的所有结果。

	curl -XGET 'localhost:9200/megacorp/employee/_search?q=last_name:Smith&pretty'

查询megacorp索引里面employee类型中last_name字段包含Smith，first_name字段包含Jane的所有结果。

	curl -XGET 'localhost:9200/megacorp/employee/_search?q=+last_name:Smith+first_name:Jane&pretty'

注意这时候如果last_name有Smith，first_name不是Jane的也会出现在结果里面，这是因为ES的结果是按照相关性排序的，由于last_name中出现了Smith，所以有一定的相关性，但score会比较低。

如果不指定索引、类型和字段，直接用：
	
	curl -XGET 'localhost:9200/_search?q=Smith
	
会把所有索引不同类型任意字段中包含Smith的都找出来了。其实ES会把文档的所有字符串字段的值连接起来放在一个大的字符串中，它被索引为一个特殊的字段_all。因此可以构造更为复杂的查询：last_name为Smith，first_name为Jane，age小于30，任意字段中包含sports的结果。

	curl -XGET 'localhost:9200/_search?q=+last_name:Smith+first_name:Jane+age:<30+sports&pretty'

### 结构化搜索（query DSL和filter DSL）

#### 一、query DSL

语法格式：
	
	curl -XGET 'localhost:9200/_search'
	{
	    "query": YOUR_QUERY_HERE
	}
	
YOUR_QUERY_HERE查询子句里面的语法格式：

	{
	    QUERY_NAME: {
	        ARGUMENT: VALUE,
	        ARGUMENT: VALUE,...
	    }
	}

如果限制在特定的字段，用：
	
	{
	    QUERY_NAME: {
	        FIELD_NAME: {
	            ARGUMENT: VALUE,
	            ARGUMENT: VALUE,...
	        }
	    }
	}
	
例如你可以查询所有姓Smith的文档
	
	curl -XGET 'localhost:9200/_search?pretty' -d '
	{
	    "query": {
	        "match": {
	            "last_name": "Smith"
	        }
	    }
	}'
	
指定多字段多查询：
	
	curl -XGET 'localhost:9200/_search?pretty' -d '
	{
	    "query": {
	        "multi_match": {
	            "query": "reading",
	            "fields": ["interests","about"]
	        }
	    }
	}'

指定关键词全部匹配：

	curl -XGET 'localhost:9200/_search?pretty' -d '
	{
	    "query": {
	        "match": {
	            "about": "like reading"
	        }
	    }
	}'

这个查询会把about字段里面只要包含like和reading其中一个都搜索出来，如果

	curl -XGET 'localhost:9200/_search?pretty' -d '
	{
	    "query": {
	        "match": {
	            "about": {
	                "query":    "like reading",
	                "operator": "and"
	            }
	        }
	    }
	}'

这样就能把about字段里面既包含like又包含reading的搜出来了。其实还有一种更灵活的方式来控制搜索结果，minimum_should_match选项可以指定最少应该匹配的数量。

	curl -XGET 'localhost:9200/_search?pretty' -d '
	{
	    "query": {
	        "match": {
	            "about": {
	                "query":    "like reading rock",
	                "minimum_should_match": "2"
	            }
	        }
	    }
	}'

由于事先不确定用户会输入的关键词数量，这里可以设置为百分比，例如75%。

合并查询子句：

	{
	    "bool": {
	        "must":     { "match": { "tweet": "elasticsearch" }},
	        "must_not": { "match": { "name":  "mary" }},
	        "should":   { "match": { "tweet": "full text" }}
	    }
	}

其中：must：必须匹配，must_no：不许匹配，should：如果匹配的话得分更高，完整的案例：

	curl -XGET 'localhost:9200/_search?pretty' -d '
	{
		"query": {
			"bool": {
				"must": {
					"match": {
						"sex": "男"
					}
				},
				"must_not": {
					"match": {
						"name": "Smith"
					}
				},
				"should": {
					"match": {
						"hobby": "reading"
					}
				}
			}
		}
	}'

如果如果没传body，即空的{}，等价于：
	
	curl -XGET 'localhost:9200/_search'
	{
		"query": {
			"match_all": {}
		}
	}

### 二、filter DSL
	
区间过滤（range）：
	
	curl -XGET 'localhost:9200/_search?pretty' -d '
	{
		"filter": {
			"range": {
				"age": {
					"gt":  20,
					"lte":   30
				}
			}
		}
	}'
	
存在性过滤（exists）

	curl -XGET 'localhost:9200/_search?pretty' -d '
	{
		"filter": {
			"exists": {
				"field": "age"
			}
		}
	}'
	
缺失值过滤（missing）

	curl -XGET 'localhost:9200/_search?pretty' -d '
	{
		"filter": {
			"missing": {
				"field": "age"
			}
		}
	}'
	
精确匹配过滤（term）

	curl -XGET 'localhost:9200/_search?pretty' -d '
	{
		"filter": {
			"term": {
				"age": "33"
			}
		}
	}'
	
term和match的区别在于match会对查询语句进行分词，而term则搜索完整的语句，不做任何分词。（注意是查询语句，而不是数据库里面的字符串字段，如非特别标注，数据库里面的字符串字段总是会分词的）
	
多条件精确匹配过滤（terms）
	
	curl -XGET 'localhost:9200/_search?pretty' -d '
	{
		"filter": {
			"terms": {
				"about": ["like","rock"]
			}
		}
	}'
	
	
排序（例如按照年龄进行倒序）：

	curl -XGET 'localhost:9200/_search?pretty' -d '
	{
		"filter" : {
			"range": {
				"age": {
					"gt":  20,
					"lte": 30
				}
			}
		},
		"sort": { "age": { "order": "desc" }}
	}'

当指定排序的时候，_score 字段就不会被计算，因为它没有用作排序。作为缩写，你可以只指定要排序的字段名称，例如："sort": "age"，那么字段值默认以顺序排列，这一点与_score默认以倒序排列不同。如果想指定多个排序：

	curl -XGET 'localhost:9200/_search?pretty' -d '
	{
		"filter" : {
			"range": {
				"age": {
					"gt":  20,
					"lte": 30
				}
			}
		},
		"sort": [
	        { "age":   { "order": "desc" }},
	        { "first_name": { "order": "asc" }}
	    ]
	}'

还有一种情况是，如果一个字段是一个集合，拥有多个值，那是否可以指定其中一个作为排序依据呢？答案是肯定的：对于数字和日期，你可以从多个值中取出一个来进行排序，你可以使用min, max, avg 或 sum这些模式， 比说你可以在 dates 字段中用最早的日期来进行排序：

	"sort": {
	    "dates": {
	        "order": "asc",
	        "mode":  "min"
	    }
	}

然而对于多值字段字符串排序这种方法就不行了。比如字符串 "fine old art"，它最终会得到三个值。例如我们想要按照第一个词首字母排序， 如果第一个单词相同的话，再用第二个词的首字母排序，以此类推，可惜 ElasticSearch 在进行排序时 是得不到这些信息的。如果使用 min 和 max 模式来排（默认使用的是 min 模式）但它是依据art 或者 old排序， 而不是我们所期望的那样。


### 查询和过滤混用

单条过滤语句：

	curl -XGET 'localhost:9200/_search?pretty' -d '
	{
	    "query": {
	        "filtered": {
	            "filter":   { "term": { "folder": "inbox" }}
	        }
	    }
	}


带过滤的查询语句：

	curl -XGET 'localhost:9200/_search?pretty' -d '
	{
	    "query": {
	        "filtered": {
	            "query":  { "match": { "email": "business opportunity" }},
	            "filter": { "term": { "folder": "inbox" }}
	        }
	    }
	}'

验证查询：

	curl -XGET 'localhost:9200/_validate/query?pretty' -d '
	{
	   "query": {
	      "employee" : {
	         "match" : "really powerful"
	      }
	   }
	}'


	curl -XGET 'localhost:9200/_validate/query?pretty&explain' -d '
	{
	   "query": {
	      "employee" : {
	         "match" : "really powerful"
	      }
	   }
	}'

### 批量操作

批量操作文档（bulk）

	curl -XPOST 'localhost:9200/_bulk
	
	{ "delete": { "_index": "website", "_type": "blog", "_id": "123" }} 
	{ "create": { "_index": "website", "_type": "blog", "_id": "123" }}
	{ "title":    "My first blog post" }
	{ "index":  { "_index": "website", "_type": "blog" }}
	{ "title":    "My second blog post" }
	{ "update": { "_index": "website", "_type": "blog", "_id": "123", "_retry_on_conflict" : 3} }
	{ "doc" : {"title" : "My updated blog post"} } 

一次性完成删除、指定id的创建、自动生成id的创建和更新。返回结果可能是这样的：

	{
	   "took": 4,
	   "errors": false, 
	   "items": [
	      {  "delete": {
	            "_index":   "website",
	            "_type":    "blog",
	            "_id":      "123",
	            "_version": 2,
	            "status":   200,
	            "found":    true
	      }},
	      {  "create": {
	            "_index":   "website",
	            "_type":    "blog",
	            "_id":      "123",
	            "_version": 3,
	            "status":   201
	      }},
	      {  "create": {
	            "_index":   "website",
	            "_type":    "blog",
	            "_id":      "EiwfApScQiiy7TIKFxRCTw",
	            "_version": 1,
	            "status":   201
	      }},
	      {  "update": {
	            "_index":   "website",
	            "_type":    "blog",
	            "_id":      "123",
	            "_version": 4,
	            "status":   200
	      }}
	   ]
	}

对查询结果进行批量删除：

	curl -XDELETE "http://localhost:9200/megacorp/employee/_query?pretty" -d '
	{
	    "query": {
	        "match": {
	            "age": "30"
	        }
	    }
	}'

