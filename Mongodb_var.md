### 由于公司存储数据有用到mongodb，主要是用来存储一些json数据信息——acmp，集群，资源池，镜像，云主机，admin，orgenazation，User等资源信息



#### 所以这次趁着假期得末尾，突然向学一下mongodb，安装和使用就不说了，集中数据库都基础都很简单，主备和副本集模式也比较常见，公司在这些业务上主要用的是副本集，当然还有cluster分片[分库分表]，由于自己工作这个框架封装的挺好的，外人是看不到我们运行代码时，资源是如何从db取出来的，其实也就是一些很简答的CURD。但是其实对于数据库，要了解的很多，例如瓶颈等问题要知道如何出现才能知道如何解决等，这次花点时间学习一下，然后今天是我对象的生日，她在吃饭饭准备出门了，我这就写点东西打发一下时间





##### BSON 文档

~~~reStructuredText

~~~

BSON就是对JSON的一个拓展延申而已

1.BSON文档的尺寸：一个document文档的尺寸为16M；**大于16M的文档需要存储在GridFS中**

2.文档内嵌深度：BSON文档的结构——tree，**深度最大为100**

##### Namespaces

1.collection命名空间：<databases>.<collection>，**最大长度为120字节**。这也限定了database和collection的名字是不能过长的

2.命名空间的个数：对于**MMAPV1引擎**，个数**最大约为24000个**，每个collection以及index都是一个namespace；**对于wiredTiger引擎则没有这个限制**

3.namespace文件的大小：对于**MMAPV1引擎而言，默认大小为16M，可以通过在配置文件中修改**，**对于wiredTiger引擎则没有这个限制**

##### Indexes

1.indexes key：每条索引的key不得超过1024字节，如果index key长度超过此长度，**将会导致write操作失败**

2.**每个collection中索引的个数不能超过64个**

3.索引名称：我们可以为index设定名称，最终全名为<database name>.<collection name>.$<index name>，**最长不得超过128字节**。默认情况下<index name>为filed名称与index类型的组合，我们可以在创建索引时显式的指定index名字，参见createIndex()方法

4.组合索引**最多能包含31个filed**

##### Data

1.Capped Collection：如果你在创建“Capped”类型的collection时**指定了文档的最大个数**，那么此个数**不能超过2的32次方**个，如果**没有指定最大个数**，则没有限制

2.Database Size ：MMAPV1引擎而言，每个database不得持有超过16000个数据文件，即单个database的总数量最大为32TB，可以通过设置‘smallFiles’来限定到8TB

3.Data Size：对于MMAVP1引擎而言，单个mongod不能管理**超过最大虚拟内存地址空间**的数据集，比如**linux64位下每个mongod实例最多可以维护64TB数据**。**wiredTiger则没有此限制**

4.每个Database中collection的个数：对于MMAPV1引擎而言，**每个database所能持有的collection个数取决于 namespace文件大小[用来保存namespace]以及每个collection中indexes的个数**，**最终总尺寸不超过namespace文件的大小**16M。**wiredTiger则不受到此限制**

##### Replica Sets

1.每个replica set中**最多支持50个members**

2.replica set中**最多能有7个voting members【投票者】**

3.如果没有显式的指定oplog的尺寸，其最大不会超过50G

##### Sharded Clusters

1.group聚合函数，在**sharding模式下**不可用。请使用**mapreduce或者aggregate方法**

2.Coverd Queries： 即查询条件中的Fields必须式index的一部分，且返回结果只包含index中的fields；对于sharding集群，如果query中不包含shard key，索引则无法进行覆盖。虽然_id不是‘shard key’，但是如果查询条件中只包含\_id,且返回结果中也只需要\_id字段值，则可以使用覆盖查询，不过这个查询——有屁的意思啊？哦，也不是，可以检查此\_id的document是否存在，嗯不要断然就说人家没用呢，你才没用呢，哼，人家超有用的，不信以后走着瞧？

