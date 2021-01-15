# HttpClient

## 简介

HttpClient是Apache Jakarta  Common下的子项目，用来提供高效的、最新的、功能丰富的支持HTTP协议的客户端编程工具包，相对于JDK自带的URLConnection更加灵活，其主要功能如下：

+ 基于java，HTTP1和HTTP1.1
+ 实现了http的所有方法（GET，POST，PUT，DELETE，HEAD, OPTION,  TRACE）
+ 支持自动转向
+ 支持HTTPS协议
+ 支持代理服务器等
+ 用于多线程应用程序中的连接管理支持。支持设置最大的总连接以及主机的最大连接数。检测和关闭陈旧的连接。
+ 自动处理Set-Cookie中的Cookie
+ 在http1.0和http1.1中利用KeepAlive保持持久连接
+ 直接获取服务器发送的response code和 headers
+ 设置连接超时的能力
+ 源代码基于Apache License 可免费获取

## 基础知识

HttpClient最重要的功能是执行HTTP方法，用户需要提供一个用于执行的请求对象，HttpClient需要向目标服务器发送请求并得到相应的响应对象，如果执行不成功则抛出异常。 

使用方法如下：

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpGet httpget = new HttpGet("http://localhost/");
CloseableHttpResponse response = httpclient.execute(httpget);
try {
    <...>
} finally {
    response.close();
}
```

### http请求

所有HTTP请求都有一个请求行，它包含一个方法名，一个请求URI和一个HTTP协议版本。

HttpClient支持HTTP /  1.1规范中定义的所有HTTP方法：GET，HEAD，POST，PUT，DELETE，TRACE和OPTIONS。每个方法类型都有一个特定的类：HttpGet，HttpHead，HttpPost，HttpPut，HttpDelete，HttpTrace和HttpOptions。

### URL

HttpClient提供URIBuilder实用程序类来简化请求URI的创建和修改。

```java
URI uri = new URIBuilder()
        .setScheme("http")
        .setHost("www.google.com")
        .setPath("/search")
        .setParameter("q", "httpclient")
        .setParameter("btnG", "Google Search")
        .setParameter("aq", "f")
        .setParameter("oq", "")
        .build();
HttpGet httpget = new HttpGet(uri);
System.out.println(httpget.getURI()); // http://www.google.com/search?q=httpclient&btnG=Google+Search&aq=f&oq=
```

### http响应

HTTP响应是服务器收到请求消息后执行相关操作后发送回客户端的消息。该消息的第一行由协议版本和数字状态码及其相关的文本短语组成。

```java
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1, HttpStatus.SC_OK, "OK");
System.out.println(response.getProtocolVersion()); // HTTP/1.1
System.out.println(response.getStatusLine().toString()); // HTTP/1.1 200 OK
```

### Message Header

HTTP消息可以包含许多描述消息属性的header，如内容长度，内容类型等等。HttpClient提供了检索，添加，删除和枚举header的方法。

```java
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1, HttpStatus.SC_OK, "OK");
response.addHeader("Set-Cookie",
                   "c1=a; path=/; domain=localhost");
response.addHeader("Set-Cookie",
                   "c2=b; path=\"/\", c3=c; domain=\"localhost\"");
response.addHeader("Set-Cookie",
                   "c4=d; path=\"/\"; domain=\"localhost\"");
// 获取header
Header h1 = response.getFirstHeader("Set-Cookie");
System.out.println(h1); // Set-Cookie: c1=a; path=/; domain=localhost
Header h2 = response.getLastHeader("Set-Cookie");
System.out.println(h2); // Set-Cookie: c4=d; path="/"; domain="localhost"
Header[] hs = response.getHeaders("Set-Cookie");
System.out.println(hs.length); // 3

// 使用HeaderIterator获取header
HeaderIterator it = response.headerIterator("Set-Cookie");

while (it.hasNext()) {
    System.out.println(it.next());
}
// 输出
// Set-Cookie: c1=a; path=/; domain=localhost
// Set-Cookie: c2=b; path="/", c3=c; domain="localhost"
// Set-Cookie: c4=d; path="/"; domain="localhost"
```

### HttpEntity

HttpEntity代表底层流的实体，是可以用Http消息发送或者接收的实体。

+ streamed: 内容是从一个流接收的，或者是随时产生的。具体来说，这个类别包括从HTTP响应中收到的实体。流派实体通常不可重复。
+ self-contained: 内容在内存中或通过独立于连接或其他实体的方式获得。该类实体通常可重复。这种类型的实体将主要用于包含HTTP请求的实体。
+ wrapping: 内容是从另一个Entity获得的。

字符串实体使用如下：

```java
HttpEntity entity = new StringEntity("字符串实体", "UTF-8");
// 内容类型
// Content-Type: text/plain; charset=UTF-8
System.out.println(entity.getContentType());
// 内容的编码格式
// null
System.out.println(entity.getContentEncoding());
// 内容的长度
// 15
System.out.println(entity.getContentLength());
// 把内容转成字符串
// 字符串实体
System.out.println(EntityUtils.toString(entity));
// 内容转成字节数组
// 15
System.out.println(EntityUtils.toByteArray(entity).length);
// 直接获得流
// 
entity.getContent();
```

对于流实体，使用之后要回收资源

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
// 发送post请求
HttpPost httpPost = new HttpPost("http://xxx");
CloseableHttpResponse response = httpclient.execute(httpget);
try {
    HttpEntity entity = response.getEntity();
    InputStream is = entity.getContent();	
    try {
        <...>
    } final {
        is.close();
    } 
} final {
    response.close();
}
```

