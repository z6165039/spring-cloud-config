#使用Spring Cloud Config搭建配置中心
笔者的微服务项目中需要使用一个统一的管理分布式系统的配置中心，之前试用过[Disconf](https://github.com/knightliao/disconf)。实际使用发现，Disconf可以很好地满足项目的需求，同时提供了一个友好的图形化界面供操作。但是，其服务器端使用了Nginx, Tomcat, MySQL, Zookeeper和Nginx，安装部署起来比较复杂；客户端也引入了不少的Jar包，可能引起版本冲突。因此，笔者也在留意是否有其它的可选方案。最近接触到Spring Cloud Config，认为也是一个搭建配置中心不错的选择，在此和大家分享一下试用的经验，供大家参考。
文章中的样例代码可以在[https://git.oschina.net/gongxusheng/spring-config-demo](https://git.oschina.net/gongxusheng/spring-config-demo)下载。
本案例共有三个项目，分别是：

 - my-sample-config: 简单项目，用于管理配置文件
 - my-config-server: Spring Cloud Config Server即服务器项目
 - my-config-client: Spring Cloud Config Client即客户端项目

##配置文件的管理
Spring Cloud Config支持在Git, SVN和本地存放配置文件，使用Git或者SVN存储库可以很好地支持版本管理，Spring默认配置是使用Git存储库。在本案例中将使用OSChina提供的Git服务存储配置文件。为了能让Config客户端准确的找到配置文件，在管理配置文件项目my-sample-config中放置配置文件时，需要了解application, profile和label的概念：

- {application}映射到Config客户端的spring.application.name属性
- {profile}映射到Config客户端的spring.profiles.active属性，可以用来区分环境
- {label}映射到Git服务器的commit id, 分支名称或者tag，默认值为master

例如在本案例中，计划将客户端my-config-client的spring.application.name设置为my-client，则上传一个文件名为my-client.yml配置文件；另外计划在uat环境上做不同的配置，则再上传一个文件名为my-client-uat.yml配置文件
##Spring Cloud Config服务器的配置
使用Spring Starter Project快速创建一个Config Server项目。

1) 增加@EnableConfigServer注解

```
@SpringBootApplication
@EnableConfigServer
public class MyConfigServerApplication {
```
2) 配置application.yml文件，设置服务器端口号和配置文件Git仓库的链接

```
server:
  port: 8888
spring:
  cloud:
    config:
      server:
        git:
          uri: https://git.oschina.net/gongxusheng/spring-config-demo.git
          searchPaths: my-sample-config
```
如果配置文件放置在Git存储库的根目录下，则无需使用searchPaths参数，本例中的配置文件在my-sample-config目录中，因此使用searchPaths参数提示Config服务器搜索my-sample-config子目录。

3)启动Config服务器MyConfigServerApplication，在浏览器中输入[http://localhost:8888/my-client/master](http://localhost:8888/my-client/master)，既可以验证Config服务器从Git存储库读到了配置文件。

```
{
  "name": "my-client",
  "profiles": ["master"],
  "label": null,
  "version": "9bace30e907ff34dc2ce377ceb7831278c856455",
  "propertySources": [
    {
      "name": "https://git.oschina.net/gongxusheng/spring-config-demo.git
        /my-sample-config/my-client.yml",
      "source": {"my-config.appName": "my-app"}
    }
  ]
}
```
Config服务器支持带多种形式的URL，读者可以尝试一下：

 - 增加profile参数，[http://localhost:8888/my-client/uat/master](http://localhost:8888/my-client/uat/master)
 - yml格式，[http://localhost:8888/my-client.yml](http://localhost:8888/my-client.yml)
 - properties格式，[http://localhost:8888/my-client.properties](http://localhost:8888/my-client.properties)

##Spring Cloud Config客户端的配置
使用Spring Starter Project快速创建一个Config Client+Web+Actuator项目。Spring Web用于演示获取属性值，Actuator的作用在于提供属性动态刷新、属性查看等功能。

1) 在bootstrap.yml中设置客户端名称spring.application.name和Config服务器的地址，注意spring.application.name要和配置文件名相符

```
spring:
  application:
    name: my-client
  cloud:
    config:
      uri: http://localhost:8888
```
2) 在@SpringBootApplication注解的MyConfigClientApplication类中增加一个方法，验证可以从Environment中获得配置

```
@Autowired
void setEnviroment(Environment env) {
	System.out.println("my-config.appName from env: " 
		+ env.getProperty("my-config.appName"));
}
```
3) 创建一个@RestController注解的类，验证使用@Value注解注入配置的值

```
@RestController
@RefreshScope
public class MySampleRestController {
	@Value("${my-config.appName}")
	private String appName;
	
	@RequestMapping("/app-name")
	public String getAppName() {
		return appName;
	}
}
```
注解@RefreshScope指示Config客户端在服务器参数刷新时，也刷新注入的属性值，详情可见后续的章节

4) 启动客户端应用MyConfigClientApplication，即可见其中日志中输出了

```
my-config.appName from env: my-app
```
在浏览器中访问[http://localhost:8080/app-name](http://localhost:8080/app-name)，可见输出my-app

5) 接下来测试一下获取参数profile指定环境的属性文件。在MyConfigClientApplication的启动VM参数中加入-Dspring.profiles.active=uat，重新启动，可见输出变为：

```
my-config.appName from env: my-app-uat
```
##动态刷新配置
无需重新启动客户端，即可更新Spring Cloud Config管理的配置

1)在Git中更新 my-client-uat.yml 文件中的配置:
```
my-config:
  appName: my-app-uat-new
```

2) 访问[http://localhost:8888/my-client/uat/master](http://localhost:8888/my-client/uat/master)，可见属性my-config.appName已经更新。但此时访问[http://localhost:8080/app-name](http://localhost:8080/app-name)，客户端读到属性值尚未更新。

3) 对Conf客户端发一个POST请求[http://localhost:8080/refresh]()，返回200 OK。再次访问[http://localhost:8080/app-name](http://localhost:8080/app-name)，可见在并未重启客户端服务的情况下，读到的属性值已经动态更新