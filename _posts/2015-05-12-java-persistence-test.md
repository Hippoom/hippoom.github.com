---

title:   用spring-test、dbunit、flyway测试Java持久层
category: continuous delivery  
layout: post

---

### 这里的持久化层是什么

&emsp;&emsp;在Java应用，持久化组件指那些负责与数据库打交道的对象。对，就是那些DAO或是Repository的实现咯。
作为应用程序中重要的一层，它们有资格拥有单独设计的测试，以便

#### 以最小的代价验证SQL操作和对象-关系映射

&emsp;&emsp;对象-关系映射的验证工作很琐碎，不适合放在验收测试中，
同样地，用验收测试来验证组合查询的SQL语句实在是大材小用。
利用好[测试金字塔](http://martinfowler.com/bliki/TestPyramid.html)，一个更底层的专项测试更经济高效。

#### 验证持久化组件的对象装配

&emsp;&emsp;持久化组件也有它的依赖，例如数据源或是持久化框架的组件（例如Hibernate的SessionFactory），
但是使用测试替身的单元测试在这里收益并不大。
一来，由于大量成熟的商业、开源组件，持久化组件变成了很薄的一层，实现代码越来越少，甚至[没有](http://spring.io/guides/gs/accessing-data-jpa/)。
你应该更关心它们能否正确地与数据库交互，而不是能否正确地与依赖交互。
二来，现代Java应用一般都会使用依赖注入容器来装配对象，通过测试可以驱动或验证持久化组件的依赖注入机制已准备就绪。  

### 测试开始

如果使用Spring作为依赖注入的容器，可以很方便地利用[spring-test](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/testing.html)来编写测试，这样就能验证对象装配机制了。


{% highlight java %}

@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = Application.class)
public class HibernateOrderRepositoryTest {

    @Autowired private PlatformTransactionManager transactionManager;

    @Autowired private HibernateOrderRepository subject;

    @Test public void should_saves_order() throws Exception {

        final String trackingId = "123456";
        final Order order = new Order(trackingId);
        order.append("item1", 1);
        order.append("item2", 2);

        subject.store(order);

        // 如果有延迟加载，需要使用TransactionTemplate，否则会出现no session异常
        new TransactionTemplate(transactionManager)
                .execute(status -> {

                    final Optional<Order> saved = subject.findByTrackingId(trackingId);
                    assertThat(saved.isPresent(), is(true));
                    final Order loaded = saved.get();
                    assertThat(order.getStatus(), equalTo(WAIT_PAYMENT));
                    assertThat(loaded.getItems().size(), is(2));

                    assertThat(loaded.getItems().get(0).getName(), equalTo("item1"));
                    assertThat(loaded.getItems().get(0).getQuantity(), is(1));

                    assertThat(loaded.getItems().get(1).getName(), equalTo("item2"));
                    assertThat(loaded.getItems().get(1).getQuantity(), is(2));

                    return loaded;
                });
    }
}

{% endhighlight %}

但是这一点也不酷，在仅有两张表，区区几个字段的情况下，测试的验证部分就变得又臭又长。

### DbUnit，分离数据和代码

[DbUnit](http://dbunit.sourceforge.net/)是一个历史悠久的开源项目了，它可以分离测试代码和数据，为你的测试“瘦身”，配合[spring-test-dbunit](http://springtestdbunit.github.io/spring-test-dbunit/)食用，口味更佳。

{% highlight java%}

@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = {Application.class})
@TestExecutionListeners({DependencyInjectionTestExecutionListener.class,
        DirtiesContextTestExecutionListener.class,
        DbUnitTestExecutionListener.class})
public class HibernateOrderRepositoryTest {

    @Autowired private PlatformTransactionManager transactionManager;

    @Autowired private OrderRepository subject;

    @ExpectedDatabase(value = "classpath:order_save_expected.xml",
                      assertionMode = NON_STRICT_UNORDERED)
    @Test public void should_saves_order() throws Exception {

        final String trackingId = "240eff2f-6c38-4998-9287-2e447dac4fd4";
        final Order order = new Order(trackingId);
        order.append("item1", 1);
        order.append("item2", 2);

        subject.store(order);
    }
}

{% endhighlight %}

{% highlight xml %}
<!--order_save_expected.xml-->
<dataset>
    <t_order tracking_id="240eff2f-6c38-4998-9287-2e447dac4fd4"
             status="WAIT_PAYMENT"/>
    <t_order_item tracking_id="240eff2f-6c38-4998-9287-2e447dac4fd4"
                  name="item1"
                  quantity="1"/>
    <t_order_item tracking_id="240eff2f-6c38-4998-9287-2e447dac4fd4"
                  name="item2"
                  quantity="2"/>
</dataset>
{% endhighlight %}

通过@ExpectedDatabase，spring-test-dbunit将数据分离到更易于编辑的文件中，简化了原本繁琐的验证部分。
值得一提的是，DbUnit本身还支持多种文件格式，比如xls文件，编辑更容易，缺点是无法方便地在版本管理系统中查看修订历史。

你还可以进一步简化测试数据的准备，用@DatabaseSetup在测试之前向数据库刷入“原型”数据，
之后利用find方法将其取出后，使用对象映射库（比如[ModelMapper](http://modelmapper.org/getting-started/)）
来克隆出一个新对象。

{% highlight java%}

@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = {Application.class})
@TestExecutionListeners({DependencyInjectionTestExecutionListener.class,
        DirtiesContextTestExecutionListener.class,
        DbUnitTestExecutionListener.class})
public class HibernateOrderRepositoryTest {

    @Autowired private PlatformTransactionManager transactionManager;

    @Autowired private OrderRepository subject;

    @DatabaseSetup("classpath:order_save_fixture.xml")
    @ExpectedDatabase(value = "classpath:order_save_expected.xml",
                      assertionMode = NON_STRICT_UNORDERED)
    @Test public void should_saves_order() throws Exception {

      final String trackingIdOfPrototype = "240eff2f-6c38-4998-9287-2e447dac4fd3";
      final String trackingIdOfToBeSaved = "240eff2f-6c38-4998-9287-2e447dac4fd4";

      final Order toBeSaved = clone(trackingIdOfPrototype, trackingIdOfToBeSaved);

        subject.store(toBeSaved);
    }

    private Order clone(String trackingIdOfPrototype, String trackingIdOfToBeSaved) {
        return new TransactionTemplate(transactionManager)
                .execute(status -> {
                    final Optional<Order> orderMaybe = subject.findByTrackingId(trackingIdOfPrototype);
                    return clone(orderMaybe.get(), trackingIdOfToBeSaved);
                });
    }

    private Order clone(Order prototype, String newTrackingId) {
      ModelMapper mapper = new ModelMapper();

      PropertyMap<Order, Order> orderMap = new PropertyMap<Order, Order>() {
          protected void configure() {
              map(newTrackingId, destination.getTrackingId());
          }
      };
      mapper.getConfiguration()
              .setFieldMatchingEnabled(true)
              .setFieldAccessLevel(PRIVATE);
      mapper.addMappings(orderMap);

      return mapper.map(prototype, Order.class);
  }
}

{% endhighlight %}

至此，测试数据与代码已经完全分离了，唯一的遗憾的是，测试数据中还有不少重复。

{% highlight xml %}
<!--order_save_expected.xml，除了tracking_id其他字段都重复了-->
<dataset>
    <t_order tracking_id="240eff2f-6c38-4998-9287-2e447dac4fd3"
             status="WAIT_PAYMENT"/>
    <t_order_item tracking_id="240eff2f-6c38-4998-9287-2e447dac4fd3"
                  name="item1"
                  quantity="1"/>
    <t_order_item tracking_id="240eff2f-6c38-4998-9287-2e447dac4fd3"
                  name="item2"
                  quantity="2"/>

    <t_order tracking_id="240eff2f-6c38-4998-9287-2e447dac4fd4"
             status="WAIT_PAYMENT"/>
    <t_order_item tracking_id="240eff2f-6c38-4998-9287-2e447dac4fd4"
                  name="item1"
                  quantity="1"/>
    <t_order_item tracking_id="240eff2f-6c38-4998-9287-2e447dac4fd4"
                  name="item2"
                  quantity="2"/>
</dataset>
<!--order_save_fixture.xml，又重复了-->
<dataset>
    <t_order tracking_id="240eff2f-6c38-4998-9287-2e447dac4fd3"
             status="WAIT_PAYMENT"/>
    <t_order_item tracking_id="240eff2f-6c38-4998-9287-2e447dac4fd3"
                  name="item1"
                  quantity="1"/>
    <t_order_item tracking_id="240eff2f-6c38-4998-9287-2e447dac4fd3"
                  name="item2"
                  quantity="2"/>
</dataset>
{% endhighlight %}

### 集成到部署流水线

在运行测试之前，需要预先为关系型数据库定义模式（Schema）。
在开发环境，这可以手工完成，但是一旦集成到部署流水线，就必须自动化了，
而且这个方案必须考虑生产环境部署，尽可能地利用部署流水线演练该方案以降低最终的发布风险，
因此简单地通过一个脚本端掉数据库再完全重建显然不可取。
一种常见的方式是采用数据库模式版本管理的工具增量迁移，比如[Flyway](http://flywaydb.org/documentation/command/migrate.html)，
方案确定后，就可以修改测试，在每次运行测试前，以同样地机制为测试数据库迁移模式。
值得一提的是Flyway提供了spring-test的扩展，通过@FlywayTest和FlywayTestExecutionListener便可集成

{% highlight java %}
@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = {Application.class})
@TestExecutionListeners({DependencyInjectionTestExecutionListener.class,
        DirtiesContextTestExecutionListener.class,
        DbUnitTestExecutionListener.class, FlywayTestExecutionListener.class})
@FlywayTest(invokeCleanDB = false)
public class HibernateOrderRepositoryTest {
}
{% endhighlight %}

@FlywayTest会自动找到classpah:db/migration/下的迁移脚本

{% highlight sql %}

//V1__create_t_order.sql
create table t_order (
    tracking_id VARCHAR(32) not null,
    status varchar(50) not null,
    CONSTRAINT pk_order PRIMARY KEY (tracking_id)
);

//V2__create_t_order_item.sql
create table t_order_item (
    tracking_id VARCHAR(32) not null,
    name varchar(200) not null,
    quantity int not null
);

{% endhighlight %}

{% highlight html %}

o.f.c.i.command.DbMigrate      : Current version of schema "ordering": << Empty Schema >>
o.f.c.i.command.DbMigrate      : Migrating schema "ordering" to version 1 - create t order
o.f.c.i.command.DbMigrate      : Migrating schema "ordering" to version 2 - create t order item
o.f.c.i.command.DbMigrate      : Successfully applied 2 migrations to schema "ordering" (execution time 00:00.078s).

{% endhighlight %}

### 测试的价值不止于验证对错

对持久化组件的测试涉及有状态的外部系统（最主要是数据库），有一定的难度和维护成本。
如果用质疑的态度去看待对这些成本是否合理，有时还可以反馈出一些设计问题。
最常见的例子就是筛选有特定领域含义的数据：
假设需要筛选超过30分钟还没有付款的订单并自动取消，常见的实现方式是使用条件查询

{% highlight java %}

    SELECT t.order_id
    FROM t_order
    WHERE t.status = 'WAIT_PAYMENT'
    AND t.placed_at <= SYSDATE - 30/(24 * 60)
{% endhighlight %}

这个实现的缺点是加重了持久化测试的负担。新增一种查询组合常常需要新增一组准备数据，用来验证筛选条件是否起作用。
而且订单作为应用程序中的重要对象，往往有很多信息，但不一定和测试有关系，准备订单的数据往往很复杂又没必要。
事实上，测试在告诉你这个设计可能需要改进。也许可以通过另一种方式来实现这个功能，每当有订单生成时，就向一张单独的表中插入一条记录，当订单的状态发生变化时就去删除该表中对应的记录。
那么原先的对t_order的条件组合查询就变为了对单表的简单查询：

{% highlight sql %}

    SELECT t.order_id
    FROM t_order_timer
    WHERE t.overdue <= SYSDATE;
{% endhighlight %}

这样测试的数据准备就简单了，而复杂度转移到了应用代码中，通过单元测试来保护，这样整体的测试成本更低
