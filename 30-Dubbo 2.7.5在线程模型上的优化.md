这是why技术的第30篇原创文章
![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0d439fb896e7?w=1280&h=853&f=jpeg&s=188928)

这可能是全网第一篇解析Dubbo 2.7.5里程碑版本中的改进点之一：客户端线程模型优化的文章。

**先劝退：文本共计8190字，54张图。阅读之前需要对Dubbo相关知识点有一定的基础。内容比较硬核，劝君谨慎阅读。**


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0d4b8ddc958a?w=157&h=53&f=png&s=3196)

**读不下去不要紧，我写的真的很辛苦的，帮忙拉到最后点个赞吧。**


# 本文目录

## 第一节：官方发布

本小节主要是通过官方发布的一篇名为《Dubbo 发布里程碑版本，性能提升30%》的文章作为引子，引出本文所要分享的内容：客户端线程模型优化。


## 第二节：官网上的介绍

在介绍优化后的消费端线程模型之前，先简单的介绍一下Dubbo的线程模型是什么。同时发现官方文档对于该部分的介绍十分简略，所以结合代码对其进行补充说明。


## 第三节：2.7.5版本之前的线程模型的问题

通过一个issue串联本小节，道出并分析一些消费端应用，当面临需要消费大量服务且并发数比较大的大流量场景时（典型如网关类场景），经常会出现消费端线程数分配过多的问题。


## 第四节：thredless是什么

通过第三节引出了新版本的解决方案，thredless。并对其进行一个简单的介绍。


## 第五节：场景复现

由于条件有限，场景复现起来比较麻烦，但是我在issues#890中发现了一个很好的终结，所以我搬过来了。


## 第六节：新旧线程模型对比

本小节通过对比新老线程模型的调用流程，并对比2.7.4.1版本和2.7.5版本关键的代码，起到一个导读的作用。



## 第七节：Dubbo版本介绍。

趁着这次的版本升级，也趁机介绍一下Dubbo目前的两个主要版本：2.6.X和2.7.X。



# 官方发布

2020年1月9日，阿里巴巴中间件发布名为[《Dubbo 发布里程碑版本，性能提升30%》](https://mp.weixin.qq.com/s/V5S_vO6Mgtq9-v2ed9eDrw)的文章：

![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0d5bb85df61e?w=709&h=731&f=jpeg&s=213220)


文章中说这是**Dubbo的一个里程碑式的版本。**



在阅读了相关内容后，**我发现这确实是一个里程碑式的跨域，对于Dubbo坎坷的一生来说，这是展现其强大的生命力和积极探索精神的一个版本。**



强大的生命力体现在新版本发布后众多的或赞扬、或吐槽的社区反馈。


探索精神体现在Dubbo在多语言和协议穿透性上的探索。


在文章中列举了9大改造点，本文仅介绍2.7.5版本中的一个改造点：**优化后的消费端线程模型。**



本文大部分源码为2.7.5版本，同时也会有2.7.4.1版本的源码作为对比。

# 官网上的介绍

在介绍优化后的消费端线程模型之前，先简单的介绍一下Dubbo的线程模型是什么。

直接看官方文档中的描述，Dubbo官方文档是一份非常不错的入门学习的文档，很多知识点都写的非常详细。

可惜，在线程模型这块，差强人意，寥寥数语，图不达意：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0d9fb29ce3ca?w=781&h=767&f=jpeg&s=216300)

官方的配图中，完全没有体现出线程"池"的概念，也没有体现出同步转异步的调用链路。**仅仅是一个远程调用请求的发送与接收过程，至于响应的发送与接收过程，这张图中也没有表现出来。**



所以我结合官方文档和2.7.5版本的源码进行一个简要的介绍，在阅读源码的过程中你会发现：


**在客户端**，除了用户线程外，还会有一个线程名称为DubboClientHandler-ip:port的线程池，其默认实现是cache线程池。


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0da60b829140?w=841&h=386&f=png&s=79966)

**上图的第93行代码的含义是，当客户端没有指定threadpool时，采用cached实现方式。**



上图中的setThreadName方法，就是设置线程名称：

`org.apache.dubbo.common.utils.ExecutorUtil#setThreadName`


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0db622393376?w=595&h=331&f=png&s=52177)

可以清楚的看到，线程名称如果没有指定时，默认是DubboClientHandler-ip:port。


**在服务端**，除了有boss线程、worker线程（io线程），还有一个线程名称为DubboServerHandler-ip:port的线程池，其默认实现是fixed线程池。


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0dc293408d10?w=1332&h=642&f=jpeg&s=260806)



