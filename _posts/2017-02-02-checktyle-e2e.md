---
title:   拯救Java Code Style强迫症
category: continuous delivery
layout: post

---



### Code Style是一项工程实践

![header](/images/checkstyle-e2e/style.jpg)    

我是右侧风格的忠实拥趸，如果在我工作的项目中看到有左侧风格的代码，你猜猜我的反应是什么

![git blame](/images/checkstyle-e2e/gitblame.jpg)

事实上，这并不是强迫症，如果没有按照惯例来编写代码，甚至会让阅读者产生疑惑。

```java
private Listener listener = new Listener() // So Listener looks like a class?  
{}; // Oops, it is an interface
```

如果代码可读性还不足以打动你，那么想象一下这个场景，你的同事告诉你他修复了两个空指针问题，请你帮忙Code Review，你比较了这个文件的修订历史，乍看之下有许多改动，看来是个大动作。然后事实上，绝大部分部分是代码格式调整，只有两处改动与需要Review的问题相关的。

![git blame](/images/checkstyle-e2e/happy-code-review.jpg)    

> 看来这位同事的IDE使用了不同的自动缩进设置，导致所有行都产生了缩进

之所以会产生以上这些影响工作效率的问题，是因为团队没有重视Code Style，没有把它当做一项工程实践，既没有对其达成一致，也没有正确地使用工具帮助实施。



### 那就按照工程实践的标准来实施Code Style

本文将重点介绍Java项目的Code Style的工具支持，但在此之前，你的团队需要一起做一些决定：

1. **使用哪种Code Style？**  
   每个人可能都有偏好的style，但在团队协作面前，需要一定的妥协。有些公司或组织有着统一的Code Style指导标准，萧规曹随是个不错的选择（但是要确保这类统一指导标准在制定时参考了开发人员的意见，是切实可行的），你的团队也可以自己裁剪，但至少要保证项目（Repository）级别上使用同一种Style。  
2. **对不符合Code Style的提交如何处理**  
   大家往往懈怠于事后补救的方式，所以我的建议是不要让不符合约定的代码流入代码库。对于遗留项目，尤其是大型项目，可以选择一部分代码作为实施范围，集中修复Style问题后严格实施，切记操之过急，最后团队疲惫不堪只得放弃。

我们都知道人工监督检查的方式是不可持续和不可靠的，来看看有哪些工具可以提供帮助吧。

### 懒惰是第一生产力

