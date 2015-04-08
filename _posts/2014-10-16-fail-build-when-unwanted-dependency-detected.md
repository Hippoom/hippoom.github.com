---

title: 用classycle自动代码走查  
category: continuous delivery  
layout: post

---

### 用classycle检查类之间的依赖关系

不少团队会进行代码走查来促进代码质量，代码走查的目标可以是设计或实现的缺陷，这往往依赖于走查者的个人经验和技术功底；也有对照代码规范的专项走查，这时需要的是细致和耐心。比如团队决定使用[slf4j](http://www.slf4j.org/manual.html)来作为日志api，以便将系统和日志实现解耦。有时因为疏忽，开发者可能会意外地导入别的日志库，比如log4j。即使团队有良好的代码走查实践，要发现如此琐碎的问题还是挺费功夫的。计算机更适合完成这类重复的工作，我们只要找到合适的工具就行了。

比如刚才的日志依赖问题，可以交给classycle，它支持自定义类之间的依赖规则，并依此检查代码库：

list-1 检查代码库不依赖于log4j，保存在src/test/resources/classycle-dependency-definition中

    show allResults

    [log4j] = org.apache.log4j.*
    [root] = your.project.root.*

    check [root] independentOf [log4j]

如果是Maven项目，可以很方便的执行检查：

list-2 使用classycle:check执行检查

{% highlight xml %}

    <plugin>
        <groupId>org.pitest</groupId>
        <artifactId>classycle-maven-plugin</artifactId>
        <version>0.4</version>
        <configuration>
            <dependencyDefinitionFile>
                src/test/resources/classycle-dependency-definition
            </dependencyDefinitionFile>
            <resultRenderer>
                classycle.dependency.DefaultResultRenderer
            </resultRenderer>
    </plugin>

{% endhighlight %}


如果classycle找到了不符合规则的依赖，会输出到target/classycle目录下的报告中。

    show onlyShortestPaths allResults
    check [root] independentOf [log4j]
    Unexpected dependencies found:
        sample.bar.Bar
    		-> org.apache.log4j.Logger



###如果发现不良的依赖就让构建失败

静态检查工具使得团队可以把更多时间投入到推敲代码上，去发现新的问题，而不是枯燥地查找已知的缺陷。既然已经有了廉价的检查手段，本着越早发现问题越容易解决的原则，不妨把检查任务集成到项目的构建过程中。如果团队已经建立了部署流水线，可以让流水线来执行检查，如果classycle发现有违反规则的依赖会使构建失败。

list-3 将检查任务绑定到maven verify中执行

{% highlight xml %}

    <plugin>
    	<groupId>org.pitest</groupId>
        <artifactId>classycle-maven-plugin</artifactId>
        <version>0.4</version>
        <executions>
        	<execution>
            	<id>verify</id>
                <phase>verify</phase>
                <goals>
                	<goal>check</goal>
                </goals>
                <configuration>
                    <dependencyDefinitionFile>
                        src/test/resources/classycle-dependency-definition
                    </dependencyDefinitionFile>
                    <resultRenderer>
                        classycle.dependency.DefaultResultRenderer
                    </resultRenderer>
                </configuration>
            </execution>
        </executions>
	</plugin>

{% endhighlight %}

最后记得让持续集成服务器把classycle生成的报告保存下来，方便团队排问题。  

如果团队有幸可以从头构建代码库，那么最好从一开始就将依赖检查加入到流水线中，并逐步按需调整。如果接手的是遗留项目，最好谨慎一些，随着重构逐步丰富检查规则。无论是哪种情况，都要根据项目的实际情况定义检查规则，如果流水线因为检查不通过而长期处于失败状态，就失去意义了。

### 这是不是有点过了

可能你会觉得如果依赖检查不通过就终止构建太严格了，不过这招往往很有效，尤其是当项目的周期较长，并有新的成员加入时，依赖检查可以帮助团队保护架构原则。比如，一个采用port & adapter架构（onion）的系统，可以检查domain不依赖于任何适配组件。

list-4 确保domain是onion架构中的“核心”

    show allResults

    [root] = your.project.root.*
    [port] = your.project.root.domain.*
    [adapters] = [root] excluding [port]

    check [port] independentOf [adapters]

另一方面，重视依赖检查也可能反过来促进制定合理的架构约束。以下两种划分package的方案，你更倾向于哪种呢？

list-5 方案一：按职责划分

	repositories/
    	UserRepository
    	TransactionRepository
    	ShipmentRepository
	serializers/
    	UserSerializer
    	TransactionSerializer
    	ShipmentSerializer
	models/
    	User
    	Transaction
    	Shipment

list-6 方案二：按领域划分

	user/
    	UserRepository
    	UserSerializer
    	User
	transaction/
    	TransactionRepository
    	TransactionSerializer
    	Transaction
	shipment/
    	ShipmentRepository
    	ShipmentSerializer
    	Shipment

假设user不应该依赖transaction的话，第一种方案需要将检查规则定义到类的级别，而第二种方案能以较粗的粒度（package）来管理依赖关系，这样更为经济高效。

### 为什么不使用embedded defination  

classycle-maven-plugin还支持把依赖规则嵌入到pom.xml中，但我更推荐使用单独的规则文件。一方面，随着项目的衍化，需要检查的规则可能会增长，一大段堆规则文本会使得pom.xml变得很臃肿，另一方面，单独的规则文件方便团队对比规历史版本，查看其修订记录及原因。

### gradle 项目  

对于gradle项目，我暂时还没有找到现成的插件，不过classycle本身支持ant task，所以可以用gradle ant插件来集成。

### 参考资料

[classycle](http://classycle.sourceforge.net/)

[classycle-maven-plugin](https://github.com/hcoles/classycle-maven-plugin)

[directory per domain model or directory per layer, role](http://stackoverflow.com/questions/21225937/directory-per-domain-model-or-directory-per-layer-role/21226273#21226273)
