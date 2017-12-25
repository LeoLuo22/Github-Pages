title: Solr高亮及搜索逻辑思考
---
# Solr高亮及搜索逻辑
   

----------
   
## 0x00 前言
   
马上就要发版本了，这次版本要新上对产品和功能的搜索。    
   
但是，却有一个问题，高亮问题。   
   
比如：  
   
搜索 111   
   
返回:中银易贷【**1243215321534**】【 12345674687】   
   
可以看到，整个数字串都被高亮了。同样，搜索 证，搜索结果中的**证券**期货，证券会被高亮。明显不是我们想要的效果。      
   
## 分析   
   
Solr对域(field)和域类型(fieldType)以及分词策略都定义在managed-schema里面。   
   
先看一下之前其他人定义的manage-schema的关键部分。    


	<fieldType name="textComplex" class="solr.TextField" positionIncrementGap="100">
      <analyzer>        
        <tokenizer class="com.chenlb.mmseg4j.solr.MMSegTokenizerFactory" mode="complex" dicPath="/bmsvr/solrdata/dic" />
		<filter class="solr.SynonymFilterFactory" synonyms="synonyms.txt" ignoreCase="true" expand="true"/>
		<filter class="solr.LowerCaseFilterFactory" />
		<filter class="solr.EdgeNGramFilterFactory" minGramSize="1" maxGramSize="50"  />
	  </analyzer>	  
    </fieldType>

	<field name="id" type="string" indexed="true" stored="true" required="true" multiValued="false"/>
	<field name="indexTypeCode" type="string" indexed="false" stored="true"/>
	<field name="indexTypeName" type="textComplex" indexed="true" stored="true"/>
	<field name="proCode" type="textComplex" indexed="true" stored="true" multiValued="false"/>
    <field name="proName" type="textComplex" indexed="true" stored="true"/>
	<field name="proUrl" type="string" indexed="false" stored="true"/>
   
    <field name="funCode" type="string" indexed="false" stored="true"/>
	<field name="funName" type="textComplex" indexed="true" stored="true"/>
	<field name="funUrl" type="string" indexed="false" stored="true"/>
	
	<field name="dataType" type="string" indexed="true" stored="true"/>
	
	<field name="product" type="string" indexed="true" stored="true"/>
	
	<field name="suggest" type="textComplex"  indexed="true"  stored="false" multiValued="true"/>
	
	<field name="text" type="textComplex" stored="false" indexed="true" multiValued="true"/>
	
    <field name="_version_" type="long" indexed="true" stored="true"/>
  
    <uniqueKey>id</uniqueKey>
   
    <defaultSearchField>text</defaultSearchField>
  
    <copyField source ="proCode" dest="text"/>                              
	<copyField source ="proName" dest="text"/>
	<copyField source ="funName" dest="text"/>
	<copyField source ="indexTypeName" dest="text"/>
	
	<copyField source ="proCode" dest="suggest"/>                              
	<copyField source ="proName" dest="suggest"/>
    <copyField source ="funName" dest="suggest"/>
	<copyField source ="indexTypeName" dest="suggest"/>
   
默认的搜索域是text，text的类型是textComplex，textComplex的分词策略如下：      
   
    <fieldType name="textComplex" class="solr.TextField" positionIncrementGap="100">
    <analyzer>        
    <tokenizer class="com.chenlb.mmseg4j.solr.MMSegTokenizerFactory" mode="complex" dicPath="/bmsvr/solrdata/dic" />
    <filter class="solr.SynonymFilterFactory" synonyms="synonyms.txt" ignoreCase="true" expand="true"/>
    <filter class="solr.LowerCaseFilterFactory" />
    <filter class="solr.EdgeNGramFilterFactory" minGramSize="1" maxGramSize="50"  />
    </analyzer>     
    </fieldType>   
   
怀疑是否是分词策略的原因。   
   
可以看到，分词策略首先是匹配停用词(stopword)，然后匹配同义词，然后转换为小写，最后使用edgengram分词。   

凭证的分词结果是凭，凭证。   
111的分词结果是 1, 11, 111,type是word   
   
