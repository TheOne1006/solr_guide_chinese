## 使用数据导入处理程序, 上传结构化数据存储数据

许多搜索应用程序要将索引的内容存储在结构化数据存储中(如 MySQL). 数据导入处理程序 (DIH: Data Import Handler) 提供了从数据存储导入内容并对其编制索引的机制.
除了关系型数据库, DIH 还可以基于 HTTP 数据源索引内容, 其中使用XPath处理器来生成字段.

在 `example/example-DIH` 目录内容包含几个集合数据导入处理程序的许多功能.
使用案例:

```bash
bin/solr -e dih
```

更多关于 数据上传程序, 详情查看<https://wiki.apache.org/solr/DataImportHandler>

### 概念和术语

导入数据程序相关几个术语:

1. Datasource 数据源
  - 定义数据的位置, 对于数据库它是 DSN,对于 HTTP 它是一个 URL
2. Entity 实体
  - 从概念上来讲, 处理实体以生成包含多个字段的文档集合,其（在可选地以各种方式变换之后）被发送到Solr用于索引.
  - 对于RDBMS数据源，实体是视图或表
  - 其将由一个或多个SQL语句处理以生成具有一个或多个列（字段）的一组行（文档）
3. Processor 处理器
  - entity 处理器将会从数据源提取数据, 并将其转化添加到索引中.
  - 可以编写自定义实体处理以扩展或替换提供实体处理器.
4. Transformer 转化器
  - 每个字段集由实体,可以随意的进行转化. 此过程可以修改字段,创建新字段, 或者从一个单行中创建 多行文档.
  - DIH 中有几个内置的 Transformer, 例如 执行修改日期或者剥离 HTML 功能
  - 可以编写 转化器 通过公共可用的接口

### Configuration 构造

配置 solrconfig.xml

导入数据必须要注册 solrconfig.xml. 例如:

```xml
<requestHandler name="/dataimport" class="org.apache.solr.handler.dataimport.DataImportHandler">
  <lst name="defaults">
    <str name="config">/path/to/my/DIHconfigfile.xml</str>
  </lst>
</requestHandler>
```

唯一必需的参数是config参数,
它指定 DIH 配置文件未知,
该文​​件包含数据源的规范, 如何获取数据, 要提取的数据以及如何处理以生成要发布到的Solr文档指数.

你可以有多个 DIH 配置文件, 每个文件将需要在solrconfig.xml文件中单独定义, 指定文件的路径.

### 配置 DIH 配置文件

基于示例服务器中的“ db”集合的注释配置文件dih如下所示（example/example-DIH/solr/db/conf/db-data-config.xml）.
它使用此模式从定义简单产品数据库的四个表中提取字段。有关此处显示的参数和选项的更多信息，请参见以下部分。

```xml
<dataConfig>
<!-- The first element is the dataSource, in this case an HSQLDB database.
     The path to the JDBC driver and the JDBC URL and login credentials are all specified here.
     Other permissible attributes include whether or not to autocommit to Solr, the batchsize
     used in the JDBC connection, a 'readOnly' flag.
     The password attribute is optional if there is no password set for the DB.
-->
  <dataSource driver="org.hsqldb.jdbcDriver" url="jdbc:hsqldb:./example-DIH/hsqldb/ex" user="sa" password="secret"/>
<!--
Alternately the password can be encrypted as follows. This is the value obtained as a result of the command
openssl enc -aes-128-cbc -a -salt -in pwd.txt
password="U2FsdGVkX18QMjY0yfCqlfBMvAB4d3XkwY96L7gfO2o="
WHen the password is encrypted, you must provide an extra attribute
encryptKeyFile="/location/of/encryptionkey"
This file should a text file with a single line containing the encrypt/decrypt password

-->
<!-- A 'document' element follows, containing multiple 'entity' elements.
     Note that 'entity' elements can be nested, and this allows the entity
     relationships in the sample database to be mirrored here, so that we can
     generate a denormalized Solr record which may include multiple features
     for one item, for instance -->
  <document>

<!-- The possible attributes for the entity element are described below.
     Entity elements may contain one or more 'field' elements, which map
     the data source field names to Solr fields, and optionally specify
     per-field transformations -->
<!-- this entity is the 'root' entity. -->
    <entity name="item" query="select * from item"
            deltaQuery="select id from item where last_modified > '${dataimporter.last_index_time}'">
      <field column="NAME" name="name" />

<!-- This entity is nested and reflects the one-to-many relationship between an item and its multiple features.
     Note the use of variables; ${item.ID} is the value of the column 'ID' for the current item
     ('item' referring to the entity name)  -->
      <entity name="feature"
              query="select DESCRIPTION from FEATURE where ITEM_ID='${item.ID}'"
              deltaQuery="select ITEM_ID from FEATURE where last_modified > '${dataimporter.last_index_time}'"
              parentDeltaQuery="select ID from item where ID=${feature.ITEM_ID}">
        <field name="features" column="DESCRIPTION" />
      </entity>
      <entity name="item_category"
              query="select CATEGORY_ID from item_category where ITEM_ID='${item.ID}'"
              deltaQuery="select ITEM_ID, CATEGORY_ID from item_category where last_modified > '${dataimporter.last_index_time}'"
              parentDeltaQuery="select ID from item where ID=${item_category.ITEM_ID}">
        <entity name="category"
                query="select DESCRIPTION from category where ID = '${item_category.CATEGORY_ID}'"
                deltaQuery="select ID from category where last_modified > '${dataimporter.last_index_time}'"
                parentDeltaQuery="select ITEM_ID, CATEGORY_ID from item_category where CATEGORY_ID=${category.ID}">
          <field column="description" name="cat" />
        </entity>
      </entity>
    </entity>
  </document>
</dataConfig>
```

