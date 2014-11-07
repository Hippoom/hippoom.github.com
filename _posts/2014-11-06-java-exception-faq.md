---

title:   JAVA异常设计FAQ
category: ramblings  
layout: default

---

### Q:受检还是非受检，这是一个问题？

A:这是一个好问题。不过强烈推荐将自定义的异常设计为非受检的，原因如下：  
1.如果期望调用者能够适当地恢复/处理错误，对于这种情况就应该使用受检异常，但在实际情况中，这个原则被大量的误用，在绝大多数的情况下，系统报告了错误，我们都很难处理，最简单（有时甚至是唯一的办法）就是直接把错误报告给用户。  
2.受检异常要求客户端一定要继续抛出或是处理异常，由于第一个原因，常常导致客户端选择继续抛出异常，如果这个异常是系统较低层的组件抛出的，你经常会看到如下场景：  

Listing-1 nested throws
{% highlight java %}
public class TopService {
    
    private MiddleService mService;
    
    public void doBusiness() throws ACustomeCheckedException {
        mService.doBusiness();
    }

}

public class MiddleService {
    
    private Dao dao;
    
    public void doBusiness() throws ACustomeCheckedException {
        dao.save();
    }

}

public class Dao {
    
   public void doBusiness() throws ACustomeCheckedException {
        //omitted persistence logic
    }

}
{% endhighlight %}

由于底层的Dao声明抛出一个受检异常，导致所有直接/间接调用它的组件也不得不声明抛出该异常，情况更糟的是，如果此时Dao由于实现变化又需要抛出另一个受检异常，这时你不得不修改所有直接/间接调用它的组件，让它们也抛出这个新的异常。这造成添加了一个底层异常，要改动大量的高层组件，使得添加新的异常非常麻烦，所以有时候干脆选择将一些系统底层（可能是框架）抛出的受检异常包裹成非受检异常，比如Spring framework就将受检异常SQLException包裹在自定义的非受检异常DataAccessException中。

基于以上两个原因，我推荐绝大多数（如果不是所有）情况下，让你的自定义异常继承 *RuntimeException* ，使其成为非受检异常。

### Q:异常中要包含哪些信息？

A:简单地说，一定要包括描述和上下文。你是否遇到这样的情况：有个任务要分析日志，检查系统运行情况，结果你在日志看到一句：
{%  highlight java %}
“Error！Cannot cancel order!”
{%  endhighlight  %}
然后就没了，这个时候，真心郁闷啊，既不知道是哪张订单，也不知道为什么不能取消订单，只能查看代码：

Listing-2 a mysterious error

{% highlight java %}
public class Order {

    public void cancel() {
        if (order.isLocked()) {
            throw CannotCancelOrderException.hasBeenLocked(orderId);
        }

        if (order.isCanceled()) {
            throw new CannotCancelOrderException(); 
        }
        //omitted code
    }
}
{% endhighlight %}

如果重构一下，修改为这样就好多了：

Listing-3 exception with context and description 
{% highlight java %}
public class Order {

    public void cancel() {
        if (order.isLocked()) {
            throw CannotCancelOrderException.hasBeenLocked(orderId);
        }

        if (order.isCanceled()) {
            throw CannotCancelOrderException.hasBeenCanceled(orderId);
        }
        //omitted code
    }
}

public class CannotCancelOrderException extends RuntimeException {
    
    public static CannotCancelOrderException hasBeenLocked(String orderId) {
        return new CannotCancelOrderException("order["+ orderId + "] has been locked");
    }

    public static CannotCancelOrderException hasBeenCanceled(String orderId) {
        return new CannotCancelOrderException("order["+ orderId + "] has already been canceled");
    }

    public CannotCancelOrderException(String message) {  
        super(message);  
    }
}
{% endhighlight %}

如果你的异常是由其他异常导致的，可别把cause落下了：

Listing-4 wrapped exception 

{% highlight java %}
public class Order {

    public void cancel() {
        //omitted code
        } catch (SupplierAccessException e) {  
            throw CannotCancelOrderException.wrapped(orderId, e);  
        } 
    }
}

public class CannotCancelOrderException extends RuntimeException {
    
    //omitted factory methods and constructors

    public static CannotCancelOrderException wrapped(String orderId, Throwable t) {  
        return new CannotCancelOrderException("Cannot cancel order[" + orderId + "] due to " + t.getMessage(), t);  
    }