![图1](http://oy2p9zlfs.bkt.clouddn.com/%E5%9B%BE1.PNG)   
   
solrconfig.xml有关高亮的配置如下：   
	   
	   <!--高亮-->
       <str name="hl">on</str>
       <str name="hl.fl">proCode proName funName</str>
       <str name="hl.encoder">html</str>
       <str name="hl.simple.pre"><b></str>
       <str name="hl.simple.post"></b></str>
       <str name="f.proCode.hl.fragsize">0</str>
       <str name="f.proCode.hl.alternateField">proCode</str>
       <str name="f.proName.hl.fragsize">0</str>
       <str name="f.proName.hl.alternateField">proName</str>
       <str name="f.funName.hl.fragsize">0</str>
       <str name="f.funName.hl.alternateField">funName</str>
     </lst>   
   
查阅了Solr的官方文档，只说明了高亮的配置，并没有介绍高亮的逻辑。   
   
之前还有一个问题，产品索引库了存在proName为凭证的索引内容，但是搜索 证券，凭证就是搜不出来。   
   
我查看了一下他俩的分词结果：   
   
凭证的分词是凭，凭证   
   
证券期货的分词是证，证券。   
   
那么，会不会是搜索结果与搜索内容和搜索结果的分词结果相关？    
   
为了测试，我修改了分词
   
默认的proCode和proName两个域都是text类型。但是后来我把proCode的分词策略改了。以搜索 111q 为例。   
   
高亮返回：   
   
    "20171110220007683664-874947726": {
      "proCode": [
        "xian-<em>111</em>5"
      ],
      "proName": [
        "xian-<em>1115</em>"
      ]
    },   
   
可以看到proCode只有111加了高亮，而proName整个1115都加了高亮。   
   
xian-1115的proCode的分词结果为：   
   
x   
xi   
i   
3   
ia   
a   
an   
n         
-1   
1   
11   
1   
11   
1   
15   
5   
   
而text的分词结果为：   
   
x   
xi   
xia   
xian   
1   
11   
111   
1115    
   
see?proCode高亮了111, proName高亮了1115。貌似是根据结果域分词最长的句子来匹配的。    
   
为了验证，我在数据库添加了一条数据，proCode和proName都是100011ab   
   
搜索100011
   
    "201711061433224775011720779125": {
      "proCode": [
        "<em>100011</em>ab"
      ],
      "proName": [
        "<em>100011ab</em>"
      ]
    },   
   
proCode分词结果：   
   
1   
10   
0   
00   
0   
00   
0   
01   
1   
11   
1   
1a   
a   
ab   
b   
   
proName的分词结果是：   
   
1   
10   
100   
1000   
10001   
100011   
100011a   
100011ab    
   
高亮的应该是结果域的分词结果能和搜索内容匹配的最多的。   
   
而且，还与搜索内容的分词结果相关。   
   
比如，搜索100011ab，默认是text分词。   
   
现象：proCode只高亮了100011   
   
搜索proCode:100011ab   
   
proCode高亮了100011ab   
   
贪心匹配。   
   
所以我对高亮策略的猜想：搜索内容的分词结果与结果的分词结果贪心匹配。   
   

## Solution   
   
第一个想法是 既然是对结果分词的贪心匹配   
   
那 如果把结果域分得足够细呢？
   
    <fieldType name="proCodeIndex" class="solr.TextField">
      <analyzer>
	    <tokenizer class="solr.NGramTokenizerFactory" minGramSize="1" maxGramSize="1" />
		<filter class="solr.LowerCaseFilterFactory" />
		<filter class="solr.ClassicFilterFactory" />
	  </analyzer>	  
    </fieldType>   
   
我把结果域使用了上述划分，划分效果如图2.   
   
   ![image](http://oy2p9zlfs.bkt.clouddn.com/%E5%9B%BE2.PNG)
   
可以看到，直接是拆成了单字的形式。同样的，我对搜索域的分词策略也进行了修改。   

    fieldType name="textComplex" class="solr.TextField" positionIncrementGap="100">
      <analyzer>        
        <tokenizer class="com.chenlb.mmseg4j.solr.MMSegTokenizerFactory" mode="complex" dicPath="/bmsvr/solrdata/dic" />
		<filter class="solr.SynonymFilterFactory" synonyms="synonyms.txt" ignoreCase="true" expand="true"/>
		<filter class="solr.LowerCaseFilterFactory" />
		<!-- <filter class="solr.EdgeNGramFilterFactory" minGramSize="1" maxGramSize="50"  /> -->
		<filter class="solr.NGramFilterFactory" minGramSize="1" maxGramSize="5"/>
		<filter class="solr.RemoveDuplicatesTokenFilterFactory" />
	  </analyzer>	  
    </fieldType>   
   
分词效果如图3   
   
   ![image](http://oy2p9zlfs.bkt.clouddn.com/%E5%9B%BE3.PNG)
   

最后，将结果域的copyField设置为了text   
   
    <copyField source ="proCode" dest="text"/>
	<copyField source ="proName" dest="text"/>
	<copyField source ="funName" dest="text"/>
	<copyField source ="indexTypeName" dest="text"/>
	
	<copyField source ="proCode" dest="suggest"/>                              
	<copyField source ="proName" dest="suggest"/>
    <copyField source ="funName" dest="suggest"/>
	<copyField source ="indexTypeName" dest="suggest"/>
   
现在的高亮效果如下。   
   
   ![image](http://oy2p9zlfs.bkt.clouddn.com/%E5%9B%BE4.PNG)
   ![image](http://oy2p9zlfs.bkt.clouddn.com/%E5%9B%BE5.PNG)
可以看到，只会对与搜索内容匹配的高亮，而不是高亮一大块。但目前还有的小问题就是不会高亮全角的字符。