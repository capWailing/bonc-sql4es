### bonc-sql4es: Elasticsearch的jdbc驱动程序

Sql-for-Elasticsearch (sql4es)   是一个用于**Elasticsearch 2.0 - 2.4**版本的能实现大多数JDBC接口的jdbc 4.1驱动程序: Connection, Statement, PreparedStatment, ResultSet, Batch 和 DataBase- /  ResultSetMetadata。下面的截图显示了可以使用驱动程序执行的SQL语句的SQLWorkbenchJ。

![SQLWorkbenchJ screenshot with examples](release/workbench_examples.png)

#### 用法

可以通过在项目的发行版目录中找到的jar文件添加到所使用的工具/应用程序中来加载sql4es驱动程序，并使用名称 '***nl.anchormen.sql4es.jdbc.ESDriver***'加载驱动程序。驱动程序需要具有以下格式的URL： ***jdbc:sql4es://host:port/index?params***. 

- host: 其中一个主机的主机名或ip（必需）
- port: 可选用于传输客户端的端口号（默认为9300）
- index: 在驱动程序中设置活动的可选索引。大多数语句（如SELECT，DELETE和INSERT）都需要一个活动索引（参见下面的USE [index / alias]语句）。然而，可以创建没有活动索引的新索引，类型和别名。
- params: 用于影响驱动程序内部的可选参数集（指定其他主机，在单个请求中获取的最大文档数量等）。如果您的群集名称不是“elasticsearch”，则应该在URL中指定群集名称（请参见下面的示例）。有关所有驱动程序特定参数的说明，请参阅本自述文件的“配置”部分。

``` java
// 注册驱动程序并获取索引'myidx' 
Class.forName("nl.anchormen.sql4es.jdbc.ESDriver");
Connection con = DriverManager.getConnection("jdbc:sql4es://localhost:9300/myidx?cluster.name=your-cluster-name");
Statement st = con.createStatement();
// 对myidx中的mytype执行查询
ResultSet rs = st.executeQuery("SELECT * FROM mytype WHERE something >= 42");
ResultSetMetaData rsmd = rs.getMetaData();
int nrCols = rsmd.getColumnCount();
// 得到像其他列信息，比如type
while(rs.next()){
	for(int i=1; i<=nrCols; i++){
  		System.out.println(rs.getObject(i));
	}
}
rs.close();
con.close();
```

### 支持的 SQL

简单来说，sql4es驱动程序将SQL语句转换为它们的Elasticsearch语意，并将结果解析成ResultSet实现。支持以下sql语句：

- SELECT: 从elasticsearch获取文档（有无评分均可）或聚合
  * COUNT (DISTINCT ...), MIN, MAX, SUM, AVG
  * DISTINCT
  * WHERE (=, >, >=, <, <=, <>, IN, LIKE, AND, OR, IS NULL, IS NOT NULL, NOT [condition])
  * GROUP BY
  * HAVING
  * ORDER BY
  * LIMIT （无偏移量，sql4es不支持偏移量）
- CREATE TABLE (AS) 创建一个索引/类型，并可选地将查询的结果索引到其中
- CREATE VIEW (AS): ：创建别名，可选地带有过滤器
- DROP TABLE/VIEW 删除索引或别名
- INSERT INTO (VALUES | SELECT): 将文档插入索引/类型; 提供查询的值或结果。可以通过指定现有文档_id来使用INSERT UPDATE文档
- UPDATE: 作为elasticsearch的Upsert执行
- DELETE FROM (WHERE):删除文档
- USE: 选择索引作为驱动程序的活动索引 (用于interpret queries)
- EXPLAIN SELECT: 返回为SELECT语句执行的Elasticsearch查询
- Table aliases like SELECT … FROM table1 as T1, table2 t2...
  - 表别名被解析，但在查询执行期间未使用

**备注**

