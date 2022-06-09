# 多Web部署

## springboot原状
我们先从传统的springboot构建的基于内置tomcat的web应用说起。其在运行main函数初始化时，使用TomcatServletWebServerFactory#getWebServer这一工厂方法，创建了一个实现WebServer接口的TomcatWebServer实例，这里的TomcatWebServer实例就是内置tomcat的映射，包括启动、停止等方法。springboot自身基于WebServer还有jetty、netty等webserver的实现，同样有其对应的工厂方法创建。对应的工厂bean基于springboot的自动装配机制加载。
## 关键问题
相较于单纯的springboot应用，一个Ark包的复杂之处在于，它可以包含多个Ark Biz，其中每个Ark Biz都是一个完整的springboot项目。因此使用内置tomat启动时会面临以下问题：
1. 多个Biz(springboot项目)需要共用一个tomcat实例
2. 需要像传统tomcat下部署多webapp一样，通过添加前缀的方式区分不同Biz的http接口

因此sofa-ark对springboot的相关实现做了替换，具体如下
|sofa-ark|springboot|
|---|---|
|ArkTomcatServletWebServerFactory|TomcatServletWebServerFactory|
|ArkTomcatEmbeddedWebappClassLoader|TomcatEmbeddedWebappClassLoader|
|ArkTomcatWebServer|TomcatWebServer|
### 多Biz共用tomcat实例
针对第一个问题——多个Biz要共用一个tomcat实例，sofa-ark定义了EmbeddedServerService接口，并使用其插件机制来扩展，插件为web-ark-plugin，里面包含了EmbeddedServerService的实现EmbeddedServerServiceImpl。

如果Biz引入了web-ark-plugin，则在ArkTomcatServletWebServerFactory中注入EmbeddedServerServiceImpl，作用持有第一个初始化的Biz调用getWebServer创建的Tomcat实例(TomcatWebServer的核心)时，并在后续初始化的其它Biz调用getWebServer获取tomcat实例时，返回持有的同一个实例，以此来保证多个Biz运行在同一个tomcat中。

### 多Biz接口区分
对于第二个问题——区分不同Biz的http接口，独立的运行的tomcat是通过contextPath这一配置来实现的，每个webapp设置不同的contextPath，作为不同webapp接口的前缀，例如
````
<context path="test1" docBase="~/Documents/web1/" reloadable = true>
<context path="test2" docBase="~/Documents/web2/" reloadable = true>
````
默认情况下使用war包解压后的文件夹名作为其contextPath
springboot中可使用以下方式指定contextPath，默认为""，一个springboot项目只能指定一个。
````
server:
  servlet:
    context-path: /myapp1
````
因此对于sofa-ark而言，参考了独立tomcat的实现方式，基于contextPath区分，并对springboot的内置tomcat实现做了改造。
每个Biz通过在其manifest文件中配置web-context-path属性的值作为其contextPath，在调用BizFactoryServiceImpl的createBiz方法创建BizModel时设置到该Biz的BizModel对象中，随后在ArkTomcatServletWebServerFactory的prepareContext方法中，为每个Biz创建其Context时，设置其对应的contextPath。