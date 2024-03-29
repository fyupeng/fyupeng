---
title: SpringMVC
top: false
cover: false
toc: true
mathjax: false
date: 2022-07-11 14:36:59
author: fyupeng
img:
coverImg:
password:
summary:
tags:
- Spring
- MVC
categories:
- Java框架
---


## 1. 声明和配置

### 1.1 自定义容器配置文件

- 在web.xml配置文件的的web-app下

```xml
<!-- 声明，注册springmvc的核心对象DispatcherServlet -->
<servlet>
    <servlet-name>springmvc</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>

    <!-- 自定义springmvc配置文件读取的位置 -->
    <load-on-startup>1</load-on-startup>

    <init-param>
        <!-- springmvc配置文件位置的属性 -->
        <param-name>contextConfigLocation</param-name>
        <!-- 指定自定义文件的位置 -->
        <param-value>classpath:springmvc.xml</param-value>
    </init-param>

</servlet>
```

- springmvc默认位置：

```xml
<!--如果不写自定义路径-->
<param-value>classpath:springmvc.xml</param-value>
```

则会找默认路径：`WebContent/WEB-INF/lib/`

**要求：**

- 声明类`org.springframework.web.servlet.DispatcherServle`t的`servlet-name`时，其对应的配置文件名称也应为`servlet-name`的值 + `"-servlet".xml`

pom.xml中加入两个依赖包：

```xml
<!-- servlet依赖 -->
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>3.1.0</version>
    <scope>provided</scope>
</dependency>

<!-- springmvc依赖 -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webmvc</artifactId>
    <version>5.2.5.RELEASE</version>
</dependency>
```

步骤：

1. 由于`tomcat`服务器每次启动会重新加载`web.xml`配置文件，从而在里面配置启动`springmvc.xml`，使它自动初始化

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/70d63fad-6c28-4243-b31e-3c388c1a2c15.png)

2. 由`index.jsp`通过链接发起，再用`MyController`处理转发给`show.jsp`

**注意：**

​	MyController通过注解实现控制器，需要在springmvc.xml配置文件中配置扫描器

`srpingmvc.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd">

    <context:component-scan base-package="org.zhkucst.controller"/>

</beans>
```

### 1.2 过滤器编码

服务器启动时自动初始化过滤器
在`web.xml`中加入

```xml
<filter>
    <filter-name>characterEncodingFilter</filter-name>

    <init-param>
        <param-name>encoding</param-name>
        <param-value>utf-8</param-value>
    </init-param>

    <init-param>
        <param-name>forceRequestEncoding</param-name>
        <param-value>true</param-value>
    </init-param>

    <init-param>
        <param-name>forceResponseEncoding</param-name>
        <param-value>true</param-value>
    </init-param>

</filter>

<filter-mapping>
    <filter-name>characterEncodingFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

### 1.3 DispatcherServlet执行源码

`springmvc`执行过程源代码分析

1. `tomcat`启动，创建容器的过程

​		通过`load-on-start`标签指定的`1`，创建`DispatcherServlet`对象，

​		`DispatcherServlet`的父类是继承`HttpServlet`的，它是一个`servlet`，在被创建是，会执行`init()`方法

​		在`init()`方法中

```java
//创建容器，读取配置文件
getServletContext().getAttribute(key, ctx);
//把容器对象放入到ServletContext()
getServletContext().setAttribute(key, ctx);	
```

上面创建容器作用：创建`@controller`注解所在的类的对象，创建`MyController`对象，

这个对象放入到`springmvc`的容器，容器时`map`，类似`map.put("myController", MyController对象)`

2. 请求的处理过程

执行`servlet`的`service()`

```java
protected void service(HttpServletRequest request, HttpServletResponse response)
    protected void doService(HttpServletRequest request, HttpServletResponse response)
    DispatcherServlet.doDispatch(request, response){
    //调用MyController的.doSome()方法
}
```

## 2. 注解式开发

### 2.1 文件上传

和`Servlet`方式的本质一样，都是通过导入两个jar包（`commons-io.jar`和`commons-fileupload.jar`）

具体步骤：（直接使用`CommonsMultipartResolver`实现上传）

- jar包

`commons-fileupload.jar`、`commons-io.jar`

- 配置`CommonsMultipartResolver`

将其加入`SpringMVC`容器中

```xml
<!-- springMVC容器初始化时，会自动寻找一个Id="multipartResolver"的bean，并将其加入MVC容器 -->
<bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
    <!-- 默认编码 -->
    <property name="defaultEncoding" value="utf-8"></property>
    <!-- 上传单个文件最大值 -->
    <property name="maxUploadSize" value="102400"></property>
</bean>
```

- 控制器：

```java
@RequestMapping(value = "testUpload")
public String testUpload(String desc, MultipartFile file) {
    System.out.println("文件描述信息：" + desc);

    try {
        InputStream input = file.getInputStream();
        String fileName = file.getOriginalFilename();

        OutputStream out = new FileOutputStream("d:\\" + fileName);

        byte[] bs = new byte[1024];
        int len = -1;
        // 循环从 输入流 input 中读取1024个字节给 bs
        while((len = input.read(bs)) != -1){
            // 每次读取到的数据 bs 写入到输入流
            out.write(bs, 0, len);
        }
        out.close();
        input.close();
    } catch (IOException e) {
        // TODO Auto-generated catch block
        e.printStackTrace();
    }

    //将file上传到服务器中的某一个硬盘中
    System.out.println("上传成功");
    return "success";
}
```

- 表单：

```html
<form action="handler/testUpload" method="post" enctype="multipart/form-data">
    <input type="file" name="file"/><br/>
    <input name="desc" type="text" /><br/>
    <input type="submit" value="上传" /><br/>
</form>
```

### 2.2 配置视图解析器

在`springmvc.xml`中增加`bean`来实例并自动配置`jsp`文件的前缀和后缀

框架会使用视图解析器的前缀 + 逻辑名称（文件名） + 后缀 组成完成路径	

- 视图路径配置

```xml
<bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
    <!--前缀 视图文件的路径-->
    <property name="prefix" value="/WEB-INF/view/"></property>
    <!-- 后缀 视图文件的扩展名 -->
    <property name="Suffix" value=".jsp"></property>
</bean>
```

- 控制器实现转发视图

```java
@RequestMapping(value = "/some.do")
public ModelAndView doSome(){
    ModelAndView mv = new ModelAndView();

    mv.addObject("msg", "欢迎使用springmvc做web开发");
    mv.addObject("fun", "执行的是doSome方法");

    mv.setViewName("show");

    return mv;
}
```

控制器不变，跟之前的写法一样

- 控制器可以实现多方法，多请求

```java
@RequestMapping(value = {"/some.do", "first.do"})
public ModelAndView doSome(){
    ModelAndView mv = new ModelAndView();

    mv.addObject("msg", "欢迎使用springmvc做web开发");
    mv.addObject("fun", "执行的是doSome方法");

    mv.setViewName("show");

    return mv;
}

@RequestMapping(value = {"/other.do", "second.do"})
public ModelAndView doOther(){
    ModelAndView mv = new ModelAndView();

    mv.addObject("msg", "欢迎使用springmvc做web开发");
    mv.addObject("fun", "执行的是doOther方法");

    mv.setViewName("other");

    return mv;
}
```

类也可以注解公共路径

```java
@Controller
@RequestMapping("/test")
public class MyController {
```

- 常见视图解析器

`InternalResourceView`、`InternalResourceViewResolver`

```java
//springMVC解析jsp时，会默认使用InternalResourceView, 如果发现JSP中包含了jstl语言，则自动转为JstlView
public class JstlView exteds InternalResourceView {
}
```

`JstlViw`	可以解析`jstl` 从而实现国际化操作

### 2.3  国际化

国际化：针对不同地区、不同国家，进行不同的显示

中国：（大陆、香港） 欢迎

美国：welcome

具体实现国际化的步骤：

- 创建资源文件

`基名_语言_地区.properties`

`基名_语言.properties`

- 配置`springmvc.xml`文件

```xml
<bean id="messageSource" class="org.springframework.context.support.ResourceBundleMessageSource">
    <property name="basename" value="i18n"></property>
</bean>
```

- 通过`jstl`使用国际化

需要两个依赖包：`jstl.jar`和 `standar.jar`

```xml
<%@ taglib uri="http://java.sun.com/jsp/jstl/fmt" prefix="fmt" %> 
```

index.jsp

```jsp
<a href="handler/testI18n">test i18n</a>
```

success.jsp

```jsp
<fmt:message key="resource.welcome"></fmt:message>
<fmt:message key="resource.exist"></fmt:message>
```

处理器：

```java
@RequestMapping(value = "testI18n")
public String testModelAttribute() {
    return "success";
}
```

不能直接访问`success`来国际化，必须通过服务器响应才能实现

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/c2e997d4-eaab-452b-8e64-510cb29f7ab7.png)

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/5753ce0a-2bfb-41d4-ac4e-2c570f05719d.png)

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/9acd994c-aaf0-4385-adcf-31ff89bdc937.png)

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/e96fbba8-7554-4d3f-8b57-08b3cf222a22.png)

`i18n_en_US.propeities`

```xml
resource.welcome=WELCOME
resource.exist=EXIST
```

`i18n_zh_CN.properties`

```xml
resource.welcome=\u4F60\u597D
resource.exist=\u9000\u51FA
```

`i18n.properties`

一般上面的找不到会自动找父类

### 2.4 处理器接收参数

- 对象接收参数

```java
@RequestMapping(value = "receiveobject.do")
public ModelAndView receiveParam(Student myStudent){
    ModelAndView mv = new ModelAndView();

    mv.addObject("myname", myStudent.getName());
    mv.addObject("myage",  myStudent.getAge());
    mv.addObject("mystudent",  myStudent);

    System.out.println(myStudent);

    mv.setViewName("show");

    return mv;
}
```

- 形参请求

请求中所携带的请求参数

1. `get`方式--通过`url`拼接，比如`localhost:8080/MyProject?name=zs`

2. 或者通过form标签输入input然后传回

**注意：**

请求参数必须要与处理器中的形参一致

```jsp
<form action="receiveparam.do">
    姓名：<input type="text" name="name"> <br/>
    年龄：<input type="text" name="age"> <br/>
    <input type="submit" value="other的post请求">
</form>
```

```java
@RequestMapping(value = "receiveparam.do")
public ModelAndView doSome(String name, int age){
}
```

额外：

`get`方法的编码默认是`utf-8`，如用`get`出现中文乱码，则使用的`tomcat`可能是`tomcat7`，使用`8`即以上，默认是`utf-8`，也可以到`apache-tomcat-8.5.63\conf`中的`server.xml`的

```xml
<Connector URIEncoding="UTF-8" connectionTimeout="20000" port="8888" protocol="HTTP/1.1" redirectPort="8443"/>
增加属性：URIEncoding="UTF-8"
```

### 2.5 静态处理

#### 2.5.1 第一种静态处理

交由 `tomcat` 本身的默认处理

在路径下：`apache-tomcat-9.0.43\conf`

```xml
<!-- The default servlet for all web applications, that serves static -->
 <!-- resources. It processes all requests that are not mapped to other --> 
<!-- servlets with servlet mappings (defined either here or in your own -->
 <!-- web.xml file).-->
```

1.访问静态资源

2.访问没有被请求映射的`servlet`

满足以上任意一点的，访问会默认交由`default`的`servlet`处理

```xml
<servlet>
    <servlet-name>default</servlet-name>
    <servlet-class>org.apache.catalina.servlets.DefaultServlet</servlet-class>
    <init-param>
        <param-name>debug</param-name>
        <param-value>0</param-value>
    </init-param>
    <init-param>
        <param-name>listings</param-name>
        <param-value>false</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
```

接着在`web.xml`使用`servlet-name`来指定默认的处理器和对要默认处理的路径

```xml
<servlet-mapping>
    <servlet-name>default</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```

#### 2.5.2 第二种静态处理

如果将所有页面交由中央调度器处理，即，`web.xml`中加入

```xml
<servlet>
    <servlet-name>springmvc</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>

