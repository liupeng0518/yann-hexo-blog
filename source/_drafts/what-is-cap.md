---
title: 什么是CAP理论
date: 2018-04-02 17:56:26
tags:
---

在蚂蚁金服的面试中关于CAP理论引申出来的问题很多，也暴露了对分布式理论不足的缺点，重新学习下吧……

<!--more-->

## CAP 基础

[CAP原则](https://baike.baidu.com/item/CAP%E5%8E%9F%E5%88%99/5712863)
![CAP](https://gss3.bdstatic.com/7Po3dSag_xI4khGkpoWK1HF6hhy/baike/c0%3Dbaike92%2C5%2C5%2C92%2C30/sign=d060223b7cc6a7efad2ba0749c93c434/5bafa40f4bfbfbed9c15b19b72f0f736aec31f81.jpg)

一图胜过千言万语，也就是在CAP的三个目标中，我们没有办法同时满足三者，只有可能满足其中两者。那我们根据排列组合，我们就会有  CA 和 CP 和 AP 这样的三种组合。

## 满足CA的系统
满足CA的系统，也就是既要保证一致性又满足可用性，那最简单也就是所谓的单机数据库，单实例的Mysql和Oracle都属于这个范畴。

## 满足AP的系统
系统只要每次对写都返回成功，对读都返回固定的某个值就可以了。这样的系统是没有意义的。

## 满足CP的系统
这个其实也就是我们最常用的Zookeeper这样的一致性服务啦，某一个节点挂掉或者在重启的过程中都会放弃提供服务直到系统恢复正常。