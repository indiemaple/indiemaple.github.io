# Elasticsearch使用

* 索引，对于需要搜索的数据，如何建立合适的索引，还需要根据特定的语言使用不同的analyzer等。
* 搜索，Elasticsearch提供了非常强大的搜索功能，如何写出高效的搜索语句？
* 数据源，我们所有的数据是存放到MySQL的，MySQL是唯一数据源，如何将MySQL的数据导入到Elasticsearch？

对于1和2，因为我们的数据都是从MySQL生成，index的field是固定的，主要做的工作就是根据业务场景设计好对应的mapping以及search语句就可以了，当然实际不可能这么简单，需要我们不断的调优。

而对于3，则是需要一个工具将MySQL的数据导入Elasticsearch，因为我们对搜索实时性要求很高，所以需要将MySQL的增量数据实时导入，笔者唯一能想到的就是通过row based binlog来完成。而近段时间的工作，也就是实现一个MySQL增量同步到Elasticsearch的服务。



## Lucene

Elasticsearch底层是基于Lucene的，Lucene是一款优秀的搜索lib，当然，笔者以前仍然没有接触使用过。:-)

Lucene关键概念：
* Document：用来索引和搜索的主要数据源，包含一个或者多个Field，而这些Field则包含我们跟Lucene交互的数据。
* Field：Document的一个组成部分，有两个部分组成，name和value。
* Term：不可分割的单词，搜索最小单元。
* Token：一个Term呈现方式，包含这个Term的内容，在文档中的起始位置，以及类型。

Lucene使用Inverted index来存储term在document中位置的映射关系。
譬如如下文档：
* Elasticsearch Server 1.0 （document 1）
* Mastring Elasticsearch （document 2）
* Apache Solr 4 Cookbook （document 3）

使用inverted index存储，一个简单地映射关系：

| Term | Count | Docuemnt |
| --- | --- | --- |
| 1.0 | 1 | <1> |
| Apache | 1 | <3> |
| Cookbook | 1 | <3> |
| Elasticsearch | 2 | <1>.<2> |
| Mastering | 1 | <2>| 


对于上面例子，我们首先通过分词算法将一个文档切分成一个一个的token，再得到该token与document的映射关系，并记录token出现的总次数。这样就得到了一个简单的inverted index。

## Elasticsearch关键概念
要使用Elasticsearch，笔者认为，只需要理解几个基本概念就可以了。

在数据层面，主要有：
* Index：Elasticsearch用来存储数据的逻辑区域，它类似于关系型数据库中的db概念。一个index可以在一个或者多个shard上面，同时一个shard也可能会有多个replicas。
* Document：Elasticsearch里面存储的实体数据，类似于关系数据中一个table里面的一行数据。document由多个field组成，不同的document里面同名的field一定具有相同的类型。document里面field可以重复出现，也就是一个field会有多个值，即multivalued。
* Document type：为了查询需要，一个index可能会有多种document，也就是document type，但需要注意，不同document里面同名的field一定要是相同类型的。
* Mapping：存储field的相关映射信息，不同document type会有不同的mapping。

对于熟悉MySQL的童鞋，我们只需要大概认为Index就是一个db，document就是一行数据，field就是table的column，mapping就是table的定义，而document type就是一个table就可以了。

Document type这个概念其实最开始也把笔者给弄糊涂了，其实它就是为了更好的查询，举个简单的例子，一个index，可能一部分数据我们想使用一种查询方式，而另一部分数据我们想使用另一种查询方式，于是就有了两种type了。不过这种情况应该在我们的项目中不会出现，所以通常一个index下面仅会有一个type。

在服务层面，主要有：
* Node: 一个server实例。
* Cluster：多个node组成cluster。
* Shard：数据分片，一个index可能会存在于多个shards，不同shards可能在不同nodes。
* Replica：shard的备份，有一个primary shard，其余的叫做replica shards。

Elasticsearch之所以能动态resharding，主要在于它最开始就预先分配了多个shards（貌似是1024），然后以shard为单位进行数据迁移。这个做法其实在分布式领域非常的普遍，codis就是使用了1024个slot来进行数据迁移。

因为任意一个index都可配置多个replica，通过冗余备份的方式保证了数据的安全性，同时replica也能分担读压力，类似于MySQL中的slave。

## Restful API

