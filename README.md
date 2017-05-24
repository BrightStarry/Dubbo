### Dubbo/Dubbox
当当网为dubbo增加了一些功能，将其命名为Dubbox（Dubbo eXtensions）

http://dubbo.io/  官网，里面有用户指南

分布式服务框架，也是SOA基础框架，以及高性能的RPC远程过程调用方案。

常用的MVC分层，dao层调用数据库，service层调用dao层，controller层调用service。（垂直化）

而dubbo将dao层和service层打成jar部署在了不同机器上（服务,SOA）；
将controller打成war放到容器tomcat中。
然后在controller中调用不同的服务。

还可以根据服务（SOA,service层和dao层的jar）部署机器的性能，根据权重，进行负载均衡。

---
核心包括：
1. 远程通讯，提供多种基于长连接的NIO框架抽象封装，包括多种线程模型，序列化，
以及“请求-响应”模式的信息交换方式。
2. 集群容错：提供基于接口方法的远程过程调用，包括多协议支持，
以及软负载均衡、失败容错、地址路由、动态配置等集群支持。
3. 自动发现：基于注册中心目录服务，使服务消费方，能动态地查找服务提供方，
使地址透明，使服务提供方可以平滑增加或减少机器。
---
#### 节点角色说明
* Provider：暴露服务的服务提供方；
* Consumer:调用远程服务的服务消费方；
* Register：服务注册与发现的注册中心；（使用zookeeper或redis作注册中心）
* Monitor:统计服务调用次数和调用时间的监控中心；
* Container：服务运行容器。
---
#### 说明
下面的几个供应商啊，消费者啊，我先简单阐述一下。

就是说，一个供应商里有AService接口，并且有对AServiceImpl的具体实现。
那么在这个项目中，进行如下配置，就可以让消费者调用该服务。

而消费方，应该也有一个AService接口，不过本地没有具体的实现，就需要进行如下配置，将上面供应商中的AServiceImpl实现类
注入成这个接口的实现类。

---
#### Dubbo供应商，实现将Service提供给其他机器调用
只需要在spring-configuration.xml配置(需要引入dubbo命名空间)

    <!-- 具体的实现bean -->
	<bean id="sampleService" class="bhz.dubbo.sample.provider.impl.SampleServiceImpl" />

	<!-- 提供方应用信息，用于计算依赖关系，name就是当前服务的名字 -->
	<dubbo:application name="sample-provider" />

	<!-- 使用zookeeper注册中心暴露服务地址 -->
	<dubbo:registry address="zookeeper://192.168.1.111:2181?backup=192.168.1.112:2181,192.168.1.113:2181" />
	<!--或者下面这种写法，随便选择一种-->
	<dubbo:registry protocol="zookeeper" address="192.168.1.111:2181,192.168.1.112:2181,192.168.1.113:2181" />
	

	<!-- 用dubbo协议在20880端口暴露服务 -->
	<dubbo:protocol name="dubbo" port="20880" />

	<!-- 声明需要暴露的服务接口  写操作可以设置retries=0 避免重复调用SOA服务,ref引用的就是具体实现的那个bean，
	retries可以设置重试次数，也就是当调用这个服务失败后，能重试几次（默认重新调用是调用这个服务的集群中的其他机器）-->
	<dubbo:service retries="0" interface="bhz.dubbo.sample.provider.SampleService" ref="sampleService" />
---
#### Dubbo消费者，调用远程服务
只需要在spring-configuration.xml配置(需要引入dubbo命名空间)

	<!-- 消费方应用名，用于计算依赖关系，不是匹配条件，不要与提供方一样 -->
	<dubbo:application name="sample-consumer" />
    
    <!--可以注册多个中心，再复制下面这句话就可以了-->
	<dubbo:registry address="zookeeper://192.168.1.111:2181?backup=192.168.1.112:2181,192.168.1.113:2181" />

	<!-- 生成远程服务代理，可以像使用本地bean一样使用demoService 检查级联依赖关系 默认为true 当有依赖服务的时候，需要根据需求进行设置 
	本地也要有和需要调用的服务类一样的接口-->
	<dubbo:reference id="sampleService" check="false"  interface="bhz.dubbo.sample.provider.SampleService" />
