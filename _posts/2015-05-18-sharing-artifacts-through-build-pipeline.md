---

title:   在基于Jenkins实现的部署流水线中共享二进制包
category: continuous delivery  
layout: post

---

### 只生成一次二进制包

&emsp;&emsp;只生成一次二进制包是部署流水线的一条重要原则，部署流水线的后续步骤会重用之前生成的二进制包。
相比每次从源代码构建二进制包，它节约了时间，更重要的是它促进了“你所测试的二进制包就是将要发布的二进制包”的实践。

现在，使用Jenkins来实现部署流水线非常流行，那么如何用Jenkins来实现共享二进制包呢？

### 拷贝/共享Workspace?

&emsp;&emsp;我所见过最常用的方案，但很遗憾，这个方案并**不**正确。
这种方案只所以流行可能要“归功”于一款插件：[Clone Workspace SCM Plug-in](https://wiki.jenkins-ci.org/display/JENKINS/Clone+Workspace+SCM+Plugin)，
这款插件可以打包Workspace，这样二进制包就被保存下来供后续步骤复用。

![只能选择最近的构建](/images/sharing-artifacts-through-build-pipeline/clone-workspace-scm-plugin.png)

遗憾的是，下游步骤只能选择**最新**的上游构建的Workspace，这在大部分情况下没有问题，
但并不是每个构建步骤都会触发下游步骤，这种情况多见于进入QA流程的步骤，在部署流水线中，
QA不再被动响应二进制包的部署，而是以“拉”模式主动选取要部署的二进制包。
因此，QA并不一定会选取最新的二进制包，但如果使用[Clone Workspace SCM Plug-in](https://wiki.jenkins-ci.org/display/JENKINS/Clone+Workspace+SCM+Plugin)，这就由不得他们了。

![可怜的家伙们](/images/sharing-artifacts-through-build-pipeline/pipeline-previsous-build-retry.png)

在上图中，如果想要部署workspace-clone-commit#7的二进制包，那么QA会点击红色方框中的build按钮，
但实际上将被部署的是workspace-clone-commit#9生成的二进制包。
这个方案还有一个邪恶的变体：下游步骤指定使用上游步骤的Workspace。

![邪恶的变体](/images/sharing-artifacts-through-build-pipeline/use-custom-workspace.png)

这个问题的可怕之处在于，你可能会将没有做好发布准备的二进制包**意外**地发布到生产环境中去。

### 关键是二进制包仓库

&emsp;&emsp;拷贝Workspace是不靠谱的，你需要建立二进制包与构建它的部署流水线之间的联系，
这样才能保证取到正确的二进制包。常见的做法是引入二进制包仓库（以下简称制品仓库）。

![制品仓库](/images/sharing-artifacts-through-build-pipeline/publish-to-artifact-repo.png)

### Jenkins自身作为制品仓库

&emsp;&emsp;Jenkins自身也可以作为制品仓库，在commit stage的Post-build Actions中加入Archive the artifacts
即可归档Workspace中的指定目录及文件。
接下来需要为后续步建立与二进制包的关联，把该次构建的build number传递下去，这样后续步骤就知道应该去哪拷贝二进制包了。
利用[Parameterized Trigger plugin](https://wiki.jenkins-ci.org/display/JENKINS/Parameterized+Trigger+Plugin)和[Build Pipeline Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Build+Pipeline+Plugin)的Manual Trigger
可以分别向自动、手动触发的下游任务传递环境变量，在变量中加入本次构建的build number：

![传递构建编号](/images/sharing-artifacts-through-build-pipeline/trigger-downstream-with-build-number.png)

下游步骤利用[Copy Artifact Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Copy+Artifact+Plugin)（一定要使用specific build）就可以精确地拷贝二进制包了：

![拷贝指定构建的制品](/images/sharing-artifacts-through-build-pipeline/copy-artifact-from-specific-build.png)

现在，二进制包就可以正确地在部署流水中流转复用了。

### 利用外部制品仓库

&emsp;&emsp;我会在后续的博客中介绍如何集成外部制品仓库，比如Nexus
