# 后台服务开发说明

# tservice代码结构
## signalstudio包中的jetty.service目录：内嵌jetty服务目录

ChangesSystemWebSocket：前端界面通知servlet，负责提供websocket连接
LoginServlet：用户登录接口
ServerMain：内嵌jetty容器的配置和启动代码

## signalstudio包中的controller目录：主要HTTP接口所在目录

AnalysisController：分析数据接口
AuthenticationFilter：接口授权过滤器
BaseController：接口基础类
DatabaseController：数据库信息接口
DeviceController：设备数据接口
DocsController：文档接口
IntersectionController：路口接口
LoginController：登录信息接口
NameRegionController：统计区域接口
NameRoadController：统计道路接口
RoleController：角色接口
SimulateController：仿真数据接口
UserController：用户数据接口
WmsGoogleController：地图背景瓦片接口
WmsTrafficController：路况渲染瓦片接口

## signalstudio包中service目录：外部数据服务

DatabaseInit：数据库初始化工具
NoticeService：前端界面通知服务
SignalService：信号数据采集服务

## 配置：

SignalStudioStronglyTypedSettings：配置文件转换接口

## 主入口：

ProgramMain：程序主入口


# 文档数据库Couchbase

https://www.couchbase.com/
couchbase是基于couchdb及memcached演化而成文档/键值数据库。在本系统中主要负责存储业务数据。

couchbase是基于bucket及json格式存储数据的nosql数据库，既可以存储json格式文档，也可以存储二进制数据。没条数据都使用唯一的key（键）来唯一标识，使用key来提取某条数据是最快速的方式。虽然couchbase支持使用类似sql的方式查询数据，但我们在实际使用中，并没有利用这一特性，而是使用类似视图的view方式来索引及查询数据的。view相当于一类索引，在view中我们设置一个索引函数map，所有bucket中的文档都会调用此函数，符合函数中设置的过滤条件的数据，将会在此view中保留一份索引数据，我们通过后期来搜索这份索引，能提取符合条件的文档的key，在通过key逐个读取需要的文档。

bucket可以理解为传统关系型数据库里的一个数据库，我们按照数据所属的大类，将系统数据分为signalstudio（基础数据）及maptile（地图数据）两个bucket，除了地图需要的数据外，所有数据都储存在signalstudio的bucket中。

这里有个问题：所有数据在一个bucket中，我们如何区分不同类型的数据。我们为bucket中的每个文档都设置了几个必须的属性：Id和Type。Id其实是这个文档key的备份，而Type则是这个文档数据所属于的类型，不同类型的数据包含不同的字段，同时在key上也进行区别。对每一类数据，如果需要进行数据查询，都需要设置一个view来提取这一类数据，我们在view中的map函数中设置

# 代码说明：

整个服务分为两部分，common和signalstudio。common包是包含了基础的工具类及依赖项目，signalstudio是主要的业务部分。

## ProgramMain

程序主入口，主要任务是调用StartService和Shutdown来启动和关闭相关所有内部服务。大部分内部服务的名称都是用service结尾的，而且服务内部的代码结构及接口都类似，表面上都是通过两个函数Setup和StartThread两个函数来初始化和启动自身的。
最后需要注意的是jetty服务的启动是调用的ServerMain这个类来配置和启动内嵌的jetty web容器的。


    private static void StartService(SignalStudioStronglyTypedSettings setting) throws Exception {
    
        CouchbaseAccess.getInstance().Setup(setting);
        CouchbaseAccess.getInstance().TestConnection();
    
        NoticeService.getInstance().Setup(setting);
        NoticeService.getInstance().StartThread();
    
        DataService.getInstance().Setup(setting);
        DataService.getInstance().StartThread();
    
        TileCacheService.getInstance().Setup(setting);
        SqlServerAccess.getInstance().Setup(setting);
    
        SignalService.getInstance().Setup(setting);
        SignalService.getInstance().StartThread();
    
        server = new ServerMain(setting);
        server.run();
    }


## ServerMain

此文件主要任务是启动内嵌的jetty服务，。本系统是使用jetty的embedded方式，在服务中启动jetty容器，并通过代码配置容器来提供http服务的，函数名称是run。在jetty的配置过车中，首先需要设置网页的主目录webRootUri，主目录的路径应有配置文件提供：

    ServletContextHandler context = new ServletContextHandler();
    context.setContextPath("/");
    context.setBaseResource(Resource.newResource(webRootUri));
    context.setWelcomeFiles(new String[]{"index.html"});
    context.setInitParameter("cacheControl", "max-age=0,public");

除了用户登录及websocket是通过简单的servlet提供外，其他数据接口都是通过jersey提供的。jersey是简化servlet开发的一种方法，具体可以使用方法可以在网上查询：https://jersey.github.io/。配置jersey的方式是将signalstudio.controller这个包中的所有文件都配置到jersey的servlet中，在加载这个servlet时，我们给出了一个基础路径就是webapi，所有signalstudio.controller中声明的服务接口，都默认包含在webapi这个路径下面了。controller包中的每个函数，都可以设置不同的子路径。


    ServerContainer wsContainer = WebSocketServerContainerInitializer.configureContext(context);
    wsContainer.addEndpoint(TimeSocket.class);
    wsContainer.addEndpoint(ChangesSystemWebSocket.class);
    wsContainer.addEndpoint(NoticesWebSocket.class);
    
    ServletHolder jerseyServlet = context.addServlet(
            org.glassfish.jersey.servlet.ServletContainer.class, "/webapi/*"); //基础路径
    jerseyServlet.setInitOrder(1);
    
    // Tells the Jersey Servlet which REST service/class to load.
    jerseyServlet.setInitParameter(ServerProperties.PROVIDER_PACKAGES, "signalstudio.controller"); //包路径
    jerseyServlet.setInitParameter(ServerProperties.PROVIDER_CLASSNAMES, "org.glassfish.jersey.filter.LoggingFilter;org.glassfish.jersey.moxy.json.MoxyFeature;org.glassfish.jersey.media.multipart.MultiPartFeature");


## AuthenticationFilter

Controller包中有个文件比较特殊，是filter类型的过滤器：AuthenticationFilter。
这个过滤器目的是过滤对包中所有接口的访问，验证其请求所携带的header中是否加入了正确的登录信息。

