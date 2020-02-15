---
title: 架构设计实践之业务核心与配置分离
category: Ramblings
---

#### 一个配置需求引发的思考 

去年我在Github上开源了一个小项目——[Daming](https://github.com/TheBund1st/daming)，为短信验证服务提供开箱即用的实现。Daming的核心是一个稳定的短信验证抽象，以便在默认实现不满足用户需求时，可以自行扩展。

一天，我收到了一条Github Issue提醒，一位朋友觉得Daming自带的阿里云验证码发送功能尚不能满足他的需求，对于如何定制扩展希望咨询我的意见：

> 目前的短信模板配置是只支持单个注入吗？短信发送前置校验是针对全局设置。假如短信发送有多模板 一个scope 一个短信模板code 有对应的发送校验 （发送次数 一天几次 类似的） 是不是就得使用数据库表做配置了？”

简要来说，Daming的短信验证抽象允许同时存在多个Scope（可以理解为短信验证的用途），但是自带的阿里云验证码发送组件在同一时间只支持一种短信模板。也就说，虽然可以同时提供多个用途的短信验证服务，但是它们只能合用一套短信模板。

![buildt-in-aliyun-sender](/images/the-configuration-sub-domain/built-in-aliyun-sender.png)

如果要扩展为每个Scope使用独立的短信模板也不困难，因为阿里云验证码发送组件可以方便地取到要发送的验证码的Scope，倒是这句“是不是就得使用数据库表做配置了”引起了我的思考:

1. 是否需要用数据库？用文件实现会有什么问题？
2. 支持多短信模板的改动有多大，会影响到短信验证抽象本身吗？



#### 数据库不是存储配置项的唯一实现方式

先来看第一个问题：是否需要用数据库？用文件实现会有什么问题？

我认为这要看情况，我们需要引入的是一组“Scope=>短信模板”的配置项，而使用数据库来存储或是使用文件来存储取决于配置变更的频率。显然，如果配置变更频率低，也能容忍需要重新部署再生效的场景里，文件实现就够用了。而如果希望给运营人员提供一个自助式的配置控制台，才需要引入数据库。

![buildt-in-aliyun-sender](/images/the-configuration-sub-domain/multiple-configurations.png)

由此看来，同样的短信验证抽象，却可能有多种配置方式。



####配置的实现方式相对于短信验证更易变

再来看第二个问题：支持多短信模板的改动有多大，会影响到短信验证抽象本身吗？

由上一问的分析可以看到，同样的短信验证抽象，却可能有多种配置方式，而且可能从一种“升级”到另一种，我们希望这种变化不应影响到短信验证本身。因此，Daming的设计中，配置不是短信验证抽象的一部分，而是通过扩展抽象来实现。这意味着配置依赖稳定的短信验证抽象，而短信验证抽象不依赖于易变的配置，从而保证短信验证抽象的稳定性。



#### 先关注业务核心，再考虑配置的实现

配置面向的是系统运维/运营，而不是用户。配置本身不独立产生业务价值，它是业务核心的辅助。只有业务核心产生价值，并且通过扩展去满足更多业务场景时，配置才有用武之地。

在去年的一个项目中，我们需要在短时间内帮助客户探索一项新兴业务。面对极为挑战的项目周期，我们在明知该业务需要大量配置功能的情况下，选择了在项目初期全力实现面向用户的功能，再回头实现配置功能的策略。这是因为：

1. 这个策略可以帮助我们以最短的时间实现了业务核心的价值验证，代价是初期需要以简陋的文件或数据库脚本来实现配置变更。这个代价是可以接受的，尤其是业务核心尚不稳定的情况下。
2. 业务核心的验证，也加强了团队对于领域知识的理解，有利于设计出稳定的业务核心抽象，再基于该抽象扩展配置功能。从而减少过度设计或破坏抽象的情况。
3. 如果项目中出现问题导致工作量膨胀，我们可以尽量保障面向用户的功能，而选择简化配置功能来应对风险。

#### 写在最后

本文尝试从一个简单的短信验证配置示例出发，探讨配置与业务核心的设计实践。如果读者有类似的经验，也欢迎一起探讨