启用线程池的dubbo.xml配置如下：

`<dubbo:protocol name="dubbo" threadpool="xxx"/>`

上面的xxx可以是fixed、cached、limited、eager，其中fixed是默认实现。当然由于是SPI，所以也可以自行扩展：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0dc495a0c603?w=633&h=544&f=png&s=47852)


所以，基于最新2.7.5版本，官方文档下面红框框起来的这个地方，描述的有误导性：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0dc5bea577d9?w=782&h=256&f=png&s=49831)

从SPI接口看来，fixed确实是缺省值。


但是由于客户端在初始化线程池之前，加了一行代码（之前说的93行），**所以客户端的默认实现是cached，服务端的默认实现是fixed。**



我也看了之前的版本，至少在2.6.0时（更早之前的版本没有查看），客户端的线程池的默认实现就是cached。


关于Dispatcher部分的描述是没有问题的：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0dca3c5bc6ec?w=766&h=210&f=png&s=40889)

Dispatcher部分是线程模型中一个比较重要的点，后面会提到。


这里配一个稍微详细一点的**2.7.5版本之前的线程模型**，供大家参考：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0de24f5a6fe9?w=1720&h=680&f=jpeg&s=122682)

`图片来源:https://github.com/apache/dubbo/issues/890`


# 2.7.5之前的线程模型的问题

那么改进之前的线程模型到底存在什么样的问题呢？


在《Dubbo 发布里程碑版本，性能提升30%》一文中，是这样描述的：

`对 2.7.5 版本之前的 Dubbo 应用，尤其是一些消费端应用，当面临需要消费大量服务且并发数比较大的大流量场景时（典型如网关类场景），经常会出现消费端线程数分配过多的问题。`

同时文章给出了一个issue的链接：

`https://github.com/apache/dubbo/issues/2013`

这一小节，我就顺着这个issue#2013给大家捋一下Dubbo 2.7.5版本之前的线程模型存在的问题，准确的说，是客户端线程模型存在的问题：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0de3ce0585af?w=741&h=388&f=png&s=49413)

首先，Jaskey说到，分析了issue#1932，他说在某些情况下，会创建非常多的线程，因此进程会出现OOM的问题。


**在分析了这个问题之后，他发现客户端使用了一个缓存线程池(就是我们前面说的客户端线程实现方式是cached)，它并没有限制线程大小，这是根本原因。**



接下来，我们去issue#1932看看是怎么说的：

`https://github.com/apache/dubbo/issues/1932`


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0de962514e71?w=995&h=981&f=jpeg&s=436818)


可以看到issue#1932也是Jaskey提出的，他主要传达了一个意思：**为什么我设置了actives=20，但是在客户端却有超过10000个线程名称为DubboClientHandler的线程的状态为blocked？这是不是一个Bug呢？**



仅就这个issue，我先回答一下这个：不是Bug！



我们先看看actives=20的含义是什么：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0ded45002583?w=901&h=197&f=png&s=18112)

按照官网上的解释：**actives=20的含义是每个服务消费者每个方法最大并发调用数为20。**



也就是说，服务端提供一个方法，客户端调用该方法，同时最多允许20个请求调用，但是客户端的线程模型是cached，接受到请求后，可以把请求都缓存到线程池中去。所以在大量的比较耗时的请求的场景下，客户端的线程数远远超过20。


这个actives配置在[《一文讲透Dubbo负载均衡之最小活跃数算法》](https://mp.weixin.qq.com/s/mxoARgB-1jyA9Kf4cRVP-g)这篇文章中也有说明。它的生效需要配合ActiveLimitFilter过滤器，actives的默认值为0，表示不限制。当actives>0时，ActiveLimitFilter自动生效。由于不是本文重点，就不在这里详细说明了，有兴趣的可以阅读之前的文章。




顺着issue#2013捋下去，我们可以看到issue#1896提到的这个问题：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0dfa56c5bb51?w=1008&h=890&f=jpeg&s=311693)

问题1我已经在前面解释了，他这里的猜测前半句对，后半句错。不再多说。


这里主要看问题2（可以点开大图看看）：服务提供者多了，消费端维护的线程池就多了。导致虽然服务提供者的能力大了，但是消费端有了巨大的线程消耗。他和下面issue#4467的哥们表达的是同一个意思：想要的是一个共享的线程池。



我们接着往下捋，可以发现issue#4467和issue#5490


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0dfd682d7434?w=758&h=571&f=png&s=77323)