##### 请求参数

请求参数可以在配置中用占位符替换 `${dataimporter.request.paramname}`

```xml
<dataSource driver="org.hsqldb.jdbcDriver" url="${dataimporter.request.jdbcurl}" user="${dataimporter.request.jdbcuser}" password=${dataimporter.request.jdbcpassword} />
```

然后, 这些参数可以穿导入命令, 或者在 `<defaults>` 部分定义 solrconfig.xml. 例如:

```bash
dataimport?command=full-import&jdbcurl=jdbc:hsqldb:./example-DIH/hsqldb/ex&jdbcuser=sa&jdbcpassword=secret
```

### 数据导入命令

DIH 命令是通过 http 请求到 Solr. 它支持以下操作

1. abort 取消
  - 终止正在进行的操作,
  - 网址: `http://<host>:<port>/solr/<collection_name>`/dataimport?command=abort
2. delta-import 增量导入
  - 用于增量导入和变化检测
  - `http://<host>:<port>/ solr/ <collection_name>/ dataimport?command=delta-import`
3. full-import 完全导入
  - 使用表单的 url 启动完全导入操作.
  - 命令立即返回, 操作将在新县城中启动,并响应中的状态属性显示为忙.操作可能需要一些时间,具体取决于数据集的大小.
  - 在完全导入件不会阻止对Solr的查询.
  - 当执行完命令时, 它将操作的时间存储在位于 conf/dataimport.properties.
4. reload-config 更新配置
  - 如果配置已更新并希望重新加载而不重启 Solr, 运行该命令
  - `http://<host>:<port>/solr/<collection_name>/command=reload-config`
5. status 状态
  - 它返回有关创建，删除，查询运行，读取行数，状态等文档数量的统计信息
  - `http://<host>:<port>/ solr/ <collection_name>/ dataimport?command=status`
6. show-config 响应配置


##### 这些参数的 full-import 命令

该full-import命令接受以下参数:

1. clean 清除
  - 默认为 true, 指示开始编制索引前是否清除索引
2. commit 承诺
  - 默认为 true, 告诉操作后是否提交
3. debug 调试
  - 默认值 false, 在调试模式下运行命令. 它由交互式开发模式使用.
  - 注意调试模式下,文档不会自动提交. 需要 commit=true
4. entity 实体
  - 直接位于`<document>`配置文件中标记下的实体的名称.使用此选项可以选择性的执行一个或多个实体.
5. optimize 优化
  - 默认为 true, 告诉 Solr 是否优化的操作
6. synchronous
  - 阻塞请求, 直到导入完成. 默认为 false.


### 属性作者

该 propertyWriter 元素定义与增量查询使用的日期格式和语言环境.
它是一个可选配置项.
将元素添加到 DIH 配置文件中, 直到 dataConfig 元素下面.

