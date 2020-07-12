# Lucene全文检索

## 一.全文检索

### 1.1数据分类

结构化数据和非结构化数据

结构化数据：指具有固定格式或有限长度的数据，如数据库，元数据等。

非结构化数据：指不定长或无固定格式的数据，如邮件，word文档等磁盘上的文件

### 1.2 结构化数据搜索

常见的结构化数据也就是数据库中的数据。在数据库中搜索很容易实现，通常都是使用sql语句进行查询，而且能很快的得到查询结果。

为什么数据库搜索很容易？

因为数据库中的数据存储是有规律的，有行有列而且数据格式、数据长度都是固定的。

### 1.3 非结构化数据查询

#### 1.3.1顺序扫描法

所谓顺序扫描，比如要找内容包含某一个字符串的文件，就是一个文档一个文档的看，对于每一个文档，从头看到尾，如果此文档包含此字符串，则此文档为我们要找的文件，接着看下一个文件，直到扫描完所有的文件。如利用windows的搜索也可以搜索文件内容，只是相当的慢。

#### 1.3.2全文检索

将非结构化数据中的一部分信息提取出来，重新组织，使其变得有一定结构，然后对此有一定结构的数据进行搜索，从而达到搜索相对较快的目的。这部分从非结构化数据中提取出的然后重新组织的信息，我们称之**索引**。

例如：字典。字典的拼音表和部首检字表就相当于字典的索引，对每一个字的解释是非结构化的，如果字典没有音节表和部首检字表，在茫茫辞海中找一个字只能顺序扫描。然而字的某些信息可以提取出来进行结构化处理，比如读音，就比较结构化，分声母和韵母，分别只有几种可以一一列举，于是将读音拿出来按一定的顺序排列，每一项读音都指向此字的详细解释的页数。我们搜索时按结构化的拼音搜到读音，然后按其指向的页数，便可找到我们的非结构化数据——也即对字的解释。

**这种先建立索引，再对索引进行搜索的过程就叫全文检索**(Full-text Search)。

虽然创建索引的过程也是非常耗时的，但是索引一旦创建就可以多次使用，全文检索主要处理的是查询，所以耗时间创建索引是值得的。

### 1.4 实现全文检索

可以使用Lucene实现全文检索。Lucene是apache下的一个开放源代码的全文检索引擎工具包。提供了完整的查询引擎和索引引擎，部分文本分析引擎。Lucene的目的是为软件开发人员提供一个简单易用的工具包，以方便的在目标系统中实现全文检索的功能。

### 1.5 应用场景

对于数据量大、数据结构不固定的数据可采用全文检索方式搜索，比如百度、Google等搜索引擎、论坛站内搜索、电商网站站内搜索等。

## 二. Lucene实现全文搜索的流程

### 2.1索引和搜索流程

