---

title:   使用乐观锁/悲观锁处理数据库并发
category: ramblings  
layout: default

---

企业应用程序基本上都是多用户同时使用的，而且很多情况下，还包括任务调度器或是外部系统触发的后台任务。所以，多个事务同时读写数据库是很常见的情况。开发人员面对的一个重大挑战就是：多个事务并发更新数据库带来的数据不一致的问题。你也许希望数据库自己就能够避免这种情况，但是保持数据一致通常是应用程序才能完成的任务。

出现数据不一致的情况主要有以下两种：
1.覆盖更新。一个事务把另一个事务的修改“盲目”地覆盖掉了，典型的场景是两个用户同时编辑一个数据，并且几乎同时提交，每个人都认为自己的更新成功了，但是有一人的提交其实已经被另一个人在不知情的情况下覆盖掉了。
2.脏读。一个事务读取了一个数据然后用这个数据去干别的事了，结果另一个事务却已经修改了这个数据，这可能造成第一个事务要去干的那个事情产生的数据出现不一致。

### 如果可以接受的话，数据库确实自己能够搞定数据一致性问题

简单的说，就是将数据库的事务隔离级别设置为serializable，使所有事务都是串行执行的。但是就像它的名字所指出的那样，串行执行使得数据库的处理性能大幅度地下降，常常是你接受不了的。所以，一般来说，数据库的隔离级别都会设置为read committed（只能读取其他事务已提交的数据），然后由应用程序使用乐观锁/悲观锁来弥补数据不一致的问题。

### 乐观锁

虽然名字中带“锁”，但是乐观锁并不锁住任何东西，而是在提交事务时检查自己上次读取这条记录后，是否有其他事务修改了这条记录，如果没有则提交，如果被修改了则回滚。如果并发的可能性并不大，那么锁定策略带来的性能消耗是非常小的。

####  实现乐观锁的方式

一般有三种方式实现乐观锁:
一是为数据表增加一个version字段，每次事务开始时，取出version，在提交事务时，检查version是否有变化，如果没有变化提交事务时将version + 1，SQL差不多是这样：

{% highlight java %}
UPDATE T_IRS_RESOURCE
set version = version + 1
where resource_id = ?
and version = ?
{% endhighlight %}

二是为数据表增加一个时间戳字段，然后通过比较时间戳检查该数据是否被其他事务修改过。

三是检查对应的字段的值有没有变化。

一般来说，如果条件允许最好采用第一种方案。因为方案二受限于时间的精度，而方案三则开发起来很麻烦，而且有些时候浮点数的比较还不是很准确或者你需要处理null的情况。

####  实现乐观锁的例子

接下来我会分别介绍使用iBATIS 2/hibernate 3实现乐观锁的简单示例。我们的假设场景非常简单，有两个用户要同时编辑一个Resource的name：

##### iBATIS

如果你使用iBATIS或其他jdbc风格的持久化框架，那么你需要自己实现乐观锁策略。方法也很简单，在更新数据之后检查一下版本号即可：

{% highlight java %}
final int rowAffected = sqlMapClientTemplate.update(NAMESPACE + ".update", resource);
if (rowAffected == 0) {
    throw new ObjectOptimisticLockingFailureException(Resource.class, resource.getId());
}
{% endhighlight %}

这样就能实现乐观锁并防止下图的问题了（注意第二个事务执行update时，第一个事务已经提交了，所以第二个事务能够读取到第一个事务修改的version）。

{% highlight java %}
| the first transaction started                       |
|                                                     |  
| select * from t_irs_resource where resource_id = ?  |
|                                                     |  
| update t_irs_resource set                           |
| version = version + 1，                             | the second transaction started
| name = ?                                            |
| where resource_id = ? and version = ?               | select * from t_irs_resource where resource_id = ?
|                                                     |  
| commit txn                                          | update t_irs_resource set                           
|                                                     | version = version + 1，                            
|                                                     | name = ?                                          
|                                                     | where resource_id = ? and version = ?    
|                                                     |  
|                                                     | rolls back because version is dirty
{% endhighlight %}

不过iBATIS无法处理下面这种极端的情况：

{% highlight java %}
| the first transaction started                       |
|                                                     | the second transaction started
| select * from t_irs_resource where resource_id = ?  |
|                                                     | select * from t_irs_resource where resource_id = ? 
| update t_irs_resource set                           |
| version = version + 1，                             | update t_irs_resource set           
| name = ?                                            | version = version + 1，
| where resource_id = ? and version = ?               | name = ?   
|                                                     | where resource_id = ? and version = ?    
| commit txn                                          |                 
|                                                     | rollback txn because version comparing fails
|                                                     |                                        
{% endhighlight %}

