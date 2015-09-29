---

title:   Given，When，Then风格的Dbunit DataSet
category: continuous delivery   
layout: post

---

&emsp;&emsp;在之前的一篇[用spring-test、dbunit、flyway测试Java持久层](/blogs/java-persistence-test.html)中，我曾介绍过使用[spring-test-dbunit](http://springtestdbunit.github.io/spring-test-dbunit/)提供的@DatabaseSetup和@ExpectedDatabase简化测试断言的方法，但使用这种办法也有痛点：
1. 在数据准备（setup）文件和数据预期（expect）文件中存在大量重复的数据定义，随着数据结构逐渐丰富，需要同时修改这两个文件
2. 调试过程中要在两个文件中不断切换，甚是麻烦，而且对比差异也不方便

{% highlight xml %}
<!--order_save_fixture.xml，这里定义“原型数据”-->
<dataset>
    <t_order tracking_id="1" status="WAIT_PAYMENT"/>
    <t_order_item tracking_id="1" name="item1" quantity="1"/>
    <t_order_item tracking_id="1" name="item2" quantity="2"/>
</dataset>
<!--order_save_expected.xml，至少需要额外定义一遍原型数据，哪怕它们不会变化-->
<dataset>
    <t_order tracking_id="1" status="WAIT_PAYMENT"/>
    <t_order_item tracking_id="1" name="item1" quantity="1"/>
    <t_order_item tracking_id="1" name="item2" quantity="2"/>

    <t_order tracking_id="2" status="WAIT_PAYMENT"/>
    <t_order_item tracking_id="2" name="item1" quantity="1"/>
    <t_order_item tracking_id="2" name="item2" quantity="2"/>
</dataset>

{% endhighlight %}

### 降低非关键数据的干扰
&emsp;&emsp;优秀的测试会尽可能地直接突出影响测试结果的关键数据，它们越是醒目，
测试就容易被他人理解、越容易维护。而现在我们却还需要编写重复的数据，这可不行。
因此，我为spring-test-dbunit提供了一个小扩展，
使得setup和expect数据可以定义在同一个文件中(省去了切换的麻烦)，而且没有重复数据！

{% highlight xml %}
<!--order_save.xml-->
<dataset>
    <given>
        <t_order tracking_id="1" status="WAIT_PAYMENT"/>
        <t_order_item tracking_id="1" name="item1" quantity="1"/>
        <t_order_item tracking_id="1" name="item2" quantity="2"/>
    <given/>

    <then>
        <added>
            <t_order tracking_id="2" status="WAIT_PAYMENT"/>
            <t_order_item tracking_id="2" name="item1" quantity="1"/>
            <t_order_item tracking_id="2" name="item2" quantity="2"/>
        </added>
    </then>
</dataset>

{% endhighlight %}

目前，需要对测试用例进行三处改动来支持这个新的数据格式，请注意以下代码的注释：

{% highlight java%}
//other annotations
@DbUnitConfiguration(dataSetLoader = GivenWhenThenFlatXmlDataSetLoader.class) //use this loader
public class HibernateOrderRepositoryTest {

    @DatabaseSetup("given:classpath:order_save.xml") //decorate xml file with "given:" prefix
    @ExpectedDatabase(value = "then:classpath:order_save.xml",//decorate xml file with "then:" prefix
                      assertionMode = NON_STRICT_UNORDERED)
    @Test public void should_saves_order() throws Exception {

      final String trackingIdOfPrototype = "1";
      final String trackingIdOfToBeSaved = "2";

      final Order toBeSaved = clone(trackingIdOfPrototype, trackingIdOfToBeSaved);

        subject.store(toBeSaved);
    }
    // other code hide for brevity
}

{% endhighlight %}

除了added(对应insert)之外，GivenWhenThenFlatXmlDataSetLoader还支持delete声明：
{% highlight xml %}
<!--order_remove_items.xml-->
<dataset>
    <given>
        <t_order tracking_id="1" status="WAIT_PAYMENT"/>
        <t_order_item tracking_id="1" name="item1" quantity="1"/>
        <t_order_item tracking_id="1" name="item2" quantity="2"/>
    <given/>

    <then>
        <deleted>
            <t_order_item tracking_id="2" name="item2"/>
        </deleted>
    </then>
</dataset>
{% endhighlight %}

你只需要把唯一标识要删除的数据的字段列出来，GivenWhenThenFlatXmlDataSetLoader会智能地找到它。
GivenWhenThenFlatXmlDataSetLoader还支持modify声明，用来支持测试update的用例，同样地，
你只需要列出有变化的字段，但遗憾的是，目前还无法为modified智能找到主键，你需要显示地告诉它 :-(

{% highlight xml %}
<!--order_remove_items.xml-->
<dataset>
    <given>
        <t_order tracking_id="1" status="WAIT_PAYMENT"/>
        <t_order_item tracking_id="1" name="item1" quantity="1"/>
        <t_order_item tracking_id="1" name="item2" quantity="2"/>
    <given/>

    <then>
        <modified pk="tracking_id, name">
            <t_order_item tracking_id="2" name="item2" quantity="3"/>
        </modified>
        <!--如果有多张表，可以定义多个<modified/>-->
    </then>
</dataset>
{% endhighlight %}

更详细的例子，可以参考Github上的[说明](https://github.com/Hippoom/spring-test-dbunit-template).
如果你有改进的好点子，不妨告诉我或是干脆来个pull request :-)