![img](https://upload-images.jianshu.io/upload_images/14883387-88ea957882421106.png?imageMogr2/auto-orient/strip|imageView2/2/w/462/format/webp)

### 2.2创建索引

对文档索引的过程，将用户要搜索的文档内容进行索引，索引存储在索引库中。

#### 2.2.1获得原始文档

**原始文档**是指要索引和搜索的内容。原始内容包括互联网上的网页、数据库中的数据、磁盘上的文件等。

从互联网上、数据库、文件系统中等获取需要搜索的原始信息，这个过程就是信息采集，信息采集的目的是为了对原始内容进行索引。

在Internet上采集信息的软件通常称为爬虫或蜘蛛，也称为网络机器人，爬虫访问互联网上的每一个网页，将获取到的网页内容存储起来。

本案例我们要获取磁盘上文件的内容，可以通过文件流来读取文本文件的内容，对于pdf、doc、xls等文件可通过第三方提供的解析工具读取文件内容，比如Apache POI读取doc和xls的文件内容。

#### 2.2.2创建文档对象

获取原始内容的目的是为了索引，在索引前需要将原始内容创建成文档，文档中包括一个一个的域（Field），域中存储内容。

这里我们将磁盘上的一个文件当成一个document，Document中包括一些file_name 、file_path、 file_size file_content. eg:

![img](https://img-blog.csdn.net/20180404080520962?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM1OTc1Njg1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

注意：每个文档可以有多个Field，不同的Document可以有不同的Field，同一个Document可以有相同的Field（域名域值都相同）

每个文档都有唯一的编号，就是文档id

####  2.2.3分析文档

将原始文档内容创建为包含域的文档，需要再对语中的内容进行分析，分析的过程是经过对原始文档提取单词、将字母转为小写、去除标点符号、去除停用词等过程生成最终的语汇单元，可以将语汇单元理解为一个一个的单词。

比如下边的文档经过分析如下：

Lucene is a Java full-text search engine.  Lucene is not a completeapplication, but rather a code library and API that can easily be usedto add search capabilities to applications.

分析后得到的单词：

lucene java full search engine

每个单词叫做一个Term，不同的域中拆分出来的相同的单词是不同的term。term中包含两部分一部分是文档的域名，另一部分是单词的内容。

例如：文件名中包含apache和文件内容中包含的apache是不同的term。

#### 2.2.4创建索引

对所有文档分析得出的语汇单元进行索引，索引的目的是为了搜索，最终要实现只搜索被索引的语汇单元从而找到Document（文档）。

创建索引是对语汇单元索引，通过词语找文档，这种索引的结构叫**倒排索引结构**。

传统方法是根据文件找到该文件的内容，在文件内容中匹配搜索关键字，这种方法是顺序扫描方法，数据量大、搜索慢。

**倒排索引结构**是根据内容（词语）找文档

![img](https://images2015.cnblogs.com/blog/855959/201707/855959-20170706154505378-610589524.png)

**倒排索引结构也叫反向索引结构，包括索引和文档两部分，索引即词汇表，它的规模较小，而文档集合较大。**

### 2.3查询索引

查询索引也是搜索的过程。搜索就是用户输入关键字，从索引中进行搜索的过程。根据关键字搜索索引，根据索引找到对应的文档，从而找到要搜索的内容。

#### 2.3.1用户查询接口

全文检索系统提供用户搜索的界面供用户提交搜索的关键字，搜索完成展示搜索结果。

Lucene不提供制作用户搜索界面的功能，需要根据自己的需求开发搜索界面。

#### 2.3.2 创建查询

用户输入查询关键字执行搜索之前需要先构建一个查询对象，查询对象中可以指定查询要搜索的Field文档域、查询关键字等，查询对象会生成具体的查询语法，

例如：

语法 “fileName:lucene”表示要搜索Field域的内容为“lucene”的文档

#### 2.3.3 执行查询

搜索索引过程：

根据查询语法在倒排索引词典表中分别找出对应搜索词的索引，从而找到索引所链接的文档链表。

比如搜索语法为“fileName:lucene”表示搜索出fileName域中包含Lucene的文档。

搜索过程就是在索引上查找域为fileName，并且关键字为Lucene的term，并根据term找到文档id列表。

#### 2.3.4 渲染结果

以一个友好的界面将查询结果展示给用户，用户根据搜索结果找自己想要的信息，为了帮助用户很快找到自己的结果，提供了很多展示的效果，比如搜索结果中将关键字高亮显示，百度提供的快照等。

## 三. 配置开发环境（以7.7.2为例）

### 3.1下载

官网下载解压后：

analysis文件夹为分析器包

core文件夹为lucene核心包

queryparser文件夹为查询分析器

版本：lucene-7.7.2	Jdk要求：1.8以上

### 3.2使用的jar包

lucene-core-7.7.2.jar

lucene-analyzers-common-7.7.2.jar

## 四. 入门程序

### 4.1 需求

实现一个文件的搜索功能，通过关键字搜索文件，凡是文件名或文件内容包括关键字的文件都需要找出来。还可以根据中文词语进行查询，并且需要支持多个条件查询。

### 4.2 创建索引

代码实现：

~~~java
public void createIndex() throws Exception{
        //把索引库保存到内存中
        //Directory directory=new RAMDirectory();
        //把索引库保存到磁盘
        Directory directory= FSDirectory.open(new File("E:\\temp\\index").toPath());
        //基于Directory对象创建一个IndexWriter对象
        IndexWriter indexWriter=new IndexWriter(directory,new IndexWriterConfig());
        //读取磁盘中的文件，对应每个文件创建一个文档对象
        File dir=new File("E:\\temp\\searchsource");
        File [] files= dir.listFiles();
        for (File f:
                files) {
            //取文件名
            String filename=f.getName();
            //文件的路径
            String filepath=f.getPath();
            //文件的内容
            String fileContent = FileUtils.readFileToString(f, "utf-8");
            //文件的大小
            long filesize = FileUtils.sizeOf(f);
            //创建Field 参数1：域的名称 参数2：域的值 参数3：是否存储到磁盘
            Field fieldName=new TextField("name",filename,Field.Store.YES);
            Field fieldPath=new TextField("path",filepath,Field.Store.YES);
            Field fieldContent=new TextField( "content",fileContent,Field.Store.YES);
            Field fieldSize=new TextField("size",filesize+"",Field.Store.YES);
            //创建文档对象
            Document document=new Document();
            document.add(fieldName);
            document.add(fieldPath);
            document.add(fieldContent);
            document.add(fieldSize);
            //文档对象写入索引库
            indexWriter.addDocument(document);
        }
        //关闭对象
        indexWriter.close();
    }
~~~

### 4.3 查询索引

代码实现：

~~~java
 public void searchIndex()throws Exception{
        //创建一个Directory对象
        Directory directory= FSDirectory.open(new File("E:\\temp\\index").toPath());
        //创建一个IndexReader对象
        IndexReader indexReader=DirectoryReader.open(directory);
        //创建一个IndexSearcher对象，构造方法中的参数 indexreader对象
        IndexSearcher indexSearcher=new IndexSearcher(indexReader);
        //创建一个Query对象 ，TermQury
        Query query=new TermQuery(new Term("content","spring"));
        //执行查询 ，得到一个TopDocs对象
        TopDocs topDocs=indexSearcher.search(query,10);
        //取结果的总记录数
        System.out.println("总记录数"+topDocs.totalHits);
        //取文档列表
        ScoreDoc [] scoreDocs=topDocs.scoreDocs;
        //打印文档中的内容
        for (ScoreDoc doc:scoreDocs
             ) {
            int docid=doc.doc;
            Document document=indexSearcher.doc(docid);
            System.out.println(document.get("name"));
            System.out.println(document.get("path"));
            System.out.println(document.get("size"));
            System.out.println(document.get("content"));
            System.out.println("-------------------------");
        }
        indexReader.close();
    }
~~~

## 五. 分析器

默认使用标准分析器StandardAnalyzer

new IndexWriterConfig()的源码如下

~~~
public IndexWriterConfig() {
        this(new StandardAnalyzer());
    }
~~~

### 5.1分析器的分析效果

使用Analyzer对象的tokenStream方法返回一个TokenStream对象，该对象中包含了最终分词效果

~~~java
public void testTokenStream() throws Exception {
        //创建一个Analyzer对象 ，StandardAnalyzer对象
        Analyzer analyzer=new StandardAnalyzer();
        //使用分析器对象的tokenStream方法获得一个TokenStream对象
        TokenStream tokenStream= analyzer.tokenStream("","The Spring Freamwork provides a comprehensive programming and configration model .");
        //向TokenStream对象中设置一个引用 ，相当于一个指针
        CharTermAttribute charTermAttribute = tokenStream.addAttribute(CharTermAttribute.class);
        //调用TokenStream对象的resst方法，如果不调用会抛异常
        tokenStream.reset();
        //遍历TokenStream对象
        while (tokenStream.incrementToken()){
            System.out.println(charTermAttribute.toString());
        }
        //关闭TokenStream对象
        tokenStream.close();

    }
~~~

### 5.2 中文分析器（IKAnalyzer）

标准分析器不能分析中文  	这里使用IKAnalyzer中文分析器

使用方法：

1.把IKAnalyzer的jar包添加到工程中

2.把配置文件和扩展词典添加到工程的classpath下

注意：扩展词典严禁使用windows记事本，保证拓展词典的编码格式是utf-8 因为windows默认utf-8是utf-8+BOM

扩展词典：添加一些新词 

停用词词典：无意义或敏感词

```java
Analyzer analyzer=new IKAnalyzer();//使用仅需要把标准分析器改为IKAnalyzer即可
```

在创建索引时使用，源码如下：

```java
public IndexWriterConfig(Analyzer analyzer) {    
    super(analyzer);    
    this.writer = new SetOnce();
}
```

正常代码中使用

~~~java
 IndexWriterConfig indexWriterConfig=new IndexWriterConfig(new IKAnalyzer());
 IndexWriter indexWriter=new IndexWriter(directory,indexWriterConfig);
~~~

## 六. 索引库的维护

### 6.1 索引库的添加

#### 6.1.1 Field域

**是否分析**：是否对域的内容进行分词处理。前提是我们要对鱼的内容进行查询。

**是否索引：**将Field分析后的词或整个Field值进行索引，只有索引方可搜索到。例如：商品名称、商品简洁分析后进行索引，订单号、身份证号不用分析但也要索引，这些都可作为查询条件。

**是否存储：**将Field值存储在文档中，存储在文档中的Field才可以从Document中获取。比如：商品名称、订单号凡是将来要从Document中获取的Field都要存储。

**是否存储的标准：是否要将内容展示给用户**

| Field类                                                      | 数据类型               | Analyzed是否分析 | Indexed是否索引 | Stored是否存储 | 说明                                                         |
| ------------------------------------------------------------ | ---------------------- | ---------------- | --------------- | -------------- | ------------------------------------------------------------ |
| StringField(FieldName,FieldValue,Store.YES)                  | 字符串                 | N                | Y               | Y/N            | 此Field用于构建一个字符串Field，但是不会进行分析，会将整个串存储在索引中，比如（姓名，订单号）。 |
| LongPoint(String name,long poing)                            | Long型                 | Y                | Y               | N              | 可以用LongPoint、intPoint等类型存储数值类型的数据。让数值类型可以进行索引。但不能存储数据，如果想存储用StoredField |
| StoredField(FieldName,FieldValue)                            | 重载方法，支持多种类型 | N                | N               | Y              | 用来构建不同类型的Field，不分析，不索引，但要Field存储在文档中 |
| TextField(FieldName,FieldValue,Store.NO)或TextField(FieldName,reader) | 字符串或流             | Y                | Y               | Y/N            | 如果是一个Reader,lucene猜测内容比较多,会采用Unstored的策略   |

#### 6.1.2 添加文档--代码实现

~~~java
public void addDocument()throws Exception{
        //创建一个IndexWriter对象
        IndexWriterConfig cinfig=new IndexWriterConfig(new IKAnalyzer());
        IndexWriter indexWriter=
                new IndexWriter(FSDirectory.open(new File("E:\\temp\\index").toPath()),cinfig);
        //创建一个Document对象
        Document document=new Document();
        document.add(new TextField("name","新加的文件", Field.Store.YES));
        document.add(new TextField("content","新加的内容", Field.Store.NO ));
        document.add(new StoredField("path","e:\\temp\\hello"));
        indexWriter.addDocument(document);
        indexWriter.close();
    }
~~~

### 6.2 索引库的删除

~~~java
 //删除全部文档
        indexWriter.deleteAll();
        indexWriter.close();
//删除查询的文档用Term
        indexWriter.deleteDocuments(new Term("name","apache"));
        indexWriter.close();
//或者查询的文档用Query
//创建一个查询条件	
Query query = new TermQuery(new Term("filename", "apache"));
indexWriter.deleteDocuments(query);
~~~

### 6.3 索引库的修改

~~~java
//创建文档对象
        Document document=new Document();
        document.add(new TextField("name","更新后的文档", Field.Store.YES));
        document.add(new TextField("name1","更新后的文档2", Field.Store.YES));
        document.add(new TextField("name2","更新后的文档3", Field.Store.YES));
        indexWriter.updateDocument(new Term("name","spring"),document);
        indexWriter.close();
~~~

## 七.Lucene索引库查询

### 7.1 TermQuery

上面已说过，不再赘述

### 7.2 数值范围查询

~~~java
 /**
     * 此方法能查询出来主要是之前使用LongPoint进行存储
     */
    public void RangeQuery()throws Exception{
        Query query = LongPoint.newRangeQuery("size", 0l, 10000l);
        TopDocs topDocs = indexSearcher.search(query, 10);
        System.out.println("总记录数"+topDocs.totalHits);
        ScoreDoc[] scoreDocs = topDocs.scoreDocs;
        for (ScoreDoc docs:scoreDocs
             ) {
            int docid=docs.doc;
            Document document=indexSearcher.doc(docid);
            System.out.println(document.get("name"));
            System.out.println(document.get("path"));
            System.out.println(document.get("size"));
            System.out.println(document.get("content"));
            System.out.println("-------------------------");
        }
        indexReader.close();
    }
~~~

### 7.3 使用queryparser查询

可以对要查询的内容进行分词，基于分词结果进行查询

需要添加一个jar包：lucene-queryparser-7.4.0.jar

~~~java
 //创建一个queryparser对象，两个参数
        QueryParser queryParser=new QueryParser("name",new IKAnalyzer());
        //参数一：默认搜索与；参数二：分析其对象
        //使用queryparser对象创建一个query对象
        Query query = queryParser.parse("lucene是一个Java开发的全文检索工具");
        //执行查询
        TopDocs topDocs = indexSearcher.search(query, 10);
        System.out.println("总记录数"+topDocs.totalHits);
        ScoreDoc[] scoreDocs = topDocs.scoreDocs;
        for (ScoreDoc docs:scoreDocs
                ) {
            int docid=docs.doc;
            Document document=indexSearcher.doc(docid);
            System.out.println(document.get("name"));
            System.out.println(document.get("path"));
            System.out.println(document.get("size"));
            System.out.println(document.get("content"));
            System.out.println("-------------------------");
        }
        indexReader.close();
~~~