---
#### Dubbo消费者+供应商 就是在一个service中，既调用别的service，又将其他service供应给别人调用
只需要在spring-configuration.xml配置(需要引入dubbo命名空间)：

    <!-- 具体的实现bean -->
    <bean id="sampleService" class="bhz.dubbo.sample.provider.impl.SampleServiceImpl" />
    
    <!-- 提供方应用信息，用于计算依赖关系，name就是当前服务的名字 -->
    <dubbo:application name="sample-provider" />

	<!-- 使用zookeeper注册中心暴露服务地址 -->
	<dubbo:registry address="zookeeper://192.168.1.111:2181?backup=192.168.1.112:2181,192.168.1.113:2181" />
	<!--或者下面这种写法，随便选择一种-->
	<dubbo:registry protocol="zookeeper" address="192.168.1.111:2181,192.168.1.112:2181,192.168.1.113:2181" />

    <!-- 用dubbo协议在20880端口暴露服务 -->
    <dubbo:protocol name="dubbo" port="20880" />
    
    <!--注意，我们在使用sampleService的时候，sampleService可能还需要调用别的service，就可以引入这个aaaService，
    也就是sampleService依赖于这个aaaService
    如果check写成true，启动时，就会检查SampleService这个服务所依赖的其他服务是否已经启动了，默认为true-->
	<dubbo:reference id="aaaService" check="false"  interface="bhz.dubbo.sample.provider.aaaService" />
    
    <!-- 声明需要暴露的服务接口  写操作可以设置retries=0 避免重复调用SOA服务,ref引用的就是具体实现的那个bean-->
	<dubbo:service retries="0" interface="bhz.dubbo.sample.provider.SampleService" ref="sampleService" />
---
#### 本地模拟启动服务，也就是运行只有service层和dao层的那个jar
可以使用一个类，ClassPathXmlApplicationContext加载spring.xml，
然后context.start(),
最后再加一句System.in.read()，让这个服务一直开着
	
---
###安装tomcat    
http://www.cnblogs.com/hanyinglong/p/5024643.html
    
解压
tar -zxvf apache-tomcat-8.0.43.tar.gz  
改名
mv apache-tomcat-8.0.43 tomcat
启动
/zx/tomcat/bin/startup.sh
停止
/zx/tomcat/bin/shutdown.sh
    
启动java web 项目，将项目放到webapps目录下。
    
查看tomcat日志
tail -f -n 500 /zx/tomcat/logs/catalina.out
   
vim catalina.sh 可以修改JVM参数：
JAVA_OPTS='-Xms256m -Xmx512m'
	
---
#### Dubbo控制台，dubbo-admin.war
从github上下载dubbo的zip文件解压。（github上目前是2.5.4，但maven中只有2.5.3的jar）。
然后需要在根目录执行mvn install -Dmaven.test.skip=true   ，如果直接去dubbo-admin目录中执行，会报找不到dubbo.jar的错。
执行完上面的语句后，本地maven仓库就有了2.5.4的jar。
并且dubbo-admin/target中也有了控制台对应的war。

放置到tomcat/webapps中后，运行tomcat后自动解压。
然后需要修改项目中WEB-INF中的dubbo.properties，
zookeeper的地址，用户名，密码等。
如果zookeeper是集群，就这么写
zookeeper://192.168.2.104:2181?backup=192.168.2.105:2181,192.168.2.106:2181

最后访问 192.168.2.104:8080/dubbo-admin 并输入上面配置的帐号密码即可

---
### Dubbox   https://github.com/dangdangdotcom/dubbox   目前2.8.4，好像也不更新了
---
#### 重要
我直接把Dubbox那段快进了。究其原因，还是觉得这个dubbo马上就要被淘汰了，而且官网和github都有中文文档，
我倒不如把那些文档好好地看一遍。
浏览器按ctrl +　p可以把网页保存成pdf。。我把dubbo用户指南弄了个pdf出来。详见项目中的pdf


