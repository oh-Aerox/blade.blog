# 结构化搜索(过滤)
## 精确值查找
### term查询
https://www.elastic.co/guide/cn/elasticsearch/guide/current/_finding_exact_values.html
## 组合过滤器
### 布尔过滤器
一个 bool 过滤器由三部分组成：
```
{
   "bool" : {
      "must" :     [],
      "should" :   [],
      "must_not" : [],
   }
}
```
* must
所有的语句都 必须（must） 匹配，与 AND 等价。
* must_not
所有的语句都 不能（must not） 匹配，与 NOT 等价。
* should
至少有一个语句要匹配，与 OR 等价。

比方说，怎样用 Elasticsearch 来表达下面的 SQL ？
```
SELECT product
FROM   products
WHERE  (price = 20 OR productID = "XHDK-A-1293-#fJ3")
  AND  (price != 30)
```
用 Elasticsearch 来表示 SQL 例子，将两个 term 过滤器置入 bool 过滤器的 should 语句内，再增加一个语句处理 NOT 非的条件：
```
GET /my_store/products/_search
{
   "query" : {
      "filtered" : { 
         "filter" : {
            "bool" : {
              "should" : [
                 { "term" : {"price" : 20}}, 
                 { "term" : {"productID" : "XHDK-A-1293-#fJ3"}} 
              ],
              "must_not" : {
                 "term" : {"price" : 30} 
              }
           }
         }
      }
   }
}
```

> 注意，我们仍然需要一个 filtered 查询将所有的东西包起来。
在 should 语句块里面的两个 term 过滤器与 bool 过滤器是父子关系，两个 term 条件需要匹配其一。
如果一个产品的价格是 30 ，那么它会自动被排除，因为它处于 must_not 语句里面。

### 嵌套布尔过滤器
尽管 bool 是一个复合的过滤器，可以接受多个子过滤器，需要注意的是 bool 过滤器本身仍然还只是一个过滤器。 这意味着我们可以将一个 bool 过滤器置于其他 bool 过滤器内部，这为我们提供了对任意复杂布尔逻辑进行处理的能力。
对于以下这个 SQL 语句：
```
SELECT document
FROM   products
WHERE  productID      = "KDKE-B-9947-#kL5"
  OR (     productID = "JODL-X-1937-#pV7"
       AND price     = 30 )
```
我们将其转换成一组嵌套的 bool 过滤器：
```
GET /my_store/products/_search
{
   "query" : {
      "filtered" : {
         "filter" : {
            "bool" : {
              "should" : [
                { "term" : {"productID" : "KDKE-B-9947-#kL5"}}, 
                { "bool" : { 
                  "must" : [
                    { "term" : {"productID" : "JODL-X-1937-#pV7"}}, 
                    { "term" : {"price" : 30}} 
                  ]
                }}
              ]
           }
         }
      }
   }
}
```
* 因为 term 和 bool 过滤器是兄弟关系，他们都处于外层的布尔逻辑 should 的内部，返回的命中文档至少须匹配其中一个过滤器的条件。
* 这两个 term 语句作为兄弟关系，同时处于 must 语句之中，所以返回的命中文档要必须都能同时匹配这两个条件。

## 查找多个精确值
如果我们想要查找价格字段值为 $20 或 $30 的文档该如何处理呢？
不需要使用多个 term 查询，我们只要用单个 terms 查询（注意末尾的 s ）， terms 查询好比是 term 查询的复数形式（以英语名词的单复数做比）。

它几乎与 term 的使用方式一模一样，与指定单个价格不同，我们只要将 term 字段的值改为数组即可：
```
GET /my_store/products/_search
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "terms" : { 
                    "price" : [20, 30]
                }
            }
        }
    }
}
```
### 包含，而不是相等
一定要了解 term 和 terms 是 包含（contains） 操作，而非 等值（equals） （判断）。 如何理解这句话呢？