Elasticsearch提供了Restful API，使用json格式，这使得它非常利于与外部交互，虽然Elasticsearch的客户端很多，但笔者仍然很容易的就写出了一个简易客户端用于项目中，再次印证了Elasticsearch的使用真心很容易。

Restful的接口很简单，一个url表示一个特定的资源，譬如/blog/article/1，就表示一个index为blog，type为aritcle，id为1的document。

而我们使用http标准method来操作这些资源，POST新增，PUT更新，GET获取，DELETE删除，HEAD判断是否存在。

这里，友情推荐httpie，一个非常强大的http工具，个人感觉比curl还用，几乎是命令行调试Elasticsearch的绝配。

一些使用httpie的例子:

```
# create
http POST :9200/blog/article/1 title="hello elasticsearch" tags:='["elasticsearch"]'

# get
http GET :9200/blog/article/1

# update
http PUT :9200/blog/article/1 title="hello elasticsearch" tags:='["elasticsearch", "hello"]'

# delete
http DELETE :9200/blog/article/1

# exists
http HEAD :9200/blog/article/1
```

## 索引和搜索

虽然Elasticsearch能自动判断field类型并建立合适的索引，但笔者仍然推荐自己设置相关索引规则，这样才能更好为后续的搜索服务。

我们通过定制mapping的方式来设置不同field的索引规则。

而对于搜索，Elasticsearch提供了太多的搜索选项，就不一一概述了。

索引和搜索是Elasticsearch非常重要的两个方面，直接关系到产品的搜索体验，但笔者现阶段也仅仅是大概了解了一点，后续在详细介绍。

## 同步MySQL数据

Elasticsearch是很强大，但要建立在有足量数据情况下面。我们的数据都在MySQL上面，所以如何将MySQL的数据导入Elasticsearch就是笔者最近研究的东西了。

虽然现在有一些实现，譬如elasticsearch-river-jdbc，或者elasticsearch-river-mysql，但笔者并不打算使用。

elasticsearch-river-jdbc的功能是很强大，但并没有很好的支持增量数据更新的问题，它需要对应的表只增不减，而这个几乎在项目中是不可能办到的。

elasticsearch-river-mysql倒是做的很不错，采用了python-mysql-replication来通过binlog获取变更的数据，进行增量更新，但它貌似处理MySQL dump数据导入的问题，不过这个笔者真的好好确认一下？话说，python-mysql-replication笔者还提交过pull解决了minimal row image的问题，所以对elasticsearch-river-mysql这个项目很有好感。只是笔者决定自己写一个出来。

为什么笔者决定自己写一个，不是因为笔者喜欢造轮子，主要原因在于对于这种MySQL syncer服务（增量获取MySQL数据更新到相关系统），我们不光可以用到Elasticsearch上面，而且还能用到其他服务，譬如cache上面。所以笔者其实想实现的是一个通用MySQL syncer组件，只是现在主要关注Elasticsearch罢了。

项目代码在这里go-mysql-elasticsearch，现已完成第一阶段开发，内部对接测试中。

go-mysql-elasticsearch的原理很简单，首先使用mysqldump获取当前MySQL的数据，然后在通过此时binlog的name和position获取增量数据。

一些限制：
* binlog一定要变成row-based format格式，其实我们并不需要担心这种格式的binlog占用太多的硬盘空间，MySQL 5.6之后GTID模式都推荐使用row-based format了，而且通常我们都会把控SQL语句质量，不允许一次性更改过多行数据的。
* 需要同步的table最好是innodb引擎，这样mysqldump的时候才不会阻碍写操作。
* 需要同步的table一定要有主键，好吧，如果一个table没有主键，笔者真心会怀疑设计这个table的同学编程水平了。多列主键也是不推荐的，笔者现阶段不打算支持。
* 一定别动态更改需要同步的table结构，Elasticsearch只能支持动态增加field，并不支持动态删除和更改field。通常来说，如果涉及到alter table，很多时候已经证明前期设计的不合理以及对于未来扩展的预估不足了。

## 具体使用
如果你有 linux ，并且恰好也有 docker ， 那么请运行如下命令：
> sudo docker run -d -p 9200:9200 -p 9300:9300 elasticsearch

