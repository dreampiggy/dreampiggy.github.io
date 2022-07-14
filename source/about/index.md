---
title: 关于
date: 2016-11-14 23:31:03
type: "about"
comments: false
---

> 迁移Again (2016/11/1)

我也是了解了国内这些云主机(PaaS)，云服务(SaaS)的作风了。之前使用的[DaoCloud](https://www.daocloud.io)，现在已经强制把所有免费的仅仅256MB的一个空间，变成了测试环境，每两天自动关闭应用，如果两周内再不管直接删数据库，也是见识了。

不多说，这次直接换静态博客Hexo，托管到GitHub Pages上，主题就用[Next](https://github.com/iissnan/hexo-theme-next)主题，真心不想再折腾了（代码高亮、评论、LaTeX支持什么都费时间），以后纯粹慢慢写博文吧，毕竟还是要注重技术博客的技术，而不是这些表面上的东西。


> 第二次迁移

由于Coding现在也改为了收费机制（每天1块，想想以前的永久免费承诺，呵呵……），所以不得不再次选择迁移。本来也打算用阿里云主机直接跑，但是想想备案之路的异常艰辛，于是只能找到了国内另一家PaaS提供商：[DaoCloud](https://www.daocloud.io)

DaoCloud允许免费的Docker镜像的部署，幸好我当时采用了七牛云来放图片资源，所以迁移异常轻松，简单写个Dockerfile，改下配置文件即可。

```docker
FROM node:0.12.14

# Create work directory
RUN mkdir -p /usr/local/app/
WORKDIR /usr/local/app/

# Install dependence
COPY . /usr/local/app/
RUN npm install --unsafe-perm --registry=https://registry.npm.taobao.org

# Expose port
EXPOSE 80

# Start
CMD ["npm", "start"]
```

同时也支持对未备案域名的绑定（自动解析到海外），免去备案因素，比起Coding，由于支持Docker的部署，相当于所有类型应用都可以Docker化以后来部署，特别方便（代价是每次改动主题什么代码，部署可能得花10分钟……不过保证稳定而且不会出错）

唉……现在想想如果之后这些PaaS提供商再抛弃用户，还是老老实实去备案了，真实服了国内的这种"良好的互联网监管机制"。

> 第一次迁移

我原先的博客是架设在[新浪云](http://www.sinacloud.com)（原先的SAE）上的，使用的是WordPress，用的PHP5.3。自己在别人基础上改进了主题和插件，也写了很多文章。然而，由于新浪云自从2016/3/21开始，对所有用户的数据库征收0.48元/天的租金（无论是否使用），并对应用实例征收0.1元/天/用户的租金，相当于一天0.58元，而一年下来总计211.7元。已经远远超出了一个空间提供商的费用，逼近主机提供商，于是看到Coding承诺对老用户的演示项目永不收费，便打算到Coding上来部署新的博客

![http://www.sinacloud.com/index/price.html](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-07-14/image/d/6d/77bd41d3257dc83b3345b8af594af.png)

而同时我也早已对WordPress不满，落后的管理方式（主题->博客->内容）和不完全的GitHub Markdown和LaTeX支持（也还是要靠插件），过于繁冗，加之新浪云的PHP空间是禁止在线上传代码和写入服务器本地磁盘的，导致很多插件无法正常使用，于是最终还是决定全部迁移到现在流行的Ghost博客上面

> Ghost的优点

+ 完全支持GitHub Markdown语法，对熟悉GitHub的md格式的人很方便

+ 方便集成[Disqus](http://www.disqus.com)，告别国内多说的各种推广广告，而且国际通用，自动同步，很不错

+ 简单的LaTeX支持，使用[MathJax](http://www.mathjax.org)来简单支持LaTeX，我配置的是使用

```latex
$ \latex $ 来行内渲染
$$ \latex $$ 来单独块渲染
```

+ 完善代码着色
采取了[Prismjs](http://prismjs.com)来支持超级全面的代码着色，无论是

```swift
print("Hello Swift")
```

还是

```haskell
qsort :: (Ord a Bool) => [a] -> [a]
qsort [] = []
qsort (x:xs) = qsort (filter (<= x) xs) ++ [x] ++ qsort (filter (> x) xs)
```

更可能是

```nasm
section     .text
global      _start                              ;must be declared for linker (ld)

_start:                                         ;tell linker entry point

    mov     edx,len                             ;message length
    mov     ecx,msg                             ;message to write
    mov     ebx,1                               ;file descriptor (stdout)
    mov     eax,4                               ;system call number (sys_write)
    int     0x80                                ;call kernel

    mov     eax,1                               ;system call number (sys_exit)
    int     0x80                                ;call kernel

section     .data

msg     db  'Hello, world!',0xa                 ;our dear string
len     equ $ - msg                             ;length of our dear string
```

都能正确着色，还有语言名提示和其他插件支持，非常爽

+ 图片存储和邮箱

这也是Coding的缺陷，由于Coding演示不支持文件持久化（虽然能运行时写入了，但是一重新部署全部丢失），所以还是把图片等资源放在国内的[七牛云](http://www.qiniu.com)上，而且迁移方便，邮箱服务器就用Gmail就行。

> 最后吐槽

Coding竟然不支持绑定多个域名或者子域名，只能绑`www`域名了……唉，免费总是没有好处，主要是不想再用一个大陆以外的主机单独跑博客（Ghost用的Node.js还是比较吃资源的）

什么？你说国内主机？我是备案一生黑。