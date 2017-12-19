---
title:  src/test/groovy——JVM混合语言开发实践
category: Ramblings
layout: post

---

![header](/images/src-test-groovy/groovy-java.png)    



## 要点

- 与其八卦哪门语言会替代Java，不如试试混合语言编程，用新兴语言的优势来弥补Java的弱项
- Groovy可以较好地与Java代码交互，自身有不少语法糖，可以尝试来写Java项目的测试代码


### Java到底什么时候死

随着Google Android 团队宣布了`Kotlin` 成为官方支持语言，互联网上一片“Java已死”的报道又一次汹涌而来。然而每次热潮一过，`Java`还是活得好好的，甚至我们将迎来半年一次更新的[计划](http://www.infoq.com/cn/news/2017/09/Java6Month)。事实上，Android团队并没有说用`Kotlin`替换`Java`，只是给大家多了一种选择。多一种选择意味着可以根据场景选择更合适的解决方案，而在一个项目中甚至可以共存多种解决方案。例如一个典型的Java Web项目，可能会有一些`Shell`或`Python`脚本来帮助构建或搭建本地开发环境。

``` shell
root
  |__src
       |__main
            |__java
  |__scripts
       |__seeder.py // 填充种子数据的Python脚本          
  
```

> 可能每个Java项目中都或多或少有其他语言的代码

### 一次Digital Marketing项目的经历

之前有幸参加了一个Digital Marketing的项目，为某个国际知名运动品牌搭建微信服务号后台。在这个细分行业中，应用的生命周期往往很短，一个支撑市场活动的应用可能需要在2周内开发出来，但只上线使用1天。`PHP`是该行业最流行的语言，但该品牌要求这次项目必须使用`Java`开发（顺便提一句，Google对Digital Marketing项目的语言要求虽然没有那么苛刻，但是`PHP`是被排除的，希望这两句话不会导致歪楼）。虽然工期很紧张，但考虑到该应用有较长的生命周期，自动化测试还是有必要的。既然生产代码本身必须是`Java`，是否可以用更灵活高效的语言来写测试呢？于是我想到了用`Groovy`来测试Java代码，一段有趣的经历开始了。

### src/test/groovy

我对`Groovy`并不熟悉，更多是由于`gradle`这个构建工对它的语言特性有所了解。例如字符串操作在`Groovy`中有一些有趣的增强。在`Java`测试代码中很麻烦的`json`字符串，可以很优雅的解决：

``` java
    @Test
    public void it_should_return_an_event() {

        stubFor(get(urlEqualTo("/v1/events/12345"))
                .willReturn(aResponse()
                .withStatus(OK.value())
                .withBody("" +
                    "{" +
                        "\"body"\:{" +
                            "\"id\": 12345," + 
                            "\"registrationOpenDate\": \"2017-07-28T09:00:00\"," +
                            "\"capacity"\": 20," + 
                            "\"currentParticipation"\": 12," + 
                        "}" +
                    "}" +
        "))); // 繁琐的转义符导致一段简单的json字符串被分隔的面目全非


        Event availability = subject.extractEvent("12345")

        assertThat(availability.eventId, is("12345"));
        // omitted asserts
    }
```
> 在Java代码中拼接字符串很繁琐，而且可读性不好，往往只能将json字符串抽取到单独的文件中，但抽取到文件中一方面降低了运行速度，另一方面在调试时需要在两个窗口中切换，体验不是很流畅

``` groovy
    @Test
    void "it should return an event"() {

        stubFor(get(urlEqualTo("/v1/events/12345"))
                .willReturn(aResponse()
                .withStatus(OK.value())
                .withBody("""
                    {
                        "body":{
                            "id": 12345,
                            "registrationOpenDate": "2017-07-28T09:00:00",
                            "capacity": 20,
                            "currentParticipation": 12,
                        }
                    }
        """)))


        def availability = subject.extractEvent("12345")

        assert availability.eventId == "12345"
        // omitted asserts
    }
```

> 通过"""可以保留json字符串的格式，并且免去了讨厌的转义符

另外一个关于`Groovy`字符串的小应用是方法名，当我们在`Java`测试方法名上尝试驼峰或是下划线分割时，`Groovy`可以直接用字符串定义方法名

``` java
    @Test
    public void itShouldEmitOrderCanceledEvent_whenCancelOrder_givenOrderIsConfirmed() {
        // omitted code
    }
```
> 常见的Java测试方法命名规则：下划线分隔given, when, then，分隔分割子句中的单词
``` groovy
    @Test
    void  "it should emit OrderCanceledEvent when I cancel the order given the order is confirmed"() {
        // omitted code
    }
```
> Groovy可以将字符串作为方法名，更容易使用自然语言来描述测试的意图

Java的强类型检查是工程上的利器，但有时在测试中就显得有些耿直了

``` java
    @Test
    public void  testPricingPolicy() {
        // omitted code
        assertThat(price.getQuantity(), is(2L)); // fails given 2
    }
```
> 在Java中，如果quantity是Long型，在测试里我们也必须精确描述，但是在数字后加个L很容易导致看错
``` java
    @Test
    void  "test pricing policy"() {
        // omitted code
        assert price.quantity == 2
    }
```
> 在Java中，如果quantity是Long型，在测试里我们也必须精确描述，但是在数字后加个L很容易导致看错

还有不少简介的类型声明方法，可以帮助简化Map, List的构造，在测试代码中这些片段往往是为了构造测试数据，其实我们更关注的是数据，甚至是测试数据的差异，希望尽可能的去除语法噪声
``` java
    @Test
    public void  testListBuilder() {
        // omitted code
        List<String> names = new ArrayList<>();
        names.add("john");
        names.add("ben");
        names.add("marry");
    }
```
> 啰嗦
``` groovy
    @Test
    void  "test list builder"() {
        // omitted code
        def names = ["john", "ben" "marry"]
    }
```
> [] 可以用来实例化List，写起来还挺顺手的

### 不只是JUnit

除了以上这些语法糖，`Groovy`社区还提供了[Spock Framework](http://spockframework.org/spock/docs/1.1/index.html)，一个强大的测试库。如果你对`Cucumber`很熟悉的话，可能会觉得它是一个更易用的替代品。相比Cucumber单独的feature文件和方法正则匹配机制，我认为Spock对单元测试更友好。单元测试的数量往往由开发者驱动编写，在同一个屏幕中可以浏览会更方便管理。

```groovy
class CancelOrderCommandHandlerTest extends Specification {

    def orderRepository = Mock(OrderRepository) // Spock自带了Mock支持
    EventPublisher eventPublisher = Mock()
    def subject = new CancelOrderCommandHandler(orderRepository, eventPublisher)

    def "it should cancel the order and publish event"() {

        given: "a to-be canceled order" // given描述
        def order = anOrder().readyToCancel().build()
        orderRepository.findBy(order.trackingId) >> order // 设置预期返回值，是不是比Mockito.when()简单呢？

        when: "ask to cancel the order"     // when描述
        def after = subject.handle(new CancelOrderCommand(order.trackingId))

        then: "the order is canceled and an event should be published"   // then描述
        assert after.isCanceled()
		1 * eventPublisher.publish(new OrderCanceledEvent(order.trackingId)) // 相当于Mockito.verify()
    }
}
```

> Spock Interaction Based Testing 不但提供了Gerkin DSL支持，还内置了强大的Mock机制

除了[Interaction Based Testing](http://spockframework.org/spock/docs/1.1/interaction_based_testing.html)的支持，`Spock`还提供了[Data Driven Testing](http://spockframework.org/spock/docs/1.1/data_driven_testing.html)的DSL:

``` groovy
class TrainingCourseTest extends Specification {


    def "it should tell if the class name matches the course"(String className, boolean matches) {//注意这里有参数

        expect: "matches the name" // 这里是测试执行和断言

        def course = aTrainingCourse().with(FUNDAMENTAL, 1).build()

        assert course.matches(className) == matches

        where: "training class name look like these" //这里是数据提供

        className                                       | matches //对应方法签名中的参数
        "i dont care FUNDAMENTAL1 i dont care either"   | true
        "i dont care FUNDAMENTAL 1i dont care either"   | true
        "i dont care FUNDAMENTAL  1i dont care either"  | true
        "i dont care FUNDAMENTAL-1i dont care either"   | true
        "i dont care FUNDAMENTAL - 1i dont care either" | true
        "i dont care FUNDAMENTAL2i dont care either"    | false
        "i dont care FUNDAMENTAL 2i dont care either"   | false
        "i dont care FUNDAMENTAL21i dont care either"   | false
        "i dont care FUNDAMENTAL - 2i dont care either" | false
        "i dont care fundamental - 1i dont care either" | true
        "i dont care 基础课程 - 1i dont care either"     | true
        "i dont care 基础课程1 - 1i dont care either"    | true
        null                                            | false
    }
}
```

> 比JUnit4 @Parameterized更友好的测试方法工厂支持

此外，`Spock`还支持与`spring-test`的集成，但还不成熟，在`spring-boot`1.4版本引入[Slice Test](https://spring.io/blog/2016/04/15/testing-improvements-in-spring-boot-1-4)后，对于需要`Spring Application Context`的测试还是推荐用`Groovy + Junit`来写。

### 总结

相比完全替换技术栈，不如考虑下混合语言编程，有目标的提升一部分代码的效能，例如测试代码，替换的风险比较低，是练手的好场景。如果你觉得`Java`编写的测试代码太啰嗦，不妨试试`Groovy`。