你要是看到一串 id ， 恭喜你， 你已经有了自己的搜索了！就是这么简单！
我们来验证一下搜索服务：
```josn
ubuntu:~$ curl localhost:9200
     {
       "status" : 200,
       "name" : "Adonis",
       "cluster_name" : "elasticsearch",
       "version" : {
         "number" : "1.7.1",
         "build_hash" : "b88f43fc40b0bcd7f173a1f9ee2e97816de80b19",
         "build_timestamp" : "2015-07-29T09:54:16Z",
         "build_snapshot" : false,
         "lucene_version" : "4.10.4"
       },
       "tagline" : "You Know, for Search"
     }
```

返回值 200 ！你成功了！这个结果除了告诉你 Elastic Search 已经启动好之外，还显示了版本号， build 信息， lucene 版本等信息。

如果你不用 Linux 或者 Docker ，放心，情况也不复杂。首先你需要 Java ，然后去官网下载，解压，运行既可。

Elastic Search 的初级功能十分易用。 Restful 请求就可以完成一切操作。所以接下来的体验，你只需要一个命令行即可，但是为了直观方便，你也可以选择官方推荐的 Marvel 或者 Postman。

相信很多人都是吃货，下面，我用中国第九大菜系食堂菜作为搜索的原数据演示 Elastic Search 的用法。

插入数据
首先来一道勉强正常的菜，苹果炖牛肉。

```
ubuntu:~$ curl -XPOST localhost:9200/food/canteen?pretty -d '{title: "苹果炖牛肉", source: "sjtu"}'
      {
       "_index" : "food",
       "_type" : "canteen",
       "_id" : "AU8HEQxfW68mQc8rk4AS",
       "_version" : 1,
       "created" : true
     }
```
有点迷糊？那来稍微介绍一点背景知识，看看上面的命令都做了啥。

Elastic Search 使用 document 方式存储， 类似 MongoDB 。它不仅高效而且分布式扩展性极佳，基于赫赫有名的 Lucene 搭建，在搜索界的地位就如同杰伦小公举在歌坛的地位一般。很多人可能有关系型数据库开发的经验，下面来张 Elastic Search 和 MySQL 的术语类比图，帮助理解。

| MySQL | Elastic Search |
| --- | --- |
| Database | Index |
| Table | Type |
| Row | Document |
| Column | Field |
| Schema | Mappping |
| Index | Everything Indexed by default |
| SQL | Query DSL |

上面例子中，localhost:9200是服务地址， Elastic Search 默认利用 9200 作为 Rest 服务的端口， 9300 作为内部通信接口和 Java 客户端接口。food是 index ，canteen是 type ，里面的数据形成 document 。值得注意的是，跟其他 NoSQL 数据库一样， Schema 不需要预先定义，直接插入数据即可， Elastic Search 会智能地分析每个 Field 的值并自动建立索引。每个 document 都有一个 id ，如果不指定则会自动生成。可以看到插入成功之后，生成了"_id" : "AU8HEQxfW68mQc8rk4AS"。插入命令使用 Post 请求，方便使用和测试。

另外，在上面的例子中，我加入了?pretty主要是让结果的 json 显示更好看。

虽然数据已经存好了，但是没有直接取到，总是心有不安，下面这条命令即可以查看数据：

```json
ubuntu:~$ curl -XGET localhost:9200/food/canteen/AU8HEQxfW68mQc8rk4AS?pretty
{
  "_index" : "food",
  "_type" : "canteen",
  "_id" : "AU8HEQxfW68mQc8rk4AS",
  "_version" : 1,
  "found" : true,
  "_source":{title: "苹果炖牛肉", source: "sjtu"}
}
```

可以看到，请求改为了GET，请求形式为/index/type/id。

再来一道菜，番茄炒菠萝，用到我们刚才说的POST请求。

```json
ubuntu:~$ curl -XPOST localhost:9200/food/canteen?pretty -d '{title: "番茄炒菠萝", source: "sjtu"}'
     {
       "_index" : "food",
       "_type" : "canteen",
       "_id" : "AU8HQW0-W68mQc8rk4Ab",
       "_version" : 1,
       "created" : true
     }
```

等等，这道菜是武汉大学的，看来我们得更新一下source。也许聪明的你已经猜到，发个PUT请求就可以搞定了。

