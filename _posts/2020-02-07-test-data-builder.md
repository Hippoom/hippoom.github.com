---
title: 浅谈自动化测试中的数据管理
category: ramblings
---

**1.听说自动化测试是活的文档**

倡导测试驱动开发（TDD）的人常说，自动化测试不仅仅是测试，它们还是活的文档（Living Document）。这样的说法是激动人心的，让人颇有“超值”的感觉。

但在实践中，我在不少项目中看到的却是脆弱的、可读性糟糕的自动化测试。糟糕的自动化测试代码最多能实现测试的目标，但它不容易随着生产代码演进（想象一下修改刚才那个测试有多痛苦）更不可能成为活的文档。

并不是TDD倡导者在说谎，而是想要提升自动化测试的效能，需要遵循最佳实践。这些实践大致可以分类为：

- 对测试友好的生产代码
- 分层的测试策略
- 良好的测试数据准备和管理

这是一个很大的话题，在本文中，我将主要讨论关于测试数据准备和管理的实践。

**2 测试数据准备和管理是提升自动化测试效能的重要实践**

我推荐的测试数据准备和管理实践：

1. **使用Test Data Builder模式：**隐藏测试数据准备的细节，在测试代码中只显示地对测试专有数据赋值，这有助于提升测试代码可读性，并有效降低测试数据准备代码变更时的副作用。
2. **尽量为每个测试设计独立的测试数据**：例如使用随机或自增长的ID，而不是固定值；这在需要数据库的测试中尤其有用，它可以显著降低由于测试数据冲突导致的假报警（False Alarm）
3. **尽量使用生产代码来准备测试数据**：尽量不要绕过生产代码，这在需要数据库的测试中特别有用，以免生产代码中的Schema变更后，还需要绞尽脑汁地修改测试数据准备脚本。

接下来，我们来谈谈这些实践是为了解决什么问题。

**2.1 使用Test Data Builder模式**

在编写自动化测试代码时，我发现并不是所有的数据都在当前测试中起到关键作用。在OrderTest案例中，虽然订单（Order）有很多属性，但在该测试中真正起到区别作用的是状态（status），但我却不得不花费大量时间来准备其他数据，这令人非常沮丧。

```java
public class OrderTest {

    @Rule
    public ExpectedException thrown = ExpectedException.none();

    @Test
    public void it_should_reject_when_cancel_given_order_is_taken() {
        //given
        Order order = new Order(Order.Identity.next());

        // 它们和测试结果不产生影响，但却浪费了大量的精力
        order.with(Location.TAKE_AWAY);
        order.setAmount(100.0);

        Customer customer = new Customer();
        customer.setSurname("周");
        customer.setTitle(Customer.Title.MALE);
        customer.setMobile("1391XXXXXX63");
        order.setCustomer(customer);

        // 在实际项目中，setter会更多

        order.taken();

        //when
        order.canel();

        //then
        thrown.expect(IllegalStateException.class);
    }
}
```

当然，有一种取巧的方式，即跳过不需要的属性赋值，但这种方法在复杂项目中，随着生产代码变更，极易造成空指针异常（NullPointerException），所以只是一种治标不治本的方法。

在阅读了一些文章后，我发现在设计测试时，可以对测试数据进行分类：

1.**应用引用数据**（Application reference data）：它们是测试无关数据，但它们是应用程序或测试启动所必需的，这些数据往往是指一些基础数据。大部分单元测试并不需要应用引用数据。

2.**测试引用数据**（Test reference data）：指那些和测试相关，但是对测试行为没有多大影响的数据。例如OrderTest案例中，除了状态（status）的其他数据。

3.**测试专用数据**（Test specific data）：真正影响测试行为的特征数据。例如OrderTest案例中的订单状态（status）

