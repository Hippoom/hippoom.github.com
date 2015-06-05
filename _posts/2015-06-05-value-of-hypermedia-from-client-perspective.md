---

title:   从消费者的角度评估Rest的价值
category: rest  
layout: post

---

&emsp;&emsp;Rest是目前业界相当火热的术语，似乎发布的API不带个Rest前缀，你都不好意思和别人打招呼了。
然而大部分号称Rest的API实际上并没有达到[Richardson成熟度模型](http://martinfowler.com/articles/richardsonMaturityModel.html)的第三个级别：Hypermedia。
而Rest的发明者Roy Fielding博士更是直言[“Hypermedia作为应用引擎”是Rest的前提，
这不是一个可选项，如果没有Hypermedia，那就不是Rest。](http://www.infoq.com/articles/roy-fielding-on-versioning)(摘自Infoq对Fielding博士的第二段访谈)

### 什么是Hypermedia？
&emsp;&emsp;那究竟什么是Hypermedia？
采用Hypermedia的API在响应（response）中除了返回资源（resource）本身外，还会额外返回一组链接（link）。
这组链接描述了对于该资源，消费者（consumer）接下来可以做什么以及怎么做。

举例来说，假设向API发起一次get请求，获取指定订单的资源表述（representation），那么它应该长得像这样：

{% highlight html %}

HTTP/1.1 200 OK
Server: Apache-Coyote/1.1
Content-Type: application/hal+json;charset=UTF-8
Transfer-Encoding: chunked
Date: Fri, 05 Jun 2015 02:54:57 GMT

{
    "tracking_id": "123456",
    "status": "WAIT_PAYMENT",
    "items": [
        {
            "name": "potato",
            "quantity": 1
        }
    ],
    "_links": {
        "self": {
            "href": "http://localhost:57900/orders/123456"
        },
        "cancel": {
            "href": "http://localhost:57900/orders/123456"
        },
        "payment": {
            "href": "http://localhost:57900/orders/123456/payments"
        }
    }
}
{% endhighlight %}


* 理解链接中的“self”的消费者知道使用get方法访问其“href”的uri可以查看该订单的详细信息
* 理解链接中的“cancel”的消费者知道使用delete方法访问其“href”的uri可以取消该订单
* 理解链接中的“payment”的消费者知道使用post方法访问其“href”的uri可以为该订单付款

这看起来很有趣，然而这对API的消费者来说有什么好处呢？

### 不再揣测如何组合使用API

&emsp;&emsp;不知道在你的开发生涯中有没有遇到过这样的事情：

    有一天，产品经理跟我说，我们要和某某酒店集团对接，在线销售它们的酒店，这是他们的联系人和详细的API说明文档。
API说明文档真够详细，有好几十页，凭着丰富的行业经验，我知道我需要找到其中的哪些API来实现基本的业务场景。
几天后，我实现了大部分的API集成，现在可以预订酒店了，订单已经在对方的测试环境生成，大功告成。
等等，这个“添加订单财务信息”的API是干嘛的？在和对方的联系人联系后，被告知“没什么用”，好了，真的大功告成了，上线！

    两周后，我们的API使用权限被对方关闭了，原因是“所有的订单都没有财务信息，无法确认对账”。  
    “等等，那谁谁谁不是说这个API没用吗？”
    “噢，他已经离职了，你们如果要恢复使用，尽快完成这个API的集成吧”  
    “我。。。”

当然，这里面还有许多别的因素，但是消费者的开发人员往往很难将业务场景和实现业务场景的API联系起来。
他们常常面对是：

* 不熟悉的业务场景
* 一套对单个API的作用描述详细，但缺乏API之间联系的文档。

Hypermedia带来的API自描述特性，使用链接的方式提示接下来做什么和怎么做，正好可以缓解这样的窘境。
如果API服务方可以提供测试环境供消费者测试，那么开发人员可以实际动手探索业务场景的衔接，这时再配合API文档情况就好多了。
回到酒店订单的例子，如果是这样，我可能就不会挨批了：

{% highlight html %}
// 预订后，提示确认订单，那么不熟悉为什么要确认以及不确认的后果的同学就可以想到去问啦
{
    "tracking_id": "123456",
    "Hotel": "A ZHAO DAI SUO",
    "status": "WAIT_ACKNOWLEDGED",
    "_links": {
        "self": {
            "href": "http://zhaodaisuo.com/orders/123456"
        },
        "cancel": {
            "href": "http://zhaodaisuo.com/orders/123456"
        },
        "acknowledge": {
            "href": "http://zhaodaisuo.com/orders/123456/payments"
        }
    }
}
{% endhighlight %}

{% highlight html %}
// 确认后，提示添加财务信息，不熟悉的同学就可以问了，然而我真的已经问了呀。。。。
{
    "tracking_id": "123456",
    "Hotel": "A ZHAO DAI SUO",
    "status": "ACKNOWLEDGED",
    "_links": {
        "self": {
            "href": "http://zhaodaisuo.com/orders/123456"
        },
        "billing": {
            "href": "http://zhaodaisuo.com/orders/123456/bill"
        }
    }
}
{% endhighlight %}

### 从此与API版本说再见

&emsp;&emsp;不知道在你的开发生涯中有没有遇到过这样的事情：

    有一天，产品经理跟我说，我们要实现一个新功能blablabla，但是依赖的API版本太老了，这是他们的联系人。
    “你好呀，请问我们需要这个信息，但是现在1.3的版本中没有提供，有什么办法吗？”
    “你可以升级到2.1的版本就有了”
    “那这个版本是不是向后兼容的啊？我们用了其中很多接口哦，我不想其它的集成点出问题”
    “那当然，放心吧”

结果当然是个悲伤的故事，“你给我过来，我保证不打死你”。
API的发布方也需要增加新功能，API自身也会随着需求变化，于是产生了版本号

    http://www.zhaodaisuo.com/api/v1.2

然而一套API一般会包括多个API，为整套API版本化的粒度太粗了。
一旦消费者希望获得其中某个API的新特性，他/她只能选择全盘升级并仔细测试或者为每个集成点配置单独的uri。
这都不够好，而Hypermedia可以改变这种局面。
由于提供了链接来告诉消费者资源的uri，相对“传统”的Rest API，uri变成了一种弱耦合，
Hypermedia API只需要公布少量入口uri就可以了。比如，以之前酒店订单的例子，只需发布

    http://www.zhaodaisuo.com/orders

后续的确认、财务信息的uri是在实际API调用的时候拿到的，无需事先准备。
消费者和发布者之间的强耦合实际上只剩下入口uri和服务契约（解释资源的含义），
当服务契约新增或是发生破坏性的变化时（例如修改了或删除了参数），只需要在资源表述中增加新的链接。

{% highlight html %}
{
    "tracking_id": "123456",
    "Hotel": "A ZHAO DAI SUO",
    "status": "ACKNOWLEDGED",
    "_links": {
        "self": {
            "href": "http://zhaodaisuo.com/orders/123456"
        },
        "billing": {
            "href": "http://zhaodaisuo.com/orders/123456/bill"
        },
        "billing-v1.1": { //billing发生了破坏性变化
            "href": "http://zhaodaisuo.com/orders/123456/bill/v1"
        },
        "coupon": { //新增了优惠券抵用的契约
            "href": "http://zhaodaisuo.com/orders/123456/coupon"
        }
    }
}
{% endhighlight %}

这样版本化的粒度就下移到了服务契约的级别，这时消费者就灵活多了，只要按需修改对应的集成点就行了。

### 彻底与API的内部实现解耦

&emsp;&emsp;不知道在你的开发生涯中有没有遇到过这样的事情：

    有一天，产品经理跟我说，我们依赖的一个API发布者通知我，现在判断订单是否能够用优惠券的条件变化，这是他们的联系人。
    “你好呀，请问现在订单能否用优惠券的判断条件有什么变化？”
    “原来，你们可以通过订单的状态来判断，现在还需要结合订单的来源，参加秒杀活动的订单不能使用优惠券”
    “好吧。。。”

于是我在集成代码中做了如下修改：

{% highlight java %}

    if (order.status().equals("WAIT_PAYMENT")) {
    if  (order.source().equals("miaosha")) {
        couponButtonEnabled = false;
    } else {
       //....
    }
} else {
   //...
}
{% endhighlight %}

好纠结，就不能把API设计成这样吗？

{% highlight html %}
{
    "tracking_id": "123456",
    "Hotel": "A ZHAO DAI SUO",
    "status": "WAIT_PAYMENT",
    "source": "normal",
    "_links": {
        "coupon": {
            "href": "http://zhaodaisuo.com/orders/123456/coupon"
        }
    }
}
{% endhighlight %}

{% highlight html %}
{
    "tracking_id": "123456",
    "Hotel": "A ZHAO DAI SUO",
    "status": "WAIT_PAYMENT",
    "source": "miaosha",
    "_links": {} //秒杀来源的订单不返回含优惠券链接的资源表述。
}
{% endhighlight %}

这样客户端代码就简单了，依赖于抽象的业务场景，而不是依赖于具体的实现

{% highlight java %}

    if (order.containsLink("coupon")) {
    couponButtonEnabled = true;
} else {
    couponButtonEnabled = false;
}
{% endhighlight %}  

### 其实你是一位API发布方的开发者？

&emsp;&emsp;嗯，Hypermedia很好，但我是API的发布者，这对我有什么好处？
很高兴你能看到这里，其他就不多说了，在这个服务竞争激烈的时代，把消费者们哄开心了，有的是你的好处。