```json
ubuntu:~$ curl -XPUT localhost:9200/food/canteen/AU8HQW0-W68mQc8rk4Ab?pretty -d '{title: "番茄炒菠萝", source: "whu"}'
     {
       "_index" : "food",
       "_type" : "canteen",
       "_id" : "AU8HQW0-W68mQc8rk4Ab",
       "_version" : 2,
       "created" : false
}
```

可以看到_version变成了 2 ，表示更新过了。再GET一下确认结果。
```json
ubuntu:~$ curl -XGET localhost:9200/food/canteen/AU8HQW0-W68mQc8rk4Ab?pretty
     {
       "_index" : "food",
       "_type" : "canteen",
       "_id" : "AU8HQW0-W68mQc8rk4Ab",
       "_version" : 2,
       "found" : true,
       "_source":{title: "番茄炒菠萝", source: "whu"}
     }
```
可以看到，_source，也就是插入的原始数据，确实改变了。

删除数据
继续来一道菜，月饼炒辣椒！
```json
ubuntu:~$ curl -XPOST localhost:9200/food/canteen?pretty -d '{title: "月饼炒辣椒", source: "fjnu"}'
     {
       "_index" : "food",
       "_type" : "canteen",
       "_id" : "AU8HRPoOW68mQc8rk4Ac",
       "_version" : 1,
       "created" : true
     }
```

算了，好像太丧心病狂了，对吃货的幼小心灵产生了暴击，我们还是把它删掉吧。
```json
ubuntu:~$ curl -XDELETE localhost:9200/food/canteen/AU8HRPoOW68mQc8rk4Ac?pretty
     {
       "found" : true,
       "_index" : "food",
       "_type" : "canteen",
       "_id" : "AU8HRPoOW68mQc8rk4Ac",
       "_version" : 2
     }
```

GET验证一下是否真的被删除了。
```json
ubuntu:~$ curl -XGET localhost:9200/food/canteen/AU8HRPoOW68mQc8rk4Ac?pretty
     {
       "_index" : "food",
       "_type" : "canteen",
       "_id" : "AU8HRPoOW68mQc8rk4Ac",
       "found" : false
     }
```
显示"found" : false，可见删除确实成功了。
搜索数据
Elastic Search 支持相当复杂的搜索情形。像“有水果有肉有主食红黄白相间成粒状”这种可以把 MySQL 的检索虐得死去活来的查询， Elastic Search 可以轻松告诉你这应该是名菜“哈密瓜年糕牛肉粒”。下面看看最简单却很实用的搜索：查询所有值。
```json
ubuntu:~$ curl -XGET localhost:9200/food/canteen/_search?pretty
     {
       "took" : 1,
        "timed_out" : false,
        "_shards" : {
          "total" : 5,
          "successful" : 5,
          "failed" : 0
        },
        "hits" : {
          "total" : 2,
         "max_score" : 1.0,
          "hits" : [ {
           "_index" : "food",
           "_type" : "canteen",
           "_id" : "AU8HEQxfW68mQc8rk4AS",
           "_score" : 1.0,
           "_source":{title: "苹果炖牛肉", source: "sjtu"}
         }, {
           "_index" : "food",
           "_type" : "canteen",
           "_id" : "AU8HQW0-W68mQc8rk4Ab",
           "_score" : 1.0,
           "_source":{title: "番茄炒菠萝", source: "whu"}
         } ]
       }
     }
```

可以看到，形如GET _search即可。结果会显示所有给定 index 和 type 下的 document 。hits下面包含找到的具体内容。_score表示相关程度，分数越高表示越相关，搜索引擎对于结果的排序都是通过类似的机制完成。
再来一个简单地查询，查找source是sjtu的菜。

```json
ubuntu:~$ curl -XGET localhost:9200/food/canteen/_search?pretty -d '{"query":{"match":{"source":"sjtu"}}}'
     {
       "took" : 35,
       "timed_out" : false,
       "_shards" : {
         "total" : 5,
         "successful" : 5,
         "failed" : 0
       },
       "hits" : {
         "total" : 1,
         "max_score" : 0.30685282,
         "hits" : [ {
           "_index" : "food",
           "_type" : "canteen",
           "_id" : "AU8HEQxfW68mQc8rk4AS",
           "_score" : 0.30685282,
           "_source":{title: "苹果炖牛肉", source: "sjtu"}
         } ]
       }
     }
```
这里用到了match query来搜索，可以看到_score的变化。