    public CannotCancelOrderException(String message, Throwable t) {  
        super(message, t);  
    }
}
{% endhighlight %}

### Q:如果没人能够处理异常，就让异常直接抛出吗？或是总要有个组件来捕获它？

A: 比如在一个MVC Controller中，调用了下单服务：

Listing-5
{% highlight java %}
String orderId = placeOrderService.placeOrder(productCode, quantity, price);
{% endhighlight %}

由于使用了Spring framework来提供声明式事务，所以如果下单时如果发生错误，我们会让placeOrder()会抛出异常从而触发回滚。为了不让用户看到500报错及满屏的错误堆栈，一般会使用try/catch块捕获错误：

Listing-6
{% highlight java %}
try {  
    orderId = placeOrderService.placeOrder(productCode, quantity, price);   
} catch (Exception e) {  
    model.addAttribute("error", e.getMessage());  
    //omitted logging code if placeOrderService doesn't log on it own
}
{% endhighlight %}

由于大多数情况下，我们只是要报告错误，一般推荐在最靠近的用户的“层”中捕获异常并根据UI接口填充错误报告（对于View可能是将错误报告放入HttpServletRequest对象待渲染，对于远程访问接口可能是翻译并返回错误码）。

### Q:我们的WebService接口需要以错误码的方式报告错误，这时怎样利用异常呢？

A: 如果要返回错误码而不是直接抛出异常，那么总会有一个组件会捕获所有的异常并进行翻译。最简单的解决方案是硬编码：

Listing-7 hard coded status code translation
{% highlight java %}
try {  
    orderId = placeOrderService.placeOrder(productCode, quantity, price);   
} catch (InsufficientInventoryException e) {  
    statusCode = INSUFFICIENT_INVENTORY;  
} catch (ExpriedPriceException e) {  
    statusCode = EXPIRED_PRICE;  
} catch (NoSuchProductException e) {  
    statusCode = NO_SUCH_PRODUCT;  
} catch (Exception e) {  
    statusCode = UNKNOWN;  
}
{% endhighlight %}

但是如果异常很多，这里的翻译篇幅就会比较大，而且如果要新增一个异常，就要回来修改catch，维护起来很麻烦。由于这些异常都是自定义的，我们可以将其设计成一个体系（Hierachy）：

Listing-8 exception hierachy
{% highlight java %}
public class PlaceOrderServiceFacadeImpl implements PlaceOrderServiceFacade {

    public PlaceOrderRs handle(PlaceOrderRq rq) {
        //omitted code
        try {  
            orderId = placeOrderService.placeOrder(productCode, quantity, price);   
            statusCode = SUCCESS;
        } catch (UncheckedApplicationException e) {  
            statusCode = e.getStatusCode();  
        } catch (Exception e) {  
            statusCode = UNKNOWN;  
        }
        return new PlaceOrderRs(statusCode, order);
    }
}

public abstract class UncheckedApplicationException {

     //omitted factory methods and constructors
     public abstract String getStatusCode();
}
{% endhighlight %}

要新增异常的话，只需要让其继承UncheckedApplicationException并实现getStatusCode()即可。同样的，如果除了要告知statusCode，如果还需要返回本地化的提示信息，还可以加上getI18nCode()、getI18nArgs()这样的方法：

Listing-9 i18 supported exception hierachy 
{% highlight java %}
public class PlaceOrderServiceFacadeImpl implements PlaceOrderServiceFacade {

    public PlaceOrderRs handle(PlaceOrderRq rq) {
        //omitted code
        try {  
            orderId = placeOrderService.placeOrder(productCode, quantity, price);   
            statusCode = SUCCESS;
            message = messageSource.getSuccess(locale));
        } catch (UncheckedApplicationException e) {  
            statusCode = e.getStatusCode();  
            message = messageSource.getMessage(e, locale));
        } catch (Exception e) {  
            statusCode = UNKNOWN;  
            message = messageSource.getUnknownError(e, locale));
        }
        return new PlaceOrderRs(statusCode, order, message);
    }
}

public abstract class UncheckedApplicationException {

    //omitted factory methods and constructors
    public abstract String getStatusCode();
    
    public abstract String getI18nCode();

    public abstract String[] getI18nArgs();
}