显然，我们应该在测试中尽可能降低准备测试引用数据的存在感，这时我找到了Nat Pryce提出的[Test Data Builder模式](http://www.natpryce.com/articles/000714.html)。简而言之，Test Data Builder是为测试定制化的Builder，在Builder的基础上指定了测试数据默认值。

如果使用Test Data Builder来重构OrderTest，其可读性会显著提升：

```java
import static com.github.hippoom.tdb.OrderTestDataBuilder.anOrder;

public class OrderTest {

    @Rule
    public ExpectedException thrown = ExpectedException.none();

    @Test
    public void it_should_reject_when_cancel_given_order_is_taken() {
        //given
        Order order = anOrder().taken().build();

        //when
        order.canel();

        //then
        thrown.expect(IllegalStateException.class);
    }
}
```

**2.2** **尽量为每个测试设计独立的测试数据**

**2.2.1** **尽可能使用随机值作为默认测试数据**

在依赖数据库的测试中，经常会遇到假警报（False Alarm），如果你排查了半天，却发现是由于测试数据冲突，而不是被测试组件造成的，一定会恼怒不已。为了缓解这个现象，最有效的方式是为每个测试设计专有的测试数据集。例如，在测试更新订单状态这个持久化操作时，我们希望被测试订单的生命周期只由该测试控制，从而避免测试数据冲突。利用Test Data Builder，我们可以将用随机生成的字符串来作为测试订单的默认ID，这样绝大多数测试都不会因为ID冲突而失败了。

```java
public class OrderTestDataBuilder {

    private Order target = new Order(Order.Identity.next());

    public static OrderTestDataBuilder anOrder() {
        return new OrderTestDataBuilder()
            //随机的ID作为测试订单的默认ID，可以降低测试数据冲突概率
            //对于需要指定ID的场景，
            // 可以调用OrderTestDataBuilder.with(Identity.of(3))的方式来实现
            .with(nextId())
            .with(IN_STORE)
            .with(aCustomer())
            .withTotal(100.0);
    }

    public OrderTestDataBuilder with(Order.Identity value) {
        target.setId(value);
        return this;
    }

    private static Order.Identity nextId() {
        return Order.Identity.of(UUID.randomUUID().toString());
    }
}
```

进一步的，如果订单里面持有客户（Customer）的ID，我也建议进一步将CustomerTestDataBuilder的ID默认值也设置为随机值，这可以防止为了构造测试订单而衍生构造出的客户，不容易和其他涉及客户的测试产生冲突。

除了ID之外，还有不少属性也是可以使用随机值的，例如电话、住址等。你可以找到一些开源库来帮助你，比如[binarywang同学的java-testdata-generator](https://github.com/binarywang/java-testdata-generator)

**2.2.2** **集中记录测试中必须使用固定值**

这是衍生自**尽可能使用随机值作为默认测试数据**的实践，因为肯定会遇到不方便使用随机值，而必须用固定值的测试。这时，推荐使用一份一页纸文档，将它们集中记录起来，以便在设计其他测试的数据时，作为参考。以下是一个例子

```java
JdbcTest:
    Order, SCOPE
    //以ORDER_作为前缀进行ID登记，方便搜索
    ORDER_1, Insert测试
    ORDER_2, 更新测试

```

**2.3** **尽量使用生产代码来准备测试数据**

让我们从一个例子开始：如果你想测试一个基于SQL的订单查询实现，那么就需要准备一批订单。一个显而易见的数据准备方法是，在测试执行前使用SQL脚本向数据库插入数据。最初我也使用这种方法，甚至尝试去扩展了DBUnit，见[spring-test-dbunit-template](https://github.com/Hippoom/spring-test-dbunit-template)。但我渐渐发现，这个方法在复杂项目中有个致命的缺陷：同一张表经常会被用在多个测试中，而为了遵循“**尽量为每个测试设计独立的测试数据**”，往往会有多份该表的测试数据文件。当该表随着功能演进而发生变更时，测试数据文件的修改工作量非常大。究其原因，其实还是由于测试数据准备过程使用了Hack的方式绕过了生产代码，自成一系。

因此，我推荐尽量使用生产代码来准备测试数据。回到例子里，假如我们要测试订单的持久化操作，建议用OrderTestDataBuilder和JdbcOrderRepository配合来准备数据：

```java
public class JdbcOrderRepositoryTest {

    @Test
    public void it_should_handle_update() {
        //given
        OrderTestDataBuilder orderBuilder = anOrder();

        Order order = orderBuilder.build();

        jdbcOrderRepository.save(order);

        Order before = jdbcOrderRepository.findBy(order.getId());
	      assertThat(after.getStatus(), is(PENDING)); //确保前置条件正确

        //when
        before.taken();

        jdbcOrderRepository.update(order);

        //then
        Order after = jdbcOrderRepository.findBy(order.getId());
        assertThat(after.getStatus(), is(TAKEN));
    }
}
```

当然，采用这种方式可能会造成测试准备数据篇幅较大，这里可以采用分层测试、自适应断言、Helper Method以及Test Data List Builder来缓解。前几个方法已经有很多文章介绍，Test Data List Builder我会在文章末尾进一步介绍。

**3 测试数据准备和管理的支撑工具还不多**

有了实践之后，自然想到的是用一些工具来提升效率，甚至将它们固化下来。但遗憾的是，这方面的工具还不多，以下是我的一些脑洞：

**3.1 自动生成TestDataBuilder**

实际上手工编写TestDataBuilder并不难，但我就是懒。如果能有一个lombok式的Annotation自动生成TestDataBuilder就好了。

**3.2** **Test Data List Builder**

在准备批量数据时，逐个使用TestDataBuilder构造还是比较麻烦的。为此，我设计了一个**Test Data List Builder，**以更简洁的方式准备批量数据，以下这个例子中，使用GenericTestDataListBuilder.listOfSize可以一次性构造5个订单，并且指定每个订单的特征。

```java
import static com.github.hippoom.tdb.GenericTestDataListBuilder.listOfSize;
import static com.github.hippoom.tdb.Location.IN_STORE
import static com.github.hippoom.tdb.Location.TAKE_AWAY

List<Order> orders = listOfSize(5, sequence -> new OrderBuilder())
   .theFirst(2, builder -> builder.is(TAKE_AWAY)) // (1)
   .number(3, builder -> builder.is(IN_STORE))    // (2)
   .theLast(2, builder -> builder.paid())         // (3)
   .build();

//(1) declaring the first two elements apply a Function<OrderBuilder, OrderBuilder> that customizes the element
//(2) declaring the third element in the list applies a Function<OrderBuilder, OrderBuilder>
//(3) declaring the last two elements apply a Function<OrderBuilder, OrderBuilder>
```

该项目已经[开源](https://github.com/Hippoom/test-data-builder)，并可以在[Maven Central](https://mvnrepository.com/artifact/com.github.hippoom/test-data-builder)上找到。

**3.3** **测试数据冲突自动提示**

在**2.2.2** **集中记录测试中必须使用固定值**中手工记录固定值的方法要求团队有很高的自律性，而且手工更新确实很麻烦。如果有一个工具，可以帮助我们自动监测测试集中的数据，并提示存在冲突就好了

**4.写在最后**

以上多来自于我项目中的实践经验，个人视野限制，有诸多不足。希望借此机会，也听听诸位的见解。