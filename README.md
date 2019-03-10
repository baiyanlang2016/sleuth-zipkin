# sleuth-zipkin
分布式服务追踪

spring-cloud-starter-sleuth 是用来实现分布式服务全链路调用的追踪，日志中的参数：
  TraceID  用来标识一条请求链路
  SpanID   表示一个基本的工作单元
  false/true  表示是否需要将信息输出到Zipkin等服务来收集和展示，在sleuth中采用了抽样收集的方式来为追踪信息打上收集标志，
              该值代表了该消息是否要被后续的追踪信息收集器获取或存储。默认情况下，Sleuth会使用PercentageBaseSampler实现的抽样策略，以请求百分比               的方式配置和收集追踪信息，我们可以通过在application.properties中设置下面的参数对其百分比进行设置，它的默认值是0.1，代表收集10%的请               求追踪信息，即请求10次会收集一次追踪信息，即true,
               spring.sleuth.sampler.probability=1
  
sleuth 追踪的实现原理，主要是通过将TraceId封装在请求头中实现的,通过Annotation来记录请求中每个事件的存在。Annotation理解为一个包含有时间戳的事件标    签，对于一个HTTP请求来说，sleuth定义了4个核心Annotation来标识一个请求的开始和结束：
    cs(Client Send) 该annotation表示客户端发起了一个请求
    sr(Server Recived)表示服务端接受了请求，并开始处理它，通过计算cs和sr两个Annotation的时间戳之差，我们可以得到当前HTTP请求的网络延时
    ss(Server Send) 记录服务器端处理完请求后准备发送请求响应信息。通过计算ss和sr的时间戳之差，可以得到当前服务端处理请求的时间消耗
    cr(Client Recived)记录客户端接受到服务器端的回复，同时也表示可这个HTTP请求的结束，通过计算cs和cr时间戳之差，可以得到HTTP请求从客户算发起到接                       受服务器端响应的总时间消耗

分散在各个服务实例的文件系统上的日志需要收集才能综合分析，sleuth可以整合以下组件实现日志分析： ELK、Zipkin

ELK ：
  ElasticSearch 开源的分布式搜索引擎
  Logstash  对日志进行收集、过滤、存储
  Kibana    为ElasticSearch和Logstash提供友好的Web界面
  
Zipkin ： 在ELK平台中的数据分析维度缺少对全链路中各阶段时间延迟的关注，为了实现对分布式系统做延迟监控等与时间消耗相关的请求，
  ELK就显得力不存心，ZipKin就可以轻松解决  