如果我们有一个 term（词项）过滤器 { "term" : { "tags" : "search" } } ，它会与以下两个文档 同时 匹配：
```
{ "tags" : ["search"] }
{ "tags" : ["search", "open_source"] } 
```
回忆一下 term 查询是如何工作的？ Elasticsearch 会在倒排索引中查找包括某 term 的所有文档，然后构造一个 bitset 。在我们的例子中，倒排索引表如下：
![image.png](https://www.hounk.world/upload/2021/08/image-bb2f7624eae0419095ee9b4811f9a63d.png)
当 term 查询匹配标记 search 时，它直接在倒排索引中找到记录并获取相关的文档 ID，如倒排索引所示，这里文档 1 和文档 2 均包含该标记，所以两个文档会同时作为结果返回。
> 由于倒排索引表自身的特性，整个字段是否相等会难以计算，如果确定某个特定文档是否 只（only） 包含我们想要查找的词呢？首先我们需要在倒排索引中找到相关的记录并获取文档 ID，然后再扫描 倒排索引中的每行记录 ，查看它们是否包含其他的 terms 。
可以想象，这样不仅低效，而且代价高昂。正因如此， term 和 terms 是 必须包含（must contain） 操作，而不是 必须精确相等（must equal exactly） 。
### 精确相等
如果一定期望得到我们前面说的那种行为（即整个字段完全相等），最好的方式是增加并索引另一个字段， 这个字段用以存储该字段包含词项的数量，同样以上面提到的两个文档为例，现在我们包括了一个维护标签数的新字段：
```
{ "tags" : ["search"], "tag_count" : 1 }
{ "tags" : ["search", "open_source"], "tag_count" : 2 }
```
一旦增加这个用来索引项 term 数目信息的字段，我们就可以构造一个 constant_score 查询，来确保结果中的文档所包含的词项数量与要求是一致的：
```
GET /my_index/my_type/_search
{
    "query": {
        "constant_score" : {
            "filter" : {
                 "bool" : {
                    "must" : [
                        { "term" : { "tags" : "search" } }, 
                        { "term" : { "tag_count" : 1 } } 
                    ]
                }
            }
        }
    }
}
```
这个查询现在只会匹配具有单个标签 search 的文档，而不是任意一个包含 search 的文档。
## 范围
我们可能想要查找所有价格大于 $20 且小于 $40 美元的产品。在 SQL 中，范围查询可以表示为：
```
SELECT document
FROM   products
WHERE  price BETWEEN 20 AND 40
```
Elasticsearch 有 range 查询，可同时提供包含（inclusive）和不包含（exclusive）这两种范围表达式，可供组合的选项如下：

gt: > 大于（greater than）
lt: < 小于（less than）
gte: >= 大于或等于（greater than or equal to）
lte: <= 小于或等于（less than or equal to）

eg.
```
GET /my_store/products/_search
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "range" : {
                    "price" : {
                        "gte" : 20,
                        "lt"  : 40
                    }
                }
            }
        }
    }
}
```
### 日期范围
range 查询同样可以应用在日期字段上：
```
"range" : {
    "timestamp" : {
        "gt" : "2014-01-01 00:00:00",
        "lt" : "2014-01-07 00:00:00"
    }
}
```
当使用它处理日期字段时， range 查询支持对 日期计算（date math） 进行操作，比方说，如果我们想查找时间戳在过去一小时内的所有文档：
```
"range" : {
    "timestamp" : {
        "gt" : "now-1h"
    }
}
```
这个过滤器会一直查找时间戳在过去一个小时内的所有文档，让过滤器作为一个时间 滑动窗口（sliding window） 来过滤文档。

日期计算还可以被应用到某个具体的时间，并非只能是一个像 now 这样的占位符。只要在某个日期后加上一个双管符号 (||) 并紧跟一个日期数学表达式就能做到：
```
"range" : {
    "timestamp" : {
        "gt" : "2014-01-01 00:00:00",
        "lt" : "2014-01-01 00:00:00||+1M" 
    }
}
```
### 字符串范围
range 查询同样可以处理字符串字段，字符串范围可采用 字典顺序（lexicographically） 或字母顺序（alphabetically）。例如，下面这些字符串是采用字典序（lexicographically）排序的：

5, 50, 6, B, C, a, ab, abb, abc, b

> 在倒排索引中的词项就是采取字典顺序（lexicographically）排列的，这也是字符串范围可以使用这个顺序来确定的原因。

想查找从 a 到 b （不包含）的字符串，同样可以使用 range 查询语法
```
"range" : {
    "title" : {
        "gte" : "a",
        "lt" :  "b"
    }
}
```
> 数字和日期字段的索引方式使高效地范围计算成为可能。但字符串却并非如此，要想对其使用范围过滤，Elasticsearch 实际上是在为范围内的每个词项都执行 term 过滤器，这会比日期或数字的范围过滤慢许多。

字符串范围在过滤 低基数（low cardinality） 字段（即只有少量唯一词项）时可以正常工作，但是唯一词项越多，字符串范围的计算会越慢。

# 全文搜索（查询）
## 基于词项与基于全文
https://www.elastic.co/guide/cn/elasticsearch/guide/current/term-vs-full-text.html

## 匹配查询
匹配查询 match 是个 核心 查询。无论需要查询什么字段， match 查询都应该会是首选的查询方式。它是一个高级 全文查询 ，这表示它既能处理全文字段，又能处理精确字段。

首先，我们使用 bulk API 创建一些新的文档和索引：

```
DELETE /my_index 

PUT /my_index
{ "settings": { "number_of_shards": 1 }} 

POST /my_index/my_type/_bulk
{ "index": { "_id": 1 }}
{ "title": "The quick brown fox" }
{ "index": { "_id": 2 }}
{ "title": "The quick brown fox jumps over the lazy dog" }
{ "index": { "_id": 3 }}
{ "title": "The quick brown fox jumps over the quick dog" }
{ "index": { "_id": 4 }}
{ "title": "Brown fox brown dog" }
```
### 单个词查询
```
GET /my_index/my_type/_search
{
    "query": {
        "match": {
            "title": "QUICK!"
        }
    }
}
```
Elasticsearch 执行上面这个 match 查询的步骤是：

1. 检查字段类型 。
标题 title 字段是一个 string 类型（ analyzed ）已分析的全文字段，这意味着查询字符串本身也应该被分析。

2. 分析查询字符串 。
将查询的字符串 QUICK! 传入标准分析器中，输出的结果是单个项 quick 。因为只有一个单词项，所以 match 查询执行的是单个底层 term 查询。

3. 查找匹配文档 。
用 term 查询在倒排索引中查找 quick 然后获取一组包含该项的文档，本例的结果是文档：1、2 和 3 。

4. 为每个文档评分 。
用 term 查询计算每个文档相关度评分 _score ，这是种将词频（term frequency，即词 quick 在相关文档的 title 字段中出现的频率）和反向文档频率（inverse document frequency，即词 quick 在所有文档的 title 字段中出现的频率），以及字段的长度（即字段越短相关度越高）相结合的计算方式

### 多词查询
```
GET /my_index/my_type/_search
{
    "query": {
        "match": {
            "title": "BROWN DOG!"
        }
    }
}
```
上面这个查询返回所有四个文档：
```
{
  "hits": [
     {
        "_id":      "4",
        "_score":   0.73185337, 
        "_source": {
           "title": "Brown fox brown dog"
        }
     },
     {
        "_id":      "2",
        "_score":   0.47486103, 
        "_source": {
           "title": "The quick brown fox jumps over the lazy dog"
        }
     },
     {
        "_id":      "3",
        "_score":   0.47486103, 
        "_source": {
           "title": "The quick brown fox jumps over the quick dog"
        }
     },
     {
        "_id":      "1",
        "_score":   0.11914785, 
        "_source": {
           "title": "The quick brown fox"
        }
     }
  ]
}
```
文档 4 最相关，因为它包含词 "brown" 两次以及 "dog" 一次。
文档 2、3 同时包含 brown 和 dog 各一次，而且它们 title 字段的长度相同，所以具有相同的评分。
文档 1 也能匹配，尽管它只有 brown 没有 dog 。

因为 match 查询必须查找两个词（ ["brown","dog"] ），它在内部实际上先执行两次 term 查询，然后将两次查询的结果合并作为最终结果输出。为了做到这点，它将两个 term 查询包入一个 bool 查询中.

以上示例告诉我们一个重要信息：即任何文档只要 title 字段里包含 指定词项中的至少一个词 就能匹配，被匹配的词项越多，文档就越相关。
#### 提高精度
用 任意 查询词项匹配文档可能会导致结果中出现不相关的长尾。这是种散弹式搜索。可能我们只想搜索包含 所有 词项的文档，也就是说，不去匹配 brown OR dog ，而通过匹配 brown AND dog 找到所有文档。

match 查询还可以接受 operator 操作符作为输入参数，默认情况下该操作符是 or 。我们可以将它修改成 and 让所有指定词项都必须匹配：
```
GET /my_index/my_type/_search
{
    "query": {
        "match": {
            "title": {      
                "query":    "BROWN DOG!",
                "operator": "and"
            }
        }
    }
}
```
#### 控制精度
在 所有 与 任意 间二选一有点过于非黑即白。如果用户给定 5 个查询词项，想查找只包含其中 4 个的文档，该如何处理？将 operator 操作符参数设置成 and 只会将此文档排除。
match 查询支持 minimum_should_match 最小匹配参数，这让我们可以指定必须匹配的词项数用来表示一个文档是否相关。我们可以将其设置为某个具体数字，更常用的做法是将其设置为一个百分数，因为我们无法控制用户搜索时输入的单词数量：
```
GET /my_index/my_type/_search
{
  "query": {
    "match": {
      "title": {
        "query":                "quick brown dog",
        "minimum_should_match": "75%"
      }
    }
  }
}
```
当给定百分比的时候， minimum_should_match 会做合适的事情：在之前三词项的示例中， 75% 会自动被截断成 66.6% ，即三个里面两个词。无论这个值设置成什么，至少包含一个词项的文档才会被认为是匹配的。
> 参数 minimum_should_match 的设置非常灵活，可以根据用户输入词项的数目应用不同的规则。完整的信息参考文档 https://www.elastic.co/guide/en/elasticsearch/reference/5.6/query-dsl-minimum-should-match.html#query-dsl-minimum-should-match

## 组合查询
在 组合过滤器 中，我们讨论过如何使用 bool 过滤器通过 and 、 or 和 not 逻辑组合将多个过滤器进行组合。在查询中， bool 查询有类似的功能，只有一个重要的区别。

过滤器做二元判断：文档是否应该出现在结果中？但查询更精妙，它除了决定一个文档是否应该被包括在结果中，还会计算文档的 相关程度 。

与过滤器一样， bool 查询也可以接受 must 、 must_not 和 should 参数下的多个查询语句。比如：
```
GET /my_index/my_type/_search
{
  "query": {
    "bool": {
      "must":     { "match": { "title": "quick" }},
      "must_not": { "match": { "title": "lazy"  }},
      "should": [
                  { "match": { "title": "brown" }},
                  { "match": { "title": "dog"   }}
      ]
    }
  }
}
```
以上的查询结果返回 title 字段包含词项 quick 但不包含 lazy 的任意文档。目前为止，这与 bool 过滤器的工作方式非常相似。

区别就在于两个 should 语句，也就是说：一个文档不必包含 brown 或 dog 这两个词项，但如果一旦包含，我们就认为它们 更相关 ：
```
{
  "hits": [
     {
        "_id":      "3",
        "_score":   0.70134366, 
        "_source": {
           "title": "The quick brown fox jumps over the quick dog"
        }
     },
     {
        "_id":      "1",
        "_score":   0.3312608,
        "_source": {
           "title": "The quick brown fox"
        }
     }
  ]
}
```
### 评分计算
bool 查询会为每个文档计算相关度评分 _score ，再将所有匹配的 must 和 should 语句的分数 _score 求和，最后除以 must 和 should 语句的总数。

must_not 语句不会影响评分；它的作用只是将不相关的文档排除。
### 控制精度
所有 must 语句必须匹配，所有 must_not 语句都必须不匹配，但有多少 should 语句应该匹配呢？默认情况下，没有 should 语句是必须匹配的，只有一个例外：那就是当没有 must 语句的时候，至少有一个 should 语句必须匹配。

就像我们能控制 match 查询的精度 一样，我们可以通过 minimum_should_match 参数控制需要匹配的 should 语句的数量，它既可以是一个绝对的数字，又可以是个百分比：
```
GET /my_index/my_type/_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "title": "brown" }},
        { "match": { "title": "fox"   }},
        { "match": { "title": "dog"   }}
      ],
      "minimum_should_match": 2 
    }
  }
}
```
这个查询结果会将所有满足以下条件的文档返回： title 字段包含 "brown"
AND "fox" 、 "brown" AND "dog" 或 "fox" AND "dog" 。如果有文档包含所有三个条件，它会比只包含两个的文档更相关。

# 多字段搜索
## 多字符串查询
https://www.elastic.co/guide/cn/elasticsearch/guide/current/multi-query-strings.html
## 最佳字段
假设有个网站允许用户搜索博客的内容，以下面两篇博客内容文档为例：
```
PUT /my_index/my_type/1
{
    "title": "Quick brown rabbits",
    "body":  "Brown rabbits are commonly seen."
}

PUT /my_index/my_type/2
{
    "title": "Keeping pets healthy",
    "body":  "My quick brown fox eats rabbits on a regular basis."
}
```
用户输入词组 “Brown fox” 然后点击搜索按钮。事先，我们并不知道用户的搜索项是会在 title 还是在 body 字段中被找到，但是，用户很有可能是想搜索相关的词组。用肉眼判断，文档 2 的匹配度更高，因为它同时包括要查找的两个词：

现在运行以下 bool 查询：
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
但是我们发现查询的结果是文档 1 的评分更高：
```
{
  "hits": [
     {
        "_id":      "1",
        "_score":   0.14809652,
        "_source": {
           "title": "Quick brown rabbits",
           "body":  "Brown rabbits are commonly seen."
        }
     },
     {
        "_id":      "2",
        "_score":   0.09256032,
        "_source": {
           "title": "Keeping pets healthy",
           "body":  "My quick brown fox eats rabbits on a regular basis."
        }
     }
  ]
}
```
为了理解导致这样的原因，需要回想一下 bool 是如何计算评分的：
* 它会执行 should 语句中的两个查询。
* 加和两个查询的评分。
* 乘以匹配语句的总数。
* 除以所有语句总数（这里为：2）。

文档 1 的两个字段都包含 brown 这个词，所以两个 match 语句都能成功匹配并且有一个评分。文档 2 的 body 字段同时包含 brown 和 fox 这两个词，但 title 字段没有包含任何词。这样， body 查询结果中的高分，加上 title 查询中的 0 分，然后乘以二分之一，就得到比文档 1 更低的整体评分。

在本例中， title 和 body 字段是相互竞争的关系，所以就需要找到单个 最佳匹配 的字段。

如果不是简单将每个字段的评分结果加在一起，而是将 最佳匹配 字段的评分作为查询的整体评分，结果会怎样？这样返回的结果可能是： 同时 包含 brown 和 fox 的单个字段比反复出现相同词语的多个不同字段有更高的相关度。

### dis_max 查询
不使用 bool 查询，可以使用 dis_max 即分离 最大化查询（Disjunction Max Query） 。分离（Disjunction）的意思是 或（or） ，这与可以把结合（conjunction）理解成 与（and） 相对应。分离最大化查询（Disjunction Max Query）指的是： 将任何与任一查询匹配的文档作为结果返回，但只将最佳匹配的评分作为查询的评分结果返回 ：
```
{
    "query": {
        "dis_max": {
            "queries": [
                { "match": { "title": "Brown fox" }},
                { "match": { "body":  "Brown fox" }}
            ]
        }
    }
}
```
得到我们想要的结果为：
```
{
  "hits": [
     {
        "_id":      "2",
        "_score":   0.21509302,
        "_source": {
           "title": "Keeping pets healthy",
           "body":  "My quick brown fox eats rabbits on a regular basis."
        }
     },
     {
        "_id":      "1",
        "_score":   0.12713557,
        "_source": {
           "title": "Quick brown rabbits",
           "body":  "Brown rabbits are commonly seen."
        }
     }
  ]
}
```
## multi_match 查询
https://www.elastic.co/guide/cn/elasticsearch/guide/current/multi-match-query.html
# 近似匹配
## 短语匹配
就像 match 查询对于标准全文检索是一种最常用的查询一样，当你想找到彼此邻近搜索词的查询方法时，就会想到 match_phrase 查询。
```
GET /my_index/my_type/_search
{
    "query": {
        "match_phrase": {
            "title": "quick brown fox"
        }
    }
}
```
类似 match 查询， match_phrase 查询首先将查询字符串解析成一个词项列表，然后对这些词项进行搜索，但只保留那些包含 全部 搜索词项，且 位置 与搜索词项相同的文档。 比如对于 quick fox 的短语搜索可能不会匹配到任何文档，因为没有文档包含的 quick 词之后紧跟着 fox 。
#### 词项的位置
当一个字符串被分词后，这个分析器不但会返回一个词项列表，而且还会返回各词项在原始字符串中的 位置 或者顺序关系：
```
GET /_analyze?analyzer=standard
Quick brown fox
```
返回信息如下：
```
{
   "tokens": [
      {
         "token": "quick",
         "start_offset": 0,
         "end_offset": 5,
         "type": "<ALPHANUM>",
         "position": 1 
      },
      {
         "token": "brown",
         "start_offset": 6,
         "end_offset": 11,
         "type": "<ALPHANUM>",
         "position": 2 
      },
      {
         "token": "fox",
         "start_offset": 12,
         "end_offset": 15,
         "type": "<ALPHANUM>",
         "position": 3 
      }
   ]
}
```
位置信息可以被存储在倒排索引中，因此 match_phrase 查询这类对词语位置敏感的查询， 就可以利用位置信息去匹配包含所有查询词项，且各词项顺序也与我们搜索指定一致的文档，中间不夹杂其他词项。
#### 什么是短语
一个被认定为和短语 quick brown fox 匹配的文档，必须满足以下这些要求：

* quick 、 brown 和 fox 需要全部出现在域中。
* brown 的位置应该比 quick 的位置大 1 。
* fox 的位置应该比 quick 的位置大 2 。
* 如果以上任何一个选项不成立，则该文档不能认定为匹配。

> 本质上来讲，match_phrase 查询是利用一种低级别的 span 查询族（query family）去做词语位置敏感的匹配。 Span 查询是一种词项级别的查询，所以它们没有分词阶段；它们只对指定的词项进行精确搜索。
值得庆幸的是，match_phrase 查询已经足够优秀，大多数人是不会直接使用 span 查询。 然而，在一些专业领域，例如专利检索，还是会采用这种低级别查询去执行非常具体而又精心构造的位置搜索。

### slop 参数
slop 参数告诉 match_phrase 查询词条相隔多远时仍然能将文档视为匹配 。 相隔多远的意思是为了让查询和文档匹配你需要移动词条多少次

https://www.elastic.co/guide/cn/elasticsearch/guide/current/slop.html

# 部分匹配
## prefix前缀查询
https://www.elastic.co/guide/cn/elasticsearch/guide/current/prefix-query.html
## 通配符与正则表达式查询
https://www.elastic.co/guide/cn/elasticsearch/guide/current/_wildcard_and_regexp_queries.html


