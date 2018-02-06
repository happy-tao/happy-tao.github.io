---
layout: post
title: "获取tomcat端口号的三种方式"
date: 2016-01-11 18:11:25 +0800
comments: true
tags: Java
keywords: Java, tomcat端口号
---
获取tomcat的端口号一般有三种方式：

1. request直接获取
2. 读取Tomcat的配置文件获取
3. 通过Tomcat的Mbean获取

<!--more-->

## request直接获取
这种方式没什么可说的，就是通过ServletRequest中的**getLocalPort**或者**getServerPort**获取。

这里的问题在于getLocalPort和getServerPort两个方法的区别。

参考官方文档，getLocalPort获取的是接收方的端口号。getServerPort获取的是发送方的端口号。这两个东西是有可能不一样的。

例如：

* 用户直接访问tomcat服务器（8080)，这里getLocalPort和getServerPort得到的端口号都是8080。
* 服务端采用nginx（80）+tomcat（8080）架构，这里的getLocalPort是tomcat的端口号8080，getServerPort是nginx的端口号（80），既用户直接访问的端口号。

## 读取tomcat的配置文件获取
这种方式的关键点有两处：

1. 如何获取tomcat的绝对路径
2. 如何读取XML

tomcat的绝对路径可以使用**ServletContext**来获取。这个对象在Servlet，Filter，Listener等中都有出现。

``` java Servlet中获得ServletContext: http://ruitao.name
public class TestServlet extends HttpServlet {
    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        ServletContext sc = request.getServletContext();
    }
}
```

``` java ServletContextListener中获取ServletContext：
public class TestListener implements ServletContextListener {
    @Override
    public void contextInitialized(ServletContextEvent event) {
        ServletContext sc = event.getServletContext();
    }
}
```

``` java Filter中获取ServletContext：
public class AppFilter implements Filter {
    @Override
    public void init(FilterConfig config) throws ServletException {
        ServletContext sc = config.getServletContext();
    }
}
```


``` java ServletContext中获取realPath：
String realPath = sc.getRealPath("/");
```

关于读取XML，网上有很多相关的介绍，就不在这里说明了。

## 通过Tomcat的Mbean获取

Tomcat实现了MBean。我们可以使用相应的MBean的方式获得Tomcat的一些配置参数。

``` java Mbean获取Tomcat的端口号
/**
 * Created by wurt on 2016/1/11.
 */
public class TomcatConfig {

    public static void getHttpPort() {
        try {
            MBeanServer server = null;
            if (MBeanServerFactory.findMBeanServer(null).size() > 0) {
                server = MBeanServerFactory.findMBeanServer(null).get(0);
            }

            Set names = server.queryNames(new ObjectName("Catalina:type=Connector,*"), null);

            Iterator iterator = names.iterator();
            ObjectName name = null;
            while (iterator.hasNext()) {
                name = (ObjectName) iterator.next();

                String protocol = server.getAttribute(name, "protocol").toString();
                String scheme = server.getAttribute(name, "scheme").toString();
                String port = server.getAttribute(name, "port").toString();
                System.out.println(protocol + " : " + scheme + " : " + port);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }

    }
}
```

在tomcat中运行，结果如下：
```
HTTP/1.1 : http : 8088
AJP/1.3 : http : 8009
```

这两个端口号正是我在server.xml中配置的端口号。



## 参考
<http://stackoverflow.com/questions/2184286/difference-between-getlocalport-and-getserverport-in-servlets>

<http://docs.oracle.com/javaee/6/api/javax/servlet/ServletRequest.html>

<https://tomcat.apache.org/tomcat-7.0-doc/funcspecs/mbean-names.html>

<http://suhuanzheng7784877.iteye.com/blog/1187356>

<http://blog.csdn.net/nieyinyin/article/details/7469088>