3.对（于）已经存有数据的collections开启sharding(之前非sharding)，则其**最大数据不能超过256G。当collection被sharding之后，那么它可以存储任意多的数据**——这段话咋读这么别扭，到底是256还是没限制啊??——事情是这样的，你把对于读称对，就好理解了——就是说要给一个已经存货的collections改成分片机制，前提是这些货你不能超过256G，改了之后随你存多少！懂了吧？总的来说就是你要采用另一种模式，那你之前的模式存货不能太多，不然人家带不动。所以说要跳槽的同学，前提是不能落后太多了哦

4.对于sharded collection，update、remove对单条数据操作（**操作选项为multi：false或者justOne**），必须指定shard key或者_id字段；**否则将会抛出错误**

5.唯一索引：shards之间不支持唯一索引，除非这个shard key是唯一所以的最左前缀。比如collection的shard key为{"zipcode":1,"name":1}，如果你想对collection创建唯一索引，那么索引必须将zipcode和name作为索引的最左前缀，比如：collection.createIndex({"zipcode":1,"name":1,"company":1},{unique:true})

6.在chunk迁移时允许的最大文档个数：如果一个chunk中documents的个数超过250000（默认chunk大小为64M）时，或者document个数大于1.3*(chunk最大尺寸(有配置参数决定)/document平均尺寸)，此chunk将无法被"move"无论时balancer还是人工干预，必须等待split之后才能被move

##### shard key

1.shard key 的长度不得查过512字节

2."shard key"索引可以为基于shard key的正序索引，或者以shard key开头的组合索引。shard key索引不能是multikey(基于数组的索引)，text索引或者geo索引

3."shard key"是不可变的，无论何时都不能修改document中的shard key，如果需要更改shard key，则需要手动清洗数据，即全量dump原始数据，然后修改并保存在全新的collection中【这就比较恐怖了哈】

4.**单调递增或者递减的shard key会限制insert的吞吐量**；如果_id是shard key，*需要知道\_id是ObjectId生成，它也是自增值*。**对于单调递增的shard key，collection上所有insert操作都会在一个shard节点上进行**，——这就是你在写测试代码用for循环去加十万条数据，可能99989条数据都在第一个shard节点上，只有剩下11条在剩下的节点上【不信的自己试试】——所以shard  key用sha1等加密算法进行处理比较好，mongo内部也有处理，但是做的不是很好。那么此shard将会承载cluster的全部insert操作，因为单个shard节点资源有限，因此整个cluster的insert量会因此受限。如果cluster主要是read、update操作，将不会有这方面的限制。为了避免整个问题，可以考虑使用"hashed shard key"或者选择非单调递增key作为shard key——**rang shard key 和 hashed shard key 各有优缺点，需要根据query的情况而定

##### Operations

1.如果mongodb不能使用索引排序来获取documents，那么参与排序的documents，尺寸需小于32M

2.aggregation Pipeline操作。Pipeline stages限制在100M内存，如果stage超过此限制将会发生错误，为了能处理较大的数据集，请开启allowDiskUse"选项，即允许pipeline stage将额外的数据写入临时文件

##### 命名规则

1.database的命名区分大小写

2.database名称中不要包含： /\\."$*<>:|?

3..database名称长度不能超过64字符

4.collection名称可以以"_"或者字母开头，但是不能包含$符号，不能为空字符或者null，不能以"system."开头，因为这是系统保留字

5.document字段不能包含"."或者null，且不能以"$"开头，因为\$是一个“引用符号”

#### 最后记录一下json嵌套中含有列表的查询方法，样例数据

~~~python
{
    "_id":ObjectId(5c6cc376a589c200018f7312),
    "id":"9472",
    "data":{
        "name":"测试",
        "publish_date":"2009-5-15",
        "authors":[
            {
                "author_id":3053,
                "author_name":"测试数据"
            }
        ],
    }
}
#我要查询authors中的author_id,query可以这样写
db.getCollection().find({"data.authors.0.author_id":3053})
#用0来代替第一个索引，点代表嵌套结构。但是spark mongo中是不能这样导入的哦，需要使用别的方法
~~~



###### 差不多就这些吧，祝我媳妇儿李嘉欣生日快乐，也不知道她许愿了没，许的什么愿，和我与没有关系，能不能很快实现。加油吧



