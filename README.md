# sleuth-zipkin
分布式服务追踪

spring-cloud-starter-sleuth 是用来实现分布式服务全链路调用的追踪
  TraceID  用来标识一条请求链路
  SpanID   表示一个基本的工作单元
  false/true  表示是否需要将信息输出到Zipkin等服务来收集和展示
  
sleuth 追踪的实现原理，主要是通过将TraceId封装在请求头中实现的

分散在各个服务实例的文件系统上的日志需要收集才能综合分析，sleuth可以整合以下组件实现日志分析： ELK、Zipkin

ELK ：
  ElasticSearch 开源的分布式搜索引擎
  Logstash  对日志进行收集、过滤、存储
  Kibana    为ElasticSearch和Logstash提供友好的Web界面
  
Zipkin ： 在ELK平台中的数据分析维度缺少对全链路中各阶段时间延迟的关注，为了实现对分布式系统做延迟监控等与时间消耗相关的请求，
  ELK就显得力不存心，ZipKin就可以轻松解决  

下图展示了ZipKin的基础架构，主要由4个和核心组件构成：
