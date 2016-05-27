---
title: 使用Daocloud持续部署Hexo
date: 2016-05-26 12:02:44
tags: docker daocloud ci lighttpd
categories: Docker
---

开始写博客，不太喜欢CSDN之类的技术博客，还是喜欢维护自己的博客网站，买了一个域名也一直没用，虽然手上也有一个ECS，但是搭建一个WordPress真的挺麻烦，而且只是想做一个安静的美男子。选来选取还是全静态的博客比较适合我这样的懒人，现在Github page使用的静态博客很轻便，大致上现在就是Jekyll 和 Hexo [优缺点对比](https://www.zhihu.com/question/21981094). 总而言之，咱们也不能沦为设备党，还是应该以内容为主。

## 前言

静态博客的核心就在于整个生成的网站是全静态，没有任何与服务器交互的部分，所以服务器只要提供HTTP服务即可，所以可以部署在Github, Coding, BAE最便宜的静态实例之上。这些都是网上的教程很多，大家百度一波就好了，就在我把自己的HEXO博客挂在BAE的静态实例之上，就在考虑为什么不把博客放在Daoclou之上呢，Docker的容器化简直是现在是最火的方向。

整理下思路，大致上也就是

1. Step 1 ：打包一个Http server
2. Step 2 ：把Pages放进web目录
3. Step 3 ：启动服务器

思路搞清楚，那我们就开始搞起来吧，首先我们需要的是一个WebServer，Nginx大家都耳熟能详，我们仅仅需要的是一个web 服务器，所以我选择使用Lighttpd，为什么选择这个可能是看见Light，我觉得占用的内存会低很多，毕竟我们使用的是免费的资源，配置还是很一般的。

## 编写Dockerfile
Dockerfile

	FROM jprjr/lighttpd
	#添加静态页面
	ADD public/ /srv/http/
	#启动服务器
	ENTRYPOINT ["lighttpd", "-D", "-f", "/etc/lighttpd/lighttpd.conf"]

内容只有这么多，注意public目录是放置Hexo generate 生成的全静态页面目录

## 部署项目
首先当然将我们的代码上传至Coding or Github，建议对国有的支持，我选择了将整个项目创建于 Coding上，最终的项目如
[Coding-Hexo-项目](https://coding.net/u/yann/p/my-blog-page/git)

1. 创建一个新的构建
![创建新构建](http://cache.yannxia.info/%E5%88%9B%E5%BB%BA%E6%96%B0%E6%9E%84%E5%BB%BA.png)
![构建成功](http://cache.yannxia.info/%E6%9E%84%E5%BB%BA%E6%88%90%E5%8A%9F.png)
2. 构建成功后进行部署
![部署成功](http://cache.yannxia.info/%E6%9E%84%E5%BB%BA%E6%88%90%E5%8A%9F.png)
![绑定域名](http://cache.yannxia.info/%E7%BB%91%E5%AE%9A%E5%9F%9F%E5%90%8D.png)
最后就是一路下一步，等待成功就Ok



## 检验成果
从执行的域名就可以访问，
![访问域名](http://cache.yannxia.info/5D5B2352-D79F-4AE1-936B-8046C3B39490.png)


## 总结
整来来说这么使用Daocloud进行Docker部署和Bae部署Hexo的静态是一样的道理，也能做到持续集成，但是我们仍然和hexo的命令没有完整的结合在一起，下一步实现一个Hook或者使用Python编写一个基于主分支修改之后，Daocloud也可以自动更新的方案。

最后的最后，感谢Daocloud提供服务。