对于issue#4467，CodingSinger说：**为什么Dubbo对每一个链接都创建一个线程池？**


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0dffba0611b8?w=752&h=391&f=png&s=42641)


从Dubbo 2.7.4.1的源码我们也可以看到确实是在WarppedChannelHandler构造函数里面确实是为每一个连接都创建了一个线程池：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e0269f5a305?w=1267&h=340&f=png&s=62053)

issue#4467想要表达的是什么意思呢？


就是这个地方为什么要做链接级别的线程隔离，一个客户端，就算有多个连接都应该用共享线程池呀？



**我个人也觉得这个地方不应该做线程隔离。线程隔离的使用场景应该是针对一些特别重要的方法或者特别慢的方法或者功能差异较大的方法。很显然，Dubbo的客户端就算一个方法有多个连接（配置了connections参数），也是一视同仁，不太符合线程隔离的使用场景。**



然后chickenij大佬在2019年7月24日回复了这个issue：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e061a4d2337?w=745&h=754&f=png&s=77955)

现有的设计就是：**provider端默认共用一个线程池。consumer端是每个链接共享一个线程池。**



同时他也说了：**对于consumer线程池，当前正在尝试优化中。**



言外之意是他也觉得现有的consumer端的线程模型也是有优化空间的。


这里插一句：chickenlj是谁呢？

`刘军，GitHub账号Chickenlj，Apache Dubbo PMC，项目核心维护者，见证了Dubbo从重启开源到Apache毕业的整个流程。现任职阿里云云原生应用平台团队，参与服务框架、微服务相关工作，目前主要在推动Dubbo开源的云原生化。`


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e0dba666f39?w=413&h=75&f=png&s=10173)

他这篇文章的作者呀，他的话还是很有分量的。



之前也在Dubbo开发者日成都站听到过他的分享：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e0f54c589cf?w=891&h=225&f=jpeg&s=54958)

如果对他演讲的内容有兴趣的朋友可以**在公众号的后台回复:1026。领取讲师PPT和录播地址。**



好了，我们接着往下看之前提到的issue#5490，**刘军大佬在2019年12月16日就说了，在2.7.5版本时会引入threadless executor机制，用于优化、增强客户端线程模型。**

![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e137313db9f?w=866&h=563&f=png&s=75897)




# threadless是什么？


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e163fd7e3a1?w=1566&h=390&f=png&s=68579)


根据类上的说明我们可以知道：



这个Executor和其他正常Executor之间最重要的区别是**这个Executor不管理任何线程。**



通过execute(Runnable)方法提交给这个执行器的任务不会被调度到特定线程，而其他的Executor就把Runnable交给线程去执行了。


这些任务存储在阻塞队列中，只有当thead调用waitAndDrain()方法时才会真正执行。简单来说就是，**执行task的thead与调用waitAndDrain()方法的thead完全相同。**



其中说到的waitAndDrain()方法如下:


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e194115b108?w=788&h=512&f=png&s=36115)


execute(Runnable)方法如下：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e1bd4d04716?w=1105&h=384&f=png&s=34371)

同时我们还可以看到，里面还维护了一个名称叫做sharedExecutor的线程池。见名知意，我们就知道了，这里应该是要做线程池共享了。


# 场景复现

上面说了这么多2.7.5版本之前的线程模型的问题，我们怎么复现一次呢？



我这里条件有限，场景复现起来比较麻烦，但是我在issues#890中发现了一个很好的终结，我搬过来即可：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e3780ce7537?w=756&h=137&f=png&s=27203)

根据他接下来的描述做出思维导图如下：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e394f5f2b5c?w=1637&h=447&f=png&s=101641)




上面说的是corethreads大于0的场景。但是**根据现有的线程模型，即使核心池数(corethreads)为0，当消费者应用依赖的服务提供者处理很慢时且请求并发量比较大时，也会出现消费者线程数很多问题**。大家可以对比着看一下。


# 新旧线程模型对比

在之前的介绍中大家已经知道了，这次升级主要是增强客户端线程模型，所以关于2.7.5版本之前和之后的线程池模型我们主要关心Consumer部分。


## 老的线程模型

老的线程池模型如下，注意线条颜色：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e3eade21568?w=1080&h=634&f=jpeg&s=68421)

1、业务线程发出请求，拿到一个 Future 实例。

2、业务线程紧接着调用 future.get 阻塞等待业务结果返回。
3、当业务数据返回后，交由独立的 Consumer 端线程池进行反序列化等处理，并调用 future.set 将反序列化后的业务结果置回。
4、业务线程拿到结果直接返回。