```xml
<propertyWriter dateFormat="yyyy-MM-dd HH:mm:ss" type="SimplePropertiesWriter" directory="data" filename="my_dih.properties" locale="en-US" />
```

可用参数:

1. dateFormat 日期格式
  - 将日期转换为文本时使用 java.text.SimpleDateFormat
  - 默认值为 "yyyy-MM-dd HH:mm:ss"
2. type 类型
  - 实现类
  - 使用 SimplePropertiesWriter 非SolrCloud安装。如果使用SolrCloud，请使用ZKPropertiesWriter。如果没有指定，它将默认为适当的类，取决于是否启用SolrCloud模式.
3. directory 目录
  - SimplePropertiesWriter仅与使用。
  - 属性文件的目录。如果未指定，默认值为`conf`。
4. filename 文件名
  - SimplePropertiesWriter仅与使用）。属性文件的名称。如果未指定，那么缺省值为requestHandler名称（solrconfig.xml由'.properties' 附加的定义）（即'dataimport.properties'）。
5. locale 语言环境
  - 区域设置
  - 如果未定义，则使用ROOT语言环境。它必须指定为语言国家（BCP 47语言标记）
  - 例如，en-US。

### 数据源

数据源指定数据的来源以及类型,在相关联的实体处理器内配置一些数据源.
也可以在 solrconfig.xml 中指定数据源, 这在你多个环境(如开发, QA 和生产) 不同的数据源时很有用.

可用的数据源类型如下所述.

> ContentStreamDataSource

这将 POST 数据作为数据源.

> FieldReaderDataSource

这可以在数据库字段包含要使用 XPathEntityProcessor 处理 xml 时使用.

> FileDataSource

> JdbcDataSource

> URLDataSource


### 实体处理器

实体处理器提取数据,对其进行转换,并添加到 Solr 索引,
实体的示例包括数据存储中的视图或表。

每个处理器都有自己的一组属性. 在下面的描述,另外,
there are non-specific attributes common to all entities which may be specified.

1. cachelmpl ?
  - 可选的. 一个类(必须实现 DIHCache) 用于从包装它的实体执行查找时缓存此实体, 提供的实现是 'SortedMapBackedCache'
2. cacheKey ?
  - 指定要用作告诉缓存密钥的次实体属性名 cacheImplif.
3. cacheLoopup ?
4. child='true'
  - 启动索引文档块 (也称为 嵌套子文档),以便使用 `Block Join Query Parsers` .
  - 它只能 根`<entity>` 下指定.它从默认的行为(合并字段值) 切换到嵌套文档作为 子文档.
  - join='zipper' 启用合并连接 aka 'zipper' 算法
5. onError
  - 允许的值为( abort| skip | continue ). 默认为 abort.
  - 出错时, 取消, 跳过, 忽略
6. pk
  - 实体的主键, 可选, 仅当使用增量导入时才需要.
  - 与 uniqueKey 无关, schema.xml 但是他们可以相同.
  - 如果执行增量导入,然后引用 `${dataimporter.delta.<column-name>}` 那么这是必需的
7. postImportDeleteQuery
  - 类似于上面, 但是在导入完成后执行
8. preImportDeleteQuery
  - 在全导入命令之前, 使用此查询来清除索引, 而不是使用 `*:*`.
9. rootEntity
  - 默认情况下, 紧接在其下的实体`<document>` 是根实体, 如果此属性设置为 false,
10. processor
  - 默认 `SqlEntityProcessor`. 仅当数据源不是 RDBMS 才需要
11. transformer
  - 可选, 要应用于此实体的一个或多个变压器
12. where
  - ?
13. name
  - 必填, 标识实体的唯一名称
14. dataSource
  - 数据源名称.如果定义多个数据源,
  - 请将此属性与此实体的数据源的名称一起使用.

提供在 DIH 的实体中的缓存以避免一个实体一次又一次的重复查找.
默认 SortedMapBackedCache是一个HashMap键是行中的一个字段,
值是同一键的一堆行.

在下面实例中, 每个 manufacturer 实体都使用 `id` 属性作为缓存件.

### SQL 实体处理器

SqlEntityProcessor是默认处理器。关联的数据源应为JDBC URL。

1. query
  - 必填, sql 查询
2. deltaQuery
  - sql 查询 如果使用的操作是 'delta-import
  - 此查询选择将作为增量更新的一部分的行的主键.
3. parentDeltaQuery






- - -