    <init-param>
        <!-- springmvc配置文件位置的属性 -->
        <param-name>contextConfigLocation</param-name>
        <!-- 指定自定义文件的位置 -->
        <param-value>classpath:springmvc.xml</param-value>
    </init-param>

    <!-- 自定义springmvc配置文件读取的位置 -->
    <load-on-startup>1</load-on-startup>
</servlet>

<servlet-mapping>
    <servlet-name>springmvc</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
<!--表示默认资源全给DispatcherServlet处理了，而DispatcherServlet本身是没有对静态资源的处理能力-->
```

会由于中央调度器没有对静态页面的默认处理器

**springMvc可以处理没有被请求映射的处理器**

导致页面转发异常，出现404，资源访问不了或不存在

需要在`springmvc.xml`（`dispatcherServlet.xml`）中添加以下配置

```xml
<mvc:default-servlet-handler/>
```

等同于在web.xml中配置：

```xml
<servlet-mapping>
    <servlet-name>default</servlet-name>
    <url-pattern>*.jpg</url-pattern>
</servlet-mapping>

<servlet-mapping>
    <servlet-name>default</servlet-name>
    <url-pattern>*.js</url-pattern>
</servlet-mapping>

<servlet-mapping>
    <servlet-name>default</servlet-name>
    <url-pattern>*.html</url-pattern>
</servlet-mapping>
```

**注意：**

对于`web.xml`、`spring.xml`配置文件的修改，需要对服务器进行热部署，即`redeloy`或者`restart`

这种方式会导致只能处理静态资源，而中央调度器本身是没有处理静态资源的能力，上面其实是转发给了tomcat处理的，所以第三种处理静态资源出现了

#### 2.5.3 第三种静态处理

```xml
<mvc:default-servlet-handler/>
```

- 如果没有加上

```xml
<mvc:annotation-driven/>
```

就只能处理静态资源，其他动态资源无法访问了，为了协调两者需要加入以上两个注解

- 上面第二种是转发给`tomcat`处理的，处理的还是`tomcat`，下面第二种处理静态资源是

`spring`专门用于处理静态资源访问请求的处理器`ResourceHttpRequestHandler`

好处是不用再交给`tomcat`处理了，完全交给`spring`

**需要spring3.0以上**

改写成

```xml
<mvc:resources mapping="/img/**" location="/img/"/>
<mvc:resources mapping="/js/**" location="/js/"/>
<mvc:resources mapping="/html/**" location="/html/"/>
<!--
  location 表示静态资源所在的目录, 目录不要使用/WEB-INF/及其子目录
  mapping 表示对该资源的请求 （mapping="/images/**"表示对 以/img/开始的请求，如
/img/beauty.jpg、/img/car.png）
-->
```

再改进： 

```xml
<mvc:resources mapping="/static/**" location="/static/"/>
```

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/78ddbc1f-6abe-46b4-81b7-7239861bcf01.png)

### 2.6 类型转换器

- Spring自带一些常见的类型转换器

`public String testDelete(@PathVariable("id") String id)` ,即可以接受`int` 类型数据id也可以接受`String` 类型数据`id`

- 自定义类型转换器

编写 自定义类型转换器的类（实现`Converter`）

```java
public class MyConverter implements Converter<String, Object>{
    @Override
    public Object convert(String source) {//source:2-zs
        String[] studentStrArr = source.split("-");
        Student student = new Student();
        student.setId(Integer.parseInt(studentStrArr[0]));
        student.setName(studentStrArr[1]);

        return student;
    }
}
```

控制器：

```java
@RequestMapping(value = "testConverter")
public String testConverter(@RequestParam(value = "studentInfo") Student student) {
    System.out.println(student.getId() + "," + student.getName());

    return "success";
}
```

其中`@RequestParam(value = "studentInfo")` 是触发转换器的桥梁

`@RequestParam(value = "studentInfo")`接收的数据 是前端传过来的：`2-zs`,但是需要将该数据复制给 修饰的 目的对象`Student`

因此`SpringMVC`可以发现接收的数据和 目标数据的不一致，并且这两种数据分别是`String`,`Student`,正好符合转化器

```java
public Student convert(String source) {//source:2-zs
```

从而触发

- 配置：将MyConverter加入到springmvc中

```xml
<!-- 自定义转换器 纳入SpringIOC容器 -->
<bean id="myConverter" class="org.zhkucst.converty.MyConverter"></bean>

<!-- 将myConverter再纳入SpringMVC提供的转换器Bean -->
<bean id="conversionService" class="org.springframework.context.support.ConversionServiceFactoryBean">
    <property name="converters">
        <set>
            <ref bean="myConverter"/>
        </set>
    </property>
</bean>

<!-- 将conversionService 注册到annotation-driven中 -->
<!-- 此配置是SpringMVC的基础配置，很多功能都需要通过该注解来协调 -->
<mvc:annotation-driven conversion-service="conversionService"/>
```

### 2.7 数据格式化

```java
SimpleDateFormat sdf = new SimpleDateFormat ("yyyy-MM hh:mm:ss");
```

`SpringMVC`提供了很多注解，方便我们数据格式化

实现步骤：

- 配置

```xml
<mvc:annotation-driven conversion-service="conversionService"/>

<!-- 配置数据格式化所依赖的bean -->
<bean id="conversionService" class="org.springframework.format.support.FormattingConversionServiceFactoryBean">	</bean>
```

- 通过注解使用

```java
public class Student {
    @NumberFormat(pattern = "###,#")//格式化数字
    private int id;
    private String name;

    @DateTimeFormat(pattern = "yyyy-MM-dd")//格式化日期
    private Date birthday;
```

处理器：

```java
@RequestMapping(value = "testDateFormat")
public String testDateFormat(Student student, BindingResult result) {

    System.out.println(student.getId() + "," + student.getName() + "," + student.getBirthday());

    if(result.getErrorCount() > 0) {
        for(FieldError error : result.getFieldErrors()) {
            System.out.println(error.getDefaultMessage());
        }
    }

    return "success";
}
```

`BindingResult` `result`传的参数相当于捕获异常，`try{}`

`catch(){}`,因此不会抛异常了

包中，数据格式化的`bean`类`FormattingConversionServiceFactoryBean`包含了转换器的`bean`类`ConversionServiceFactoryBean`

```xml
<!--既可以实现转换、也可以实现数据格式-->
<bean id="conversionService"
      class="org.springframework.format.support.FormattingConversionServiceFactoryBean">
```

```xml
<!--只能实现转换-->
<bean id="conversionService"
      class="org.springframework.context.support.ConversionServiceFactoryBean"></bean>
```

#### 2.7.1 错误信息

```java
public String testDateTimeFormat（Student student, BindingResult result, Map<String,Object> map）
```

需要验证的数据是 `Student`中的`birthday`, `SpringMVC`要求 如果校验失败 则将错误信息 自动放入 该对象之后 的参数中，

即`Student student, BindingResult result`之间 不能有其他参数

如果要将控制台的错误信息传到`jsp`中显示，则可以将 错误信息对象放入`request`域中然后在jsp中 从 `request`取出

#### 2.7.2数据校验

`JSR-303` 是`JAVA EE6` 中的一项子规范，叫做`Bean Validation`，`Hibernate Validator` `是 Bean Validation` 的参考实现，提供了 `JSR 303` 规范中所有内置 `constraint` 的实现，除此之外还有一些附加的 `constraint`。

数据校验

​	1.`JSR303`

​	2.`Hibernate Validator`

使用Hibernate Validator步骤：

- 导包

| classmate-1.5.1.jar                                      |
| -------------------------------------------------------- |
| hibernate-validator-5.4.1.Final.jar                      |
| hibernate-validator-annotation-processor-5.4.1.Final.jar |
| jboss-logging-3.3.0.Final.jar                            |
| validation-api-1.1.0.Final.jar                           |

- 配置

此时`mvc:annotation-driven`的作用，要实现`Hibernate Validator/JSR303`校验(或者其他各种校验) 必须实现`SpringMVC`提供的一个接口：

`ValidatorFactory`

`LocalValidatorFactoryBean`是`ValidatorFactory`的一个实现类

`<mvc:annotation-driven></mvc:annotation-driven>`会在`springmvc`容器中 自动加载一个`LocalValidatorFactoryBean`

所以配置只需要配置

```xml
<mvc:annotation-driven/>
```

- 直接使用注解

```java
public class Student {
    @NumberFormat(pattern = "###,#")
    private int id;
    private String name;

    @Past
    @DateTimeFormat(pattern = "yyyy-MM-dd")
    private Date birthday;

    @Email
    private String email;
```

要求：在属性上使用注解后，需要在控制器中的参数Student前使用注解@Valid，标明上是要有合法性的参数

```java
@RequestMapping(value = "testDateFormat")
public String testDateFormat(@Valid Student student, BindingResult result) {
```

### 2.8 路径

#### 2.8.1 绝对路径和相对路径

jsp请求路径：

```jsp
<form action="some.do" method="post">
    姓名：<input type="text" name="name"> <br/>
    年龄：<input type="text" name="age"> <br/>
    <input type="submit" value="other的post请求">
</form>
```

- **不以 “/" 开头**

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/d48db25e-1732-42e6-808a-66ca8a8eed8f.png)

参考的地址为 `http://localhost:8888/ch06-path/`

即 `http://localhost:8888/ch06-path/` + `some.do`

**隐患：**  

由于多次请求导致路径发生变化，从而请求不到资源，原因：使用了相对路径

常见例子：

1. index.jsp访问 `user/some.do`

2. `http://localhost:8080/project/user/some.do` 中返回到`index.jsp`

3. `http://localhost:8080/project/user/some.do` (是因为是由处理器发起了转发，因而地址没有改变)

4. 再次访问`some.do`就会出现 `http://localhost:8080/project/user/user/some.do`

**解决方法：**

1. 使用base标签

```jsp
<base href="http://localhost:8888/${pageContext.request.ContextPath}/">
```

则`form`表单或`a`标签每次请求的基地址都是以`http://localhost:8888/${pageContext.request.ContextPath}/`开头 + `some.do`

2. 更加动态、灵活的方式

```jsp
<%
String basePath = request.getScheme() + "://" +
    request.getServerName() + ":" + request.getServerPort() +
    request.getContextPath() + "/";
%>

<base href="<%=basePath%>">
```

- **以 “/”开头**

参考的地址为 `http://localhost:8888/` 

即 `http://localhost:8888/` + `some.do`

处理方法：

```jsp
<form action="/ch06-path/some.do" method="post">
    姓名：<input type="text" name="name"> <br/>
    年龄：<input type="text" name="age"> <br/>
    <input type="submit" value="other的post请求">
</form>
```

不够灵活：

**解决方法：**

加上`${pageContext.request.ContextPath}`

`${pageContext.request.ContextPath}`表示项目根

即`http://localhost:8888/` + ch06_path 

即

```jsp
<form action="${pageContext.request.ContextPath}/some.do" method="post">
    姓名：<input type="text" name="name"> <br/>
    年龄：<input type="text" name="age"> <br/>
    <input type="submit" value="other的post请求">
</form>
```

**扩展：**

`ant`风格的请求路径：

`？`：单字符

\*   ：任意字符（0或多个）

`**`：任意目录

#### 2.8.2 路径访问问题

`web`项目中，`WEB-INF`是受保护的，通过url路径访问会出现`404`,不允许访问

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/39dfad24-cc57-4074-8d3c-062e0fba5820.png)

`WEB-INF`目录中的文件只能通过转发来实现

### 2.9 RequestMapping

#### 2.9.1 @RequestParam注解

如果表单的传值和控制器中的形参变量不一致，则需要通过在变量名前加上注`RequestParam`

表单提交：

```jsp
<form action="receiveparam.do" method="post">
    姓名：<input type="text" name="rname"> <br/>
    年龄：<input type="text" name="rage"> <br/>
    <input type="submit" value="other的post请求">
</form>
```

控制器：

添加注解：`@RequestParam(value = "rname")`

​				`@RequestParam(value = "rage")`

表示请求中的参数，例如`url`：`localhost:8080/MyProject?rname=&rage=`

 `value`值与表单的传值一致

```java
@RequestMapping(value = "receiveparam.do")
public ModelAndView receiveParam(@RequestParam(value = "rname") String name,
                                 @RequestParam(value = "rage") Integer age){
    ModelAndView mv = new ModelAndView();

    mv.addObject("myname", name);
    mv.addObject("myage",  age);

    mv.setViewName("show");
    return mv;
}
```

**注意：**  不能直接访问`servlet`地址，必须先通过表单传参或地址栏传参

出错`400`

![img](file://C:\Users\fyp01\Documents\FocusNote\assets\5c42bac3-3766-450c-9753-45d194f5deae.png?t=1658047446525)

所以，需要在方法前再加另外一个注解表明是否表示请求中必须包含此参数

```java
@RequestParam(value = "rname", required = false) String name,
@RequestParam(value = "rage", required = false) Integer age)
```

默认`require`为`true`，即包含的参数是必须的

`localhost:8080/MyProject?rname=&rage=`

`false`则非必须，可以有，也可以没有

#### 2.9.2 @RequestMapping属性

- **传参数**

```java
@RequestMapping(value="welcome",method="RequestMethod.Post",
                params={"name=zs","age!=23","!height"})
```

说明：

1. `age`可以为空，不传值

2. `height`不能传值

- **请求头信息**

设置请求头信息：

```java
@RequestMapping(value="welcome",method="RequestMethod.Post",
                headers ={"Accept=text/html,application/xhtml....","...."}
```

获取请求头信息：

```java
@RequestMapping(value = "testRest/{id}", method = RequestMethod.GET)
public String welcome4(@PathVariable("id") Integer id, 
                       @RequestHeader("Accept-Language") String al) {
}
```

说明：需要支持或兼容该请求头的浏览器

- **路径请求参数**

表单：

```jsp
<a href=""handler/welcome/zs">路径传参</a>
```

后台接收：

```java
@RequestMapping(value="welcome/{name}",method="RequestMethod.Post",
                public String welcome(@PathVariable("name") String name) {
                }                 
```

- **Cookie信息**

通过`mvc`获取`cookie`值（`JESSIONID`）

`@CookieValue`

(前置知识：服务端在接受客户端第一次请求时，会给客户端分配一个`session`(该`session`包含一个`sessionId`))

小结：

`SpringMVC`处理各种参数的流程、逻辑：

请求：前端发请求`a->@RequestMapping("a")`;

处理请求中的参数：

​		`@RequestMapping("a")`

​		`public String aa("@Xxx注解(xyz)" xyz)`

#### 2.9.3 指定请求方式Method属性

在方法名上标明注解

```java
@RequestMapping(value = {"/some.do", "/first.do"}, method = RequestMethod.GET)
public ModelAndView doSome(){
//如果不指定请求方式，则没有限制
```

`form`标签指定`get`方法，控制器不能指定`post`方法

否则会出现`405`错误

```jsp
<form action="test/some.do" method="post">
    <input type="submit" value="other的post请求">
</form>
```

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/c1c2d500-b542-43d3-83bd-aebb4be2501e.png)

### 2.10 处理模型数据

- **数据模型**：`ModelAndView、Model、ModelMap、Map`

```java
@RequestMapping(value = "testMap")
public String testMap(Map<String,Object> map) {

    Student student = new Student();
    student.setId(3);
    student.setName("zs");

    map.put("student", student);

    return "success";
}

@RequestMapping(value = "testModelMap")
public String testModelMap(ModelMap modelMap) {

    Student student = new Student();
    student.setId(3);
    student.setName("zs");

    modelMap.put("student", student);

    return "success";

}

@RequestMapping(value = "testModel")
public String testModel(Model model) {

    Student student = new Student();
    student.setId(3);
    student.setName("zs");

    model.addAttribute("student", student);

    return "success";
}

@RequestMapping(value = "testModelAndView")
public ModelAndView testModelAndView() {
    ModelAndView modelAndView = new ModelAndView();

    Student student = new Student();
    student.setId(3);
    student.setName("zs");

    modelAndView.addObject("student", student);
    modelAndView.setViewName("success");

    return modelAndView;

}
```

- **将数据放入`request`后，也放入`session`**

类名上添加

1. `@SessionAttributes("student4")` : 静态写法

参数名与`requesst`域的`key`值相同，表示指定哪一个k-v对

2. `@SessionAttributes(types = Student.class)` ： 动态代理写法

具体是以调用哪个方法中的`student`来动态同步到`session`

- **@ModelAttribute**

1. 有该注解的方法在会在每次请求前先执行

2. 并且该方法的参数`map.put()`可以将对象 放入即将查询的参数中

必须满足的约定：

1. `map.put(k,v)`其中的`k`必须是即将查询方法参数的 首字母小写`testModelAtribute(Student xxx)` , 即`student`;

2. 如果不一致,需要通过`@ModelAttribute`声明。

​	 `map.put("stu",student)`

`testModelAtribute( @ModelAttribute("stu"） Student student )`

示例：

```jsp
<form action="handler/testModelAttribute">
    name:<input type="text" name="name"><br/>
    <input type="submit" value="testModelAttribute">
</form>
```

```java
/*先执行该方法*/  
@ModelAttribute
public void queryStudents(Map<String, Object> map) {
    Student student = new Student();
    student.setId(5);
    student.setName("ww");

    System.out.println(student.getId()+"," + student.getName());


    map.put("student", student);
}

/*执行完后，该注解会被标记为通知，会执行框架中的另外的方法（method,map,student,handler）
      目的：把map中包含的student传给请求的方法体参数中,并通过前端的传值，覆盖student
      原理：面向切面，即AOP 
      method:    public String testModelAttribute(Student student)


      map:       map.put("student", student);


      student:   Student student = new Student();
            	 student.setId(5);
        		 student.setName("ww");


      handler:   @Controller
@RequestMapping(value = "handler")
public class SpringMVCHandler {

    */

/*前端请求处理的方法，先调用有注解@ModelAttribute的方法*/
@RequestMapping(value = "testModelAttribute")
public String testModelAttribute(Student student) {
    System.out.println(student.getId()+"," + student.getName());

    return "success";
}
```

### 2.11 返回值

#### 2.11.1 返回 `ModelAndView`、`String`

- **ModeAndView**

前面已经演示过：返回数据 + 视图

- **String**，返回String（视图）

可分为逻辑视图和完整视图

​	（1）逻辑视图需要配置视图解析器

```java
@RequestMapping(value = "returnString-view.do")
public String doReturnView(HttpServletRequest request, String name, int age){
    request.setAttribute("myname", name);
    request.setAttribute("myage", age);

    return "show";
}
```

​	（2）完整视图不能配置视图解析器，根目录是`WebContent`或者`webapp`

```java
    //处理器方法返回String, 表示完整路径，不能配置视图解析器
    @RequestMapping(value = "returnString-view2.do")
    public String doReturnView2(HttpServletRequest request, String name, int age){
        request.setAttribute("myname", name);
        request.setAttribute("myage", age);

        return "/WEB-INF/view/show.jsp";
    }
```

#### 2.11.2 返回 `void`、`Object`

- **void** **用于处理ajax**

需要使用`ajax`，导入`jquery-3.5.1.js` 以及 `pom`中加入依赖包（`jackson-databind` 和`jackson-core`）

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.9.0</version>
</dependency>

<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-core</artifactId>
    <version>2.9.0</version>
</dependency>
```

前端：

```javascript
<script type="text/javascript" src="js/jquery-3.5.1.js"></script>
<script type="text/javascript">

    function button_data(){
    $.ajax({
        url:"returnVoidAjax.do",
        data:{
            name:"张三",
            age:"21"
        },
        type:"post",
        dataType:"json",
        success:function(msg){
            alert(msg.name + "," + msg.age);
        }
    });
};

</script>
```

```javascript
dataType:"json",//是指定控制返回的类型，如果没有它，jquery默认也是当成json处理
```

但处理器中`response`没有设置响应头，则数据无法传到前端

```jsp
<button id="btn" onclick="button_data()">发起ajax请求</button>
```

控制器：

```java
@RequestMapping(value = "returnVoidAjax.do")
public void returnVoidAjax(HttpServletResponse response, String name, Integer age) throws IOException {
    Student student = new Student();
    student.setName(name);
    student.setAge(age);

    String json = "";
    ObjectMapper om = new ObjectMapper();
    json = om.writeValueAsString(student);

    //设置响应头，设备响应编码，不设置会乱码
    response.setContentType("application/json;charset=utf-8");
    PrintWriter pw =  response.getWriter();
    pw.print(json);

    pw.close();

}
```

响应头：增加

```java
//设置响应头，设备响应编码，不设置会乱码
response.setContentType("application/json;charset=utf-8");
```

后显示：

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/156d3838-2dec-45aa-8360-70afffcc0998.png)

`json`格式字符串转成`json`对象

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/7b15803c-f2cb-497b-8fdd-b51d66f1aa94.png)

没有增加显示：

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/57d3ec45-bc61-46b7-8f8c-c232e751a46d.png)

导致的乱码：

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/0bb3fc98-4b4c-422f-9d24-9673981e0fc8.png)

虽然前端显示出的数据正常，但`F12`控制台中`response`的`json`格式的字符串和`preview`的`json`对象

`response`从服务器返回的是`json`格式的字符串{name:"张三", age: 21}

因为`jquery`可以把`json`格式字符串转换成`json`对象,并赋值给`success`中的形参`msg`

但是`request`的请求头是默认有的：

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/afddc5cf-4168-473c-8546-c09f07c27542.png)



- **`Object`**

例如`String`, `Integer`, `Map`, `List`, `Student`等等都是对象

对象有属性， 属性就是数据，所以返回`Object`表示数据，和视图无关

可以使用对象表示的数据，响应`ajax`请求。

```xml
<!--加入<mvc:annotation-driven> 注解驱动-->
<mvc:annotation-driven></mvc:annotation-driven>
```

```java
@RequestMapping(value = "returnObjectAjax.do")
@ResponseBody
public Student returnStudentAjax(String name, Integer age){
    Student student = new Student();
    student.setName(name);
    student.setAge(age);

    return student;
}
```

```js
function button_data(){
    $.ajax({
        url:"returnObjectAjax.do",
        data:{
            name:"张三同学",
            age:"21"
        },
        type:"post",
        dataType:"json",
        success:function(resp){
            alert(msg.name + "," + msg.age);
        }
    });
};
```

#### 2.11.3 返回 `List<Object>`

与`Object`类似

控制器：

```java
@RequestMapping(value = "returnObjectArrayAjax.do")
@ResponseBody
public List<Student> returnStudentArrayAjax(String name, Integer age){


    List<Student> list = new ArrayList<>();


    Student student = new Student();
    student.setName("张三");
    student.setAge(21);
    list.add(student);


    student = new Student();
    student.setName("李四");
    student.setAge(24);
    list.add(student);


    return list;
}
```

前端：

```js
function button_data(){
    $.ajax({
        url:"returnObjectArrayAjax.do",
        data:{
            name:"张三同学",
            age:"21"
        },
        type:"post",
        dataType:"json",
        success:function(resp){
            $.each(resp,function (i, n){
                alert(n.name + " , " + n.age + "/" )
            })


        }
    });
};
```

#### 2.11.4 返回 `String` 文本数据

```java
public String returnString(HttpServletResponse response){
    return "返回String类型";
}
```

```js
$.ajax({
    //url:"returnVoidAjax.do",
    // url:"returnObjectAjax.do",
    //url:"returnObjectArrayAjax.do",
    url:"returnString.do",
    data:{
        name:"张三同学",
        age:"21"
    },
    type:"post",
    /**
    * 这里不能用上json了，因为文本数据转换不了json对象
    * 如果文本数据是 "{name:"zs",age:"21"}"
    * 则可以转换
    */
    dataType:"text",
    success:function(resp){
        alert(resp);
    }
});
```

#### 2.11.5 json 数据手动处理

需要`jackson-core`的两个包

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.9.0</version>
</dependency>

<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-core</artifactId>
    <version>2.9.0</version>
</dependency>
```

```java
@RequestMapping(value = "returnVoidAjax.do")
public void returnVoidAjax(HttpServletResponse response, String name, Integer age) throws IOException {
    Student student = new Student();
    student.setName(name);
    student.setAge(age);

    String json = "";
    ObjectMapper om = new ObjectMapper();
    json = om.writeValueAsString(student);

    response.setContentType("application/json;charset=utf-8");
    PrintWriter pw =  response.getWriter();
    pw.print(json);

    pw.close();
}
```

#### 2.11.6 @ResponseBody的 json 数据处理原理

- **实现步骤**

1. 加入处理`json`的工具库的依赖

`springmvc`默认使用的`jackson`

2. 在springmvc配置文件之间加入`<mvc:annotation-driven>` 注解驱动

```java
json = om.writeValueAsString(student);
```

3. 在处理器的方法的上面加入`@ResponseBody`注解

```java
response.setContexType("application/json;charset=utf-8");
PrintWriter pw = response.getWriter();
pw.println(json);
```

`springmvc`处理器方法的返回`Object`,可以转为`json`输出到浏览器，响应`ajax`的内部原理

1.`<mvc:annotation-driven>`注解驱动

注解驱动实现的功能是完成`java`对象到`json`、`xml`、`text`、`二进制`等数据格式的转换的7个实现类对象，包括MappingJackson2HttpMessageConvertor(使用`jackson`工具库中的`ObjectMapper`实现`java`对象转为`json`对象 )

`HttpMessageConveter`接口：消息转换器

功能：定义了`java`转为`json`，`xml`等数据格式的方法。这个接口有很多实现类。

这些实现类完成`java`对象到`json`、`java`对象到`xml`、`java`对象到二进制数据的转换

下面有两个方法是控制器类把结果输出给浏览器时使用的：

```java
boolean canWrite(Class<?> var1, @Nullable MediaType var2);
void write(T var1, @Nullable MediaType var2, HttpOutputMessage var3)
```

```java
//例如处理器方法
@RequestMapping(value = "returnObject.do")
public Student returnVoidAjax(HttpServletResponse response, String name, Integer age) throws IOException {
    Student student = new Student();
    student.setName(name);
    student.setAge(age);

    return student;
}
```

```ruby
1)canWrite:作用检查处理器方法的返回值，能不能转为var2表示的数据格式。
检查student(name, age)能不能转为var2表示的数据格式，如果检查能转成json，返回true
MediaType：表示数据格式的，例如json、xml等等
StringHttpMessageConverter
MappingJackson2HttpMessageConverter
```

```ruby
2)write: 把处理器方法的返回值对象，调用jackson中的ObjectMapper转为json字符串
json = om.writeValueAsString(student);
```

未加入注解驱动时的状态：

```xml
<!--加入<mvc:annotation-driven> 注解驱动-->
<mvc:annotation-driven></mvc:annotation-driven>
```

自动实例化`MessageConverter`4个实现类

```ruby
org.springframework.http.converter.ByteArrayHttpMessageConverter, 
org.springframework.http.converter.StringHttpMessageConverter, org.springframework.http.converter.xml.SourceHttpMessageConverter, org.springframework.http.converter.support.AllEncompassingFormHttpMessageConverter
```

加入注解驱动时的状态：

自动实例化`MessageConverter`7个实现类

```ruby
org.springframework.http.converter.ByteArrayHttpMessageConverter, 
org.springframework.http.converter.StringHttpMessageConverter, **org.springframework.http.converter.ResourceHttpMessageConverter,** 
**org.springframework.http.converter.ResourceRegionHttpMessageConverter,** org.springframework.http.converter.xml.SourceHttpMessageConverter, org.springframework.http.converter.support.AllEncompassingFormHttpMessageConverter, **org.springframework.http.converter.json.MappingJackson2HttpMessageConverter**
```

#### 2.11.7 乱码问题

通过请求过来mapping设置响应头

```java
@RequestMapping(value = "returnString.do", produces = "text/plain;charset=utf-8")
```

设置前（默认）

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/58ab19a6-3a96-4a02-8ee1-9018851cf89a.png)

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/c4b0498f-9bcb-430b-80b3-6339b1b95f70.png)

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/526aa3e2-358a-4c43-a1b9-225aeaeeae16.png)

设置后：

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/044aa562-ed78-4acf-9e91-3f8fe8f89090.png)

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/452406ff-a9db-45da-8876-1ca3a364fffa.png)

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/8a867b30-14b7-4d4c-9244-b356abdef32f.png)