工程实践不能没有自动化工具支持，在java生态圈中，Code Style工具最出名的应该是Checkstyle了，它可以通过XML形式的[外部DSL](https://martinfowler.com/books/dsl.html)来定义Code Style的检查风格，比如你可以从这里找到[Google的Java Checkstyle配置文件](https://github.com/checkstyle/checkstyle/blob/master/src/main/resources/google_checks.xml)。这里我不会详细介绍Checkstyle本身，相反，我会更多地探讨如何如何工程化地使用Checkstyle，在交付代码的各个活动中，我们都可以用到Checkstyle，360°无死角检查。

![git blame](/images/checkstyle-e2e/lifecycle-overview.jpg)

> 和Code Style相关的代码交付生命周期



### 守住提交的质量关口

为了贯彻不让不符合约定的代码流入代码库的决定，可以优先在服务端设置Code Style的检查关卡。

<img src="/images/checkstyle-e2e/lifecycle-ci.jpg" width="350px"/>

> 优先守住代码提交时的服务端检查，可以考虑使用CI服务器来实现

从实现层面上说，有两种方式：

一是在SCM（Source Control Management，例如Git/SVN）服务端设置检查项，如果不达标则拒绝提交，但这种方式相对不容易实现，而且一般SCM服务端也不由开发团队管理，设置起来不灵活也不方便。  

二是利用[持续集成](https://www.thoughtworks.com/continuous-integration)服务器，开发团队的每一次提交都会触发一次构建，我们可以在构建脚本中加入Checkstyle检查，如果有不达标的代码则让构建失败以便告诉提交者立即修复Style问题。我更推荐这个方案，因为相关的工具支持都很成熟，实现简单，而且构建过程可以在开发者的本地环境复制，以便在后续改进中将Checkstyle检查前移，提供更快的反馈。如果团队使用Maven/Gradle等构建工具，可以用插件的方式实现Checkstyle检查并嵌入到整个构建过程中。这样CI服务器只要调用构建脚本就行了。

### 在开发者本地验证Style

<img src="/images/checkstyle-e2e/lifecycle-local.jpg" width="350px"/>

> 在开发者本地实现验证，反馈关口前移

在实现了CI验证后，就可以着手实现开发者本地验证了，这样开发者就不用等到提交代码到服务端后才会获得反馈了。由于之前采用的是构建工具的插件方案，所以开发者在本地运行构建就能实现验证了。这里以Gradle为例，简单介绍一下所需的配置

``` groovy
# build.gradle
# omitted plugins
apply plugin: 'checkstyle'

checkstyle {
    configFile = file("config/checkstyle.xml") //指定checkstyle配置文件
    toolVersion = "7.4" //指定checkstyle工具的版本，部分style规则有版本要求
}

checkstyleTest.exclude "**/ContractVerifierTest**" // 忽略检查生成代码，这个锅我们不背

// 如果出现checkstyle warning也使构建失败，插件默认只支持checkstyle error失败 https://github.com/gradle/gradle/issues/881
tasks.withType(Checkstyle).each { checkstyleTask ->
    checkstyleTask.doLast {
        reports.all { report ->
            def outputFile = report.destination
            if (outputFile.exists() && outputFile.text.contains("<error ")) {
                throw new GradleException("There were checkstyle warnings! For more info check $outputFile")
            }
        }
    }
}
```

现在只需要一条命令，每个开发者就能在本地验证Code Style了。你可以在[这里](https://docs.gradle.org/current/userguide/checkstyle_plugin.html)找到Gradle Checkstyle Plugin的详细配置文档，如果你使用Maven，则可以参考[这里](https://maven.apache.org/plugins/maven-checkstyle-plugin/)。

```shell
➜  court-booking-backend (master) ✗ ./gradlew check
Starting a Gradle Daemon (subsequent builds will be faster)
:compileJava
:compileGroovy UP-TO-DATE
:processResources UP-TO-DATE
:classes
:checkstyleMain
[ant:checkstyle] [WARN] /Users/twer/Workspace/restbucks/court-booking-backend/src/main/java/com/restbucks/courtbooking/http/CourtRestController.java:16: 'method def' child have incorrect indentation level 4, expected level should be 8. [Indentation]
:checkstyleMain FAILED

FAILURE: Build failed with an exception.

* Where:
Build file '/Users/twer/Workspace/restbucks/court-booking-backend/build.gradle' line: 83

* What went wrong:
Execution failed for task ':checkstyleMain'.
> There were checkstyle warnings! For more info check /Users/twer/Workspace/restbucks/court-booking-backend/build/reports/checkstyle/main.xml

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output.

BUILD FAILED

Total time: 15.539 secs
```

### 本地验证很不错，但我有时候会忘记执行

<img src="/images/checkstyle-e2e/lifecycle-git-hook.jpg" width="350px"/>

> 让机器代劳琐事

有时候，开发者修改了代码后会忘记执行本地检查就提交代码了，最好能够在提交代码前自动执行检查。如果你使用Git的话，可能会想到Git commit hook，比如这是我常用的pre-commit hook

```shell
#!/bin/sh
# From gist at https://gist.github.com/chadmaughan/5889802

# stash any unstaged changes
git stash -q --keep-index

# run the tests with the gradle wrapper
./gradlew clean build

# store the last exit code in a variable
RESULT=$?

# unstash the unstashed changes
git stash pop -q

# return the './gradlew build' exit code
exit $RESULT
```

将该脚本拷贝到`.git/hooks/`下，在执行`git commit`的时候就会自动触发检查了，如果检查失败则提交失败。但问题是`.git`并不能提交到远程代码仓库，那么除了人工分发和拷贝外，有没有更好的方式在团队中共享这个机制呢？

可以曲线救国！把pre-commit纳入版本控制(如下面的`config/pre-commit`)，再使用构建工具的扩展机制来自动完成拷贝工作，这样可以间接实现git hooks的团队间共享。

```groovy
# build.gradle

task installGitHooks(type: Copy) { //将pre-commit拷贝到指定位置
    from new File(rootProject.rootDir, 'config/pre-commit')
    into {
        new File(rootProject.rootDir, '.git/hooks')
    }
    fileMode 0755
}

build.dependsOn installGitHooks //设置执行build任务时会自动触发installGitHooks任务
```

### 关闭包围圈，编辑时反馈

<img src="/images/checkstyle-e2e/lifecycle-edit-time.jpg" width="350px"/>

> 实时反馈

之前基于构建工具的方案都很好，但是对于开发者来说，最好的还能将反馈前移到编辑时，并且可视化。所幸的是，Checkstyle的生态系统非常成熟，各主流IDE都有插件支持，以Intellij Idea为例，可以使用[checkstyle-idea](https://github.com/jshiell/checkstyle-idea)插件，让团队成员手工设置插件，使用项目的checkstyle配置文件即可(我目前还没有找到自动化配置的方式，或许gradle idea插件可以？)

<img src="/images/checkstyle-e2e/idea-checkstyle-settings.jpg" width="600px"/>
<img src="/images/checkstyle-e2e/idea-checkstyle-plugin.jpg" width="600px"/>
> checkstyle-idea插件配置和效果

有了自动实时检查，最好还能将IDE的自动格式化与Checkstyle配置文件挂钩，否则自动格式化反倒给你添麻烦了。
<img src="/images/checkstyle-e2e/idea-format.jpg" width="600px"/>
> 为IDE导入checkstyle配置文件作为自动格式化的依据

如果你连自动格式化都懒得按，那可以试试[Save Actions](https://github.com/dubreuia/intellij-plugin-save-actions)插件，它可以在Intellij保存文件时自动执行代码格式化等动作。

<img src="/images/checkstyle-e2e/save-actions.jpg" width="350px"/>
> 这个插件目前对部分文件有些问题，可以通过File path exclusion忽略

### 总结

1. Code Style影响工作效率，团队应将其当做工程实践予以重视
2. Code Style不能靠人工监督和检查，应该提供端到端的工具支持

- - 服务端检查（推荐集成到CI的构建步骤中）
  - 开发环境检查（使用各构建工具的Checkstyle插件）
  - 自动提交检查（git pre-commit hook与共享）
  - IDE增强（checkstyle插件实时可视化反馈/自动的自动格式化！）

好了，希望以上这些招数可以解救Java Code Style强迫症 ：）