public class DelegateToSpringMessageSourceFacadeImpl implments MessageSourceFacade {
    public String getMessage(UncheckedApplicationException e, Locale locale) {  
        return delegate.getMessage(e.getI18nCode(), e.getI18nArgs(), locale);//delegate to Spring MessageSource
    }
}
{% endhighlight %}

另外，因为往往一个WebService的实现中有多个方法，如果使用之前的方案，那么每个方法都要编写一次try/catch块，但是内容是一样的，对于这类情况，我推荐使用AOP的方式来捕获异常：

Listing-10 a web service impl example
{% highlight java %}
public class PlaceOrderServiceFacadeImpl implements PlaceOrderServiceFacade {
    public QuoteRs handle(QuoteRq rq) {
        //omitted code
        try {  
            //omitted quoting code 
            statusCode = SUCCESS;
            message = messageSource.getSuccess(locale));
        } catch (UncheckedApplicationException e) {  
            statusCode = e.getStatusCode();  
            message = messageSource.getMessage(e, locale));
        } catch (Exception e) {  
            statusCode = UNKNOWN;  
            message = messageSource.getUnknownError(e, locale));
        }
        return new QuoteRs(statusCode, quotes, message);
    }

    public PlaceOrderRs handle(PlaceOrderRq rq) {
        //omitted code
        try {  
            orderId = placeOrderService.placeOrder(productCode, quantity, price);   
            statusCode = SUCCESS;
            message = messageSource.getSuccess(locale));
        } catch (UncheckedApplicationException e) {  
            statusCode = e.getStatusCode();  
            message = messageSource.getMessage(e, locale));
        } catch (Exception e) {  
            statusCode = UNKNOWN;  
            message = messageSource.getUnknownError(e, locale));
        }
        return new PlaceOrderRs(statusCode, order, message);
    }
}
{% endhighlight %}

Listing-10 an around advice catching exception
{% highlight java %} 
public class LogAndReturnHandler implements MethodInterceptor {
	@Setter
	private LoggingSupport loggingSupport;

	@Override
	public Object invoke(MethodInvocation invocation) throws Throwable {
		//omitted logging code
		try {
			//omitted logging code
			Object returning = invocation.proceed();
			//omitted logging code
			return returning;
		} catch (UncheckedApplicationException e) {
			//omitted logging code
			return populateRs(invocation, e.getStatusCode(), e.getMessage());
		} catch (Throwable t) {
			//omitted logging code
			return populateRs(invocation, StatusCode.UNKNOWN_ERROR,
					t.getMessage());
		}
	}

	private Class<?> returnTypeOf(MethodInvocation invocation) {
		return invocation.getMethod().getReturnType();
	}

	private GenericRs aGenericRsWith(String statusCode, String message) {
		return new GenericRs(statusCode, message);
	}

	private Object populateRs(MethodInvocation invocation, String statusCode,
			String message) {
		final GenericRs aGenericRs = aGenericRsWith(statusCode, message);
		Object object = mapper(invocation, aGenericRs);
		return object;
	}

	private Object mapper(MethodInvocation invocation,
			final GenericRs aGenericRs) {
		ModelMapper mapper = new ModelMapper();
		Object object = mapper.map(aGenericRs, returnTypeOf(invocation));
		return object;
	}
}

public class PlaceOrderServiceFacadeImpl implements PlaceOrderServiceFacade {
    public QuoteRs handle(QuoteRq rq) {
       
        //omitted quoting code 
       
        return new QuoteRs(SUCCESS, quotes, successMessageFor(locale));
    }

    public PlaceOrderRs handle(PlaceOrderRq rq) {
        //omitted code
      
        orderId = placeOrderService.placeOrder(productCode, quantity, price);            
        
        return new PlaceOrderRs(SUCCESS, order, successMessageFor(locale));
    }
    
    private String successMessageFor(Locale locale) {
        return messageSource.getSuccess(locale));
    }
}

spring xml
<bean id="irs.MemberBookingServiceFacadeImpl" class="org.springframework.aop.framework.ProxyFactoryBean">
    <property name="target" ref="irs.MemberBookingServiceFacadeImplTarget" />
    <property name="interceptorNames">
        <list>
            <value>irs.LogAndReturnHandler</value>
        </list>
    </property>
</bean>
{% endhighlight %}

