## Solr索引简介

本节介绍了创建索引的过程: 添加内容到 Solr 的索引,必要时需要删除相关内容. 通过向一个索引添加内容, 然后我们可以通过 solr 进行搜索.

Solr 的索引可以接收来自不同的数据源, 包含 XML, CSV, 从数据库中提取的数据, 以及常见的文件格式,例如 Microsoft Word 或者 PDF.

下面有三种方式将数据添加到 Solr 的索引中:

* 使用 Solr Cell 框架(基于 Apache Tika构建), 用于接收二进制文件或结构化文件, 例如 Office, Word, PDF 和其他专有格式.
* 通过生成此类请求的任何环境相 Solr 服务器发送 HTTP 请求来上传 XML 文件.
* 通过 Solr 的 java 客户端 API 像 solr 写入数据.
