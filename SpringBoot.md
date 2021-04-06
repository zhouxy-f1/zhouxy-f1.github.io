##### 快速生成demo

```shell
#生成一个demo
1.通过IDEA生成
文件 > 新建 > 项目 > Sping Initializr > 下一步 > 填写项目设置（组：一般用项目名；工件：用工程名）> 下一步 > 依赖 Spring Web > 完成

2.在官网上生成
创建方法跟idea一样，地址是： https://start.spring.io/


#制作demo主页
1.通过mapping，在默认主类下新建文件controller/UserController，内容如下
@Controller
@RequestMapping(value = "/user",method = {RequestMethod.GET,RequestMethod.POST})
public class UserController {

    @RequestMapping("/hello")
    @ResponseBody
    public String hello(){
        return "Spring Boot Demo";
    }
2.自建index.html，默认会读/src/resoucres/static/index.html文件
默认静态文件存放于/src/resources/static或者/src/resources/resources目录下的index.html
```

##### 访问资源

```bash
#处理静态资源
静态资源默认存放于/src/resources/static或/src/resources/resources目录下，默认读取index.html文件

#处理动态资源
1.处理动态资源需要在pom.xml中加载thymeleaf模块
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-thymeleaf</artifactId>
		</dependency>
2.主类
/src/main/java/com.herelink.herelinkdemo/controller/UserController
package com.herelink.herelinkdemo.controller;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.*;
import java.util.Map;

@Controller
@RequestMapping(value = "/user",method = {RequestMethod.GET,RequestMethod.POST})
public class UserController {

    @RequestMapping("/hello")
    @ResponseBody
    public String hello(){
        return "Spring Boot Demo";
    }

    @RequestMapping("/test")
    public String test(Map<String,String>map){
        String str="zhouxy";
        map.put("str",str);
        return "content";

    }
}
3.新建模板目录和文件
新建/src/resources/templates/content.html，内容如下
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<span th:text="${str}"> thymeleaf-demo</span>
</body>
</html>

```

##### 配置文件

```bash
默认的配置文件是/src/main/resources/application.properties，如果需要根据不同环境写不同的配置文件，

1.定义配置文件，如下
application-develop.yml		=>	对应开发测试环境
application-release.yml		=>	对应集成测试环境
application-master.yml		=>	对应生产环境

2.激活配置
CMD java  -Xms700m -Xmx980m -Dspring.profiles.active="ENV_PROFILE" -jar /maven/PROJECT_NAME.jar

docker run -it --spring.profiles.active="develop" 


```