这里的要点在于虽然每个方法都会catch异常，但是每个方法返回类型是不一致的（比如QuoteRs/PlaceOrderRs），需要找到一种方法能根据返回类型自动实例化并填充statusCode和message的方法（一般发生错误时，没有业务数据需要返回），我这里使用的是一个开源的library：ModelMapper，只要所有的Rs（比如QuoteRs/PlaceOrderRs）都和GenericRs一样有statusCode、message两个属性，ModelMapper会自动完成映射Rs的任务，非常方便（可以让所有的Rs都继承GenericRs降低属性名称不一致的风险）。

### Q:自定义异常的粒度到什么程度比较好？

A: 这主要看应用程序准备怎么使用这个自定义的异常体系了。理论上来讲，即使是为了翻译statusCode，整个异常体系也只需要一个自定义异常就可以满足使用：

Listing-10 GenericApplicationException
{% highlight java %}
public class GenericApplicationException extends RuntimeException {
    @Getter
    private String statusCode;
    @Getter
    private String i18nCode;
    @Getter
    private String[] i18nArgs;

    public GenericApplicationException(String statusCode, String message, String i18nCode, String[] i18nArgs, Throwable cause) {
        super(message, cause);
        this.statusCode = statusCode;
        this.i18nCode = i18nCode;
        this.i18nArgs = i18nArgs;
    }
}
{% endhighlight %}

这个异常就非常通用，但是不易用，因为抛出它的组件需要自己负责输入statusCode, message, i18nCode, i18nArgs， cause。这样来看的话，你可以根据错误的来源设计一些特定的异常以简化异常的抛出代码，比如：

Listing-10 CannotCancelOrderException
{%  highlight java %}
public class CannotCancelOrderException extends UncheckedApplicationException {
    
    private String orderId;
    private String i18nCode;
    
    public static CannotCancelOrderException hasBeenLocked(String orderId) {
        return new CannotCancelOrderException("order["+ orderId + "] has been locked", orderId, "i18n.order.orderHasBeenLocked");
    }

    public static CannotCancelOrderException hasBeenCanceled(String orderId) {
        return new CannotCancelOrderException("order["+ orderId + "] has already been canceled", orderId, "i18n.order.orderHasBeenCanceled");;
    }

    public CannotCancelOrderException(String message, String orderId, String i18nCode) {  
        super(message);  
        this.orderId = orderId;
        this.i18nCode = i18nCode;
    }
   
    @Override
    public String getStatusCode() {
         return CANNOT_CANCEL_ORDER;
    }

    @Override
    public String getI18nCode() {
         return i18nCode;
    }

    @Override
    public String[] getI18nArgs() {
         return new String[] {orderId};
    }
}

i18n-zh-CN.properties

i18n.order.orderHasBeenCanceled = 订单{0}已被取消
i18n.order.orderHasBeenLocked = 订单{0}已被锁定
{%  endhighlight  %}

CannotCancelOrderException将statusCode、i18n、message的拼装隐藏了起来，这样抛出CannotCancelOrderException的组件就轻松多了，只需要输入orderId即可。如果编写单元测试，这种情况就更明显了，如果设计得当，抛出异常的组件在测试中也不用关心异常的信息细节，异常的信息细节可以在异常的单元测试中验证。

Listing-11 unit test for exception
{%  highlight java %}
public class CannotChangeOrderExceptionUnitTests {

	private CannotChangeOrderException target;

	@Test
	public void tellsOrderIsCanceled() throws Exception {
		final String orderId = "1";
		target = CannotChangeOrderException.orderIsCanceled(orderId);

		Assert.assertEquals("订单[1]已被取消", target.getMessage());
		Assert.assertEquals(StatusCode.CANNOT_CHANGE_ORDER,
				target.getStatusCode());
		Assert.assertEquals(
				"com.springtour.irs.application.exception.CannotChangeOrderException.orderIsCanceled.message",
				target.getI18nCode());
	}

	@Test
	public void tellsOrderIsHolding() throws Exception {
		final String orderId = "1";
		target = CannotChangeOrderException.orderIsHolding(orderId);

		Assert.assertEquals("订单[1]预占中", target.getMessage());
		Assert.assertEquals(StatusCode.CANNOT_CHANGE_ORDER,
				target.getStatusCode());
		Assert.assertEquals(
				"com.springtour.irs.application.exception.CannotChangeOrderException.orderIsHolding.message",
				target.getI18nCode());
	}

}
{%  endhighlight  %}