### 2.12 RESTFul 风格

`REST`风格：软件编程风格

`Springmvc`:

`GET` : 查

`POST`: 增

`DELETE` : 删

`PUT` : 改

普通浏览器 	只支持`get / post`  方式请求， 其他请求方式 如 `delete / put` 需借助过滤器

**过滤器的约定：**

- 过滤的是`input`标签；

- 标签中类型为隐藏域，而且name必须为"`_method`"

- 请求方式为`post`

- 根据其`value`的值来判断

如果`value`为`delete` 则将请求改为`delete`，如果为`put`，改为`put`

- 不指定方法请求方式，默认为`get`

- 使用`restful`风格时，不建议使用其他方式传参，比如"`?id=1`"，不建议混合使用`restful`和一般方式，即使可以使用

`ant`风格的请求路径：

`？`：单字符

\*： 任意字符（0或多个）

`**`： 任意目录

#### 2.12.1 Get

```jsp
<form action="handler/testRest/1234" method="get">
    <input type="submit" value="get">
</form>
```

```java
@RequestMapping(value = "testRest/{id}", method = RequestMethod.GET)
public String welcome4(@PathVariable("id") Integer id) {
    System.out.println("get: 改" + id);

    return "success";
}
```

#### 2.12.2 Post

```jsp
<form action="handler/testPost/1234">
    <input type="hidden" name="_method" value="POST">
    <input type="submit" value="post">
</form>
```