## 新的线程模型

新的线程池模型如下，注意线条颜色：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e40b4c8af28?w=1080&h=806&f=jpeg&s=74610)

1、业务线程发出请求，拿到一个 Future 实例。
2、在调用 future.get() 之前，先调用 ThreadlessExecutor.wait()，wait 会使业务线程在一个阻塞队列上等待，直到队列中被加入元素。
3、当业务数据返回后，生成一个 Runnable Task 并放ThreadlessExecutor 队列。
4、业务线程将 Task 取出并在本线程中执行反序列化业务数据并 set 到 Future。
5、业务线程拿到结果直接返回。


可以看到，相比于老的线程池模型，新的线程模型由业务线程自己负责监测并解析返回结果，免去了额外的消费端线程池开销。


## 代码对比

接下来我们**对比一下2.7.4.1版本和2.7.5版本的代码**，来说明上面的变化。



**需要注意的是，由于涉及到的变化代码非常的多，我这里仅仅起到一个导读的作用，如果读者想要详细了解相关变化，还需要自己仔细阅读源码**。



首先两个版本的第一步是一样的：**业务线程发出请求，拿到一个Future实例。**



但是实现代码却有所差异，在2.7.4.1版本中，如下代码所示：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e48c76bdfcc?w=1157&h=697&f=jpeg&s=203379)

上图圈起来的request方法最终会走到这个地方，可以看到确实是返回了一个Future实例：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e49c786cbe2?w=1000&h=490&f=png&s=77812)



而newFuture方法源码如下，**请记住这个方法，后面会进行对比：**


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e4c9a918565?w=908&h=421&f=png&s=50041)



同时通过源码可以看到在获取到Future实例后，紧接着调用了subscribeTo方法，实现方法如下：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e506d88e58c?w=619&h=273&f=png&s=27927)

用了Java 8的CompletableFuture，实现异步编程。


但是在2.7.5版本中，如下代码所示：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e53b3eeb125?w=1167&h=742&f=jpeg&s=238323)

在request方法中多了个executor参数，而该参数就是的实现类就是ThreadlessExecutor。


接下来，和之前的版本一样，会通过newFuture方法去获取一个DefaultFuture对象：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e561bf9d6a2?w=1154&h=535&f=png&s=90879)

通过和2.7.4.1版本的newFuture方法对比你会发现这个地方就大不一样了。**虽然都是获取Future，但是Future里面的内容不一样了。**



直接上个代码对比图，一目了然：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e58a00d32b3?w=1020&h=422&f=png&s=46922)


**第二步：业务线程紧接着调用 future.get 阻塞等待业务结果返回。**



由于Dubbo默认是同步调用，而同步和异步调用的区别我在第一篇文章[《Dubbo 2.7新特性之异步化改造》](https://mp.weixin.qq.com/s/_WPuFsjNRjLc2vQnXam8Gg)中就进行了详细解析：



![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e5c5daef9f8?w=602&h=275&f=png&s=67328)


我们找到异步转同步的地方，先看2.7.4.1版本的如下代码所示：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e695c7c100f?w=821&h=573&f=png&s=78202)


而这里的asyncResult.get()对应的源码是，CompletableFuture.get()：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e6b4f49143d?w=798&h=487&f=png&s=71180)


而在2.7.5版本中对应的地方发生了变化：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e6c5a93adb3?w=1126&h=679&f=jpeg&s=176791)


**变化就在这个asyncResult.get方法上。**


在2.7.5版本中，该方法的实现源码是：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e6f4482a75b?w=1159&h=250&f=png&s=36505)

先说标号为②的地方，和2.7.4.1版本是一样的，都是调用的CompletableFuture.get()。但是多了标号为①的代码逻辑。而**这段代码就是之前新的线程模型里面体现的地方，下面红框框起来的部分：**


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e768ec2a7d7?w=614&h=472&f=jpeg&s=43942)


**在调用 future.get() 之前（即调用标号为②的代码之前），先调用 ThreadlessExecutor.wait()（即标号为①处的逻辑），wait 会使业务线程在一个阻塞队列上等待，直到队列中被加入元素。**



接下来再对比两个地方：


**第一个地方：之前提到的WrappedChannelHandler**，可以看到2.7.5版本其构造函数的改造非常大：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e7a7860ead3?w=1266&h=514&f=png&s=93246)


