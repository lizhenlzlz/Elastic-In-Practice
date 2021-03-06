## 16-深度探秘搜索技术-使用copy_to定制组合field解决cross-fields搜索弊端

上一讲，我们其实说了，用most_fields策略，去实现cross-fields搜索，有3大弊端，而且搜索结果也显示出了这3大弊端

**第一个办法**：用copy_to，将多个field组合成一个field

问题其实就出在有多个field，有多个field以后，就很尴尬，我们只要想办法将一个标识跨在多个field的情况，合并成一个field即可。比如说，一个人名，本来是first_name，last_name，现在合并成一个full_name，不就ok了吗。。。。。

```json
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



**用了这个copy_to语法之后，就可以将多个字段的值拷贝到一个字段中，并建立倒排索引**

```json
POST /forum/article/_bulk
{ "update": { "_id": "1"} }
{ "doc" : {"new_author_first_name" : "Peter", "new_author_last_name" : "Smith"} }	
{ "update": { "_id": "2"} }	
{ "doc" : {"new_author_first_name" : "Smith", "new_author_last_name" : "Williams"} }
{ "update": { "_id": "3"} }
{ "doc" : {"new_author_first_name" : "Jack", "new_author_last_name" : "Ma"} }		
{ "update": { "_id": "4"} }
{ "doc" : {"new_author_first_name" : "Robbin", "new_author_last_name" : "Li"} }		
{ "update": { "_id": "5"} }
{ "doc" : {"new_author_first_name" : "Tonny", "new_author_last_name" : "Peter Smith"} }

GET /forum/article/_search
{
  "query": {
    "match": {
      "new_author_full_name":       "Peter Smith"
    }
  }
}
```



很无奈，很多时候，我们很难复现。比如官网也会给一些例子，说用什么什么文本，怎么怎么搜索，是怎么怎么样的效果。es版本在不断迭代，这个打分的算法也在不断的迭代。所以我们其实很难说，对类似这几讲讲解的best_fields，most_fields，cross_fields，完全复现出来应有的场景和效果。

更多的把原理和知识点给大家讲解清楚，带着大家演练一遍怎么操作的，做一下实验

期望的是说，比如大家自己在开发搜索应用的时候，碰到需要best_fields的场景，知道怎么做，知道best_fields的原理，可以达到什么效果；碰到most_fields的场景，知道怎么做，以及原理；碰到搜搜cross_fields标识的场景，知道怎么做，知道原理是什么，效果是什么。。。。



**问题1**：只是找到尽可能多的field匹配的doc，而不是某个field完全匹配的doc --> **解决**，最匹配的document被最先返回

**问题2**：most_fields，没办法用minimum_should_match去掉长尾数据，就是匹配的特别少的结果 --> **解决**，可以使用minimum_should_match去掉长尾数据

**问题3**：TF/IDF算法，比如Peter Smith和Smith Williams，搜索Peter Smith的时候，由于first_name中很少有Smith的，所以query在所有document中的频率很低，得到的分数很高，可能Smith Williams反而会排在Peter Smith前面 --> **解决**，Smith和Peter在一个field了，所以在所有document中出现的次数是均匀的，不会有极端的偏差