关闭内容流和关闭响应的区别在于，前者将尝试通过消耗实体内容来保持底层连接的活动，而后者立即关闭并放弃连接。

请注意HttpEntity＃writeTo（OutputStream）方法也需要确保一旦实体被完全写出，正确释放系统资源。如果此方法通过调用HttpEntity＃getContent（）来获取java.io.InputStream的实例，则还应该在finally子句中关闭该流。

在使用流式实体时，可以使用EntityUtils＃consume（HttpEntity）方法确保实体内容已被完全消耗，并且基础流已关闭。

然而，可能有这样的情况，当只需要检索整个响应内容的一小部分，并且消耗剩余内容或使得连接可重用的性能损失太高时，在这种情况下，可以通过关闭响应来终止内容流。连接不会被重用，它所拥有的所有关卡资源将被正确释放。

### EntityUtils

在代码中得到的response是不会有详细信息的，必须使用HttpClient中的EntityUtils工具类转为String才行。

```java
CloseableHttpClient httpClient = HttpClients.createDefault();
HttpGet httpget = new HttpGet("http://www.baidu.com/");

CloseableHttpResponse response;

{
    try {
        response = httpClient.execute(httpget);
        System.out.println(response);

        System.out.println(EntityUtils.toString(response.getEntity()));
    } catch (IOException e) {
        e.printStackTrace();
    }
}

// 输出
// HttpResponseProxy{HTTP/1.1 200 OK [Server: bfe/1.0.8.18, Date: Thu, 19 Apr 2018 07:45:43 GMT, Content-Type: text/html, Last-Modified: Mon, 23 Jan 2017 13:27:36 GMT, Transfer-Encoding: chunked, Connection: Keep-Alive, Cache-Control: private, no-cache, no-store, proxy-revalidate, no-transform, Pragma: no-cache, Set-Cookie: BDORZ=27315; max-age=86400; domain=.baidu.com; path=/] org.apache.http.client.entity.DecompressingEntity@6d00a15d}

// <!DOCTYPE html>
// <!--STATUS OK--><html> <head><meta http-equiv=content-type content=text/html;charset=utf-8><meta http-equiv=X-UA-Compatible content=IE=Edge><meta content=always name=referrer><link rel=stylesheet type=text/css href=http://s1.bdstatic.com/r/www/cache/bdorz/baidu.min.css><title>ç¾åº¦ä¸ä¸ï¼ä½ å°±ç¥é</title></head> <body link=#0000cc> <div id=wrapper> <div id=head> <div class=head_wrapper> <div class=s_form> <div class=s_form_wrapper> <div id=lg> <img hidefocus=true src=//www.baidu.com/img/bd_logo1.png width=270 height=129> </div> <form id=form name=f action=//www.baidu.com/s class=fm> <input type=hidden name=bdorz_come value=1> <input type=hidden name=ie value=utf-8> <input type=hidden name=f value=8> <input type=hidden name=rsv_bp value=1> <input type=hidden name=rsv_idx value=1> <input type=hidden name=tn value=baidu><span class="bg s_ipt_wr"><input id=kw name=wd class=s_ipt value maxlength=255 autocomplete=off autofocus></span><span class="bg s_btn_wr"><input type=submit id=su value=ç¾åº¦ä¸ä¸ class="bg s_btn"></span> </form> </div> </div> <div id=u1> <a href=http://news.baidu.com name=tj_trnews class=mnav>æ°é»</a> <a href=http://www.hao123.com name=tj_trhao123 class=mnav>hao123</a> <a href=http://map.baidu.com name=tj_trmap class=mnav>å°å¾</a> <a href=http://v.baidu.com name=tj_trvideo class=mnav>è§é¢</a> <a href=http://tieba.baidu.com name=tj_trtieba class=mnav>è´´å§</a> <noscript> <a href=http://www.baidu.com/bdorz/login.gif?login&amp;tpl=mn&amp;u=http%3A%2F%2Fwww.baidu.com%2f%3fbdorz_come%3d1 name=tj_login class=lb>ç»å½</a> </noscript> <script>document.write('<a href="http://www.baidu.com/bdorz/login.gif?login&tpl=mn&u='+ encodeURIComponent(window.location.href+ (window.location.search === "" ? "?" : "&")+ "bdorz_come=1")+ '" name="tj_login" class="lb">ç»å½</a>');</script> <a href=//www.baidu.com/more/ name=tj_briicon class=bri style="display: block;">æ´å¤äº§å</a> </div> </div> </div> <div id=ftCon> <div id=ftConw> <p id=lh> <a href=http://home.baidu.com>å³äºç¾åº¦</a> <a href=http://ir.baidu.com>About Baidu</a> </p> <p id=cp>&copy;2017&nbsp;Baidu&nbsp;<a href=http://www.baidu.com/duty/>ä½¿ç¨ç¾åº¦åå¿è¯»</a>&nbsp; <a href=http://jianyi.baidu.com/ class=cp-feedback>æè§åé¦</a>&nbsp;äº¬ICPè¯030173å·&nbsp; <img src=//www.baidu.com/img/gs.gif> </p> </div> </div> </div> </body> </html>
```

### 使用

HttpClient本质功能是执行http方法，对于每个http方法都有一个对应的类

+ GET -> HttpGet
+ POST -> HttpPost
+ PUT -> HttpPut
+ HEAD -> HttpHead
+ DELETE -> HttpDelete
+ OPTION -> HttpOptions
+ TRACE -> HttpTrace