下图展示了ZipKin的基础架构，主要由4个和核心组件构成：
![image](https://github.com/baiyanlang2016/sleuth-zipkin/tree/master/images/zipkin.jpg)

Collector  收集器组件，处理从外部系统发送过来的追踪信息，将这些信息转化为zipkin内部处理的Span格式，以支持后续的储存、分析、展示等
Storge    存储组件 处理收集器接受到的追踪信息，默认会将信息存储在内存中，我们也可以修改存储策略，通过使用其他存储组件将追踪信息存储到数据库中
RESTful API  API组件，提供外部访问接口。比如给客户端展示追踪信息，或是外接系统访问以实现监控
WEB UI 基于API组件实现的上层应用。通过UI组件，用户可以方面又直观的查询和分析追踪系统

（一） Collector--->日志收集：
  HTTP收集： 1.创建一个Zipkin-server应用，并通过http://localhost:9092 访问Zipkin管理界面 
                <dependency>
                    <groupId>io.zipkin.java</groupId>
                    <artifactId>zipkin-server</artifactId>
                    <version>2.10.1</version>
                </dependency>
                <dependency>
                    <groupId>io.zipkin.java</groupId>
                    <artifactId>zipkin-autoconfigure-ui</artifactId>
                    <version>2.12.3</version>
                </dependency>
            2.为应用引入和配置zipkin服务，并在配置文件中配置zip的服务端
                <dependency>
                    <groupId>org.springframework.cloud</groupId>
                    <artifactId>spring-cloud-sleuth-zipkin</artifactId>
                </dependency>
                
   消息中间件收集： 客户端将追踪消息输出到消息中间件上，同时zipkin服务端从消息中间件上异步的消息这些追踪消息
                 1. 修改客户端应用
                   1.1 引入依赖，并application.properties中去掉HTTP方式实现时使用的spring.zipkin.base-url
                   <dependency>
                        <groupId>org.springframework.cloud</groupId>
                        <artifactId>spring-cloud-sleuth-stream</artifactId>
                        <version>1.3.5.RELEASE</version>
                    </dependency>
                    <dependency>
                        <groupId>org.springframework.cloud</groupId>
                        <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
                    </dependency>

                    1.2.在配置文件中添加rabbitmq的配置信息
                      spring.rabbitmq.addresses=
                      spring.rabbitmq.host=
                      spring.rabbitmq.port=
                      spring.rabbitmq.password=
                      
                      #项目启动后将在rabbitmq创建sleuth exchange和zipkin queue
                      #发送的队列名称，默认zipkin，正常情况不要修改因为zipkin-server默认监控队列也是zipki
                      #https://blog.csdn.net/xjune/article/details/79870054
                  
                  2.修改Zipkin-server服务端
                      <dependency>
                          <!-- 该依赖时实现从消息中间件收集追踪信息的核心封装-->
                      <dependency>
                          <groupId>org.springframework.cloud</groupId>
                          <artifactId>spring-cloud-sleuth-zipkin-stream</artifactId>
                          <version>1.3.5.RELEASE</version>
                      </dependency>
                      <dependency>
                          <groupId>org.springframework.cloud</groupId>
                          <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
                      </dependency>
                      <!--zipkin的前端依赖-->
                      <dependency>
                          <groupId>io.zipkin.java</groupId>
                          <artifactId>zipkin-autoconfigure-ui</artifactId>
                          <version>2.12.3</version>
                      </dependency>
                   
    日志收集机制：
       sleuth中的Span对象包含：Span的开始时间、结束时间、Span的名称、TraceID、SpanID、Tags(对应Zipkin中的BinaryAnnotation(记录了服务名、类名、方法名、时间戳、IP地址、端口号等信息))、Logs(对应Zipkin中的Annotation)等
       Zipkin的Span对象包含：TraceID、name、id、parentID、timestamp、annotation(包含时间戳)、binaryAnnotation()等
       
       所以收集机制中存在Sleuth的Span对象到Zipkin的Span对象的转换函数converted
                     
(二) Storge ---->数据存储                     
     默认情况下，Zipkin-server会将追踪信息存储在内存中，每次重启Zipkin-server都会使之前的追踪信息丢失。
     ZipKin的Storge组件默认中提供了对MySql的支持。下面基于消息中间件实现的zipkin-server应用，对其进行mysql存储扩展：
      1. 为zipkin-server添加依赖
      <!--添加mysql存储日志-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>
        <dependency>
            <!--解决zipkin收集并入库时的bug,必须选择3.8.0版本-->
            <groupId>org.jooq</groupId>
            <artifactId>jooq</artifactId>
            <version>3.8.0</version>
        </dependency>
        
       2. 在MySQL中创建基于Zipkin存储的shema,不同版本的zipkin对数据库表结构有一些变化，所以尽量使用对应版本的脚本来创建Schema。同时Zipkin实现的          mysql存储仅在MySql 5.6-5.7版本中测试过，所以尽量选择对应版本的MySql.从依赖的jar(zipkin-storge-mysql)中找到mysql.sql脚本。
       
          也可以通过在程序中进行配置的方式让其自动初始化，只需在application.properties中增加如下配置：
          spring.datasource.schema=classpath:/mysql.sql
          spring.datasource.url=
          spring.datasource.username=
          spring.datasource.password=
          spring.datasource.continue-on-error=true
          spring.datasource.initialization-mode=always
          
          程序启动后可得到两张表：
            zipkin_spans: 存储span信息的表
            zipkin_annotations: 存储annotation信息的表
            
        3. 切换存储类型，让zipkin-server连接到MySql,配置如下：
            zipkin.storage.type=mysql
                     
(三) API ------> RESTful Api
![image](https://github.com/baiyanlang2016/sleuth-zipkin/blob/master/images/zipkin-api.jpg)
        
   更多关于A{PI页面，可以访问官网  https://zipkin.io/zipkin-api/
                     

  