Elasticsearch不支持事务处理。因此，执行批次不能在失败时回滚（也不能提交语句）。文档完整索引也需要一些时间，因此直接执行一个INSERT后跟一个SELECT可能不包括刚插入的文档。

一些尚未支持的 SQL语句或elasticsearch功能:


- 无法指定偏移（偏移偏移或极限偏移，数字）
- ~~不支持父子关系。目前无法索引或检索此类型的字段。~~
  - 新添加功能：SELECT * FROM my_parent WHERE _child.my_child='age:20'
- elasticsearch功能不支持建议和模板。

### 概念

由于elasticsearch是一个NO-SQL数据库，它不包含大多数人熟悉的确切关系对象（如数据库，表和记录）。但是elasticsearch确实有一个类似的对象层次（索引，类型和文档）。sql4es使用的概念映射如下：

- Database = Index 数据库=索引
- Table = Type 表=类型
- Record = document 记录=文档
- Column = Field 列=字段
- View = Alias 视图=别名 （这不符合层次的观点，但是符合视图/别名的目的）

Elasticsearch 的响应，搜索结果和聚合结果都被放入到ResultSet的实现中. 默认情况下，任何嵌套对象都将“分解”为侧视图；这意味着嵌套对象被视为连接的表，放在同一行中（请参阅[此页面](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+LateralView)查看解释） 可以将嵌套对象表示为嵌套ResultSet，请参阅“配置”部分。（URL中添加result.nested.lateral=false参数）请注意，虽然对象会被分解，但是具有基本类型的数组不会！它们被JDBC支持的java.sql.Array实现。

Sql4es需要从一个活动的索引/别名启动，这意味着它只解析了这个索引的类型。例如myIndex当前处于活动状态，SELECT * FROM sometype将只返回myindex的类型的所有结果。在索引中不存在的类型上执行SELECT将返回一个空的结果。可以通过执行USE [otherIndex]来更改活动索引，如下所述。

### 查询

