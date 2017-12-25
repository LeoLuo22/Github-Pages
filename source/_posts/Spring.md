title: Spring学习笔记之Spring MVC
---
# Spring学习笔记之Spring MVC

----------
## 0x00 前言
我为什么要学习Spring?   
   
之前一直用的是django，但是公司用的是Spring。来公司之后在公司搭好的架子上面写代码，也不用关心各种配置和配置文件。但是，周末有一个技能比赛，要求用Spring框架，于是我从头开始一步一步开始做，复制粘贴各种配置文件。结果，卡在了登陆上，打死获取不到我放在session里面的值。   
   
## 0x01   
   
Spring MVC全称为Spring Web MVC framework，它提供了Model-View-Controller(MVC)架构。MVC可以分散应用的不同部分（输入逻辑，业务逻辑，UI逻辑)，并且使得他们的耦合性很低。   
   
- **Model**: 封装应用数据，通常由**POJO**组成。   
- **View**: 负责渲染model的数据，通常产生浏览器可以解析的HTML文件。
- **Controller**: 负责处理用户请求，构造合适的model然后传递给view渲染。   
   
## 0x02 DispatcherServlet   
   
Spring MVC框架围绕着一个*DispacherServlet*设计，DispacherServlet是用来处理所有的HTTP请求和响应的。如下图所示：   
   
![图1](http://oy2p9zlfs.bkt.clouddn.com/spring_dispatcherservlet.png)   

下面的是发送给DispacherServlet的HTTP的处理流程：   
   
- 接收到HTTP请求之后，DispacherServlet询问*HandlerMapping*来调用合适的*Controller*
- *Controller*接收request然后调用合适的service方法。service方法会基于定义的业务逻辑设置model数据，把view名称返回给*DispacherServlet*
- *DispacherServlet*询问*ViweResolver*来选择为请求定义的view
- 当viwe好了之后，*DispacherServlet*将model数据床底给view,view渲染给浏览器
   
上述所有的组件，比如HandlerMapping，Controller以及ViewResolver都是*WebApplicationContext*的一部分。   
   
## 0x03 Required Configuration   
   
你需要将你想要让*DispatcherServlet*处理的请求进行映射，通过在 **web.xml** 里面的URL映射。下面的例子是用来对 **HelloWeb** *DispatcherServlet* 进行声明和映射。
   
    <?xml version="1.0" encoding="UTF-8"?>
	<display-name>Spring MVC</display-name>

	<web-app id="WebApp_ID" version="2.4"
		 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
         xmlns="http://xmlns.jcp.org/xml/ns/javaee" 
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee 
         http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd" >
	<servlet>
    	<servlet-name>HelloWeb</servlet-name>
    	<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
     	<load-on-startup>1</load-on-startup>
	</servlet>

	<servlet-mapping>
     	<servlet-name>HelloWeb</servlet-name>
    	<url-pattern>*.jsp</url-pattern>
	</servlet-mapping>

	</web-app>
   
**web.xml** 在你应用的WebContent/WEB-INF文件夹下。当初始化 **HelloWeb** DispatcherServlet时，框架会尝试从一个位于WebContent/WEB-INF文件下的 **[servlet-name]servlet.xml** 的文件载入应用的上下文（context）。本次我们的文件叫做 **HelloWebServlet.xml** 。   
   
接下来,<servlet-mapping>标签表明URLs会被哪个DispatcherServlet处理。在这里所有的HTTP请求以 **.jsp** 结尾的都会被 **HelloWeb** DispatcherServlet处理。   
   
如果你不想要默认的 **[servlet-name]servlet.xml** 文件名以及路径，你可以通过在你的web.xml添加servlet listener *ContextLoaderListener*  来自定义文件名和路径。   
   
	<web-app...>

	<!------------------- DispatcherServlet definition goes here ---------------->
	....
    <context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>
			/WEB-INF/HelloWeb-servlet.xml,
		</param-value>
	</context-param>

	<listener>
		<listener-class>
			org.springframework.web.context.ContextLoaderListener
		</listener-class>
	</listener>
   
现在，来看一下 **HelloWeb-servlet.xml** 必需的配置。   
   
    <?xml version="1.0" encoding="UTF-8" ?>
	<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
    xmlns:p="http://www.springframework.org/schema/p"
    xmlns:context="http://www.springframework.org/schema/context"
    xmlns:util="http://www.springframework.org/schema/util" 
    xmlns:mvc="http://www.springframework.org/schema/mvc"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
       http://www.springframework.org/schema/context 
       http://www.springframework.org/schema/context/spring-context-3.0.xsd
       http://www.springframework.org/schema/util 
       http://www.springframework.org/schema/util/spring-util-3.0.xsd 
       http://www.springframework.org/schema/mvc 
       http://www.springframework.org/schema/mvc/spring-mvc.xsd
      ">
	
	<!-- 扫包 -->
    <context:component-scan base-package="com.springmvc.*"></context:component-scan>
	
	<bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
    	<!-- 制定页面存放的路径 -->
        	<property name="prefix" value="/WEB-INF/pages/"></property>
            <!-- 文件的后缀 -->
            <property name="suffix" value=".jsp"></property>
    </bean>
	</beans>   
   


- *[servlet-name]-servlet.xml* 被用来创建定义的beans
- *<contextcomponent-scan...>* 标签被用来激活Spring MVC的注解扫描，比如@Controller和@RequestMapping等。   
- *InternalResourceViewResolver* 有定义的规则来解析view的名称。在上述定义的规则中，一个名为 **hello** 的逻辑视图会被代表为在 *WEB-INF/jsp/hello.jsp* 的实现。   
   
 