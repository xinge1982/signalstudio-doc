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

这个文件在Controller包中比较特殊，是filter类型的过滤器。
这个过滤器目的是过滤对包中所有接口的访问，验证其请求所携带的header中是否加入了正确的登录信息。在此文件的filter函数中，程序校验了请求中名称为**Authorization**的header数据，如果数据符合系统要求就可以通过过滤，否则程序会报出异常终止请求继续执行。

## IntersectionController

信号路口的接口文件，定义了对应路口存取数据的所有接口。
大部分接口存取数据都使用到了与couchbase建立的数据库连接来操作数据，这就涉及到使用CouchbaseAccess这个类维护的数据库连接。CouchbaseAccess在系统中只存在一个唯一的实例，用以管理系统服务与数据库的连接。CouchbaseAccess在系统启动时，在ProgramMain程序中被初始化并与数据库建立连接，每个连接都对应唯一一个bucket，建立的连接被储存在实例的一个Map中，键值就是bucket的名称。之后如果有其他代码需要使用数据库的连接，就需要调用CouchbaseAccess的getBucket函数取到对应名称的bucket连接。一般的代码如下：

    Bucket bucket = CouchbaseAccess.getInstance().getBucket(defaultBucketName);

取到数据库连接bucket对象后，可以使用它来查询view或者操作文档，一般我们使用这样几个函数来操作数据：

1. **使用文档的key直接获取文档内容：**
    JsonDocument document = bucket.get(documentKey);


2. **使用JsonObject对象构造新的文档，并添加或覆盖之前的文档：**
    JsonDocument newdoc = JsonDocument.create(documenetKey, jsonObject);
    bucket.upsert(newdoc, 3, TimeUnit.SECONDS);

或者使用CouchbaseAccess提供的修改数据函数直接修改数据：

    JsonDocument documentChanged = CouchbaseAccess.getInstance().upsert(defaultBucketName, document);


3. **查询view读取数据**

查询view需要用到几个类：

- ViewQuery 构建查询
- ViewResult 遍历结果
- RxJava用于异步批量处理查询结果

一般的查询过程如下：

    ViewQuery viewQuery = ViewQuery.from(viewDoc, view)
            .stale(Stale.FALSE);
    
    ViewResult result = bucket.query(viewQuery);
    if (result.success()) {
        List<JsonObject> results = rx.Observable.from(result.allRows())
                .flatMap(r -> bucket.async().get(r.id()))
                .map(r -> {
                    。。。
                })
                .toList()
                .toBlocking()
                .single();
    
        return JsonObject.create().put("total", result.totalRows())
                .put("data", results).toString();
    }

说明：

- 1-2行是构建查询的view对象，需要提供view的文档名称及名称
- 4行是执行查询。
- 5行判断查询是否成功
- 6-13行是使用了RxJava的方式，从之前查询的ViewResult中遍历查询的结果。我们在此首先取ViewResult中的所有数据allRows，此列表总记录了查询的结果，包括查询出的文档id和值。之后是遍历这个列表，通过结果中的文档id读取每个文档的具体内容。此处用了几个函数：
  - flatmap：通过bucket对所有文档id读取文档内容
  - map：对文档内容进行后续处理，并返回最终需要的JsonObject。
  - toList：将结果组成列表的形式
  - toBlocking和single：等待所有数据都返回后，继续执行下面的代码