```java
@RequestMapping(value = "testRest/{id}", method = RequestMethod.POST)
public String welcome1(@PathVariable("id") Integer id) {
    System.out.println("post: 增" + id);

    return "success";
}
```

#### 2.12.3 Delete

```jsp
<form action="handler/testDelete/1234">
    <input type="hidden" name="_method" value="DELETE">
    <input type="submit" value="post">
</form>
```

```java
@RequestMapping(value = "testRest/{id}", method = RequestMethod.DELETE)
public String welcome2(@PathVariable("id") Integer id) {
    System.out.println("delete: 删" + id);

    return "success";
}
```

#### 2.12.4 Put

```jsp
<form action="handler/testPut/1234">
		<input type="hidden" name="_method" value="PUT">
		<input type="submit" value="post">
</form>
```

```java
@RequestMapping(value = "testRest/{id}", method = RequestMethod.PUT)
public String welcome3(@PathVariable("id") Integer id) {
    System.out.println("put: 改" + id);

    return "success";
}
```

#### 2.12.5 声明隐藏域过滤器

```xml
<filter>
    <filter-name>hiddenHttpMethodFilter</filter-name>
    <filter-class>org.springframework.web.filter.HiddenHttpMethodFilter</filter-class>
</filter>
<filter-mapping>
    <filter-name>hiddenHttpMethodFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

版本：

1.`tomcat7`版本及以下，`tomcat`默认不支持`RESTful`风格的`DELETE`和`PUT`, 配置过滤器后`tomcat`就可以访问了

2.`tomcat8`版本即以上，配置过滤器后，执行表单后会跳转到`405`页面，说明高版本的`tomcat`是不支持`RESTful`风格

方法：在跳转成功的`jsp`页面，添加错误页面参数`isErrorPage=“true”`

不配置过滤器默认是支持`RESTful`风格，也就是不用自己配置过滤器

## 3. SSM 整合开发

### 3.1 Spring 容器和SpringMVC 容器

依赖包：

```xml
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>3.1.0</version>
    <scope>provided</scope>
