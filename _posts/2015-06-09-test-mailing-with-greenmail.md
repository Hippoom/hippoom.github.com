---

title:   邮差必备——Java邮件发送的测试金字塔
category: continuous delivery  
layout: post

---

&emsp;&emsp;企业应用开发中或多或少会遇到通知用户的场景，比如用户预订酒店后邮件通知确认等等。
如何测试这类功能呢？“那还不简单，打开邮箱收一下不就知道了？”
但作为一位高（lan）效（duo）的Java开发者，我还是希望能找到一种自动化的测试方案。

### 单元测试，通知用户 VS 向用户发送邮件

&emsp;&emsp;先来看一则用户故事

    作为酒店的预订管理员
    我希望在确认订单后向客户发送预订成功的邮件
    以便让客户知晓预订已经确认

场景明确，但也有一丝“臭味”，这则故事掺入了“实现细节”，既然目的是通知客户，那么发短信是否可以呢？让我们修改一下：

    作为酒店的预订管理员
    我希望在确认订单后通知客户
    以便让客户知晓预订已经确认


那么在代码中，是否体现了恰当的抽象层级呢？如果顺着最初的故事，代码可能会写成这样：

{% highlight java %}

import org.springframework.mail.MailSender;

public class OrderingHandler {
    @Setter
    private MailSender mailSender;
    @Setter
    private String from;
    @Setter
    private String acknowledgeMailSubject;

    public void acknowledge(TrackingId orderId) {
        final Order order = orderingService.acknowledge(orderId);
        final SimpleMailMessage acknowledgeMail = new SimpleMailMessage();
        acknowledgeMail.setFrom(from);
        acknowledgeMail.setTo(order.getCustomer().getEmail());
        acknowledgeMail.setSubject(acknowledgeMailSubject);
        acknowledgeMail.setText("Hello, your order [" + order.getTrackingId() + "] is acknowledged.");
        mailSender.send(acknowledgeMail);
    }
{% endhighlight %}

诚然，可以用抽取方法把发送邮件的细节提取到一个私有方法中，但在单元测试中，还是不免要与它们打交道。
一方面，你不免要mock无法控制的外部代码（比如上例中spring-framework的MailSender）；
另一方面，如果要求增加或替换通知客户的方式时，单元测试也要随之修改。
这些现象实际上在说：**不同抽象级别的代码混在一起了**。

{% highlight java %}

public class OrderingHandler {
    @Setter
    private ApplicationEvents applicationEvents;

    public void acknowledge(TrackingId orderId) {
        final Order order = orderingService.acknowledge(orderId);
        applicationEvents.notifyOrderAcknowledged(order);
    }
{% endhighlight %}

事实上，发送邮件的代码涉及外部框架或库、配置信息和外部服务，最好将它们隔离在应用代码中有限的区域。
现在应用代码基于通知用户的抽象，不管是发短信还是发邮件，是同步发还是异步发都不会影响单元测试了。

### 组件测试，验证你的集成代码和配置机制

在单元测试中，我们绕开了外部依赖的复杂性，那么邮件发送的测试由哪部分没有办法验证这段代码能够真正地发送邮件，因为诸如subject、from、甚至关键的mailSender都是注入的，我们还需要集成测试来验证它们被正确地配置起来了。这里有个问题是，如果使用真正的邮件服务器，我们怎么在自动化测试中验证邮件是否发送成功呢？如果需要手工地去刷新收件箱检查邮件，那么自动化测试就没有意义了。让我们先来看看最重要的mailSender是如何配置的，见Listing-4：

<pre>
<bean id="crs.MailSender" class="org.springframework.mail.javamail.JavaMailSenderImpl">
    <property name="host">
        <value>${runtime.crs.email.host}</value>
    </property>
    <property name="port">
        <value>${runtime.crs.email.port}</value>
    </property>
    <property name="javaMailProperties">
        <props>
            <prop key="mail.smtp.auth">true</prop>
        </props>
    </property>
    <property name="username">
        <value>${runtime.crs.email.username}</value>
    </property>
    <property name="password">
        <value>${runtime.crs.email.password}</value>
    </property>
</bean>
</pre>
spring framework已经为我们提供了MailSender的开箱实现JavaMailSenderImpl，而邮件服务器的搭建和配置属于基础架构，一般会有专门的团队负责并测试，这意味着我们并不需要测试JavaMailSenderImpl的代码（spring framework的团队负责它的测试）以及邮件服务器本身，而我们只需要测试JavaMailSenderImpl的配置是否有遗漏。如此一来，只要我们有一个和邮件服务器使用同一个传输协议的测试替身就可以完成测试了，而green mail正是我们要找的东西。green mail支持多种邮件传输协议，包括常见的smtp、pop3等，下面来看一下它是如何验证邮件发送的，见Listing-5：

Listing-5
<pre>
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = { "classpath:applicationContext.xml" })
public class MailSenderIntegrationTests {
    @Autowired
    private MailSender target;

    private GreenMail mailServer;

    @Before
    public void startMailServer() {
        mailServer = new GreenMail();
        mailServer.start();
    }

    @After
    public void shutdownMailServer() {
        mailServer.stop();
    }

    @Test
    public void mailSendingIsFine() throws Exception {
        final SimpleMailMessage test = new SimpleMailMessage();
        test.setFrom("from@test.com");
        test.setTo("to@test.com");
        test.setSubject("test subject");
        test.setText("test content");

        target.send(test);
        assertTrue(mailServer.waitForIncomingEmail(2000, 1));
        assertThat(GreenMailUtil.getBody(mailServer.getReceivedMessages()[0]),
equalTo("test content"));
        //omitted asserts using GreenMailUtil
    }
}
</pre>

只要在properties文件中，将配置修改一下，就可以运行这个测试了（见Listing-6）

Listing-6
<pre>
runtime.crs.email.username=test
runtime.crs.email.password=test
runtime.crs.email.host=localhost
runtime.crs.email.port=3025
</pre>

如果测试通过了，那么恭喜你的JavaMailSenderImpl配置方案是正确的，只要在部署时替换不同的地址和认证信息就行了。你可以直接对OrderApplicationEvents进行集成测试，从而验证subject、from以及mailSender是否正确地注入了，不过如果你的应用中如果有多处发送邮件的需求且有自动化验收测试，我建议使用一个针对JavaMailSenderImpl的集成测试即可，其他的配置可以使用验收测试来验证。

当然，当部署到手工验收测试环境和生产环境时，你不会使用greenmail，届时只要简单验证一下任一邮件发送成功就可以了（只要你所有的邮件发送代码都注入的是同一个测试过的mailSender）。

### 验收测试，

### 等等，邮筒在哪里？

h2. *Q：Further reading?*
A:你可以访问greenmail的站点了解使用例子（+www.icegreen.com/greenmail+）

h2. *Q：Maven GAV?*
A:<pre>
<dependency>
    <groupId>com.icegreen</groupId>
    <artifactId>greenmail</artifactId>
    <version>1.3.1b</version>
    <scope>test</scope>
</dependency>
</pre>

h2. *Q：我如何可以获得该成就*
A:使用spring framework MailSender和green mail完成一次helloworld的发送，在测试中验证from、to、subject和content，你将获得成就“邮差马龙”
