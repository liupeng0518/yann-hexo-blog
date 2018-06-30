---
title: Netty源码分析-(5)-其他
date: 2018-06-21 13:00:48
tags:
toc: true
categories: ["java", "网络编程", 'Netty']
---

# 其他部分

Netty最为核心部分我们已经大致上了解完了，Netty中还有一些其他可能会问到的知识点，我们在这里一并提一下。

<!-- more -->

## epoll空转bug

[JDK-6670302 : (se) NIO selector wakes up with 0 selected keys infinitely [lnx 2.4]
](http://bugs.java.com/bugdatabase/view_bug.do?bug_id=6670302) 这个Bug就是在Select阶段应该会阻塞的，但是JDK在实现调用系统底层函数，由于Epoll会获取不到任何时间，而Select又不断的在唤醒，最终导致CPU 100%

> 在部分Linux的2.6的kernel中，poll和epoll对于突然中断的连接socket会对返回的eventSet事件集合置为POLLHUP，也可能是POLLERR，eventSet事件集合发生了变化，这就可能导致Selector会被唤醒。==》这是与操作系统机制有关系的，JDK虽然仅仅是一个兼容各个操作系统平台的软件，但很遗憾在JDK5和JDK6最初的版本中（严格意义上来将，JDK部分版本都是），这个问题并没有解决，而将这个帽子抛给了操作系统方，这也就是这个bug最终一直到2013年才最终修复的原因，最终影响力太广。

Nett解决思路记录select空转的次数，定义一个阈值，这个阈值默认是512，可以在应用层通过设置系统属性io.netty.selectorAutoRebuldThreshold传入，当空转的次数超过了这个阈值，重新构建新Selector，将老Selector上注册的Channel转移到新建的Selector上，关闭老Selector，用新的Selector代替老Selector。

```java
private void select(boolean oldWakenUp) throws IOException {
    Selector selector = this.selector;
    try {
        int selectCnt = 0;
        long currentTimeNanos = System.nanoTime();
        long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);

        for (;;) {
            //略
            if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
                // timeoutMillis elapsed without anything selected.
                selectCnt = 1;
            } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                    selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) { ➊
                rebuildSelector(); ➋
                selector = this.selector; ➌

                // Select again to populate selectedKeys.
                selector.selectNow();
                selectCnt = 1;
                break;
            }

            currentTimeNanos = time;
        }
    } catch (CancelledKeyException e) {
    }
}
```

➊ 处我们发现 当空转的次数超过 `SELECTOR_AUTO_REBUILD_THRESHOLD` 就会触发下面这个流程分支
➋ 在此处重建 Selector 在 `io.netty.channel.nio.NioEventLoop#rebuildSelector0`
➌ 替换原有的 Selector