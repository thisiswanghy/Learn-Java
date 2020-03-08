这是 why 技术的第 28 篇原创文章

之前在[《Dubbo 一致性哈希负载均衡的源码和 Bug，了解一下？》](https://juejin.im/post/5dedd263518825124b120298 "《Dubbo一致性哈希负载均衡的源码和Bug，了解一下？》")中写到了我发现了一个 Dubbo 一致性哈希负载均衡算法的 Bug。

对于解决方案我是这样写的：

`特别简单，把获取identityHashCode的方法从System.identityHashCode(invokers)修改为invokers.hashCode()即可。此方案是我提的issue里面的评论，这里System.identityHashCode和 hashCode之间的联系和区别就不进行展开讲述了，不清楚的大家可以自行了解一下。`

我说：**这里 System.identityHashCode 和 hashCode 之间的联系和区别就不进行展开讲述了，不清楚的大家可以自行了解一下。**

但是有读者在后台问我详细原因，我已经和他聊清楚了。

![](https://user-gold-cdn.xitu.io/2020/1/6/16f79156ac23676e?w=283&h=270&f=png&s=70182)
再加上这个**BUG 已于近期修复了，且只用了一行代码就修复了**，那我就写一下解决方案，以及背后的原理。

即是对之前文章的一个补充，也是一个独立的知识点。

所以本文主要是回答下面这三个问题：

**1.什么是 System.identityHashCode？**

**2.什么是 hashCode？**

**3.为什么一行代码就修复了这个 BUG？**

注：本文 Dubbo 源码 2.7.4.1 版本。如果阅读过[《Dubbo 一致性哈希负载均衡的源码和 Bug，了解一下？》](https://juejin.im/post/5dedd263518825124b120298 "《Dubbo一致性哈希负载均衡的源码和Bug，了解一下？》")可以更好的理解这篇文章。但是没有读过也不会影响阅读。

# 前情回顾

先通过一个前情回顾，引出本文所要分享的内容。

Dubbo 一致性哈希负载均衡算法的**设计初衷**应该是如果没有服务上下线的操作，后续请求根据已经映射好的哈希环进行处理，不需要重新映射。

然而我在研究其源码时，我发现**实际情况**是即使在服务端没有上下线操作的时候，一致性哈希负载均衡算法每次都需要重新进行 hash 环的映射。

**实际情况与设计初衷不符。**

于是给 Dubbo 提了一个 issue，地址如下：

`https://github.com/apache/dubbo/issues/5429`

![](https://user-gold-cdn.xitu.io/2020/1/6/16f79174f8bffa09?w=847&h=401&f=png&s=76413)

以下内容是对该 issue 的详细说明：

**在 Dubbo 对应的源码中，只需要一行代码。就可以判断是否有服务上下线的操作：**

![](https://user-gold-cdn.xitu.io/2020/1/6/16f79178c4638ee4?w=1323&h=442&f=png&s=91616)

就是下面这一行代码：

`int identityHashCode = System.identityHashCode(invokers);`

通过判断 invokers(服务提供方 List 集合)的 identityHashCode 是否发生了变化，从而判断是否有服务上下线的操作。

但是这行代码，在**Dubbo2.7.0 版本之后就失效了。**

问题出在 Dubbo2.7.0 版本引入的新特性之一：标签路由。

其对应的源码如下：

`org.apache.dubbo.rpc.cluster.router.tag.TagRouter#filterInvoker`

![](https://user-gold-cdn.xitu.io/2020/1/6/16f7918281a2ebf5?w=1105&h=700&f=png&s=92991)

通过源码可以看出：**在 TagRouter 中的 stream 操作，改变了 invokers，导致每次调用时其 System.identityHashCode(invokers)返回的值不一样。**

所以每次调用都会进行哈希环的映射操作，在服务节点多，虚拟节点多的情况下**一定会有性能问题。**

该问题对应的 PR 链接如下：

`https://github.com/apache/dubbo/pull/5440`

修复方法也是特别简单：把获取 identityHashCode 的方法从 System.identityHashCode(invokers)修改为 invokers.hashCode()即可。如下图所示：

![](https://user-gold-cdn.xitu.io/2020/1/6/16f7918d227fb6e0?w=985&h=372&f=png&s=71034)

# 为什么一行代码就能修复？

为什么把获取 identityHashCode 的方法从 System.identityHashCode(invokers)修改为 invokers.hashCode()就可以了呢？

要回答这个问题，我们首先得明白什么是 identityHashCode？什么是 hashCode？

**什么是 identityHashCode？**我们看看 API 里面的注释：

![](https://user-gold-cdn.xitu.io/2020/1/6/16f791942cd8b963?w=825&h=264&f=png&s=16791)

**返回与默认方法 hashCode()返回的给定对象相同的哈希码，无论给定对象的类是否覆盖了 hashCode()。空引用的哈希码为零。**

另外关于 identityHashCode 还有下面的三条规则：

1.所以如果两个对象 A == B，那么 A、B 的 System.identityHashCode() 必定相等；

2.如果两个对象的 System.identityHashCode() 不相等，那他们必定不是同一个对象；

3.但是如果两个对象的 System.identityHashCode()相等，并不保证 A==B，因为 identityHashCode 的底层实现是基于一个伪随机数实现的。

**什么是 hashCode？** 大家应该都比较熟了，还是看 API 上的注释：

![](https://user-gold-cdn.xitu.io/2020/1/6/16f7919d50863a0f?w=1000&h=354&f=png&s=37520)

再结合下面两个示例代码，深入理解。

示例一：WhyHashCodeDto**没有重写 hashCode()方法**，所以 identityHashCode 和 hashCode 的值是一样的：

![](https://user-gold-cdn.xitu.io/2020/1/6/16f791a675deddb7?w=850&h=430&f=png&s=62809)

示例二：如下所示，String 是**重写了 hashCode()的方法**，所以在下面的例子中 identityHashCode 不等于 hashCode：

![](https://user-gold-cdn.xitu.io/2020/1/6/16f791a966b08c0e?w=757&h=455&f=png&s=66033)

![](https://user-gold-cdn.xitu.io/2020/1/6/16f791b0d88b5902?w=496&h=326&f=png&s=23346)

# 带入场景

有了前面的知识铺垫，我们就可以回到 Dubbo 的一致性哈希算法的场景中去了。

在 PR 中有一行注释是这样写的：

`using the hashcode of list to compute the hash only pay attention to the elements in the list`

**我们应该只注意 list 里面的元素就可以了。** 而这个 list 里面的元素，就是一个个的服务提供方。

所以，在 Dubbo 的一致性哈希算法的场景中，我们**只需要关心 List 里面的服务提供方是否有上下线的操作，而不关心这个 List 是否每次都是新的。**

我们再回到源码中，**结合源码，然后简化源码：**

![](https://user-gold-cdn.xitu.io/2020/1/6/16f791be84d60ab8?w=1151&h=727&f=png&s=96433)

把上面的源码抽离一下，简化一下，如下：

![](https://user-gold-cdn.xitu.io/2020/1/6/16f791c1950be4b0?w=997&h=506&f=png&s=73706)

filterInvoker 方法是根据条件过滤 invokers，并返回一个 List。而我传入的条件是，过滤出 invokers 中 invoker 大于 0 的数据：

`filterInvoker(invokers, invoker -> invoker > 0);`

执行结果如下：

![](https://user-gold-cdn.xitu.io/2020/1/6/16f791c7878d1d30?w=384&h=144&f=png&s=11499)

可以看到经过 filterInvoker 方法后，由于集合中所有的元素都满足条件，所以过滤前后，集合中的元素并没有发生变化，导致 hashCode 没有变化。但是由于装元素的容器(集合)已经不是原来的容器了，所以 identityHashCode 发生了变化。

"**因为集合中的元素没有发生变化，导致 hashCode 没有变化。**"这句话的理由是什么？

因为 List 重写了 hashCode()方法，其算出的 hashCode 只和 list 中的元素相关：

![](https://user-gold-cdn.xitu.io/2020/1/6/16f791cbee6066fa?w=749&h=369&f=png&s=46282)

经过 filterInvoker 方法后元素还是【1,2,3】，与过滤之前一样，所以 hashCode 没有变。

"**由于装元素的容器(集合)已经不是原来的容器了，所以 identityHashCode 发生了变化。**"这句话的理由又是什么？

可以看到在源码中，Collectors.toList()方法会 new List。**所以都是新的，那么每次的 identityHashCode 必不相同。**

![](https://user-gold-cdn.xitu.io/2020/1/6/16f791d35b72770c?w=904&h=541&f=png&s=66043)

**上面的示例代码，模拟的是没有服务上下线的操作。**

**接下来，我们模拟一下服务下线的场景：**

这次传入的过滤条件为，过滤出 invokers 中 invoker 大于 1 的数据：

`filterInvoker(invokers, invoker -> invoker > 1);`

输出结果如下：

![](https://user-gold-cdn.xitu.io/2020/1/6/16f791e054ddc802?w=389&h=141&f=png&s=11357)

可以看到，过滤后的集合中只有【2,3】了，所以 hashCode 发生了变化。

上面的示例在 Dubbo 的一致性哈希算法的场景中相当于 1 号服务器下线了，服务列表发生了变化，需要重新进行哈希环的映射。

![](https://user-gold-cdn.xitu.io/2020/1/6/16f791e3fcd41f2a?w=324&h=319&f=jpeg&s=12942)

对应源码如下(PR 提交的源码)：

![](https://user-gold-cdn.xitu.io/2020/1/6/16f791e5e2237457?w=739&h=256&f=png&s=40422)

因为在标号为 ① 处得到的 invokersHashCode 和之前的不一样了，所以在标号为 ② 处判断条件为真，进入标号为 ③ 的代码处，重新进行 Hash 环的映射，并选择某个虚拟节点执行该请求。

通过上面模拟的两个示例，再结合下面的源码：

![](https://user-gold-cdn.xitu.io/2020/1/6/16f791e76a87c5ae?w=955&h=300&f=png&s=56354)

也就回答了为什么把上图中编号为 ① 处的代码替换为标号为 ② 的代码，这一行代码就能修复这个 Bug，**核心思想就是只关心 List 集合里面的元素变化，而不关心 List 集合容器是否发生变化。**

# 最后说一句

最开始找到这个 BUG 的时候，我自己也是有一套解决方案的。思路也是只关心 List 里面的元素，而不关心 List 这个容器，但是实现方式比较复杂，改动点较多，还需要写一个工具类。

但是看到 issue 下面的这个评论，

![](https://user-gold-cdn.xitu.io/2020/1/6/16f791ebf60371cb?w=496&h=82&f=png&s=7946)

我才一下回过神来，原来一行代码就能代替我写的工具类了啊。而对于这个知识点，我之前其实是知道的。

我反思了一下自己为什么没有想到这个方案。

**其实就是对于已知道的知识点，掌握不够深刻导致的，没有达到融会贯通的地步。知其然，也知其所以然，可惜在需要使用的场景稍稍一变的情况下，就想不起来了。**

知道知识点，但是该用的时候却记不起来，这种情况其实挺常见的，那怎么解决呢？

**这篇文章就是我的解决方案，记录下来嘛。就像高中的时候人手一本的错题本，做错的题，不会的题都抄下来嘛。没事的时候翻一翻，总有下次碰到的时候。再次碰到时，就是"一雪前耻"的机会。**

![](https://user-gold-cdn.xitu.io/2020/1/6/16f791ef6d049f00?w=300&h=396&f=jpeg&s=24222)

好了。

才疏学浅，难免会有纰漏，如果你发现了错误的地方，还请你留言给我指出来，我对其加以修改。

感谢您的阅读，感谢您的关注。

以上。

欢迎关注公众号【why 技术】,坚持输出原创。愿你我共同进步。

![](https://user-gold-cdn.xitu.io/2020/1/6/16f791f7cf532687?w=258&h=258&f=png&s=55217)
