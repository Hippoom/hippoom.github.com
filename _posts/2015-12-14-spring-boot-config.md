---

title:   为什么我粉Spring Boot——应用配置篇
category: continuous delivery   
layout: post

---

今天先从应用配置的角度谈谈为什么Spring Boot很对我的胃口。
这里的应用配置指的是一组能够是应用程序可以在不同环境中工作的参数，比如数据库地址或者某个特性的开关。
许多工具或是Library尝试提供了解决方案，但我目前对Spring Boot提供的特性最为满意，先来看看作为开发人员我们对应用配置有哪些需求。

### 第一个用户故事

    作为Developer
    我希望将配置项纳入版本控制
    以便自动获得Teamate提交的最新配置项
    但其不应该自动覆盖我自定义过的配置项

这个场景在团队协作中很常见，随着特性的增加，不断会有新的配置项加入。在Developer本机开发调试时，
大部分配置项可以使用默认值工作，比如大家约定数据库都使用localhost:3306，但有些配置项确实需要因人而异，
比如某个特性正在开发中，负责这个特性的开发人员将特性开关设为“打开”以便自己可以继续开发，而于此同时需要将默认值设为关闭，以便团队中的其他人不受影响，并且可以应用也处于随时可发布的状态。

对此，有些团队采用“注释大法”，比如这样：

shared_config.yml

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

每当开发人员要提交的时候，需要格外留意将配置项恢复到默认值，实在是太不人性化了。
为了避免人为错误，有的团队采用隔离配置文件的实践，每个开发人员使用单独的配置文件，每个开发人员维护把所有的配置项（即使大部分用相同的值），这在需要加入新配置项时尤其麻烦。

     |___config.yml.sample #sample中含有当前所有的配置项和默认值，请手工更新到自己的配置文件中。。。
     |___john_config.yml
     |___mike_config.yml
     |___jane_config.yml
     |___will_config.yml

现在来看看Spring Boot的方案，它提供了一种覆盖机制，可以让Developer使用环境变量，Profile或是文件覆盖部分配置项的默认值。
举例来说，我们有一个HelloWorld应用程序，其中HostConfig用来映射我们的配置文件，在src/main/resources/application.yml中有各个配置项的默认值。

    // src/main/java/configsample/Application.java
    @RestController
    @SpringBootApplication
    public class Application {

        @Resource private HostConfig host;

        @RequestMapping public String hello() {
            return new ST("Welcome to <hostName>, you can call me <nickName>")
                    .add("hostName", host.getName())
                    .add("nickName", host.getNickName())
                    .render();
        }

        public static void main(String[] args) { SpringApplication.run(Application.class args);}
    }

    // src/main/resources/application.yml
    host:
      name: prod
      nickName: John

    // src/main/java/configsample/host/HostConfig
    @Configuration
    @ConfigurationProperties(prefix = "host")
    @Setter @Getter
    public class HostConfig {
        @NotNull private String name;

        @NotNull private String nickName;
    }

现在我需要在本机调试时自定义host.name为“dev”，接下来利用Profile和Spring Boot来解决这个问题：
首先，在src/main/resources/下添加一个名为“application-dev.yml”的文件，修改host.name。

    // src/main/resources/application-dev.yml
    host:
      name: dev

接下来，在启动Application时加上“-Dspring.profiles.active=dev”的参数并在浏览器中访问应用（http://localhost:8080），会得到“Welcome to dev, you can call me John”，说明我们成功地覆盖了host.name并且复用了host.nickName。最后一步则是在.gitignore中忽略application-dev.yml，这样团队成员就不会互相影响了。

    # .gitignore  
    #local
    application-dev.yml

事实上，Spring Boot还支持使用环境变量，或是命令行来覆盖配置项，比如--host.name="dev"，但我觉得对于这个场景还是Profile更好用一些。

### 第二个用户故事

    作为Developer
    我希望可以将配置文件外置
    以便单独部署应用配置的变化

提到这个故事是因为我曾经遇到过一个大型应用程序，团队也为搭建了自动化构建和部署程序，其构建时间大约需要20分钟，每次部署（测试、准生产）需要30分钟。经过简单地调查后，我发现该团队为每个环境都准备一份配置文件，而该应用程序的Artifact是一个包含配置文件的War文件，所以当需要为每个特定的环境准备Artifact时，会从源代码开始重新编译。这显然是一个巨大的浪费，花费20分钟的时间只是为了替换一个文件，既然构建War文件的时间成本如此高，那么在构建完成后，我们应该把它保存起来。一个临时缓解方案是当需要向特定的环境部署的时候，解开War文件并替换对应的配置文件，在重新打包并部署。长期的方案则是修改应用程序，让其可以支持外置的配置文件，比如很多应用程序都会读取/etc/目录下的配置文件。而Sprint Boot是天生支持外置化配置文件的，比如当我将Artifact像这样部署到服务器上：

    |__boot-build-0.0.1.jar # contains application.yml as default config
    |__application.yml

    # application.yml
    host:
      name: uat

并启动应用程序的话（java -jar boot-build-0.0.1.jar），Spring Boot会合并当前目录的application.yml以及jar文件中的application.yml，所以当我们访问应用程序，它会显示
“Welcome to uat, you can call me John”。如果团队采用了持续交付的实践，可以通过分离应用程序和配置文件的代码库来进一步强化外置化配置文件的优势，例如下图的Pipeline Value Stream，在UAT的配置变化时，我们可以复用boot-build生成的jar文件（不触发构建boot-build），直接部署UAT环境。

<img src="/images/spring-boot-config/fan-in.png" width="600px"/>

有许多工具和Library都能够满足以上两个用户故事之一，或者我们自己也可以想办法结合这些工具同时实现这个用户故事，但是Spring Boot天生就支持它们，是不是很方便呢？
