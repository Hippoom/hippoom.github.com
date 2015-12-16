---

title:   为什么我粉Spring Boot——应用配置篇
category: continuous delivery   
layout: post

---

本文尝试从应用配置的角度谈谈为什么Spring Boot很对我的胃口 :-)

这里的应用配置指的是一种实践，在不修改应用程序代码的情况下，使其可以适应不同环境，常见的做法是将因环境而已的参数（比如数据库地址或者某个特性的开关）从代码中抽取出来，放到一个text文本中。
在构建应用程序的过程中，至少会经历本地开发环境，测试环境和生产环境，所以现在很少能找到哪个应用程序没有采用这种实践的，但我的Java项目经验中，有许多Library或工具尝试为应用配置提供友好的支持，但很少有像Spring Boot那样切中要害的，先来看看（被我代表的）开发人员对应用配置有哪些需求。

### 用户故事-团队协作

    作为Developer
    我希望将配置项纳入版本控制
    以便自动获得Teamate提交的最新配置项
    但其不应该自动覆盖我自定义过的配置项

这个场景在团队协作中很常见，随着特性的增加，不断会有新的配置项加入。在Developer本机调试时，
大部分配置项可以使用默认值工作，比如大家都约定使用相同的数据库参数，但有些配置项确实需要因人而异。
举例来说，当开发与微信集成的应用时，每个Developer可能希望使用自己的沙箱账号。有些团队采用“注释大法”，像这样：

    # shared_config.yml

    # John' sandbox
    # weChatAppId: abcd

    # Mike's sandbox
    # weChatAppId: abcde

    # Will's sandbox
    # weChatAppId: abcdef

    # Jane's sandbox
    # weChatAppId: abcdefg

    # Finally, after 30 lines of comment, this is my weChatAppId
    weChatAppId: me

每当开发人员要提交的时候，需要格外留意将配置项恢复到默认值，实在是太不人性化了，你可以想象团队成员之间关于带有“污染”的提交的有趣评论。为了避免人为错误，有的团队采用隔离配置文件的实践，每个开发人员使用单独的配置文件，每个开发人员维护所有的配置项（即使大部分用相同的值），这在需要加入新配置项时尤其麻烦。

     |___config.yml.sample #sample中含有当前所有的配置项和默认值，请手工更新到自己的配置文件中。。。
     |___john_config.yml
     |___mike_config.yml
     |___jane_config.yml
     |___will_config.yml

于是有些团队开始修改应用程序，在Java代码中定义默认值，这倒是个很好的实践，而Spring Boot为此提供了更好的支持，它提供了一种覆盖机制，可以让Developer使用环境变量，Spring Profile或是文件覆盖部分配置项的默认值。

让我们用一个经典的HelloWorld应用程序来体验下，其使命是在用户访问时打招呼。。。

    // src/main/java/configsample/Application.java
    @RestController
    @SpringBootApplication
    public class Application {

        @Resource private HostConfig host;

        @RequestMapping public String hello() {
            // 根据配置项输出Welcome消息
            return new ST("Welcome to <hostName>, you can call me <nickName>")
                    .add("hostName", host.getName())
                    .add("nickName", host.getNickName())
                    .render();
        }

        public static void main(String[] args) { SpringApplication.run(Application.class args);}
    }

    // src/main/resources/application.yml, 有各个配置项的默认值。
    host:
      name: prod
      nickName: John

    // src/main/java/configsample/host/HostConfig 用来映射配置文件
    @Configuration
    @ConfigurationProperties(prefix = "host")
    @Setter @Getter
    public class HostConfig {
        @NotNull private String name;

        @NotNull private String nickName;
    }

假设我需要在本机调试时设置host.name为“dev”，可以先在src/main/resources/下添加一个名为“application-dev.yml”的文件，修改host.name。

    // src/main/resources/application-dev.yml
    host:
      name: dev

第二步，在启动Application时加上“-Dspring.profiles.active=dev”的参数。
现在如果我们在浏览器中访问应用（http://localhost:8080），会得到

    Welcome to dev, you can call me John

说明我们成功地覆盖了host.name并且复用了host.nickName。
最后一步则是在.gitignore中忽略application-dev.yml，这样一来我的自定义配置不会自动覆盖其他人的了。
如果有新的配置项要增加，可以添加到/src/main/resource/application.yml中，所有成员通过更新最新的代码就可以自动获得，所有需要自定义的配置项则在/src/main/resource/application-dev.yml中配置。

    # .gitignore  
    #local
    application-dev.yml

事实上，Spring Boot还支持使用环境变量，或是命令行来覆盖配置项，比如--host.name="dev"，但我觉得对于这个场景还是Spring Profile更好用一些，编辑和修改更容易一些。

### 用户故事-单独部署应用配置

    作为Developer
    我希望可以将配置文件外置
    以便单独部署应用配置的变化

提到这个故事是因为我曾经遇到过一个大型应用程序，以War文件的形式发布，团队也为其配备了自动化构建和部署程序，其构建时间大约需要20分钟，每次部署（测试、准生产环境）需要30分钟。经过简单地调查后，我发现该团队为每个环境都准备了一份配置文件，当需要为每个特定的环境准备Artifact时，会从源代码开始重新编译，这显然是一个巨大的浪费，花费20分钟的时间只是为了替换一个无需编译的文件，既然构建War文件的时间成本如此高，那么在构建完成后，我们应该把它保存起来。

     // dangerous, please don't do this at home
    mvn clean package -Pdev
    mvn clean package -Puat
    mvn clean package -Pprod

一个临时缓解方案是当需要向特定的环境部署的时候，解开之前已经生成的War文件并替换对应的配置文件，在重新打包并部署。长期的方案则是修改应用程序，让其可以支持外置的配置文件，比如很多应用程序都会读取/etc/目录下的配置文件。

Spring Boot天生支持外置化配置文件，比如可以将Hello World的Artifact部署到服务器上:

    |__boot-build-0.0.1.jar # contains application.yml as default config
    |__application.yml # an externalized config file

    # application.yml
    host:
      name: uat

运行java -jar boot-build-0.0.1.jar，当我们访问应用程序，它会显示

    Welcome to uat, you can call me John。

这说明Spring Boot不但可以支持外置配置文件，而且根据优先级覆盖、复用配置项。
这个特性非常好用，一来，只需要自定义差异的配置项，可以很清楚地看到各个环境之间的配置差异。
二来，外置化配置文件使得单独部署配置变化变得更容易。想象一下，在应用程序不变的情况下，我们需要修改uat环境的某个特性开关，此时只需要部署最新的配置文件就可以了，省时省力省带宽。特别对于采用了持续交付实践的团队，可以通过分离应用程序和配置文件的代码库来进一步强化外置化配置文件的优势，例如下图的Pipeline Value Stream，在UAT的配置变化时，我们可以复用boot-build生成的jar文件（不触发构建boot-build），直接部署UAT环境。

<img src="/images/spring-boot-config/fan-in.png" width="600px"/>

### 粉点
有许多工具和Library都能够满足以上两个用户故事之一，或者我们自己也可以想办法结合这些工具同时实现这个用户故事，但是Spring Boot天生就支持它们，是不是很方便呢？
