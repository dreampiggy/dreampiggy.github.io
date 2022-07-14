---
title: Web项目3-基于Node.js的多人协作编辑平台的设计实现
categories: Web
tags:
  - Web
  - Node.js
  - JavaScript
updated: '2016-03-30 01:30:31'
date: 2016-03-30 01:19:21
---

终于，我参加的第三个Web项目基本已经完成了Beta版本的整体架构和实现。

这次的Web项目要求，是制作一个基于Web（或者移动Web）的在线协作编辑平台，主要支持Markdown语法及同步预览，类似于<a title="Google Docs" href="https://docs.google.com/document" target="_blank">Google Docs</a>或者<a title="Office Online" href="https://office.live.com/start/default.aspx" target="_blank">Office Online</a>这样的东西（但是绝对不可能达到国际一流大厂的水平）

由于只有我们4个人开发（其中2个还算专门前端开发），所以尽可能减少后端开发压力，经过初步的讨论，最终选择了使用Node.js作为后端开发语言。Node.js强力的非阻塞IO和异步事件非常适合我们这种IO密集型的应用。而对于协作的部分，我们选择使用<a title="socket.io" href="http://socket.io/" target="_blank">socket.io</a>来进行这种密集文本数据的通信和广播，也能很大的减少开发周期。

嘛，整体架构大概这样，虽然很丑陋，对于我这种没有实际架构Web经验以及Beta版来说，已经足够了。

![](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-07-14/image/e/bc/22d0e801243e1c22305ce85f897b3.png)

话不多说，作为一个多人编辑的平台，首先就需要传统的Web应用的老三样：用户管理，文档管理，项目管理，今天就主要说这些。


### **一、用户管理**

作为最基础的Web提供的服务，也许不用多说，用PHP当年也是基本这个样子，包括注册登录，信息验证，加上增删改查(REST) <span style="font-size: 13px;"> </span> 借助于Node的模块导出(Module.exports)，我们可以把不同层次的模块分层，比如说，我们可以建立一个user.js模块，存储底层的对user数据库的操作，在上层的account.js模块中直接进行整合，达到上层的用户注册，登陆，编辑，查询，删除，邀请，接受，拒绝的功能。

![](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-07-14/image/4/2c/cbc5296c83216b037c0bd6ac607d0.png)

![](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-07-14/image/8/fd/7de6621bfaa012093edbdc6194923.png)

上篇已经说过，Node.js是异步事件编程的，所有对数据库的访问都是异步的，也就是说你不能用同步编程控制顺序来进行验证。怎么办呢？两种方法，要么用回调函数处理（对于著名的callback hell问题，可以使用<a title="async" href="https://github.com/caolan/async" target="_blank">async</a>或者Promises处理，如<a title="Q" href="https://github.com/kriskowal/q" target="_blank">Q</a>），个人的丑陋方法是通过将数据库查询结果传入回调函数来进行调用，分层调用起来也能向面向对象时候使用对象调用一样方便。

信息验证，主要就是指Session，关于这个可以在之后写一下，在此只提供方法：我使用Redis数据库来存取sessionID，存入一个Hashmap中，顺便可以绑定一些常用的信息，比如验证码，状态，减少对MySQL数据库的压力。相关的用法可以查阅：<a title="node-redis" href="https://github.com/mranney/node_redis" target="_blank">node-redis</a>、<a title="express-session" href="https://github.com/expressjs/session" target="_blank">express-session</a>、<a title="cookie-parser" href="https://github.com/expressjs/cookie-parser" target="_blank">cookie-parser</a>等

![](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-07-14/image/0/fd/64e2f6885fda48496392b72787e42.png)

### **二、文档管理**

做为比较重要的文档管理，其实也莫过于增删改查四个接口，只不过所有修改你当然需要对用户进行验证，所以这里在设计时需要数据库同步user表的内容到docs表，而且验证的时候可以复用user.js模块中的验证。
![](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-07-14/image/9/b4/c5f6392ba72369c19f6ffb1eaa5ef.png)

这里现在是第一版的，之后我会写怎么讲markdown文本内容转入Redis数据库，来满足实时协作编辑的需要。甚至为了功能可以提供一个预览文档的功能，在Redis中进行缓存，可以快速的提取预览，减少对真正读写文档的IO压力。

### **三、项目管理**

项目管理，当然不再是简单的增删改查啦，不过还是REST那一套，你只需要复用user.js和docs.js中对于文档，对于用户的接口，然后再次封装一下，就可以满足文档的管理（主要是项目新建，删除，编辑，项目用户邀请，权限管理-这个暂时没做，项目中新建文档，查询文档，编辑文档，删除文档这几个接口），数据库表也就随意做一个

![](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-07-14/image/0/f2/f8a644c7bd838a96205147c28a7bf.png)

主要的接口大概就是这些，配合路由以及Session验证，就可以出来一个简单的API调用。对于我们这个项目，路由就是这种丑陋的写法：

```javascript
	app.post('/api/user/register',function(req,res){
		handler.userRegister(req,res);
	});
//User Login
	app.post('/api/user/login',function(req,res){
		handler.userLogin(req,res);
	});
//User Captcha
	app.get('/api/user/captcha',function(req,res){
		handler.userCaptcha(req,res);
	});
//User Logout
	app.post('/api/user/logout',function(req,res){
		handler.userLogout(req,res);
	})
//User Info
	app.post('/api/user/info',function(req,res){
		handler.userInfo(req,res);
	});
//User Invite
	app.post('/api/user/invite',function(req,res){
		handler.userInvite(req,res);
	});
//User Accept
	app.post('/api/user/accept',function(req,res){
		handler.userAccept(req,res);
	});
//User Reject
	app.post('/api/user/reject',function(req,res){
		handler.userReject(req,res);
	});
```

而最后handler负责对路由来的请求进行验证，进行缓存，以及进行POST参数提取，静态文件返回等等这种中间件的杂事，controller不需要request对象，model不需要request和response对象两者，这样也能减少藕合度，便于单元测试。（其实感觉这层和controller有重复，不过暂时先这样，加快开发）

一些小东西，比如说验证码，session处理什么的，灵活使用网上的node_modules进行封装，然后便可以快速使用了，想要了解底层实现也可以自己看看源码，对之后开发很重要。

说实话，Node开发比当年用PHP开发难度要大，因为你不可避免要接触到很多HTTP协议的规定，网络编程的知识，而PHP就是纯粹一个模版让你使用，不需要太多编程或者网络知识，对深入学习其实是很不利的。当年对于PHP中Session的理解就是`$_SESSION['name']`这样用，然后另一个也看就可以访问到，却不知道Session的原理，真是太天真了……

先到这里吧，下一篇主要说明

**如何设计协作以及对<a title="socket.io" href="http://socket.io" target="_blank">socket.io</a>的使用**，来满足实时协作编辑。