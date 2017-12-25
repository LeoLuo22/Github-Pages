title: 如何在SolrCloud中添加删除节点
---
# 如何在SolrCloud中添加删除节点  
   
   ----------

> 本人原创，转载请注明出处
> Blog:[Why So Serious](http://leoluo.top "LeoLuo")   
> Github: [LeoLuo22](https://github.com/LeoLuo22/)   
> CSDN: [我的CSDN](http://blog.csdn.net/u013271714)   

----------
## 0x00 准备   
   
*注: 默认你已经搭建好了zookeeper集群。并且已经把配置文件夹上传到了zookeeper。*   
   
创建一个Solrhome文件夹。这个文件夹里保存的是solr.xml文件。   
   
    mkdir <solr.home for new solr node>
    cp <existing solr.xml path> <new solr.home>

solr.xml文件内容大概是这样：   
   
![图1](http://oy2p9zlfs.bkt.clouddn.com/%E5%9B%BE1.png)   
   
## 0x01 方案1
   
之前采用的方案是当节点增加时创建新的分区，删除节点删除对应的分区就可以了。   

-  创建集合   
   
`http://22.11.97.12:8983/solr/admin/collections?action=CREATE&name=product-collection&router.name=implicit&shards=shard1,shard2&collection.configName=myconf`   
   
其中，注意route.name要设为impicit的，这样的话以后才能动态添加删除分区。shards根据你的当前的Solr服务器数目决定，目前我有两台服务器，所以设为shard1和shard2。 还有要指定配置文件夹名称，配置文件夹我是放在Zookeeper的/configs/myconf。   
   
此时的集群如下：   
   
![图2](http://oy2p9zlfs.bkt.clouddn.com/%E5%9B%BE2.png)
   
- 创建分区   
   
启动你要新增的节点。运行创建分区的命令。   
   
    http://22.11.97.18:8987/solr/admin/collections?action=CREATESHARD&shard=shard3&collection=function-collection   
   
   
- 删除分区   
   
    http://22.11.97.18:8987/solr/admin/collections?action=DELETESHARD&shard=shard3&collection=function-collection   
   
满足了动态添加删除节点的需求。但是有个问题，我发现这种方式添加的节点都是主节点。这样的话如果我把某台服务器停掉的话，会是什么效果？   
   
试验了一下，直接报了500错误。   
   
所以，这种方案不可取。   

----------

## 0x02 方案2   
   
这是目前采用的方案。   
该方案是我在官方文档上找到了一个ADDREPLICA的API，顾名思义，就是添加副本(replica)的。   
   
- 创建集合   
   
    http://22.11.97.11:8983/solr/admin/collections?action=CREATE&name=function-collection&replicationFactor=3&collection.configName=myconf&numShards=1   
   
![图4](http://oy2p9zlfs.bkt.clouddn.com/%E5%9B%BE4.png)
   
replicationFactor是副本的个数，以你当前的服务器数为准。numShards是分区数，目前设置为1个分区。   
   
- 添加节点   
   
    http://22.11.97.18:8983/solr/admin/collections?action=ADDREPLICA&collection=function-collection&shard=shard1   
   
- 删除节点   
   
默认，从上往下是core_node1,code_node2,core_node3   
   
    http://22.11.97.11:8983/solr/admin/collections?action=DELETEREPLICA&collection=function-collection&shard=shard1&replica=core_node1   
   
![图5](http://oy2p9zlfs.bkt.clouddn.com/%E5%9B%BE5.png)
   

  
## 0x03 小结   
   
目前我所知道的动态添加删除节点的方式就两种。一种是通过将集合切割分区的方式，但是这种方式有两个弊端。其一是目前的集合的文档数还不多，不到2w，切割太多的分区没有意义，一个就好。其二是不知道为什么这种方式导致新增的节点都是主节点，所以一台节点挂掉之后整个集群就挂了。   
   
而第二种方式，也就是是添加分区的副本(replica)的方式，这种方式的话Solr会自动选举主节点，避免了单个节点挂掉整个集群就奔溃。而且日后如果索引太多的话，可以通过切割分区的方式来划分。   
   
后续如果索引太多可以使用   
   
    http://22.11.97.18:8983/solr/admin/collections?action=SPLITSHARD&name=function-collection&shard=shard1   
   
来切割分区。