</dependency>


<dependency>
    <groupId>javax.servlet.jsp</groupId>
    <artifactId>jsp-api</artifactId>
    <version>2.2.1-b03</version>
    <scope>provided</scope>
</dependency>


<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webmvc</artifactId>
    <version>5.2.5.RELEASE</version>
</dependency>


<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-tx</artifactId>
    <version>5.2.5.RELEASE</version>
</dependency>


<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jdbc</artifactId>
    <version>5.2.5.RELEASE</version>
</dependency>


<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-core</artifactId>
    <version>2.9.0</version>
</dependency>


<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.9.0</version>
</dependency>






<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis-spring</artifactId>
    <version>1.3.1</version>
</dependency>


<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.5.1</version>
</dependency>


<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.9</version>
</dependency>


<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.1.12</version>
</dependency>
```

另加：用于注解`@Resource`

```xml
<dependency>  
    <groupId>javax.annotation</groupId>  
    <artifactId>jsr250-api</artifactId>  
    <version>1.0</version>  
</dependency>  
```

```xml
<!--build中添加以下，把java下的xml文件配置一起打包到target中-->
<build>
    <resources>
        <resource>
            <directory>src/main/java</directory>
            <includes>
                <include>**/*.xml</include>
            </includes>
            <filtering>false</filtering>
        </resource>
    </resources>
</build>
```

1. 如果xml文件写在与`dao`同包名下的`java`资源中，则需要以上`build`指定`xml`到打包到`target`，否则
   将报错`--Request processing failed; nested exception is org.apache.ibatis.binding.BindingException:`
   `mybatis和mapper绑定失败`

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/a6e4aa3b-4a29-4c94-af85-dab7549528a4.png)

2. 如果`xml`文件写在`resources`中，则只需新建同包名的`xml`即可

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/e300724d-9596-4e3a-943c-bce2ce002fc4.png)![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/242e90c9-73de-450a-9d09-c0dc8d4ada00.png)

### 3.2 整合内置 tomcat

为什么要整合内置`tomcat`?

内置`tomcat`可以在别人部署项目时不用纠结到使用`tomcat`版本的问题

即可以在没有`tomcat`环境下使用

步骤：

- 1.导入依赖

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.tomcat.maven</groupId>
            <artifactId>tomcat7-maven-plugin</artifactId>
            <configuration>
                <port>8080</port>
                <server>tomcat7</server>
                <path>/</path>
                <useBodyEncodingForURI>true</useBodyEncodingForURI>
                <uriEncoding>UTF-8</uriEncoding>
            </configuration>
        </plugin>
    </plugins>
</build>
```

- 手动配置`maven`启动内置`tomcat`

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/630ea270-274b-4737-8e30-3c4defcebe02.png)

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/094fd1bd-ce43-45bc-8474-65ed163cfdde.png)	

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/ab7800b2-3cc5-4bfa-afbb-c22cfec53697.png)

最后直接运行即可

### 3.3 xml 配置

**1.dispatcherServlet.xml : 中央调度器**

（1）声明注解`@controller`所在包的扫描器；

（2）内部资源视图解析器；

（3）声明`mvc`注解驱动；

**2.applicationContext.xml : springIOC容器**

（1）声明`db.properties`所在位置；

（2）声明数据源；

（3）声明`SqlSessionFactoryBean`,创建`SqlSessionFactory`；

（4）声明`mapper`所在包扫描器，自动创建`dao`对象；

（5）声明注解`@Service`所在包的扫描器

**3.mybatiis.xml : mybatis容器**

（1）声明别名

（2）挂载所有`mapper`配置文件，配置文件的路径默认在`resources`下（pom没有声明`java`中对`xml`文件的打包）

**4.mapper.xml : mapper容器**

（1）配置`namespace`,用于绑定`Dao`接口的，即面向接口编程

（2）增删查改标签

### 3.4 Ajax 请求

```js
<script type="text/javascript" src="js/jquery-3.5.1.js"></script>
<script type="text/javascript">


    $(function(){
    loadStudentData();


    $("#btnLoader").click(function (){
        loadStudentData();
    })
})


function loadStudentData(){
    $.ajax({
        url:"student/queryStudents.do",
        type:"post",
        dataType:"json",
        success:function (data){
            $("#info").html("");


            $.each(data, function(i,n){
                $("#info").append("<tr>")
                    .append("<td>" + n.id + "</td>")
                    .append("<td>" + n.name + "</td>")
                    .append("<td>" + n.age + "</td>")
                    .append("</tr>")
            });
        }
    });
}


</script>
```

### 3.5 三层整合

#### 3.5.1 Controller

```java
@Controller
@RequestMapping(value = "/student")
public class StudentController {
    @Resource
    private IStudentService studentService;


    //注册学生
    @RequestMapping(value = "/addStudent.do")
    public ModelAndView addStudent(Student student){
        ModelAndView mv = new ModelAndView();
        String tips = "注册失败";


        int nums = studentService.addStudent(student);
        if(nums > 0){
            //注册成功
            tips = "学生【" + student.getName() + "】注册成功";
        }
        //添加数据
        mv.addObject("tips", tips);
        //指定结果页面
        mv.setViewName("result");
        return mv;
    }


    @RequestMapping(value = "/queryStudents.do")
    @ResponseBody
    public List<Student> queryStudents(){


        List<Student> students = studentService.findStudents();
        System.out.println(students);
        return students;
    }
}
```

#### 3.5.2 Service

```java
@Controller
@RequestMapping(value = "/student")
public class StudentController {
    @Resource
    private IStudentService studentService;


    //注册学生
    @RequestMapping(value = "/addStudent.do")
    public ModelAndView addStudent(Student student){
        ModelAndView mv = new ModelAndView();
        String tips = "注册失败";


        int nums = studentService.addStudent(student);
        if(nums > 0){
            //注册成功
            tips = "学生【" + student.getName() + "】注册成功";
        }
        //添加数据
        mv.addObject("tips", tips);
        //指定结果页面
        mv.setViewName("result");
        return mv;
    }


    @RequestMapping(value = "/queryStudents.do")
    @ResponseBody
    public List<Student> queryStudents(){


        List<Student> students = studentService.findStudents();
        System.out.println(students);
        return students;
    }
}
```

#### 3.5.3 Dao

```java
public interface StudentDao {

    int insertStudent(Student student);

    List<Student> selectStudents();

}
```

#### 3.5.4 xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<!-- namespace :该mapper.xml的 唯一标识
用于绑定Dao接口的，即面向接口编程
-->
<mapper namespace="org.zhkucst.dao.StudentDao">
    <!-- 后续通过 namespace.id -->
    <!-- parameterType:输入参数的类型
    resultType:查询返回值的类型，返回类型 -->

    <select id="selectStudents" resultType="student">
        select id, name, age from student
    </select>

    <insert id="insertStudent" parameterType="student">
        insert into student(name,age) values(#{name},#{age})
    </insert>


</mapper>
```

#### 3.5.5前端

index.jsp

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%
String basePath = request.getScheme() + "://" +
    request.getServerName() + ":" + request.getServerPort() +
    request.getContextPath() + "/";
%>
<html>
    <head>
        <base href="<%=basePath%>/">
        <title>功能入口</title>
    </head>
    <body>


        <div align="center">
            <p>SSM整合</p>
            <img src="imges/ssm.jpg"/>


            <table>
                <tr>
                    <td><a href="addStudent.jsp">注册学生</a></td>
                </tr>
                <tr>
                    <td><a href="listStudent.jsp">浏览学生</a></td>
                </tr>
            </table>


        </div>
    </body>
</html>
```

addStudent.jsp

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%
String basePath = request.getScheme() + "://" +
    request.getServerName() + ":" + request.getServerPort() +
    request.getContextPath() + "/";
%>
<html>
    <head>
        <base href="<%=basePath%>/">
        <title>Title</title>
    </head>
    <body>
        <div align="center">
            <form action="student/addStudent.do" method="post">
                <table>
                    <tr>
                        <td>姓名</td>
                        <td><input type="text" name="name"></td>
                    </tr>
                    <tr>
                        <td>年龄</td>
                        <td><input type="text" name="age"></td>
                    </tr>
                    <tr>
                        <td>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</td>
                        <td><input type="submit" value="注册"></td>
                    </tr>
                </table>
            </form>
        </div>

    </body>
</html>
```

listStudent.jsp

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%
String basePath = request.getScheme() + "://" +
    request.getServerName() + ":" + request.getServerPort() +
    request.getContextPath() + "/";
%>
<html>
    <head>
        <base href="<%=basePath%>/">
        <script type="text/javascript" src="js/jquery-3.5.1.js"></script>
        <script type="text/javascript">


            $(function(){
                loadStudentData();


                $("#btnLoader").click(function (){
                    loadStudentData();
                })
            })


            function loadStudentData(){
                $.ajax({
                    url:"student/queryStudents.do",
                    type:"post",
                    dataType:"json",
                    success:function (data){
                        $("#info").html("");


                        $.each(data, function(i,n){
                            $("#info").append("<tr>")
                                .append("<td>" + n.id + "</td>")
                                .append("<td>" + n.name + "</td>")
                                .append("<td>" + n.age + "</td>")
                                .append("</tr>")
                        });
                    }
                });
            }


        </script>
        <title>Title</title>
    </head>
    <body>
        <div align="center">
            <table style="border:1px solid">
                <thead style="border:1px solid">
                    <tr>
                        <td>学号</td>
                        <td>姓名</td>
                        <td>年龄</td>
                    </tr>
                </thead>
                <tbody id="info" style="border:1px solid">


                </tbody>
            </table>
            <input id="btnLoader" type="button" value="查询数据">
        </div>


    </body>
</html>
```

result.jsp

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>Title</title>
</head>
<body>


    result.jps 结果页面 注册结果：${tips}


</body>
</html>
```

## 4. 转发和重定向

### 4.1 转发

转发和重定向是不受视图解析器影响的，就当没有视图解析器

`controller`

```java
@RequestMapping(value = "doForward.do")
public ModelAndView doForward(){
    ModelAndView mv = new ModelAndView();

    mv.addObject("msg", "欢迎使用springmvcweb开发");
    mv.addObject("fun", "执行的是dodoForward方法");
    mv.setViewName("forward:/WEB-INF/view/show.jsp");

    return mv;

}
```

总结：

1.转发可以访问`/WEB-INF/`目录下的视图以及其他文件

2.转发如果携带了数据（请求是实例化在服务器中，数据保存在该`request`中）

可以在其他视图中视图`EL`表达式`${requestScope.name}`获取属性值为“`name`”的值

3.也可以通过`request.getAttribute("name")`获取，两者等价

### 4.2 重定向

`controller`

```java
@RequestMapping(value = "doRedirect.do")
public ModelAndView doRedirect(String name, Integer age){
    ModelAndView mv = new ModelAndView();

    mv.addObject("myname", name);
    mv.addObject("myage", age);
    mv.setViewName("redirect:/hello.jsp");

    return mv;

}
```

2.视图：

```jsp
<body>
    <h3>hello.jsp</h3>
    <h3>myname数据1：${param.myname}</h3>
    <h3>myage数据2：${param.myage}</h3>
    <h3>myage数据2：<%=request.getParameter("myname")%></h3>
</body>
```

总结：

1.重定向是第二次请求转发，请求的数据会丢失，不会保留在服务器的`request`中，是拿了一个新的请求

2.重定向如果携带了数据，携带的数据不是保存在服务器中，而是在新的请求的形参中，例如：`http://locahost:8080/MyProject?name=zs`

3.重定向到新的视图中，可以通过`${param.name}`获取形参中的变量值，注意，只能获取4个基本类型+`String`，

其他类型会以地址`@***`显示

4.也可以通过request.getParameter("name")获取属性值为“`name`”的值

5.注意，通过视图传到控制器，再重定向的，视图中的参数值是获取不到的，两个请求不一样，第一个请求

结束后已经自动销毁

### 4.3 e额外知识

```js
${param.name} == request.getParameter("name")
${requestScope.name} == request.getAttribute("name")
${sessionScope.name} == session.getAttribute("name")
${applicationScope.name} == application.getAttribute("name")
```

`session`和`applicaton`如果要在同一个请求中，可以通过`request.getSession()`和 `request.getSession().getServletContext()`获取得到

如果在同一视图，直接`appliction.setAttribute()`、`session.setAttribute()`就可以了

## 5. 集中统一处理异常

### 5.1 理论

异常处理：

- **通过注解实现异常统一处理：@ExceptionHandler**

springmvc框架采用的是统一、全局的异常处理。

把controller中的所有异常处理都集中到一个地方。采用的是`aop`的思想，把业务逻辑和异常处理代码分开，解耦合。

使用两个注解

1. `@ExceptionHandler`

2. @ControllerAdvice

```ruby
异常处理步骤：
1.新建maven web项目
2.加入依赖
3.新建一个自定义异常类 MyUserException , 再定义它的子类NameException, AgeException
4.在controller抛出NameException , AgeException
5.创建一个普通类，作用全局异常处理类
    1)在类的上面加入@controllerAdvice
    2)在类中定义方法，方法的上面加入@ExceptionHandle
6.创建springmvc的配置文件
    1)组件扫描器，扫描@Controller注解
    2)组件扫描器，扫描@ControllerAdvice所在的包名
    3)声明注解驱动
```

结果图：

![img](file://C:\Users\fyp01\Documents\FocusNote\assets\43f7dfb0-ec87-47fe-8935-4a37f6d4ab57.png)



原理：

自定义普通类，通过继承`Exception`使它成为异常类，然后再用注解让`springmvc`对它管理：

发生异常，有`springmvc`来抛异常，不交给`jvm`虚拟机

**总结：**

- 如果有方法抛出一个异常，对该异常的处理有两种方法，则优先级：最短优先

- 如果一个方法用于处理异常，并且只处理当前类中的异常：`@ExceptionHandler`

- 如果一个方法用于处理异常，并且处理所有类中的异常，类前加`@ControllerAdvice`, 处理异常的方法前加`@ExceptionHandler`

**优先级：**

1. 使用`@ExceptionHandler`处理本`Controller`内部异常优先级最高；
2. 使用`@ExceptionHandler`+`@ControllerAdvice`处理外部`Controller`异常优先级第二； 
3. 自定义实现`HandlerExceptionResolver`接口的类优先级第三；
4. `spring-context.xml`中配置`SimpleMappingExceptionResolver`优先级第四；
5. `web.xml`配置`error-page`优先级第五；
6. 不做任何处理，会跳转到`tomcat`默认的异常页面；

- **自定义异常显示页面：@ResponseStatus**

`ResponseStatusExceptionResolver`:自定义异常显示页面 @ResponseStatus

自定义异常显示页面：`@ResponseStatus(value=HttpStatus.FORBIDDEN, reason="数组越界222")`

```java
@public class MyArrayIndexOutofBoundsException extends Exception{//自定义异常类

}
```

`@ResponseStatus`也可以标注在方法前： 

自定义异常类：

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/697fe1d6-bde9-47be-854b-b00c70066bc8.png)

```java
@ResponseStatus(value = HttpStatus.FORBIDDEN, reason = "数组越界222")
public class MyArrayIndexOutofBoundException extends Exception{
	
}
```



![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/d565c99b-4ef6-4ea2-9435-2cd7633e51e4.png)



- **异常处理的实现类：DefaultHandleExceptionResolver**

`DefaultHandleExceptionResolver:SpringMVC`一些常见异常的基础上（`300,500,405`），新增一些异常，例如：

```ruby
 * @since 3.0
 * @see org.springframework.web.servlet.mvc.method.annotation.ResponseEntityExceptionHandler
 * @see #handleNoSuchRequestHandlingMethod
 * @see #handleHttpRequestMethodNotSupported ： 如果请求限制为post，实际请求为get，则会触发此异常
 * @see #handleHttpMediaTypeNotSupported
 * @see #handleMissingServletRequestParameter
 * @see #handleServletRequestBindingException
 * @see #handleTypeMismatch
 * @see #handleHttpMessageNotReadable
 * @see #handleHttpMessageNotWritable
 * @see #handleMethodArgumentNotValidException
 * @see #handleMissingServletRequestParameter
 * @see #handleMissingServletRequestPartException
 * @see #handleBindException
```

这些实现了框架已经实现了，只要出现该异常就会触发

- **通过配置实现异常处理：SimpleMappingExceptionResolver**

配置：

```xml
<!-- SimpleMappingExceptionResolver:以配置的方式处理异常 -->
<bean class="org.springframework.web.servlet.handler.SimpleMappingExceptionResolver">
    <!-- 可以省略，默认值为exception -->
    <property name="exceptionAttribute" value="ex"></property>
    <property name="exceptionMappings">
        <props>
            <!-- 相当于catch(ArithmeticException ex){跳转:error} -->
            <prop key="java.lang.ArithmeticException">error</prop>
        </props>
    </property>
</bean>
```

控制器：

```java
@RequestMapping(value = "testSimpleMappingException")
public String testSimpleMappingException() {
    System.out.println(1/0);//ArithmeticException
    return "success";
}
```

两种方式异常总结：

1.通过继承+注解`@ExceptionHandler`统一处理异常

2.通过配置实现异常处理，与1无异

3.1和2同时实现，同种异常只由1捕获，即1优先级高于2

### 5.2 全局异常处理类

全局异常处理类相当于一个控制器，只是之前的控制器是控制用户输入数据处理，

而全局异常处理类是控制用户操作出错处理的控制器

```java
package org.zhkucst.handle;


import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.servlet.ModelAndView;
import org.zhkucst.exception.AgeException;
import org.zhkucst.exception.NameException;

/**
 * ControllerAdvice:控制器增强（给控制器诶增强功能，异常处理功能）
 *  定义：在类的上面
 *  特点：必须让框架指定这个注解所在的包名，需要在springmvc配置文件声明组件扫描器
 *  指定@ControllerAdvice所在的包名
 *
 */
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(value = NameException.class)
    public ModelAndView doNameException(Exception exception){
        //处理NameException异常
        ModelAndView mv = new ModelAndView();
        mv.addObject("msg", "姓名必须是zs，其他用户不能访问");
        mv.addObject("ex",exception);
        mv.setViewName("nameError");

        return mv;
    }

    @ExceptionHandler(value = AgeException.class)
    public ModelAndView doAgeException(Exception exception){
        //处理NameException异常
        ModelAndView mv = new ModelAndView();
        mv.addObject("msg", "你的年龄不能大雨80");
        mv.addObject("ex",exception);
        mv.setViewName("ageError");

        return mv;
    }

    @ExceptionHandler
    public ModelAndView doOtherException(Exception exception){
        //处理NameException异常
        ModelAndView mv = new ModelAndView();
        mv.addObject("msg", "其他错误!!");
        mv.addObject("ex",exception);
        mv.setViewName("defaultError");

        return mv;
    }

}
```

### 5.3 自定义具体处理类

步骤：

写一个自定义父类，继承`Exception`

再分别写具体的异常类

1.`MyUserException`

```java
package org.zhkucst.exception;
public class MyUserExcepton extends Exception{
    public MyUserExcepton(){
        super();
    }
    public MyUserExcepton(String message){
        super(message);
    }
}
```

2.`NameException`

```java
package org.zhkucst.exception;
public class NameException extends MyUserExcepton{
    public NameException(){
        super();
    }
    public NameException(String message){
        super(message);
    }
}
```

3.`AgeException`

```java
package org.zhkucst.exception;
public class AgeException extends MyUserExcepton{
    public AgeException(){
    }
    public AgeException(String message){
        super(message);
    }
}
```

### 5.4 前端

1.`nameError`

```jsp
<body>
    <h3>nameError.jsp</h3>
    <h3>msg数据：${msg}</h3>
    <h3>message数据：${ex.message}</h3>