**第二个地方**：之前提到的Dispatcher，是需要再写一篇文章才能说的清楚的，我这仅仅是做一个抛砖引玉，提一下：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e7cee2d10f2?w=1372&h=910&f=jpeg&s=314821)



AllChannelHandler是默认的策略，证明代码如下：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e7e413d5fae?w=649&h=310&f=png&s=21234)

首先还是看标号为②的地方，看起来变化很大，其实就是对代码进行了一个抽离，封装。sendFeedback方法如下，和2.7.4.1版本中标号为②的地方的代码是一样的：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e80420c5f46?w=1049&h=321&f=png&s=46658)

所以我们重点对比一下两个标号为①的地方，它们获取executor的方法变了：
```
2.7.4.1版本的方法是getExecutorService()
2.7.5版本的方法是getPreferredExecutorService()
```
代码如下，大家品一品两个版本之前的差异：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e855c45a085?w=1111&h=1080&f=jpeg&s=284695)

主要翻译一下getPreferredExecutorService方法上的注释：
```
Currently, this method is mainly customized to facilitate the thread model on consumer side.
1. Use ThreadlessExecutor, aka., delegate callback directly to the thread initiating the call.   
2. Use shared executor to execute the callback.
```
目前，使用这种方法主要是为了客户端的线程模型而定制的。

1.使用ThreadlessExceutor，aka.，将回调直接委托给发起调用的线程。
2.使用shared executor执行回调。


```小声说一句：这里这个aka怎么翻译，我实在是不知道了。难道是嘻哈里面的AKA？大家好，我是宝石GEM，aka(又名) 你的老舅。又画彩虹又画龙的。```

![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e95fe32fc82?w=304&h=89&f=png&s=22521)



好了，导读就到这里了。**能看到这个地方的人我相信已经不多了。还是之前那句话由于涉及到的变化代码非常的多，我这里仅仅起到一个导读的作用，如果读者想要详细了解相关变化，还需要自己仔细阅读源码。希望你能自己搭个Demo跑一跑，对比一下两个版本的差异。**



# Dubbo版本介绍

趁着这次的版本升级，也趁机介绍一下Dubbo目前的主要版本吧。


据刘军大佬的分享：Dubbo 社区目前主力维护的有 2.6.x 和 2.7.x 两大版本，其中：


**2.6.x 主要以 bugfix 和少量 enhancements 为主，因此能完全保证稳定性。**



**2.7.x 作为社区的主要开发版本，得到持续更新并增加了大量新 feature 和优化，同时也带来了一些稳定性挑战。**



为方便 Dubbo 用户升级，社区在以下表格对 Dubbo 的各个版本进行了总结，包括主要功能、稳定性和兼容性等，从多个方面评估每个版本，以期能帮助用户完成升级评估：


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e9c98024201?w=660&h=589&f=png&s=56185)


![](https://user-gold-cdn.xitu.io/2020/1/20/16fc0e9d9796c107?w=532&h=199&f=png&s=20177)


可以看到社区对于最新的2.7.5版本的升级建议是：**不建议大规模生产使用。**

同时你去看Dubbo最新的issue，有很多都是对于2.7.5版本的"吐槽"。


但是我倒是觉得2.7.5是Dubbo发展进程中浓墨重彩的一笔，该版本打响了对于 Dubbo向整个微服务云原生体系靠齐的第一枪。对于多语言的支持方向的探索。实现了对 HTTP/2 协议的支持，同时增加了与 Protobuf 的结合。

**开源项目，共同维护。我们当然知道Dubbo不是一个完美的框架，但是我们也知道，它的背后有一群知道它不完美，但是仍然不言乏力、不言放弃的工程师，他们在努力改造它，让它趋于完美。我们作为使用者，我们少一点"吐槽"，多一点鼓励。只有这样我们才能骄傲的说，我们为开源世界贡献了一点点的力量，我们相信它的明天会更好。**



向开源致敬，向开源工程师致敬。


总之，牛逼。

# 最后说一句


才疏学浅，难免会有纰漏，如果你发现了错误的地方，还请你留言给我指出来，我对其加以修改。

感谢您的阅读，**我坚持原创**，十分欢迎并感谢您的关注。

以上。

欢迎关注公众号【why技术】。在这里我会分享一些技术相关的东西，主攻java方向，用匠心敲代码，对每一行代码负责。偶尔也会荒腔走板的聊一聊生活，写一写书评，影评。愿你我共同进步。

![公众号-why技术](https://user-gold-cdn.xitu.io/2019/10/22/16df1d25d396c5c6?w=258&h=258&f=png&s=44160)  
