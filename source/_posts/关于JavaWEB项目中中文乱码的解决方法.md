---
title: Java WEB项目中中文乱码
categories:
  - Web
tags:
  - 中文乱码
abbrlink: 25480
date: 2017-05-12 16:44:33
---

# Java WEB项目中中文乱码的解决方法

------
## 例子
如下表单：
```shell
<form action="${pageContext.request.contextPath}/UserServlet" method="post">
    姓名：<input type="text" name="name">
    <input type="submit" value="提交">
</form>    
```
在这个表单中如果‘姓名’为中文，提交至web后端进行获取，如：

```shell
    name = request.getParameter("name");
```
此时获取到的name的值一定是乱码，此时就要进行中文乱码处理了。
## 处理方案可根据请求的类型分为两类:
**POST请求**：
```shell
request.setCharacterEncoding("UTF-8");//在接收请求参数之前，将request的缓冲区字符集设置为UTF-8
request.getParameter("name");//此时再获取表单中的中文数据就不会乱码了
```
**GET请求**
```shell
String param = reques.getParameter("name");//首先获取表单数据
String name = new String(param.getBytes("ISO-8859-1"),"UTF-8");//将获取到的表单数据以ISO-8859-1字符集转成字节数组，再按照UTF-8字符集编码成新的字符串，即可解决中文乱码
```

**重点：**
    表单数据之所以乱码，是因为字符集不匹配，导致的，在Post请求时，我们在获取请求参数之前，将request缓冲区的字符集设置成UTF-8，即可在表单数据发送过来时，将表单数据按照UTF-8的编码放入缓冲区，在取出来时，自然就已经不会乱码了。在get请求时较为繁琐。需要首先将表单数据以ISO-8859-1字符集转成字节数组，因为request缓冲区的默认字符集是ISO-8859-1,所以以该字符集来将乱码数据转换成字节数组，不会丢失字节，从而保证了表单数据的完整性，进而以UTF-8字符集构造成新的字符串，将中文读出。
    但是如果需要将中文数据在页面展示，还需要：
```shell
    response.setContentType("text/html;charset=UTF-8");//向浏览器指明要以UTF-8字符集解析返回的数据
```

***以上是针对单个网页的中文乱码解决方法***

<!--more-->

### 在一个web项目中，中文乱码的解决方案

#### 我认为最佳解决方案就是使用过滤器，拦截所有包含有中文的请求数据，转码后再传递给请求的目标地址。

依旧以上述表单数据的中文乱码为例：
Filter代码如下：
```shell
public class GenericEncodingFilter implements Filter {

    @Override
    public void destroy() {
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
            FilterChain chain) throws IOException, ServletException {
        // 转型为与协议相关对象
        HttpServletRequest httpServletRequest = (HttpServletRequest) request;
        // 对request包装增强
        HttpServletRequest myrequest = new MyRequest(httpServletRequest);
        chain.doFilter(myrequest, response);
    }

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
    }

}
```
在上述代码中，我们思考到，之所以在获取表单数据时，拿到的中文都是乱码，是因为request对象本身并不具备对中乱码的解决能力，因此，我们想到以装饰者模式对request进行增强。(当然也可以使用动态代理来解决)
**关于装饰者模式的注意点：**

 1. 增强的类要和被增强的类实现同一个接口 
 2. 增强的类需要传入一个被增强类的引用

以下是增强HttpServletRequest对象的实现：
```shell
// 自定义request对象
class MyRequest extends HttpServletRequestWrapper {

    private HttpServletRequest request;

    private boolean hasEncode;

    public MyRequest(HttpServletRequest request) {//此处传入了request的引用
        super(request);// super必须写
        this.request = request;
    }

    // 对需要增强方法 进行覆盖
    @Override
    public Map getParameterMap() {
        // 先获得请求方式
        String method = request.getMethod();
        if (method.equalsIgnoreCase("post")) {
            // post请求
            try {
                // 处理post乱码
                request.setCharacterEncoding("utf-8");
                return request.getParameterMap();
            } catch (UnsupportedEncodingException e) {
                e.printStackTrace();
            }
        } else if (method.equalsIgnoreCase("get")) {
            // get请求
            Map<String, String[]> parameterMap = request.getParameterMap();
            if (!hasEncode) { // 确保get手动编码逻辑只运行一次
                for (String parameterName : parameterMap.keySet()) {
                    String[] values = parameterMap.get(parameterName);
                    if (values != null) {
                        for (int i = 0; i < values.length; i++) {
                            try {
                                // 处理get乱码
                                values[i] = new String(values[i]
                                        .getBytes("ISO-8859-1"), "utf-8");
                            } catch (UnsupportedEncodingException e) {
                                e.printStackTrace();
                            }
                        }
                    }
                }
                hasEncode = true;
            }
            return parameterMap;
        }

        return super.getParameterMap();
    }

    @Override
    public String getParameter(String name) {
        Map<String, String[]> parameterMap = getParameterMap();
        String[] values = parameterMap.get(name);
        if (values == null) {
            return null;
        }
        return values[0]; // 取回参数的第一个值
    }

    @Override
    public String[] getParameterValues(String name) {
        Map<String, String[]> parameterMap = getParameterMap();
        String[] values = parameterMap.get(name);
        return values;
    }

}
```
在上述方法中，对request进行了增强，根据请求方法的不同，对request所携带的表单数据进行中文乱码的处理。然后将增强后的request向后传递：
```shell
// 对request包装增强
HttpServletRequest myrequest = new MyRequest(httpServletRequest);
chain.doFilter(myrequest, response);//将包装后的myrequest对象作为request对象向后传递
```

在web.xml中做如下配置
```shell
<filter>
    <display-name>GenericEncodingFilter</display-name>
    <filter-name>GenericEncodingFilter</filter-name>
    <filter-class>com.wfm.web.filter.GenericEncodingFilter</filter-class>
  </filter>
  <filter-mapping>
    <filter-name>GenericEncodingFilter</filter-name>
    <url-pattern>/*</url-pattern>
  </filter-mapping>
```
上述配置中``` <url-pattern></url-pattern>  ```,决定了该编码过滤器对哪些请求进行过滤，```/*```意思是指，对访问该web项目的所有请求都进行过滤。
## 如何对所有的servlet进行请求的过滤？
以下是一个servlet的配置：
```shell
<servlet>
    <description></description>
    <display-name>FormServlet</display-name>
    <servlet-name>FormServlet</servlet-name>
    <servlet-class>com.wfm.web.servlet.FormServlet</servlet-class>
  </servlet>
  <servlet-mapping>
    <servlet-name>FormServlet</servlet-name>
    <url-pattern>/FormServlet</url-pattern>
  </servlet-mapping>
```
在上述配置中，请求通过访问```/FormServlet```路径触发FormServlet，因此，我们可以拦截所有对该路径的请求。但是如果我们要拦截的是指定的几个Servlet怎么办？此时，想到了如下方法：
```shell
<servlet>
    <description></description>
    <display-name>FormServlet</display-name>
    <servlet-name>FormServlet</servlet-name>
    <servlet-class>com.wfm.web.servlet.FormServlet</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>FormServlet</servlet-name>
    <url-pattern>/servlet/FormServlet</url-pattern>
</servlet-mapping>
```
我们可以在```FormServlet```的请求路径中加上一个虚拟路径，并将所有待拦截的Servlet的路径都添加```/servlet```即可构造出一个请求的目录，然后将Filter的``` <url-pattern>/servlet/*</url-pattern>```,这样就实现了对特定几个servlet的请求过滤。