</body>
```

2.`ageError`

```jsp
<body>
    <h3>ageError.jsp</h3>
    <h3>msg数据：${msg}</h3>
    <h3>message数据：${ex.message}</h3>
</body>
```

3.`defaultError`

```jsp
</head>
<body>
    <h3>defaultError.jsp</h3>
    <h3>msg数据：${msg}</h3>
    <h3>message数据：${ex.message}</h3>
</body>
```

### 5.5 处理器

```java
@RequestMapping(value = "some.do")
public ModelAndView doSome(String name, Integer age) throws MyUserExcepton {
    ModelAndView mv = new ModelAndView();

    if(!"zs".equals(name)){
        throw new NameException("姓名不正确！！");
    }

    if(age == null || age > 80){
        throw new AgeException("年龄比较大！！");
    }

    mv.addObject("myname", name);
    mv.addObject("myage", age);
    mv.setViewName("show");

    return mv;

}
```

## 6. 拦截器

### 6.1 *.do 和 *.action 的请求调度

规则不仅仅只有`*.do`和`*.action`，自定义也可以

![img](https://yupeng-tuchuang.oss-cn-shenzhen.aliyuncs.com/462ae275-e9a1-41c1-8a9f-9e0e2d83b557.png)

```xml
<!-- springmvc的前端控制器 -->
<servlet>
    <servlet-name>tony-video-admin-mng</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <!-- contextConfigLocation不是必须的， 如果不配置contextConfigLocation， springmvc的配置文件默认在：WEB-INF/servlet的name+"-servlet.xml" -->
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:spring/springmvc.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>

