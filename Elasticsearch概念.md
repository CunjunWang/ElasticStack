常见术语：
    文档 Document
        - 用户存储在ES中的数据文档
        - 相当于sql表中的一行数据
    索引 Index
        - 由具有相同字段的文档列表组成
        - 相当于数据库中的table
        - 6.0: 一个index下面只能有一个type
    节点 Node
        - 一个ES的运行实例，是集群的构成单元
    集群 Cluster
        - 一个或多个节点组成，对外提供服务

Document 介绍
    Json Object, 由字段(field)组成，常见的数据类型：
        - 字符串
            - text: 分词
            - keyword: 不分词
        - 数值类型
            - long, integer, short, byte, double, float, half_float, scaled_float
            - half_float, scaled_float, short, byte 在字段范围有限时可以节省空间
        - 布尔类型 boolean
        - 日期 date
        - 二进制 binary
        - 范围类型 
            - integer_range, float_range, long_range, double_range, date_range
    每个文档有唯一的Id标识
        - 自行指定
        - ES自动生成
    Document Metadata
        - 元数据，用于标注文档的相关信息
            - _index: 文档所在的索引名
            - _type: 文档所在的类型名
            - _id: 文档唯一Id
            - _uid: 组合Id, 由_type和_id组成，6.x版本之后_type不起作用，本字段和_id一样
            - _source: 文档原始Json数据，可以从这里获取每个字段的内容
            - _all: 整合多有字段内容到该字段，默认禁用

Index介绍
    索引中存储具有相同结构的文档(Document)
        - 每个索引都有自己的mapping定义，用于定义字段名和类型
    一个集群可以有多个索引

