# ES, Kibana配置文档-入门篇



| 编撰人 |    时间    |  描述  |
| :----: | :--------: | :----: |
|  Lee   | 2021/06/04 | 初始化 |
|        |            |        |

## Docker启动 Elasticsearch和 Kibana

本文将使用 `docker-compose.yml` 文件来创建 ES及Kibana容器，此文件包含镜像配置、端口映射等。

创建docker工作目录：

```
mkdir -p elastic-kibana/src/docker
cd elastic-kibana/src/docker
```

创建好文档之后，创建`docker-compose.yml` 文件，并写入如下配置：

```yml
version: '3.1'

services:

  elasticsearch:
   container_name: elasticsearch
   image: docker.elastic.co/elasticsearch/elasticsearch:7.9.2
   ports:
    - 9200:9200
   volumes:
    - elasticsearch-data:/usr/share/elasticsearch/data
   environment:
    - xpack.monitoring.enabled=true
    - xpack.watcher.enabled=false
    - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    - discovery.type=single-node
   networks:
    - elastic

  kibana:
   container_name: kibana
   image: docker.elastic.co/kibana/kibana:7.9.2
   ports:
    - 5601:5601
   depends_on:
    - elasticsearch
   environment:
    - ELASTICSEARCH_URL=http://localhost:9200
   networks:
    - elastic
  
networks:
  elastic:
    driver: bridge

volumes:
  elasticsearch-data:
```

bash进入该目录下, 在docker文件夹中运行`docker compose`命令来启动容器:

```bash
docker-compose up -d
```

第一次运行`docker compose`命令时，它将从docker Hub加载ElasticSearch和Kibana的镜像。

运行该命令后，检查ElasticSearch和Kibana是否已启动并运行：`docker ps -a`

进入 http://localhost:9200 查看 Elasticsearch 是否运行，运行成功则返回如下json：

```json
{
  "name" : "34b9b680a416",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "mbWUsAm8T-OIZpmGvOmx7w",
  "version" : {
    "number" : "7.9.2",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "d34da0ea4a966c4e49417f2da2f244e3e97b4e6e",
    "build_date" : "2020-09-23T00:45:33.626720Z",
    "build_snapshot" : false,
    "lucene_version" : "8.6.2",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

以上是该端口下ES服务的基本信息；然后进入http://localhost:5601下确保kibana在线。

## .NET Core与SerialLog配置

项目需要安装的包：

```
Serilog.AspNetCore
Serilog.Enrichers.Environment
Serilog.Sinks.Debug
Serilog.Sinks.Elasticsearch
```

在项目目录下添加`elk.json`文件：

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information", //最低级别日志
      "Override": {
        "Microsoft": "Information",
        "System": "Warning"
      }
    }
  },
  "ElasticConfiguration": {
    "Uri": "http://localhost:9200"
  },
  "AllowedHosts": "*"
}
```

这块如果将配置放到`appsetting.json`文件中，后续好像无法读取到该配置，有兴趣的可试一下。

接下来，在启动程序的Main方法中，在创建Host之前设置Logger。如果启动失败，便可以记录，示例如下：

```c#
 public static void Main(string[] args)
 { 
     //配置Logger
     ConfigLogger();
     
     CreateHostBuilder(args).Build().Run();
 }
```

`ConfigLogger`方法实现如下：

```c#
 private static void ConfigLogger() 
{
	var environment = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT");
	var configuration = new ConfigurationBuilder()
	                .AddJsonFile("elk.json", optional: false, reloadOnChange: true)
	                .Build();
	Log.Logger = new LoggerConfiguration()
	               .Enrich.FromLogContext()
	               .Enrich.WithMachineName()
	               .WriteTo.Debug()
	               .WriteTo.Console()
	               .WriteTo.Elasticsearch(ConfigureElasticSink(configuration, environment)) //配置读取
	               .Enrich.WithProperty("Environment", environment)
	               .ReadFrom.Configuration(configuration)
	               .CreateLogger();
}
private static ElasticsearchSinkOptions ConfigureElasticSink(IConfigurationRootconfiguration,string environment) 
{
	var option = new ElasticsearchSinkOptions(new Uri(configuration["ElasticConfiguration:Uri"])) 
	{
		AutoRegisterTemplate = true,
		IndexFormat = $"{Assembly.GetExecutingAssembly().GetName().Name.ToLower().Replace(".", "-")}-{environment?.ToLower().Replace(".", "-")}-{DateTime.UtcNow:yyyy-MM}"
		            };  //这里的索引名称可通过天、月份配置
		return option;
	}
```

当然，更合理的方式是要记录Host启动的日志，因为以上代码运行完后，logger就可使用了：

```c#
private static void CreateHost(string[] args)
{
	try
	{
		CreateHostBuilder(args).Build().Run();
	}
	catch (System.Exception ex)
	{
		Log.Fatal($"Failed to start {Assembly.GetExecutingAssembly().GetName().Name}", ex);
		throw;
	}
}
```

最后，别忘了在Builder中启用SerialLog配置：

```c#
 public static IHostBuilder CreateHostBuilder(string[] args) =>
            Host.CreateDefaultBuilder(args)
            .UseServiceProviderFactory(new AutofacServiceProviderFactory())
            .ConfigureWebHostDefaults(webBuilder =>
            {
               webBuilder.UseStartup<Startup>().UseUrls("http://*:5000").UseSerilog(); 
            });
```

这块直接`UseSerilog()`就可以，目前好像是只能有一种方式处理日志，即写本地或者写ES等等。

## 日志发送

在`ConfigureElasticSink`方法中，`new`出来的`ElasticsearchSinkOptions`中有一个`indexPattern`属性，这就是Es中区分数据源的索引，在Es中将根据这个索引来对日志进行基本的处理，如果发送成功，Kibana的索引列表中将显示对应的索引名称：

![image-20210604103923608](https://raw.githubusercontent.com/GFLEE/PicBed/main/img/image-20210604103923608.png)

Kibana目前还不会显示任何日志。在查看日志数据之前，必须指定索引，因此必须创建新的索引，索引创建可根据Kibana文档进行，不赘述：

![image-20210604104152449](https://raw.githubusercontent.com/GFLEE/PicBed/main/img/image-20210604104152449.png)

 ![image-20210604105436466](https://raw.githubusercontent.com/GFLEE/PicBed/main/img/image-20210604105436466.png)

