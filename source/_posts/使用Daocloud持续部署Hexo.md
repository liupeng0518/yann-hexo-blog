---
title: 使用Daocloud
date: 2016-05-26 12:02:44
tags: docker daocloud ci lighttpd
---

开始写博客，不太喜欢CSDN之类的技术博客，还是喜欢维护自己的博客网站，买了一个域名也一直没用，虽然手上也有一个ECS，但是搭建一个WordPress真的挺麻烦，而且只是想做一个安静的美男子。选来选取还是全静态的博客比较适合我这样的懒人，现在Github page使用的静态博客很轻便，大致上现在就是Jekyll 和 Hexo [优缺点对比](https://www.zhihu.com/question/21981094). 总而言之，咱们也不能沦为设备党，还是应该以内容为主。


静态博客的核心就在于整个生成的网站是全静态，没有任何与服务器交互的部分，所以服务器只要提供HTTP服务即可，所以可以部署在Github, Coding, BAE最便宜的静态实例之上。这些都是网上的教程很多，大家百度一波就好了，就在我把自己的HEXO博客挂在BAE的静态实例之上，就在考虑为什么不把博客放在Daoclou之上呢，Docker的容器化简直是现在是最火的方向。

整理下思路，大致上也就是

1. Step 1 ：打包一个Http server
2. Step 2 ：把Pages放进web目录
3. Step 3 ：启动服务器

思路搞清楚，那我们就开始搞起来吧，