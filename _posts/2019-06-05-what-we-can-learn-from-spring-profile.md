---
title:  条件型业务规则约束的抽象与实现——从Spring Profile得到的灵感
category: ramblings
layout: post
---



最近，有幸参与了一个平台型的项目，该平台支持多种类型的产品预订，并且对于不同的产品类型，支持不同的预订规则。开发团队想尽可能地将主流程实现得更通用，以便在将来更快速地支持新的产品类型。因此，团队决定在主流程中，以产品类型作为条件，决定是否应用某个给定的预订规则。

例如其中有一个对于配送地址的验证规则，它只对特定产品类型（火车票）生效：

> (经过简化的用户故事——火车票预订)
>
> 作为用户，当我预订火车票时，我应该被告知配送地址无法送达，以便我调整配送地址或选择上门取票

该平台还支持预订酒店，不过由于没有凭据需要配送，所以并不需要检查配送地址是否可达。于是有了以下实现：

```java
public class AddressIsAvailableToDelivery implements PlaceOrderRule {

    @Override
    public void verify(PlaceOrderCommand command) { 
        if (command.getProduct().isTypeOf(RAILWAY)) {
            // check if the adress is available for delivery the ticket
        } else {
            // hotel, makes no sense of deliering tickets
        }
    }
}
```

预订主流程会依次执行所有的`PlaceOrderRule`，并由各个`PlaceOrderRule`的实现决定需要对哪些产品生效。

几个迭代过后有了新的产品需要支持：观光景点，需要配送门票给用户，所以一个类似的用户故事诞生了：

> (经过简化的用户故事——门票预订)
>
> 作为用户，当我预订景点门票时，我应该被告知配送地址无法送达，以便我调整配送地址或选择上门取票

于是，团队修改了条件表达式，增加了对门票景点的判断：


```java
public class AddressIsAvailableToDelivery implements PlaceOrderRule {

    @Override
    public void verify(PlaceOrderCommand command) { 
        if (command.getProduct().isTypeOf(RAILWAY) || command.getProduct().isTypeOf(SIGHTSEEING)) {
            // check if the adress is available for delivery the ticket
        } else {
            // hotel, makes no sense of deliering tickets
        }
    }
}
```

