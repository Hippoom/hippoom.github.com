---
title: 【翻译】使用Gradle脚本为每次构建自动生成唯一的构建版本号
category: Continuous delivery
---

文本为翻译，如果对原文感兴趣，可以移步至[Use different build numbers for every build — automatically using a gradle script](https://medium.com/@passsy/use-different-build-numbers-for-every-build-automatically-using-a-gradle-script-35577cd31b19#.k2gx0pbqn)。



# 正文 #

又一个JIRA ticket，来源于客户的故障报告（crash report）: “你们的App在搜索产品时总是崩溃”。没有更多信息。这是一个新的崩溃吗？怎么重现？他使用的是哪个版本的App？在和客户通话后，我发现该崩溃只出现在他的三星平台设备上，App版本是1.1.2。我没有办法在我们的设备上重现该崩溃。故障检测机制捕获了这个崩溃但并没有什么帮助。难不成是注释中的一个空指针异常（A NullPointerException inside a comment）？

最终的原因，让我长话短说，该客户使用的是一个我在最后一次会议中做过本地代码修改的调试（debug）版本。这个1.1.2版本和我们git仓库里标记（tagged）的1.1.2版本是不一样的。一般来说，发送给该客户的版本应该还有一个构建编号（build number），即1.1.2-**65**。这个丢失的构建编号表示该App版本是一个本地构建的调试版本。

## 使用Jenkins增加构建编号

使用Jenkins为每次构建自动新增构建编号，这是一个常规实践，唯一困扰我的是，**一个更大的Jenkins构建编号并不意味着这是一个更新的App（原文为app state）**。不管是我构建2.3.6还是1.0.1，该编号总是在自动增长。

在构建编号202中加入的特性X，在构建编号203中却没有了，这是令人费解的。我曾经收到过数封“特性丢失”的邮件，其实是因为我们为了展示开发进展而将开发中的特性分支构建出预览版本。

为了解决这个问题，我们为预览版本设置了独立的Jenkins Job。当我们发布尚未合并代码的预览版本时，也从不公布指向“最新”版本的链接，每个链接总是指向一个特定的版本。

我们有另一个Jenkins Job，它通过构建develop分支来发布“最新”版本。尽管这个策略还不错，但它无法解决本地构建（local builds）的问题。不管你是用你的本地机器还是通过Jenkins来构建，如果它们都能使用同一个构建编号就好了。而且这个方案还可以照顾到你没有使用Jenkins的私人娱乐项目。

## 好的构建编号

在我看来，一个好的构建编号可以反映软件版本的当前状态。它既不是一个时间戳，也不是一个随机增长的编号，因此，它不应该因为我构建相同的代码两次就产生变化。一个更大的编号应该代表这个软件有更新的版本，因为我们都知道：更高的编号总是更好的，这是众所周知的。

微软在Windows的构建编号上做得相当不错。你可以清楚地看到14342代表的版本要比11082代表的版本新很多，并且它和最新的14352版本差别不大。

SVN通过为每个提交增加修订版本（revision）也提供了不错的版本策略——它为本地和远程构建总是提供一样的修订版本。然后这在git中并不可行，因为分支是如此常见（这是个好事）。为当前分支的提交（commit）计数并不精确。“342个提交”并不指向一个特定的提交，并且不同分支上的多个提交都可能有341个历史提交（ancestor）。

## 为Git提供静态构建编号

我为解决这个问题，思考了很多并总结为以下的解决方案：

对每一次构建，我将其提交计数到主分支上（对大多数人来说，即“master”或“develop”）。这个提交计数是一个好的开始，但还不够，因为我认为它暴露了过多的项目信息。一旦有客户知道构建编号和提交计数是一样的，他们会开始说三道四：为什么最新的发布的提交数这么多/这么少。这促使我将项目年龄（project age）作为提交计数的一部分，并将其相加。

我决定为每一年设置1000的初始值——这意味每8.67小时会增加一次提交。那么当项目开一年半后，你就会有325个提交，而提交计数应该在825左右。这个数字很容易理解，对每个提交（本地或在Jenkins上）都很稳定，并且总是会增长。

## 多分支

这个方案本身不解决多分支问题。为此我对构建编号再增加一个后缀：一个当前分支名的两位字母Hash，以及从最后一次合并默认分支起，该特性分支上的提交计数（例子：825-**ud4**）。当你看到另一个有同样构建编号风格（825-za2）的版本时，这很容易就可以识别出这两个版本都衍生自同一次提交（构建编号825）。并且显然，不同分支构建会有不同的分支识别符。

当你告诉一个客户，自构建450开始引入了特性X，他再也不会问起，为什么在构建436-as16上为什么没有该特性了。并且当开发特性Y时，你可以提供带有“wt”-test-builds后缀的版本，这样客户们理解特性Y先是基于421-wt6的，而当你将最新的532构建合并到该特性分支后，它是基于532-wt6的。

## 本地变更

向客户提供调试版本并不常见。但当某个在特定设备上出现的缺陷需要调试时，这很有用。我也在和QA合作时使用这种方法。因此，识别带有本地变更的构建是很重要的。

我为带有本地变更的构建增加了大写的`-SNAPSHOT`后缀

```
1083-dm4(6)-SNAPSHOT
```

这个编号代表：特性分支（dm）相对主分支有4次提交，并且带有6次本地变更（但尚未Push）

## 在你的项目中使用该方案

It’s so easy. I wrote this logic as a gradle scriptscript which you can apply with a single line and access the version name with the ext.gitVersionName variable. The configuration is optional.

这个方案很简单，我使用Gradle脚本编写了代码。你可以仅用一行代码就可以将其添加到你的项目，并且使用`ext.gitVersionName`获得计算好的构建编号。而且还提供了自定义配置项：

```groovy
/ Top-level build file where you can add configuration options common to all sub-projects/modules.

buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.1.0'
        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

// Optional: configure the versioner
/*ext.gitVersioner = [
        defaultBranch           : "develop",  // default "master"
        yearFactor              : 1200,       // default "1000", increasing every 8.57h
        snapshotEnabled         : false,      // default false, the "-SNAPSHOT" postfix
        localChangesCountEnabled: false       // default false, the (<commitCount>) before -SNAPSHOT
]*/

// import the script which runs the version generation
apply from: 'https://raw.githubusercontent.com/passsy/gradle-GitVersioner/master/git-versioner.gradle'
```

```groovy
android {
    defaultConfig {
        ...
        buildConfigField 'String', 'REVISION', "\"$gitVersionName\""
    }
    
    productFlavors {
        ...
        beta {
            applicationIdSuffix '.beta'
            versionCode gitVersion.version
            versionName gitVersionName
        }
        
        playStore {
            versionCode 21
            versionName '3.0.1'
        }
    }
}
```

你也可以通过`gitVersion`变量获得编号的各个组成部分，以便你将它们组成自定义格式

更多信息和代码示例可以参考这个Github项目 https://github.com/passsy/gradle-GitVersioner

## PS:

可能你们中的一些人已经在是用 [Jake Whartons’ lazy strategy](https://plus.google.com/+JakeWharton/posts/6f5TcVPRZij) 来生成构建编号。这个策略也很棒，但我总是忘记手工增加构建编号。

```groovy
def versionMajor = 3
def versionMinor = 0
def versionPatch = 0
def versionBuild = 0 // bump for dogfood builds, public betas, etc.

android {
  defaultConfig {
    versionCode versionMajor * 10000 + versionMinor * 1000 + versionPatch * 100 + versionBuild
    versionName "${versionMajor}.${versionMinor}.${versionPatch}"
  }
}
```

可以使用我的脚本来实现他的策略。该版本对每个默认分支的提交都会增加编号。 #automateeverything

```groovy
// Optional: configure the versioner (before applying the script)
/* ext.gitVersioner = [
        defaultBranch           : "develop",  // default "master"
        yearFactor              : 1200,       // default "1000", increasing every 8.57h
        snapshotEnabled         : false,      // default false, the "-SNAPSHOT" postfix
        localChangesCountEnabled: false       // default false, the (<commitCount>) before -SNAPSHOT
] */
apply from: 'https://raw.githubusercontent.com/passsy/gradle-GitVersioner/master/git-versioner.gradle'

android {
  defaultConfig {
    versionCode gitVersion.version // will change after every commit
    versionName "1.0" // or the JakeWharton naming
    
    // Don't use a dynamic version name. This will change the AndroidManifest.xml for
    // every build and forces instantrun to reinstall the app instead of sending the 
    // diff to the device. Instant run will not work!
    // use Buildconfig.REVISION instead.
    // versionName gitVersionName
    buildConfigField 'String', 'REVISION', "\"$gitVersionName\""
  }
}
```

# 这很容易，对吗？

我很期待你对我的Git Versioner的看法和反馈，只需要几秒钟你就可以在现在的项目中测试一下！