<servlet-mapping>
    <servlet-name>tony-video-admin-mng</servlet-name>
    <url-pattern>*.action</url-pattern>
</servlet-mapping>
```

请求方可以是视图也可以是控制器

- 1.请求方为视图：
- 2.请求方为控制器：

通过`<jsp:forward page="${base}/center.action"></jsp:forward>`

或通过`form`表单请求，请求的表单`action`可以不写`.action`，但浏览器直接请求需要写后缀`.action`，`base`取当前页面所在的路径，经过中央调度器处理后，即上面在`web.xml`配置`*.action`的所有请求给中央调度器，解析完后如果控制器上有配置`requestMapping`会捕捉给控制器，控制器上的`requestMapping`的`value`属性可以不写`.action`后缀，这是中央调度器的解析协议。没有就会转给视图，最后才会报`404`

### 6.2 拦截器的介绍

- 拦截器是springmvc中的一种，需要实现HandlerInterceptor接口

- 拦截器和过滤器类似，功能方向侧重点不同

  - 过滤器是用来过滤请求超时，设置编码字符集等工作；

  - 拦截器是拦截用户的请求，对请求做判断处理的。

- 拦截器是全局的，可以对多个Controller做拦截

  - 一个项目中可以有0个或多个拦截器，他们在一起拦截用户的请求;

  - 拦截器常用在：用户登录处理，权限检查，记录日志。

- 拦截器的使用步骤

  - 定义类实现HandlerInterceptor接口

  - 在springmvc配置文件中，声明拦截器，让框架指定拦截器的存在

- 拦截器的执行时间

  - 在请求处理之前，也就是controller类中的方法执行之前先被拦截；

  - 在控制器方法执行之后也会执行拦截器

  - 在请求处理完成之后也会执行拦截器

拦截器：看做是多个Controller中公用的功能，集中到拦截器统一处理，使用的是aop的思想

### 6.3 拦截器三个方法

```java
package org.zhkucst.handle;

import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.util.Date;

public class MyInterceptor implements HandlerInterceptor {
    private long btime;
    /**
     * preHandle叫做预处理方法
     * 参数：
     * Object handle：被拦截的控制对象
     * 返回值boolean
     * true:
     * false:
     *
     * 特点：
     *  1.方法在控制方法（MyController的doSome）之前先执行的
     *      用户的请求首先到达此方法
     *  2.在这个   方法中可以获取请求的信息，验证请求是否符合要求
     *      可以验证用户是否登录，验证用户是否有权限访问某个连接地址（url）
     *      如果验证失败，可以戳断请求，请求不能被处理
     *      如果验证成功，可以放行请求，此时控制器方法才能执行。
     *
     */
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        btime = System.currentTimeMillis();
        System.out.println("拦截器的MyInterceptor的preHandle()");

        return true;
    }

    /**
     * postHandle:后处理器方法
     * 参数：
     *  Object handler:被拦截的处理器对象MyController
     *  ModeAndView mv:处理器方法的返回值
     *
     *  特点：
     *   1.在处理器方法之后执行的（MyController.doSome()）
     *   2.能够获取处理器方法的返回值ModeAndView,可以修改ModeAndView中的
     *   数据和视图，可以影响到最后的执行结果
     *   3.主要是对原来的执行结果做第二次修正
     */
    @Override
    public void postHandle(HttpServletRequest request,
                           HttpServletResponse response,
                           Object handler, ModelAndView modelAndView) throws Exception {
        System.out.println("拦截器的MyInterceptor的postHandle()");

        if(modelAndView != null){
            System.out.println(11);
            modelAndView.addObject("mydata", new Date());
            modelAndView.setViewName("other");
        }

    }

    /**
     * +afterCompletion:最后执行的方法
     * 参+数：
     *  Ob+ject handle:被拦截的处理器对象
     *  Exception ex：程序中发生的异常
     *  特点：
     *   1.在请求处理完成后执行的，框架中规定是当你的视图处理完成后，对视图执行了forward,就认为请求处理完成
     *   2.一般做资源回收工作，程序请求过程中创建了一些对象，在这里可以
     */
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        long etime = System.currentTimeMillis();

        System.out.println("拦截器的MyInterceptor的afterCompletion()");

        System.out.println("计算机从preHandle到请求处理完成的时间:" + (etime - btime));
    }

}
```

### 6. 4 拦截器的声明

web.xml

```xml
<!--声明拦截器：拦截器可以有0或多个-->
<mvc:interceptors>
    <!--声明第一个拦截器-->
    <mvc:interceptor>
        <!--指定拦截的路径-->  
        <mvc:mapping path="/**"/>        
        <!--指定不拦截的路径-->
        <mvc:execlude-mapping path="/handler/testUpload"/>
        <!--声明拦截器对象-->
        <bean class="org.zhkucst.handle.MyInterceptor"/>
    </mvc:interceptor>
</mvc:interceptors>
```

### 6.5 处理步骤

```ruby
拦截处理步骤：
1.新建maven web项目
2.加入依赖
3.创建Controller类
4.创建一个普通类，作为拦截器使用
    1)实现HandlerInterceptor接口
    2)实现接口中的三个方法
5.创建show.jsp
6.创建springmvc的配置文件
    1)组件扫描器，扫描@Controller注解
    2)声明拦截器，并指定拦截的请求uri地址
