---
title:  单页应用的HATEOAS实战
category: Rest
layout: post

---

![header](/images/hateoas-spa/HATEOAS.png)    



## 要点

- HATEOAS是**Hypertext As The Engine Of Application State**的缩写。在 [Richardson Maturity Model](http://martinfowler.com/articles/richardsonMaturityModel.html)中, 它是REST的最高级形态
- 单页应用正越来越受到欢迎，前后端分离的开发模式进一步细化了分工、消除了瓶颈，但同时也引入了不少重复的工作，例如一些业务规则在后端必须实现的情况下，前端也需要再实现一遍已获得更好的用户体验。HATEOAS虽然不是唯一消除这些重复的方法，但作为一种架构原则，它更容易让团队找到消除重复的“套路”

### 什么是HATOEAS
HATEOAS是**Hypertext As The Engine Of Application State**的缩写。采用Hypermedia的API在响应（response）中除了返回资源（resource）本身外，还会额外返回一组Link。 这组Link描述了对于该资源，消费者（consumer）接下来可以做什么以及怎么做。

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
    "_Links": {
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

* 理解Link中的“self”的消费者知道使用get方法访问其“href”的uri可以查看该订单的详细信息
* 理解Link中的“cancel”的消费者知道使用delete方法访问其“href”的uri可以取消该订单
* 理解Link中的“payment”的消费者知道使用post方法访问其“href”的uri可以为该订单付款


REST是目前业界相当火热的术语，似乎发布的API不带个REST前缀，你都不好意思和别人打招呼了。 然而大部分号称REST的API实际上并没有达到[Richardson成熟度模型](http://martinfowler.com/articles/richardsonMaturityModel.html)的第三个级别：Hypermedia。 而REST的发明者Roy Fielding博士更是直言[HATEOAS是REST的前提， 这不是一个可选项，如果没有Hypermedia，那就不是REST。](http://www.infoq.com/articles/roy-fielding-on-versioning)(摘自Infoq对Fielding博士的第二段访谈)


那么HATOEAS带来了什么优势？

一个显而易见的好处是，只要客户端总是使用Link Rel来获取URI，那么服务端可以在不破坏客户端实现的情况下实现URI的修改，从而进一步解耦客户端和服务端。

另一个容易被忽视的优势是它可以帮助客户端开发者探索API，Links实际上提示了开发者接下来可以进行何种业务操作，开发者虽然精通技术，但往往对于业务不甚了解，这些提示可以帮助他们理解业务，至少是一个查询API文档的好起点。想象一下，如果某个API的响应中多了一个新的Link，敏感的开发者可能就会询问这个Link是用来做什么的，是一个新的特性吗？虽然看起不起眼，但这往往使两个团队的成员沟通起来更容易。




### 单页应用和HATEOAS

在过去的几年里，WEB开发技术发生了很多重大的变革，其中之一就是单页应用，它们往往能带来更平滑的用户体验。在这一领域，分工进一步细化，前端工程师专精客户端程序构建和HTML、CSS等效果的开发，后端工程师则更偏重高并发、DevOps等技能，大部分特性需要前后端工程师配合完成。或许有人会质疑，为什么不是全栈工程师？诚然，如果一个人就能端到端的交付特性，那自然会减少沟通成本，但全栈工程师可不好找，细化分工才能适应规模化的开发模式。继Ajax之后，单页应用和前后端分离架构进一步催生了大量的API，我们急需一些方法来管理这些API的开发和演进，而HATEOAS应该在此占有一席之地。

###### 在摸索中前进，自由地重命名你的资源
我们常说在敏捷开发中，应该拥抱变化。所以敏捷开发中推崇重构、单元测试、持续集成等技术，因为它们可以使变化更容易、更安全。HATOEAS也是这样一种技术。想象一下，在项目初始阶段，团队对业务的理解还不深入，很有可能会得出错误的业务术语命名，或者业务对象的建模也不完全合适。反映在API上，可能你希望能够修正API的URI，在非HATOEAS的项目中，由于URI是在客户端硬编码的，即使你把它们设计的非常漂亮（准确的HTTP动词，以复数命名的资源，禁止使用动词等等），也不能帮助你更容易地修改它们，因为你的重构需要前端开发者的配合，而他/她不得不停下手头的其他工作。但在采用了HATEOAS的项目中，这很容易，因为客户端是通过Link来查找API的URI，所以你可以在不破坏API Scheme的情况下修改它的URI。当然，你不可能保证所有API的URI都是通过Link来获取的，你需要安排一些Root Resource，例如 `/api/currentLoggedInUser`，否则客户端没有办法发起第一次请求。

{% highlight html %}

HTTP/1.1 200 OK
Path: /api/currentLoggedInser (1)
{

​     …… //omitted content

​     "_Links": {  (2)
        "searchUserStories": {
            "href": "http://localhost:8080/userStories/search{?page, size}"
        },

​       "searchUsers": {
            "href": "http://localhost:8080/users/search{?page, size, username}"
        },

​        "logout": {
            "href": "http://localhost:8080/logout"
        }

​    }
}
{% endhighlight %}

1. Root Resource，它们是API的入口，客户端通过他们浏览当前用户有哪些资源可以访问，你可以定义多个Root Resource，并确保它们的URI不会改变
2. Link引入的URI可以自由地变化，可能是因为需要重命名资源，也可能是需要抽取出新的服务（域名变化）



###### 消除重复的业务规则校验实现，更容易得适应变化

经验告诉我们，不能相信客户端的请求，所以在服务端我们需要根据业务规则校验当前的请求是否合法。这样确保了业务正确，但当用户发起了请求后才告诉他们请求失败，有时候是一件令人沮丧的事情。为了用户体验，可能会要求某些组件根据业务规则展示。例如，对于某个业务对象，要求编辑按钮只在当前用户可以编辑的情况下才展示。在传统的服务端渲染架构下，一般都可以复用校验的代码，而在单页应用中，往往由于技术栈不同，代码无法直接共用，业务规则在前后端都分别实现了一次。例如，在我们最近的一次项目中，前后端分别实现了如下规则：

> 给定一个用户故事，
>
> 只有它的作者才能编辑它

服务端通过在用户故事的API中暴露作者帮助前端完成编辑按钮的有条件渲染。

{% highlight html %}

HTTP/1.1 200 OK
Path: /api/userStories/123
{

​     "author": "john.doe@gmail.com"  (1)

}
{% endhighlight %}

1. 与当前用户比较判断是否渲染编辑按钮

但如果规则发生变化，前后端都需要适应这一改变，所以我们用HATEOAS重构了一下：

{% highlight html %}

HTTP/1.1 200 OK
Path: /api/userStories/123
{

​     "author": "john.doe@gmail.com",
     "_links": {

​	"updateUserStory": { "href": "http://localhost:8080/api/userStories/123" }  (1)

​     } 

}
{% endhighlight %}

1. 现在前端会根据 `updateUserStory`link是否出现来验证当前用户是否具有编辑用户故事的能力


后来业务规则变为`除了作者之外，系统管理员也可以编辑用户故事`，这时候只需要后端去响应这个变化就行了。你可能会质疑，通过为用户故事暴露一个 `isCurrentLoggedInUserAvailableToUpdate`的计算属性也可以做到。没错，HATOEAS并不是唯一的办法，但作为一种架构约束，团队会自然而然地想到它，而计算属性则要求团队成员有更强的抽象技能。

### 总结

HATEOAS提倡在响应返回Link来提示对该资源接下来的操作。这种方式解耦了服务端URI，也可以让客户端开发者更容易地探索API。最后，通过Link来判断业务状态，还能有效地消除单页应用中的业务规则重复实现。