Rest API介绍
    ES集群对外提供Restful API
        - REST: Representational State Transfer
        - URI指定资源，如Index, Document等
        - Http方法指明资源操作类型，如GET, POST, PUT, DELETE等
    常用两种交互方式
        - Curl命令行
            curl -XPUT "host/xxx/xxx" -i -H "headerXXX" -d {data}
        - Kibana Devtools
    索引API
        ES有专门的Index API, 用于创建，更新，删除索引配置等
            - 创建：PUT /index_name
            - 查看：GET _cat/indices
            - 删除：DELETE /index_name
    文档API
        - 创建：
            如果索引不存在，ES会自动创建对应的index和type
            - 指定ID创建文档：PUT方法
                方法    index   type id
                PUT /test_index/doc/1
                {
                    "username": "cunjun",
                    "age": 1
                }
            - 不指定ID创建文档：POST方法
                POST /test_index/doc
                {
                    "username": "tom",
                    "age": 11
                }
        - 查询
            - 指定想要查询的文档的id
                GET /test_index/doc/1
                返回：200response
                {
                  "_index": "test_index",
                  "_type": "doc",
                  "_id": "1",
                  "_version": 2,
                  "found": true,
                  "_source": {
                    "username": "cunjun",
                    "age": 1
                  }
                }
                _source存储了文档的完整原始数据
            - 查询所有文档，用到_search
                GET /test_index/doc/_search
                { query body }
        - 批量创建：
            ES允许一次创建多个文档
            - endpoint为_bulk
            - action_type: index, update, create, delete
                POST _bulk
                {"index": {"_index": "test_index", "_type": "doc", "_id": "3"}}
                {"username": "cunjun", "age": "10"}
                {"delete": {"_index": "test_index", "type": "doc", "_id": "1"}}
                {"update:" {"_id": "2", "_index": "test_index", "_type": "doc"}}
                {"doc": {"age"" "20"}}
        - 批量查询：
            - endpoint为_mget
                GET /_mget
                {
                    "docs": [
                        {
                            "_index": "test_index",
                            "_type": "doc",
                            "_id": 1
                        },
                        {
                            "_index": "test_index",
                            "_type": "doc",
                            "_id": "2"
                        }
                    ]
                }

倒排索引与分词
    正排索引：
        文档Id到文档内容、单词的关联联系
    倒排索引：
        单词到文档Id的关联，包含两部分：
            - 单词词典 (Term Dictionary)
                - 记录所有文档的单词，一般都比较大
                - 记录单词到倒排列表的关联信息
                - 一般的实现是B+树
            - 倒排列表 (Posting List)
                - 记录了单词对应的文档集合，由倒排索引项组成
                - 倒排索引项(Posting)主要包含如下信息：
                    - 文档Id, 用于获取原始信息
                    - 单词频率(TF, Term Frequency), 记录该单词在该文档中的出现次数，用于后续相关性算分
                    - 位置(Position)，记录单词在文档中的分词位置(多个)，用于做词语搜索(Phrase Query)
                    - 偏移(Offset), 记录单词在文档开始和结束位置，用于做高亮显示
    查询包含"xxx"的文档：
        - 通过倒排索引获得"xxx"对应的文档Id
        - 通过正排索引查询对应Id的完整内容
        - 返回用户最终结果

分词
    把文本转换成一系列单词的过程，也叫做文本分析
    ES中负责分词的组件叫分词器，Analyzer
        - Character Filters
            - 针对原始文本进行处理，比如出去html特殊标记符
        - Tokenizer
            - 将原始文本按照一定规则切分称为单词
        - Token Filters
            - 针对tokenizer处理的单词进行再加工，比如转小写，删除或新增等处理
        - 调用顺序：
            Character Filters -> Tokenizer -> Token Filters
    *Analyze API
        ES提供了一个测试分词的API，方便验证分词效果，endpoint是_analyze
        - 可以直接指定analyzer进行测试
            请求：
            POST _analyze
            {
                "analyzer": "standard",
                "text": "hello world!"
            }
            结果：
            {
                "tokens": [
                  {
                    "token": "hello",
                    "start_offset": 0,
                    "end_offset": 5,
                    "type": "<ALPHANUM>",
                    "position": 0
                  },
                  {
                    "token": "world",
                    "start_offset": 6,
                    "end_offset": 11,
                    "type": "<ALPHANUM>",
                    "position": 1
                  }
                ]
            }
        - 可以直接指定索引中的字段进行测试
            POST test_index/_analyze
            {
                "field": "username",
                "text": "hello world"
            }
        - 可以自定义分词器进行测试
            POST _analyze
            {
                "tokenizer": "standard",
                "filter": ["lowercase"],
                "text": "Hello World!"
            }
    ES预定义的分词器：
        - Standard
            - 默认分词器
            - 按词切分，支持多语言
            - 小写处理
            Tokenizer: Standard
            Token Filters: ["Standard", "Lower case", "Stop"(disabled by default)]
        - Simple
            - 按照非字母切分
            - 小写处理
            Tokenizer: Lower case
        - Whitespace
            - 按照空格切分
            - Tokenizer: Whitespace
        - Stop
            - stop word指语气助词等修饰性词语，比如the,an, 的，这等等
            - 相比simple多了stop word处理
            Tokenizer: Lower Case
            Token Filters: Stop
        - Keyword
            - 不分词，直接将输入作为一个单词输出
            Tokenizer: Keyword
        - Pattern
            - 通过正则表达式自定义分隔符
            - 默认是\W+, 即非字词的符号作为分隔符
            Tokenizer: Pattern
            Token Filters: [Lower case, Stop(disabled by defaults)]
        - Language
            - 提供了30+常见语言的分词器
    中文分词：
        - 难点：汉语中词没有一个形式上的分解符，比如空格，划线
        - 上下文不同，分词结果迥异，比如交叉歧义问题
        - 常用分词系统：
            - IK
                - 实现中英文单词的划分，支持ik_smart, ik_maxword等模式
                - 可自定义词库，支持热更新分词词典
            - jieba
                - python中最流行的分词系统，支持分词和词性标注
                - 支持繁体分词、自定义词典、并行分词
        - 基于NLP的分词处理系统
            - Hanlp
                - 由一系列模型和算法组成的java包，普及NLP在生产环境的应用
            - THULAC
                - 清华大学NLP实验室推出的中文词法分析工具包
    自定义分词
        - 定义Character Filter, Tokenizer, Token Filter实现
        - Character Filters
            - 在Tokenizer之前对原始文本进行处理，增加、删除或替换字符等
            - 自带的如下：
                - HTML Strip 去除html标签和转换html实体
                - Mapping 进行字符替换操作
                - Pattern Replace 进行正则匹配替换
            - 会影响后续的tokenizer解析的position和offset信息
            - 测试：
                POST _analyze
                {
                    "tokenizer": "keyword",
                    "char_filter": ["html_strip"],
                    "text": "<p>ewhflwklhekl</p>"
                }
        - Tokenizer
            - 将原始文本按照一定规则切分为单词
            - 自带的如下：
                - standard              按照单词进行分割
                - letter                按照非字符类进行分割
                - whitespace            按照空格进行分割
                - UAX URL Email         按照standard进行分割，但不会分割邮箱和url
                - NGram和Edge NGram     连词分割(智能提示列表)
                - Path Hierarchy        按照文件路径进行分割 
            - 测试
                POST _analyze
                {
                    "tokenizer": "path_hierarchy",
                    "text": "/one/two/three"
                }
        - Token Filter
            - 针对tokenizer输出的单词(term)进行增加、删除、修改等操作
            - 自带的如下：
                - lowercase             所有的term变为小写
                - stop                  删除stop words
                - NGram和Edge NGram     连词分割
                - Synonym               添加近义词的term
            - 测试
                POST _analyze
                {
                    "text": "a Hello, world!",
                    "tokenizer": "standard",
                    "filter": [
                        "stop",
                        "lowercase",
                        {
                            "type": "ngram",
                            "min_gram": 4,
                            "max_gram": 4
                        }
                    ]
                }
        - API配置
            可以自定义的配置：char_filter, tokenizer, filter, analyzer
            PUT test_index
            {
                "settings": {
                    "analysis": {
                        "char_filter": {},
                        "tokenizer": {},
                        "filter": {},
                        "analyzer": {}
                    }
                }
            }
            - 示例
                Custom Analyzer:
                    - Character Filters: HTML Strip
                    - Tokenizer: Standard
                    - Token Filters: [Lower case, ASCII Folding]
                    - 结构
                    PUT test_index
                    {
                        "settings": {
                            "analysis": {
                                "analyzer": {
                                    "my_custom_analyzer": {
                                        "type": "custom",
                                        "tokenizer": "standard",
                                        "char_filter": [
                                            "html_strip"
                                        ],
                                        "filter": [
                                            "lowercase", "asciifolding"
                                        ]
                                    }
                                }
                            }
                        }
                    }
                    - 测试请求
                        POST test_index/_analyze
                        {
                            "analyzer": "my_custom_analyzer",
                            "text": "is this <b>a box</b>?"
                        }
    分词使用说明
        - 会在如下两个时机使用：
            - 创建或更新文档时(index time), 会对相应的文档进行分词处理
            - 查询时会对查询语句进行分词
        - 索引时分词
            - 索引时分词通过配置index mapping中的每个字段的analyzer属性实现
            - 不指定分词时，默认使用standard
        - 查询时分词
            - 通过analyzer指定分词器
            - 通过index mapping设置search_analyzer实现


Mapping设置
    Mapping是什么
        - 类似于数据库中的表结构定义(schema)
            - 定义Index下的字段名
            - 定义字段类型
            - 定义倒排索引的相关配置，比如是否索引，记录position等
        - API
            - 请求获取mapping
            GET /test_index/_mapping
    自定义Mapping
        - API
            PUT my_index
            {
                "mappings": {
                    "doc": {
                        "properties": {
                            "title": {
                                "type": "text"
                            },
                            "name": {
                                "type": "keyword"
                            },
                            "age": {
                                "type": "integer"
                            }
                        }
                    }
                }
            }
        - Mapping中的字段类型一旦设定，就禁止直接修改
            - 重新建立新的索引，然后做reindex操作
        - 允许新增字段
            - 通过dynamic参数来控制字段的新增
                - true (默认)：允许自动新增字段
                - false：不允许自动新增子弹，但是文档可以正常写入，无法对字段进行查询操作
                - strict 文档不能写入，报错
                PUT my_index
                {
                    "mappings": {
                        "my_type": {
                            "dynamic": false,
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
                }
        常见参数设置
            - copy_to
                - 将该字段的值复制到目标字段，类似实现_all的作用
                - 不会出现在_source中，只用来搜索
                PUT my_index
                {
                    "mappings": {
                        "doc": {
                            "properties": {
                                "first_name": {
                                    "type": "text",
                                    "copy_to": "full_name"
                                },
                                "last_name": {
                                    "type": "text",
                                    "copy_to": "full_name"
                                },
                                "full_name": {
                                    "type": "text"
                                }
                            }
                        }
                    }
                }
            - index
                - 控制当前字段是否索引。默认为true，即记录索引，false为不记录，即不可搜索
                    PUT my_index
                    {
                        "mappings": {
                            "doc": {
                                "properties": {
                                    "cookie": {
                                        "type": "text",
                                        "index": false
                                    }
                                }
                            }
                        }
                    }
                - 请求
                    PUT my_index/doc/1
                    {
                        "cookie": "name=alfred"
                    }
                    GET my_index/_search
                    {
                        "query": {
                            "match": {
                                "cookie": "name"
                            }
                        }
                    }
                    返回的结果为error，因为未记录索引
                - 当存储不希望被检索的信息时，可以用这个属性
            - index_options
                - 用于控制倒排索引记录的内容，有以下4中配置
                    - docs 只记录doc id
                    - freqs 记录doc id和term frequencies
                    - positions 记录doc id, term frequencies和term position
                    - offsets 记录doc id, term frequencies和character offsets
                - text类型默认配置为positions, 其他默认为docs
                - 记录内容越多，占用空间越大
                    PUT my_index
                    {
                        "mappings": {
                            "doc": {
                                "properties": {
                                    "cookie": {
                                        "type": "text",
                                        "index_options": "offsets"
                                    }
                                }
                            }
                        }
                    }
            - null_value
                - 当字段遇到null值的处理策略，默认为null，即为空值，此时ES会忽略该值
                可以通过设定该值设定字段的默认值
                    PUT my_index
                    {
                        "mappings": {
                            "my_type": {
                                "properties": {
                                    "status_code": {
                                        "type": "keyword",
                                        "null_vallue": "NULL"
                                    }
                                }
                            }
                        }
                    }
    数据类型
        - 核心数据类型
            - 字符串
                - text: 分词
                - keyword: 不分词
            - 数值类型
                - long, integer, short, byte, double, float, half_float, scaled_float
                - half_float, scaled_float, short, byte 在字段范围有限时可以节省空间
            - 布尔类型 boolean
            - 日期 date
            - 二进制 binary
            - 范围类型 
                - integer_range, float_range, long_range, double_range, date_range
        - 复杂数据类型
            - 数组 array
            - 对象 object
            - 嵌套类型 nested object
            - 地理位置相关
                - geo_point
                - geo_shape
        - 专用类型
            - ip地址 ip
            - 自动补全 completion
            - 记录分词数 token_count
            - 记录字符串hash值 murmur3
            - percolator
            - join
        - 多字段特性 multi-fields
            - 允许对同一个字段采用不同的配置
    Dynamic Mapping
        - ES可以自动识别文档字段类型
            依靠Json文档的字段类型来实现自动识别字段类型
            |==========|========================================|
            | Json类型 |                ES 类型                 |
            |==========|========================================|
            | null     | 忽略                                   |
            |----------|----------------------------------------|
            | boolean  | boolean                                |
            |----------|----------------------------------------|
            | 浮点     | float                                  |
            |----------|----------------------------------------|
            | 整数     | long                                   |
            |----------|----------------------------------------|
            | object   | object                                 |
            |----------|----------------------------------------|
            | array    | 由第一个非null的值决定                 |
            |----------|----------------------------------------|
            | string   | 匹配为日期则为date (默认开启)          |
            |          | 匹配为数字则设为float或long (默认关闭) |
            |          | 设为text类型并附带"keyword"为子字段    |
            |==========|========================================|
        - 日期与数字的识别
            - 日期的自动识别可以自行配置日期格式，以满足各种需求
                - dynamic_date_formats可以自定义日期类型
                - date_detection可以关闭日期自动识别的机制
                    PUT my_index
                    {
                        "mappings": {
                            "my_type": {
                                "dynamic_date_formats": ["MM/dd/yyyy"]
                            }
                        }
                    }
            - 字符串是数字时，默认不会识别为整型，因为字符串中出现数字时合理的
                - numeric_detection可以开启字符串中数字的自动识别
                    PUT my_index
                    {
                        "mappings": {
                            "my_type": {
                                "numeric_detection": true
                            }
                        }
                    }
    Dynamic Template
        - 允许根据ES自动识别的数据类型、字段名来动态设定字段类型，可以实现以下效果
            - 所有字符串类型都设定为keyword类型，默认不分词
            - 所有以message开头的字段都设置为text
            - 所有long_开头的字段都设置为long
            - 所有自动匹配为double类型的都设定为float以节省空间
        - API
            PUT test_index
            {
                "mappings": {
                    "doc": {
                        "dynamic_templates": [
                            {
                                "strings": {
                                    "match_mapping_type": "string",
                                    "mapping": {
                                        "type": "keyword"
                                    }
                                }
                            }
                        ]
                    }
                }
            }
            - 匹配规则有如下常用参数：
                - match_mapping_type匹配ES自动识别的字段类型，如boolean, long, string等
                - match, unmatch 匹配字段名
                - path_match, path_unamtch 匹配路径
    自定义Mapping的建议
        - 自定义mapping的操作步骤一般如下
            - 写入一条文档到ES临时索引中，获取ES自动生成的mapping
            - 修改该mapping，自定义设置相关配置
            - 使用修改后的mapping创建实际所需索引
    索引模板
        - Index Template
            主要用于在新建索引时自动应用预先设定的配置，简化索引创建的操作步骤
                - 可以设定索引的配置和mapping
                - 可以由多个模板，根绝order设置，order大的覆盖小的配置
        - API
            PUT _template/test_template
            {
                "index_patterns": ["te*", "bar*"],
                "order": 0,
                "settings": {
                    "number_of_shards": 1
                },
                "mappings": {
                    "doc": {
                        "_source": {
                            "enabled": false
                        },
                        "properties": {
                            "name": {
                                "type": "keyword"
                            }
                        }
                    }
                }
            }


Search API
    介绍
        - 实现对ES中存储的数据进行查询分析，endpoint为_search
            GET /_search
            GET /my_index/_search
            GET /my_index1, my_index2/_search
            GET /my_*/_search 指定索引查询，可一次查询多个
        - 查询主要有两种形式
            - URI Search
                - 操作简单，方便通过命令行测试
                - 仅包含部分查询语法
                    GET /my_index/_search?q=user:alfred
            - Request Body Search
                - ES提供的完备的查询语法Query DSL (Domain Specific Language)
                    GET /my_index/_search
                    {
                        "query": {
                            "term": {
                                "user": "alfred"
                            }
                        }
                    }
    URI Search
        - 通过url query参数来实现搜索，常用参数如下：
            - q: 指定查询的语句，与乏味Query String Syntax
            - df: defalut field的简称，q中不指定字段时默认查询的字段，如果不指定，ES会查询所有字段
            - sort: 排序
            - timeout: 指定超时时间，默认不超时
            - from, size: 用于分页
            GET /my_index/_search?q=alfred&df=user&sort=age:asc&from=4&size=10&timeout=1s (查询user字段包含alfred的文档，结果按照age圣墟排列，返回第5~14个文档，如果超过1s没有结束则以超时结束)
        - Query String Syntax
            - profile参数：看ES通过什么语句查询的
            - term与phrase
                - alfred way: term查询，等于alfred OR way
                - "alfred way": phrase查询，看做整体查询
            - 泛查询
                - alfred: 等于在所有字段去匹配改term(未指定df参数)
            - 指定字段
                - name:alfred: 在name字段查询alfred
            - Group 分组设定，使用括号指定匹配的规则
                - (quick OR brown) AND fox
                - status: (active OR pending) title: (full text search)
            - Boolean Operator
                - AND(&&), OR(||), NOT(!)
                    - name: (tom NOT lee)
                    - 注意大写，不能小写
                - + - 分别对应must和must_not
                    - name: (tom + lee - alfred)
                        = name: ((lee && !alfred) || (tom && lee && !alfred))
                    - + 在url中会被解析为空格，要使用encode之后得俄国才可以，为%2B
            - 范围查询，支持数值和日期
                - 区间写法，闭区间用[]，开区间用{}
                    - age:[1 TO 10] 1<= age <=10
                    - age:[1 TO 10} 1<= age < 10
                    - age:[1 TO ]   age >=1
                    - age:[* TO 10] age <=10
                - 算数符号写法
                    - age:>=1
                    - age:(>=1 && <=10)或者age:(+>=1+<=10)
            - 通配符
                - ?代表一个字符, *代表0或多个字符
                    - name:t?m
                    - name:tom*
                    - name:t*m
                - 通配符执行效率低且占用较多内存，不建议使用
                - 如果没有特殊要求，不要把?/*放在最前面
            - 正则表达式
                - name:/[mb]oat/
            - 模糊匹配 fuzzy query
                - name:roam~1
                - 匹配与roam差一个字母的词，比如foam, roams等
            - 近似度查询 proximity search
                - "fox quick"~5
                - 以term为单位进行差异比较，比如"quick fox", "quick brown fox"都会被匹配
    Query DSL
        将查询语句通过http request body发送到ES，主要包含以下参数
            - query 复合Query DSL语法的查询语句
            - from, size
            - timeout
            - sort
            - ...
            GET /my_index/_search
            {
                "query": {
                    "term": {
                        "user": "alfred"
                    }
                }
            }
        基于Json定义的查询语言，主要包含两种类型：
            - 字段类查询
                - 如term, match, range等，只针对一个字段进行查询
            - 复合查询
                - 如bool查询，包含一个或多个字段类查询或者复合查询语句
        字段类查询
            - 主要包含两类
                - 全文匹配
                    - 针对text类型的字段进行全文检索，会对查询语句先进行分词处理，如match, match_phrase等query类型
                - 单词匹配
                    - 不会对查询语句做分词处理，直接去匹配字段的倒排索引，如term, terms, range等query类型
            - match query
                - API
                    GET test_search_index/_search
                    {
                        "query": {
                            "match": {
                                "username": "alfred way"
                            }
                        }
                    }
                - 流程
                    对查询语句分词, 根据字段的倒排索引进行匹配得分， 汇总得分，根据得分排序，返回匹配文档
                    算分算法：TF/IDF, BM25
                - 默认OR, 提供operator来控制
                    GET test_search_index/_search
                    {
                        "query": {
                            "match": {
                                "username": {
                                    "query": "alfred way",
                                    "operator": "and"
                                }
                            }
                        }
                    }
                    minimum_should_match参数：最小匹配单词数
        相关性算分
            - 指的是文档与查询语句间的相关度，relevance
                - 通过倒排索引可以获取与查询语句相匹配的文档列表
            - 重要概念
                - TF: Term Frequency 词频，单词在文档中出现的次数。词频越高相关性越高。
                - DF: Document Frequency 文档频率，单词出现的文档数
                - inverse Document Frequency(IDF) 逆向文档频率, 1/DF
                - Field-length Norm 文档越短，相关性越高
            - ES目前有两个主要的相关性算分模型
                - TF/IDF
                - BM25: 5.x之后默认的模型
        Match Phrase Query
            - 对字段做检索，有顺序要求，API示例如下
                GET test_search_index/_search
                {
                    "query": {
                        "match_phrase": {
                            "job": "java engineer"
                        }
                    }
                }
            - 通过slop参数可以控制单词间的间隔(模糊查询)
                GET test_search_index/_search
                {
                    "query": {
                        "match_phrase": {
                            "job": {
                                "query": "jave engineer",
                                "slop": "1"
                            }
                        }
                    }
                }