```

### 6.6 多拦截器

多拦截器的初始化和执行顺序按照声明顺序，声明在前的先初始化

拦截器的初始化是存放在一个ArrayList数组中，数组的顺序跟先后添加是一致的

先初始化的拦截器先拦截，后初始化的后拦截

**第一个拦截器preHandle：true ；第二个拦截器preHandle：false 执行结果：**

1拦截器的`MyInterceptor`的`preHandle()`

2拦截器的`MyInterceptor`的`preHandle()`

1拦截器的`MyInterceptor`的`afterCompletion()`

原因：1.第二个拦截器没有放行，被拦截下来的处理器方法没有执行，`postHandle()`不会执行

​		  2.第一个放行，故而会执行`afterCompletion()`

**第一个拦截器preHandle：true ；第二个拦截器preHandle：true 执行结果：**

1拦截器的`MyInterceptor`的`preHandle()`

2拦截器的`MyInterceptor`的`preHandle()`

===执行`doSome`===

2拦截器的`MyInterceptor`的`postHandle()`

1拦截器的`MyInterceptor`的`postHandle()`

2拦截器的`MyInterceptor`的`afterCompletion()`

1拦截器的`MyInterceptor`的`afterCompletion()`

**第一个拦截器preHandle：false ；第二个拦截器preHandle：true | false 执行结果：**

1拦截器的`MyInterceptor`的`preHandle()`

原因：第一个拦截器不放行，自然第二个拦截器也不会执行，处理器方法不会执行，后面的`postHandle`、`afterCompletion`都不会执行

### 6.7 拦截器和过滤器的区别

1.过滤器是servlet中的对象，拦截器是框架中的对象

2.过滤器实现Filter接口的对象，拦截器是实现HandleInterceptor接口

3.过滤器是用来设置request,response的参数，属性的，侧重对数据过滤的拦截器是用来验证请求的，能截断请求

4.过滤器是在拦截器之前先执行的

5.过滤器是tomcat服务器创建的对象，拦截器是springmvc容器中创建的对象

6.过滤器是一个执行时间点，拦截器有三个执行时间点

7.过滤器可以处理jsp,js,,html等等，拦截器是侧重拦截对Controller的对象，如果你的请求不能被DispatcherServlet接收，这个请求不会执行拦截器内容

8.拦截器拦截普通类方法执行，过滤器过滤servlet请求响应

## 7. 其他处理器

```java
ApplicationContext ctx = new ClassPathXmlApplication("beans.xml");
StudentService service = (StudentService) ctx.getBean("service");
```

`springmvc`内部请求的处理流程：也就是`springmvc`接收请求，到处理完成的过程

**1.用户发起请求some.do**

**2.DispatcherSerrvlet接收请求some.do,把请求转发给处理器映射**

处理器映射器：springmvc框架中的一种对象，框架把实现了`HandlerMapping`接口的类都叫做映射器（多个）

处理器映射器作用：根据请求，从`springmvc`容器对象中获取处理器对象

```java
MyController controller = ctx.getBean("some.do")
```

框架把找到的处理器对象放到一个叫做处理器执行链（`HandleExecutionChain`）的类中保存

`HandlerExecutionChain`：类中保存着：

(1) 处理器对象（`MyController`）;

(2) 项目中的所有的拦截器`List<HandlerInterceptor> interceptorList`

```java
HandlerExecutionChain mapperHandler = getHandler(processedRequest)
```

**3.DispatcherServlet把2中的HandlerExecutionChain中的处理器对象交给了处理器适配器对象（多个）**

处理器适配器：springmvc框架中的对象，需要实现HandlerAdapter接口

处理器适配器作用：执行处理器方法（调用`MyController.doSome()` 得到返回值`ModeAndView`）

中央调度器调用适配器：

```java
HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHndler())
```

执行处理器方法：

```java
mv = ha.handle(processedRequest, response, mapperHadler.getHandler())*
```

**4.DispatherServlet把3中获取的ModeAndView交给了视图解析器对象**

​	视图解析器：springmvc中的对象，需要实现ViewResolver接口（可以有多个）

​	视图解析器作用：组成视图完整路径，在框架中`jsp`,	`html`不是`string`表示，而是使用`View`和他的实现类表示视图

​	`InternalResourceView`：视图类，	表示jsp文件，视图解析器会创建`InternalResourceView`类对象

**5.DispatcherServlet把4步骤中创建的View对象获取到，调用View类自己的方法，把Model数据放入到requesst作用域**

图解：

![img](file://C:\Users\fyp01\Documents\FocusNote\assets\515278c9-a47e-4c3c-8f10-a436207069e1.png?t=1658452664101)

​	

## 8. 表单标签

表单标签：

​	自定义标签：`el`、`jstl`

​	`Spring EL`：

**1.支持各种类型的请求方式（查询doGet, 增加doPost, 删除doDelete，修改doPut）**

（1）编写`method="put|delete"`

  (2)	过滤器：为了让 浏览器能够支持`put`和`delete`请求

`get post`

`put delete` ->过滤器 `HiddenHttpMethodFilter`

`HiddenHttpMethodFilter`会将全部请求 名为 "`_method`" 的隐藏域 进行 `put | delete` 处理

如果使用的是`SpringMVC`标签：

`method="put | delete"`

如果不是`SpringMVC`标签：是传统的`html`标签：

`method="post"`

```jsp
<input type="hidden" name="_method" value="put | delete">
```

```jsp
<form:form action="controller/testMethod" method="get">
    <input type="submit" value="查看">
</form:form>


<form:form action="controller/testMethod" method="post">
    <input type="submit" value="增加">
</form:form>


<form:form action="controller/testMethod" method="delete">
    <input type="submit" value="删除">
</form:form>


<form:form action="controller/testMethod" method="put">
    <input type="submit" value="修改">
</form:form>
```

优点：省略步骤：隐藏域

**可参考原生态的RESTful风格**		

  **2.可以将对象和 表单绑定起来**

`SpringMVC`项目：

```ruby
  1. 环境搭建：
  2. 引入标签库：<%@ taglib prefix="form"  uri="http://www.springframework.org/tags/form"%>
  3. 使用
```

**input：**

```jsp
<form:form>
    姓名：<form:input path="stuName"/><br/>
    年龄：<form:input path="stuAge"/>
    <input type="submit" value="提交">
</form:form>
```

1.默认实例化`bean`标签的`id`值为`command`，`map.put`的k值必须为`command`

2.自定义，`modelAttribute="person"`，通过`form`中该属性可以指定`map.put`的`k`值

3.`path`：绑定中对象的属性值

控制器：

```java
@Controller
@RequestMapping(value = "/controller")
public class MyController {
    @Resource(name = "studentServiceImpl")
    IStudentService studentService;

    public IStudentService getStudentService() {
        return studentService;
    }

    @RequestMapping(value = "/testFormTag")
    public String testFormTag(Map<String, Object> map){

        Student student = studentService.queryStudentByStuNo(1);

        map.put("command",student);

        return "success";
    }

}
```

`mapper`:

```jsp
<select id="selectStudents" resultType="student">
    select stuno,stuname,stuage from student
</select>
```

`entity`：

```java
public class Student {
    private Integer stuNo;
    private String stuName;
    private Integer stuAge;
```

**checkbox和checkboxes：**

`checkbox`：

​	自动绑定`request`域中的值

​	**a.通过boolean值绑定**

```jsp
<form:form modelAttribute="stu">
    <form:checkbox path="stuSex"/>
    <input type="submit">
</form:form>
```

```java
@RequestMapping(value = "testCheckBox")
public String testCheckBox(Map<String,Object> map){
    Student student = new Student();
    student.setStuSex(true);

    map.put("stu",student);

    return "success";
}
```

**b.绑定 集合（List、Set）、数组的中枢**

`checkboxes`：多个`checkbox`的组合

`path`：选中的选项

`items`：所有的选项

```jsp
<form:form modelAttribute="stu">
    <form:checkbox path="hobbies" value="basketball"/>
    <form:checkbox path="hobbies" value="football"/>
    <form:checkbox path="hobbies" value="pingpong"/>
    <input type="submit">
</form:form>
```

```java
@RequestMapping(value = "testCheckBoxes")
public String testCheckBoxes(Map<String, Object> map){
    Student student = new Student();
    List<String> hobbies = new ArrayList<>();
    hobbies.add("football");
    hobbies.add("basketball");

    student.setHobbies(hobbies);

    map.put("stu",student);

    return "success";
}
```

等价于：

```jsp
<form:form modelAttribute="stu">
    <form:checkboxes path="hobbies" items="${allHobbies}"/>
    <input type="submit">
</form:form>
```



```java
@RequestMapping(value = "testCheckBoxes")
public String testCheckBoxes(Map<String, Object> map){
    Student student = new Student();
    //选中选项
    List<String> hobbies = new ArrayList<>();
    hobbies.add("football");
    hobbies.add("basketball");
    hobbies.add("pingpong");
    //全部选项
    /* List<String> allHobbies = new ArrayList<>();
    allHobbies.add("football");
    allHobbies.add("basketball");
    allHobbies.add("pingpong");
    allHobbies.add("d");*/
    Map<String, String> allHobbies = new HashMap<>();
    allHobbies.put("football","足球");
    allHobbies.put("basketball","篮球");
    allHobbies.put("pingpong","乒乓球");
    allHobbies.put("d","其他");
    student.setHobbies(hobbies);


    map.put("stu",student);
    map.put("allHobbies",allHobbies);


    return "success";
}
```

**radiobuttons：**

同理

**select:**

方式一：

```JSP
<form select path="默认的值" items="所有的可选项">
```

方式二：

```jsp
<form:select path="默认的值">
    <form:option value="football" 足球></form:option>
    <form:option value="basketball"></form:option>
    <form:option value="pingpong"></form:option>
</form:select>
```

方式三：

```jsp
<form:select path="默认的值">
    <form:options items="${allBallMap}"></form:options>
</form:select>
```

总结：

如果方式二、方式三同时存在，则使用方式二

如果方式一、方式二同时存在，则使用方式一

原生态优先级高于框架，而且最原生态的`<option>`没有绑定的功能 

**c.（了解）嵌套对象的toString()的返回值**

```java
public class Student {
    private Integer stuNo;
    private String stuName;
    private Integer stuAge;
    private boolean stuSex;

    private Other other;
}
```

```java
public class Other {
    @Override
    public String toString(){
        return "pingpong";
    }
}
```

```jsp
<form:form modelAttribute="stu">
    <%--<form:checkbox path="other" value="basketball"/>
    <form:checkbox path="other" value="football"/>
        <form:checkbox path="other" value="pingpong"/>--%>
    <input type="submit">
</form:form>
```

```java
@RequestMapping(value = "testCheckBoxes")
public String testCheckBoxes(Map<String, Object> map){
    Student student = new Student();
    Other other = new Other();
    student.setOther(other);

    map.put("stu",student);

    return "success";
}
```