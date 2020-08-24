# 体系结构

![20150730152113099](/Users/jimmylsc/Desktop/20150730152113099.jpeg)

连接层：Connection Pool接收客户端请求开启线程提供服务

服务层：Management Services and Utilities, SQL Interface, Parser, Optimizer, Cashes and Buffers

存储引擎层：Pluggable Storage Engines(插件式存储引擎)

存储层：File System, File and Logs(日志，错误信息，二进制信息，索引信息)



整个MYSQL SERVER 由以下组成

- Connection Pool：连接池组件
- Mangement Services & Utilities：管理服务和工具组件
- SQL Interface：SQL接口组件
- Parser：查询分析器组件
- Optimizer：优化器组件
- Caches & Buffers：缓冲池组件
- Pluggable Storage Engines：存储引擎
- File System：文件系统