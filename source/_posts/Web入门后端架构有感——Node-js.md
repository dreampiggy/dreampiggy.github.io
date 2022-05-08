---
title: Web入门后端架构有感——Node.js
categories: Web
tags:
  - Node.js
  - JavaScript
updated: '2016-03-30 00:59:35'
date: 2016-03-30 00:56:15
---

说来也是惭愧，从大一暑假接触到第一门Web后端语言——PHP之后，自己对Web应用程序的认识却一直没有多大改变，对整个Web服务器的运行也没有清晰的概念。不过，这个大二的寒假里面，由于需要架设一个Web多人协作编辑平台，我接触到了这个非常热门尽管国内应用不多的后端架设方式——`Node.js`

说到Node.js，其实很多人也都知道，一个JavaScript在本地的运行时，基于Google的V8引擎。很可惜，由于自己当时学习Web主要走的后端路线，自己对JavaScript的认识也就靠的是w3school的简单说明和了解以及简单的jQuery用法罢了。学Node.js对我印象最大的，就是事件驱动的回调函数。（其实还是与自己并未真正接触到函数式编程有关）

举个例子，就是最简单的Hello World Web应用程序。 

```javascript
var http = require("http");

http.createServer(function(request, response) {
  response.writeHead(200, {"Content-Type": "text/plain"});
  response.write("Hello World");
  response.end();
}).listen(8888);
```

在JavaScript里面，函数是和变量同地位的一等公民，它可以被传递进当为参数，这种写法初次接触JavaScript的人肯定感觉非常不适应（尤其从Java,PHP转来的），不过习惯之后便发现这样带来了不错的易读性。这里面，那个匿名函数function(request,response)就是一个回调函数，Node.js内置的http对象通过listen进行8888端口的监听，一旦有任何访问便会调用这个回调函数，执行那三行来发送一个Hello World的HTML文本回去

Node.js编写Web程序的体验绝对是Java和PHP很难达到的，因为你不仅要写一个Web应用程序，必须自己写一个HTTP服务器程序。不过这也意味着你可以干更多的事而不用局限于PHP这种解释器一样执行的代码，很多人甚至不知道在一行简单的echo "Hello World!" 的背后的Web应用程序究竟是怎样工作的，你也可以不用再和Apache，Nginx这种Web应用程序的使用打交道，你可以自己写一个类似的出来，定制化更高。   上面关于Node.js的简单认识也就先这样，下面我想说说关于Web应用程序的非常简单的架构组织。

一个Web应用程序，可以分这几个模块来进行编写： 

1.  **HTTP模块**
2.  **路由模块**
3.  **请求处理模块**
4.  **数据库模块**
5.  **外部模块**


结合MVC模式，可以在请求处理模块这个大模块中进行分层，专门分出View层（使用模版来生成HTML也好，直接生成也好），Controller层（进行一定的业务逻辑判断），Model层（一般抽象出来的类来封装数据，包括了数据库，IO，外部数据等）。

简单的说明一下：首先需要一个HTTP服务器模块，通过监听计算机的端口（一般是80端口）来发现所有对80端口的访问。正如上面的http.listen()一样。 

```javascript
var express = require('./node_modules/express/lib/express');
var app = express();
app.listen(80);
```

接下来，你便需要一个简单的路由模块。所谓路由，就是把对不同URL的访问根据URL的不同，指派给不同的请求处理程序来进行相应功能的使用。比如`http://dreampiggy.com/help`这个URL，你可以获取/help来指定用一个专门负责帮助说明的请求处理程序，返回一个关于帮助说明的HTML页面。对Web应用非常重要的Ajax访问的接口一般也是特定的URL，当然，需要POST/GET过来一定的AppID和Session来防止其他人恶意调用。 

```javascript
if(pathname == "/"){
	handler.home(request,response);
}
else{
	response.writeHead(404,{"Content-Type":"text/plain"});
	response.write("404 not found");
	response.end();
}
```

路由的目的一般就是把URL请求分离出各参数，然后向请求处理程序传递，不推荐在这个模块进行过多的判断处理。对于请求处理程序而言，就是主要我们需要写Web应用的地方。我们可以通过MVC的方式进行分层，这时候就是你大展伸手的时候了。请求处理程序的Model层少不了和数据库打交道，不过我还是推荐把数据库访问抽象出来，放到一个新的模块，用请求处理模块来调用它。Node.js的模块导出和函数导出也是非常常见的，合理运用让自己的Web应用逻辑更为清晰。

外部模块包含很多，比如你要对XSS注入加以防范，可以通过一定的外部XSS模块或者自己手写一个XSS来使用，然后在请求处理程序的View层进行过滤（当然这种做法不是很好，简单举例）。使用Node.js的一大好处就是你可以随时加入很多非常好的他人写好的模块（npm大法），而且对于不满意的地方自己进行修改，可以加快开发进度（比如我这个协作编辑平台需要的Markdown语法解析器就是用他人的加以改造的）

题外话，对于习惯OO的人来说，JavaScript在继承，多态方面等常见OO的用法上面比较薄弱，而且又由于JavaScript的弱类型语言，你不能使用重载函数来对不同的参数进行判断，所以建议对于Node.js来说，可以使用比如typeof来进行判断再加以封装，多态方面可以通过虚函数来使用。JavaScript透露出的函数式编程还需要我一点点理解，正所谓学什么语言用什么语言思考，谨记不要随意就把PHP/Java的套路往JavaScript上套。

就到这吧……或许几个月回过头来看这篇文章基本是废话，不过这确实是现在的我的一点感悟。

**学才能深入，才能知不足。**