在这种情况下，第二个事务的update由于不能读取第一个事务未提交的数据，它会认为version没有问题从而提交成功，破坏了数据一致性。这时最好使用hibernate这样的orm框架，通过试验（我在update语句之后，方法(即事务边界)退出前，让线程sleep 10秒，这时用另一个事务修改数据来检查结果）它可以解决这个问题，这是因为hibernate只有在事务提交时才把sql语句发送到数据库执行（而iBATIS则是按照代码顺序依次执行）。

##### Hibernate

使用Hibernate 3的话，实现乐观锁非常简单，只需要在映射文件中添加一行配置即可：

{% highlight java %}
<hibernate-mapping default-access="field"
	package="com.gmail.hippoom.irs.domain.model.resource">
	<class name="Resource" table="T_IRS_RESOURCE"  dynamic-update="true">
		//omitted id config
		<version name="version" column="version" />
		//omitted properties config
	</class>
</hibernate-mapping>
{% endhighlight %}

hibernate会自动帮我们完成乐观锁的检查工作。

总得来说，使用乐观锁并不一定安全（请考虑刚才说的极端的情况并且你在使用iBATIS），并且你必须保证所有对数据更新的代码都遵循乐观锁机制（否则相当于有人走后门），但是乐观锁来的额外开销最小，对于数据不是特别敏感的情况还是使用乐观锁为宜。

### 悲观锁

和乐观锁相比，悲观锁则是一把真正的锁了，它通过SQL语句“select for update”锁住数据，这时如果其他事务来更新时会等待：
{% highlight java %}
| the first transaction started                       |
|                                                     |  
| select * from                                       |
| t_irs_resource where resource_id = ? for update     |
|                                                     |  
| update t_irs_resource set                           |
| version = version + 1，                             | the second transaction started
| name = ?                                            |
| where resource_id = ? and version = ?               | select * from                                       |
|                                                     | t_irs_resource where resource_id = ? for update 
| commit txn                                          | 
|                                                     | wait until lock released
|                                                     | 
{% endhighlight %}

##### iBATIS

很简单，在SQLMAP中的查询语句中增加 for update即可。不过要注意的是，如果查询语句中有表关联，只是用for update会锁住所有表的相关记录，建议使用for update of t.id。如果你不想让你的事务等待其他事务的锁，可以在for update 后加上 nowait，这样当遇上冲突时数据库会抛出异常。

##### Hibernate

也很简单，在session.get()/load()时添加LockMode/LockOption参数即可，LockMode选择 LockMode.UPGRADE/LockMode.UPGRADE_NO_WAIT。

总的来说，悲观锁相对乐观锁更安全一些，但是开销也更大，甚至可能出现数据库死锁的情况，建议只在乐观锁无法工作时才使用。

### 利用乐观锁/悲观锁解决重复新增的或读取数据不一致的问题

前面我们讨论了如何使用乐观锁/悲观锁处理数据更新时并发问题，我们再来看看怎么使用它们处理重复新增的问题。假设这样一个场景，两个用户同时准备为同一个Resource新增SalesPlan，而且其DateRange都是2013-01-01到2013-01-01，但是需求要求SalesPlan的DateRanges不能重叠，但这时你又无法使用数据库唯一键来实现这个需求（因为日期段可能存在 2013-01-01到2013-01-10和 2013-01-02到2013-01-03 的重叠情况）。要解决这种问题的关键在于锁住Resource，虽然我们并不需要更新Resource，但是如果锁住了它，另一个新增SalesPlan的事务中由于也要获取Resource，最后就会由于Resource冲突而失败。

这里要注意，如果使用hibernate，且使用乐观锁，那么你需要手工的更新一下Resource，否则hibernate会认为Resource没有变化而不触发Resource的更新导致整个策略失效。我使用了一个小花招来应对这个情况：

{% highlight java %}
public class Resource {
    //omitted fields
    private int version = 1;

    private boolean dirty = false;

    public void alwaysMakeDirty() {
        this.dirty = !dirty;
    }
}

public class HibernateResourceRepositoryImpl implements ResourceRepository {

    //omitted code
    @Override
    public void store(Resource resource) {
        resource.alwaysMakeDirty();
        if (resource.isUnsaved()) {
            hibernateTemplate.save(resource);
        } else {
            hibernateTemplate.update(resource);
        }
        resource.saved();
    }
}
{% endhighlight %}
这时由于dirty总是会变化（本来是true变为false，本来是false变为true），hibernate会认为Resource已被修改从而触发update语句验证version。

###  这是不是意味着我要锁住所有东西？

如果你担心的是比如有一张主表，还有若干字表通过外键关联主表，是否需要锁住所有的行/检查所有的行的version。确实，这样做会给开发带来很大麻烦，需要非常细心的检查以免遗漏了某把锁，不过一般对于这种情况，推荐采取Aggreate策略，即以主表作为锁定对象，不锁定子表，但是所有对子表的访问和操作都必须先获得主表的锁。这样相对来说，开发工作量就小多了，开发人员实现锁策略时也不用绞尽脑汁判断某个表是否要锁定。