到这里，我们闻到到了一些"坏味道"：随着需要验证地址是否达的产品类型增加，代码的[圈复杂度]([https://zh.wikipedia.org/wiki/%E5%BE%AA%E7%92%B0%E8%A4%87%E9%9B%9C%E5%BA%A6](https://zh.wikipedia.org/wiki/循環複雜度))会随之升高，意味着需要更多的测试用例来保护。如果将来再有一个新的类型需要检查配送地址是否可达，可以预见此处还会修改；如果系统中有越来越多的条件型业务规则使用当前的方式实现，系统将会越来越脆弱。

### 找到稳定的抽象

那么问题出在哪里？我认为这是由于没有找到正确的抽象，对于条件型的业务规则，其实是有稳定的步骤的：

> 1. 检测当前情况是否需要验证给定的业务规则
> 2. 如需要，执行验证；如不需要则略过

如果将`AddressIsAvailableToDelivery`修改为：

```java
public class AddressIsAvailableToDelivery implements PlaceOrderRule {

    @Override
    public void verify(PlaceOrderCommand command) { 
        if (command.getProduct().isDeliverableAddressRequired()) {
            // check if the adress is available for delivery the ticket
        } else {
            // hotel, makes no sense of deliering tickets
        }
    }
}
```

这样，条件表达式依赖了稳定的抽象。代码不需要再关心产品类型了，当新的产品加入平台时，只需要知道该产品是否需要验证配送地址就行了。这样就做到了当新产品加入时，核心的规则验证逻辑不需要变更，系统更加稳定。

### 但这样好难用

工程师对这个重构感到满意，于是找到了BA(业务分析师)，尝试对用户故事做一些变化

> (经过简化的用户故事——产品预订)
>
> 1. 作为用户，当我预订需要检查配送地址是否可达的产品时，我应该被告知配送地址无法送达，以便我调整配送地址或选择上门取票
> 2. 作为运营人员，我可以设置产品在预订时是否需要检查配送地址，以避免预订后无法配送凭证的情况

BA对此提出了担心：

> 1. 在这个实现方案中，平台运营团队需要为不同的产品设置不同的规则吗？如果规则数量很多，配置起来是不是很麻烦？因为对于某个产品类型，几乎不需要做规则的调整，要求运营团队去配置这些功能在现阶段反而使他们的工作变复杂了
> 2. 平台运营团队在平时的工作中，还是按照产品类型的思维在工作的，他们更习惯于"如果产品类型是火车，那么。。。"这样的沟通方式，想要改变这样的思维方式不是那么容易
> 3. 修改后的用户故事似乎太抽象了，这样能否帮助团队有效地理解真实的业务场景？

![product-configuration](/images/what-we-can-learn-from-spring-profile/property-based-configuration.png)

> 当有大量规则的时候，细粒度的产品配置方式确实有些繁琐，可能需要"配置专家"才能搞定

这些担忧不无道理，团队一下子陷入了两难的境地。

### 意外的灵感

我在阅读该项目一段配置代码的时候发现了这样一个细节：

```java
if (isSmsEnabled()) {
   //enable sms sending
}

if (isEmailEnabled()) {
   //enable email sending
}



// application.properties
sms.enabled: false
email.enabled: false

// application-dev.properties
sms.enabled: false
email.enabled: false
  
// application-qa.properties
sms.enabled: false 
email.enabled: true
  
// application-prod.properties
sms.enabled: true 
email.enabled: true


```

这段代码表示，在不同的环境中，通过细粒度的配置项，可以精确地控制某个特定功能是否起效。配置项的控制范围很小，而且可能会有许多这样的配置项，但团队根据各个环境上的测试约定，将这些配置项归拢到以环境命名的配置文件中，这是`spring boot`提供的[`Profile`](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-profiles.html)机制。在启动应用的时候，并不需要一一指定各个配置项的值，而是指定粗粒度的`profile`即可: `--spring.profiles.active=prod`

这个方案给了我一个灵感：能否将之前的预订规则表达式类比为配置项，产品类型类比为`Profile`呢？

在这个思路下，我们保持`AddressIsAvailableToDelivery`依赖稳定的`isDeliverableAddressRequired`：

```java
public class AddressIsAvailableToDelivery implements PlaceOrderRule {

    @Override
    public void verify(PlaceOrderCommand command) { 
        if (command.getProduct().isDeliverableAddressRequired()) {
            // check if the adress is available for delivery the ticket
        } else {
            // hotel, makes no sense of deliering tickets
        }
    }
}
```

而在实例化`Product`时，注入预先设置的配置项，将产品类型和配置项的转换从核心的规则校验中剥离出去。

```properties
# railway
placeOrderRule.RAILWAY.deliverableAddressRequired=true
placeOrderRule.RAILWAY.anotherConstraint1=false
placeOrderRule.RAILWAY.anotherConstraint2=false
# sightseeing
placeOrderRule.SIGHTSEEING.deliverableAddressRequired=true
placeOrderRule.SIGHTSEEING.anotherConstraint1=false
placeOrderRule.SIGHTSEEING.anotherConstraint2=true
```

这样，既能让核心的规则校验依赖稳定的抽象，又暂时避免了给运营团队带来繁琐的配置工作。

### 遗留的问题

回顾这个过程，实在有些偶然，而且我认为我们只是用了最熟悉的技术手段暂时缓解了之前BA提出的第一点担心。

>2. 平台运营团队在平时的工作中，还是按照产品类型的思维在工作的，他们更习惯于"如果产品类型是火车，那么。。。"这样的沟通方式，想要改变这样的思维方式不是那么容易
>3. 修改后的用户故事感觉太抽象了，这样能否帮助团队有效地理解真实的业务场景？

而2、3则涉及到项目团队和干系人对产品的思考方式，当我们更倾向于使用具体的场景沟通的时候，团队更不容易意识到需要从中寻找稳定的抽象。那么我们需要花费精力去改变用户的思维方式吗，如果需要又应该使用什么样的方式？又或者我们需要使用更抽象的方式来撰写用户故事吗？在这里，想听听大家的意见。