本节介绍如何将SQL解释并转换为SE语句。presto解析器用于解析SQL语句，请参阅[presto 网站](https://prestodb.io/docs/current/sql.html)上的语法定义。

#### SELECT

``` sql
/* basic syntax */
SELECT [field (AS alias)] FROM [types] WHERE [condition] GROUP BY [fields] HAVING [condition] ORDER BY [field (ASC|DESC)] LIMIT [number]
```

- fields (AS alias): 定义要从elasticsearch检索并放在ResultSet中的字段。可以使用*来表示应该检索所有字段（包括_id和_type）。字段可以通过他们的名字来表示，嵌套字段可以使用点状表示法使用它们的层次结构名称来表示，如：nesteddoc.otherdoc.field。使用一个星号将简单地获取文档中存在的所有字段，包括嵌套字段。可以指定对象的根以获取其所有字段。像SELECT nesteddoc FROM type这样的查询将获取nesteddoc中存在的所有字段。因此，如果nesteddoc有数百个字段，它可能返回数百列。
- types: 执行查询的类型。这只能是当前处于活动状态的索引或别名中的类型（也可参见'use'语句）。
- condition: 标准SQL条件使用=，>，> =，<，<=，<>，IN和LIKE运算符。Sql4es不支持NOT运算符，而可以使用'<>'。使用AND和OR组合条件。
- limit: 仅适用于非聚合查询。不支持使用偏移量！（不支持分页）

``` sql
/* 以下结果会将nested对象解析成侧视图 */
SELECT * from mytype

SELECT _id as id, myInt, myString FROM mytype WHERE myInt >= 3 OR (myString IN ('hello','hi','bye') AND myInt <= 3)

/* 如果nestedDoc包含两个字段，结果会解析成[myInt,nestedDoc.field1, nestedDoc.field2] */
SELECT myInt, nestedDoc FROM mytype WHERE myInt > 3 AND myString <> 'bye'

/* 如果数组包括3个对象，ResultSet将包含3条记录，即使使用了LIMIT! */
SELECT array_of_nested_objects FROM mytype LIMIT 1
```

**表/类型**

只能在FROM子句中处理活动索引或别名的部分类型。如果必须在单个查询中访问来自不同索引的类型，则必须创建一个别名（有关创建别名的参照CREATE VIEW）。*查询缓存* 是唯一的例外。当在FROM中使用查询缓存标识符（默认'query_cache'）时，表示使用查询缓存。每当这个时候，从缓存中提取查询而不是执行查询，这样在查询时最小化查询时间。

``` sql
/* 从类型查询*/
SELECT DISTINCT field, count(1) FROM type, query_cache
/* 与上述完全相同，但现在也从缓存查询 */
SELECT DISTINCT field, count(1) FROM type
```

**文本匹配，搜索和评分**

默认情况下，查询将作为过滤器执行，这意味着elasticsearch不会对结果进行分数，并以任意顺序返回。将“_score”添加为所选列之一，以便更改此行为并请求评分。默认情况下，结果返回按_score DESC排序（可以更改为ORDER BY _score ASC）。在另一个字段上排序将禁用评分！另外，可以通过分别指定_id和_type来获取文档的id和类型。

Sql4es在文本字段的搜索和匹配之间没有区别。行为完全取决于对正在查询/搜索的文本字段执行的analysis（如果有）。在底层，使用几个简单的规则来确定应该使用什么类型的查询：

- 单个词放入TermQuery (example: myString = 'hello')
- 多个单词放入 MatchPhraseQuery (example: myString = 'hello there')
- 通配符的存在（％和_）将触发使用 WildcardQuery (% 代替* 、 _ 代替 ?)。(Examples: mystring = '%something' is the same as mystring LIKE '%something')
- IN（...）的使用将被放在一个 TermsQuery中

此外，可以执行ES支持的所有功能的常规搜索。搜索是通过在虚构字段“_search”上执行匹配来完成的（见下面的例子）。可以使用高亮功能为任何文本字段请求高亮，如：SELECT highlight(field), ...可通过全局配置设置片段大小和数字。

``` sql
/* term query */
SELECT _score, myString FROM mytype WHERE myString = 'hello' OR myString = 'there'
/* 同上 */
SELECT _score, myString FROM mytype WHERE myString IN ('hello', 'there')
/* use of NOT; find all documents which do not contain 'hello' or 'there' */
SELECT _score, myString FROM mytype WHERE NOT myString IN ('hello', 'there')

/* 检查NULL值（缺少字段） */
SELECT myInt FROM mytype WHERE myString NOT NULL
SELECT myInt FROM mytype WHERE myString IS NULL

/* 短语查询 */
SELECT _score, highlight(myString), myString FROM mytype WHERE myString = 'hello there'
/* 通配符查询 */
SELECT _score, myString FROM mytype WHERE myString = 'hel%'
/* a search for exactly the same as the first two */
SELECT _score, highlight(mystirng) FROM mytype WHERE _search = 'myString:(hello OR there)'
```

**通过_id获取文档**

可以通过在_id字段上指定'='或IN谓词来执行文档ID的搜索。可以将_id与其他字段的匹配组合，但是应该始终使用IN完成匹配多个_id。

``` sql
SELECT * FROM mytype WHERE _id = 'whatever_id'
SELECT * FROM mytype WHERE _id = 'whatever_id' AND myInt > 3
SELECT * FROM mytype WHERE _id = 'whatever_id' OR _id = 'another_ID' /* 错误 */
SELECT * FROM mytype WHERE _id IN ('whatever_id', 'another_ID') /* 正确 */
```

**通过_search查询**

在_search字段上通过'='执行对文档的queryString查询，通常格式为：_search='type:value AND age:[1 TO 10} OR content:value'。ES官方网站有更多关于queryString的语法说明。

**通过_child、_parent执行父子关系查询**

我们支持了新的父子关系查询，在有父子关系的两个type之间执行hasChild和hasParent查询。在_child、_parent字段上指定子表或者父表的表名：_child.<childType>、_parent.<parentType>。'='之后的值将会返回queryStringQuery的查询结果，queryString查询语法参照_search。

``` sql
//在父表my_parent中查询子表my_child满足age为0到50的条件的记录
SELECT * FROM my_parent WHERE _child.my_child = 'age:{0 TO 50}';

//在子表my_child中查询父表my_parent满足下列条件的记录
SELECT * FROM my_child WHERE _parent.my_parent = 'number: 10';
```

关于父子关系查询的注释:

- 父子关系必须在创建index时在mapping里定义，也就是说子表必须有_parent字段。详见ES官方网站父子关系说明。
- 必须声明正确的父子表关系。_child之后的值必须为所查询的表的子表，_parent之后的值必须为所查询表的父表。

**聚合**

Sql4es在检测到DISTINCT，GROUP BY 或者聚合函数（MIN，MAX，SUM，AVG或者COUNT）而且没有普通字段时，会返回一个聚合请求。聚合请求不会返回任何搜索结果

Sql4es支持一些基本的算术函数：*，/，+，- 和％（modulo）。也可以在诸如AVG(field)/100和SUM(field)/count(1)之类的计算中组合来自结果集的不同字段。请注意，在当前实现中，一旦从Elasticsearch获取数据，这些计算就在驱动程序内执行。可以使用括号 *[offset]*来引用函数中其他行中的值。例如，SUM(volume)/ SUM(volume)[-1]将将行X的列的和与行X-1中的值进行比较。如果无法计算值，例如上例中的行号0，将获得值Float.NaN。

``` sql
/* Aggregates on a boolean and returns the sum of an int field in desc order */
SELECT myBool, sum(myInt) as summy FROM mytype GROUP BY myBool ORDER BY summy DESC

/* This is the same as above */
SELECT DISTINCT myBool, sum(myInt) as summy ROM mytype ORDER BY summy DESC

/* Aggregates on a boolean and returns the sum of an int field only if it is larger than 1000 */
SELECT myBool, sum(myInt) as summy ROM mytype GROUP BY myBool HAVING sum(myInt) > 1000

/* Gets the average of myInt in two different ways... */
SELECT myBool, sum(myInt)/count(1) as average, avg(myInt) FROM mytype GROUP BY myBool

/* Calculates the percentage of growth of the myInt value acros increasing dates */
SELECT myDate, sum(myInt)/sum(myInt)[-1]*100 FROM mytype GROUP BY myDate ORDER BY myDate ASC

/* aggregation on all documents without a DISTINCT or GROUP BY */
SELECT count(*), SUM(myInt) from mytype

/* the following will NOT WORK, a DISTINCT or GROUP BY on mytext is required */
SELECT mytext, count(*), SUM(myInt) from mytype
```

关于SELECT的一些注释:

- limit 仅适用于非聚合查询。聚合上的任何'limit'将被省略
- 字段的计算仅在当前在驱动程序中执行
- having (对聚合结果进行过滤) 仅在当前在驱动程序中执行
- 聚合结果的排序仅在当前在驱动程序中执行

#### Explain

Explain可以通过执行以下操作来查看为SELECT语句执行的ES查询:

***EXPLAIN [SELECT statement]***

#### 使用

Sql4es使用活动索引/别名。默认情况下，这是用于获取连接的URL所指定的索引/别名（如果有）。可以通过执行以下操作来更改活动索引/别名：

***USE [index / alias]***

这个动作只会影响驱动器，对elasticsearch没有影响

#### 创建和删除

Sql4es支持创建索引，类型（创建表）和别名（创建视图）。这些语句需要了解ES工作原理，如映射，类型定义和别名。 

***CREATE TABLE (index.)type ([field] "[field definition]" (, [field2])...) WITH (property="value" (, property2=...) )***

这将在当前活动的索引或使用点符号指定的索引中为[type]创建映射。每当使用点号时，默认第一个点之前的部分指的是索引。 如果指定的索引已经存在，则只将该类型添加到此索引。

字段定义是将json定义放在映射中而不引用json元素！字符串类型可以定义如下：*CREATE TABLE mytype (stringField "type:string, index:analyzed, analyzer:dutch")*。任何映射元素（如模板）都可以使用WITH子句设置（参见下面的示例）。所有这些json部分将被正确引用，并将它们混合成一个映射请求。

``` sql
/*为newindex中的mytype创建映射，并使用模板存储任何不经过分析的字符串 */
CREATE TABLE index.mytype (
	myInt "type:integer",
  	myDate "type:date, format:yyyy-MM-dd"
  	myString "type:string, index:analyzed, analyzer:dutch"
) WITH (
  dynamic_templates="[{
    default_mapping: { 
    	match: *,
    	match_mapping_type: string, 
    	mapping: {type: string, index: not_analyzed }
    }
  }]"
)
```

可以使用CREATE TABLE index.type（_id“type：string”）创建一个空索引。_id字段被省略，因为它是一个标准ES字段。

***CREATE TABLE (index.)type AS SELECT ...***

根据SELECT语句的结果创建新的索引/类型。新的字段名称取自SELECT，可以使用列别名来影响字段名称。例如CREATE TABLE myaverage AS SELECT avg（somefield）AS average将导致当前活动索引中的一个新类型myaverage，单个Double字段称为“average”。请注意，驱动器通过两步过程执行。首先执行查询，其次创建索引，并将结果（批量）写入新类型。

``` sql
/*创建基于之前创建的映射与类型映射另一个index*/
CREATE TABLE index.mytype AS SELECT myDate as date, myString as text FROM anyType

/* create a type with a (possibly expensive to calculate) aggregation result */
CREATE TABLE index.myagg AS SELECT myField, count(1) AS count, sum(myInt) AS sum from anyType GROUP BY myField ORDER BY count DESC
```

***CREATE VIEW [alias] AS SELECT * FROM index1 (, [index2])... (WHERE [condition])***

创建一个包含指定索引的新ES别名，或将索引添加到现有别名。可选的WHERE子句在指定的索引别名对象上添加一个过滤器。有关别名的信息，请参阅[elasticsearch文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-aliases.html) for information on aliases

***DROP TABEL [index] / DROP VIEW [alias]***

删除指定的索引或别名

``` sql
/*Create an elasticsearch alias which includes two indexes with their types */
CREATE VIEW newalias AS SELECT * FROM newindex, newindex2

/* Same as above but with a filter*/
CREATE VIEW newalias AS SELECT * FROM newindex, newindex2 WHERE myInt > 99

/*Use the alias so it can be queried*/
USE newalias

/* removes myindex and remove newalias */
DROP TABLE myindexindex
DROP VIEW newalias
```

#### 插入和删除

介绍通过sql插入和删除数据

***INSERT INTO (index.)type ([field1], [field2]...) VALUES ([value1], [value2], ...), ([value1], ...), …***

在索引中添加一个或多个文档到指定的类型。必须定义字段，值的数量必须与定义的字段数匹配。可以在单个INSERT语句中添加多个文档。可以在insert语句中指定_id字段。在这种情况下，它将强制ES来插入指定的文档ID。 如果_id已经存在，则插入作为一个UPDATE！插入嵌套对象是不可能的，因为它们不能在SQL语言中指定。

***INSERT INTO  (index.)type SELECT …***

将SELECT语句中的所有结果添加到索引中指定的类型。要从结果中取出要插入的字段名称（即可以使用列别名）。请注意，类似于“CREATE TABLE .. AS SELECT”，结果将被拉入驱动程序，然后索引（使用Bulk）。

``` sql
/* Insert two documents into the mytype mapping */
INSERT INTO mytype (myInt, myDouble, myString) VALUES (1, 1.0, 'hi there'), (2, 2.0, 'hello!')

/* insert a single document, using quotes around nested object fields */
INSERT INTO mytype (myInt, myDouble, "nestedObject.myString") VALUES (3, 3.0, 'bye, bye')

/* update or insert a document with specified _id */
INSERT INTO mytype (_id, myInt, myDouble) VALUES ('some_document_id', 4, 4.0)

/* copy records from anotherindex.mytype to myindex.mytype that meet a certain condition */
USE anotherindex
INSERT INTO myindex.mytype SELECT * from newtype WHERE myInt < 3 
```

***DELETE FROM type (WHERE [condition])***

从符合条件的指定类型中删除所有文档。如果未指定WHERE子句，则将删除所有文档。由于Elasticsearch只能根据它们的_id删除文档，这意味着这个语句是分两步执行的。首先从满足条件的文档中收集所有_id，其次使用批量API删除这些文档。

``` sql
/* delete documents that meet a certain condition*/
DELETE FROM mytype WHERE myInt == 3

/*delete all documents from mytype*/
DELETE FROM mytype 
```

### UPDATE

可以使用标准SQL语法更新索引/类型中的文档。请注意，嵌套对象名称必须用双引号括起来：

***UPDATE index.type SET field1=value, fiedl2='value', "doc.field"=value WHERE condition***

更新将分两步执行。首先获取与条件匹配的所有文档的_id，之后使用Upsert API批量更新这些文档的指定字段。

### 配置

可以通过提供的URL来设置参数。所有参数都暴露于Elasticsearch，这意味着可以设置客户端参数，参见 [elasticsearch docs](https://www.elastic.co/guide/en/elasticsearch/client/javascript-api/current/configuration.html). 可以设置以下驱动程序的具体参数：

- es.hosts: 带逗号分隔列表，带有可选端口的附加主机，格式为host1（：port1），host2（：port2）...）当未指定端口时，将使用默认端口9300。
- fetch.size (int default 10000): 在单个请求中获取的最大结果数（10000是elasticsearch的最大值）。可以降低文件以避免记录问题，当文档获取非常大时。
- results.split (default: false): 设置它将将整个结果分成多个ResultSet对象，每个对象具有最大的*fetch.size*记录数。可以使用Statement.getMoreResults（）获取下一个ResultSet。默认值为*false* ，驱动程序将把所有结果放在一个ResultSet中。当客户端没有足够的内存将单个ResultSet中的所有结果保存下来时，应使用此设置。
- scroll.timeout.sec (int, default 10): 滚动ID保持有效的时间，可以调用“getMoreResults（）”。应根据实际情况增减。
- query.timeout.ms (int, default 10000): 查询上设置的超时值。可以根据用例进行更改。
- default.row.length (int, default 250): 为结果创建的列的初始数。只有在搜索结果被解析时触发的结果不适合（通常由数组索引超出范围异常）时才增加此属性。
- query.cache.table (string, default 'query_cache'): 用于指示使用elasticsearch查询缓存的虚构表名。可以更改，使其更短或更方便。
- result.nested.lateral (boolean, default true): 指定嵌套结果必须分解（默认）与否。从您自己的代码处理驱动程序时可以设置为false。在这种情况下，包含嵌套对象（包装在ResultSet中）的列将具有java.sql.Types = Types.JAVA_OBJECT，并可用作（ResultSet）rs.getObject（colNr）。
- fragment.size (int, default 100): 以字符为单位指定首选片段长度。
- fragment.count (int, default 1): 指定请求突出显示时返回的最大片段数。
- precision.threshold (int, default 3000): 指定用于[基数聚合的精度](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-metrics-cardinality-aggregation.html)

