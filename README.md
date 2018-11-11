# Gradle基础 构建生命周期和Hook技术
[https://juejin.im/post/5afec54951882542715001f2](https://juejin.im/post/5afec54951882542715001f2)
任何Gradle的构建过程都分为三部分：初始化阶段、配置阶段和执行阶段。

**一定要注意，配置阶段不仅执行build.gradle中的语句，还包括了Task中的配置语句。**从上面执行结果中可以看到，在执行了dependencies的闭包后，直接执行的是任务test中的配置段代码（Task中除了Action外的代码段都在配置阶段执行）。 另外一点，无论执行Gradle的任何命令，**初始化阶段和配置阶段的代码都会被执行**

![图片](https://user-gold-cdn.xitu.io/2018/7/3/1645f7712096f3e6?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

# dependOn DAG
[https://blog.csdn.net/sergeycao/article/details/54341813](https://blog.csdn.net/sergeycao/article/details/54341813)
使用dependOn属性将任务插入到有向无环图中。
讨论
在初始化阶段，Gradle根据它们的依赖性将任务组合成一个序列。结果是DAG。例如，Gradle文档形成了Java插件的DAG，如图4-1所示。

# 自定义Task并使得原有Task依赖它--afterEvaluate
app的buile.gradle
```
apply from: rootProject.file('tasks/tasks1.gradle')
project.afterEvaluate {
    println 'app的build文件************************* '
    assembleDebug.dependsOn beforeAssembleDebug
}
```

task1.gradle
```
task afterAssembleDebug() {
    doLast {
        println '111doLast afterAssembleDebug'
    }
}
```

# 自定义Task并依赖原有Task--whenTaskAdded 
[https://blog.csdn.net/zhaoyanjun6/article/details/78523958?locationNum=3&fps=1](https://blog.csdn.net/zhaoyanjun6/article/details/78523958?locationNum=3&fps=1)
在 app的 build.gradle 添加
apply from:"../util.gradle"
这样在 app的build.gradle 就可以用 copyFile task 了

//在app的build.gradle中添加如下
apply from:"../util.gradle"
//在task被添加的时候定义依赖关系
```
tasks.whenTaskAdded {
    task ->
        if (task.name.contains("assembleDebug")) {
            task.getDependsOn().add({
                println 'app getDependsOn doLast afterAssembleDebug'
                afterAssembleDebug//可以直接执行这个task或者说是依赖
            })
        }
}
```

util.gradle如图
![图片](https://img-blog.csdn.net/20171117165014556?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhhb3lhbmp1bjY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
# 
# [Gradle的执行顺序](https://segmentfault.com/q/1010000004503896)和生命周期函数
## gradle的解析顺序如下
rootproject 的setting.gradle,然后是rootproject的build.gradle,然后是各个subproject。所以project下的build.gradle会先于app下的build.gradle。
在build.gradle中，我们可以通过apply plugin:*** 引入插件，也可以通过 ******apply from****** ***.gradle引入其他gradle脚本中的函数定义或task等。
## gradle里的钩子
**一般hook我们指的是gradle的生命周期**：
在解析setting.gradle之后，开始解析build.gradle之前，这里如果要干些事情（更改build.gradle校本内容），可以写在beforeEvaluate
>举个例子，我们将我们的一个subproject中的apply plugin改掉，原来是一个library工程，我们希望它被当作application处理：
```
    project.beforeEvaluate {
                // Change android plugin from `lib' to `application' dynamically
                // FIXME: Any better way without edit file?
    
                if (mBakBuildFile.exists()) {
                    // With `tidyUp', should not reach here
                    throw new Exception("Conflict buildFile, please delete file $mBakBuildFile or " +
                            "${project.buildFile}")
                }
    
                def text = project.buildFile.text.replaceAll(
                        'com\\.android\\.library', 'com.android.application')
                project.buildFile.renameTo(mBakBuildFile)
                project.buildFile.write(text)
            }
```
>在所有build.gradle解析完成后，开始执行task之前，此时所有的脚本已经解析完成，task，plugins等所有信息可以获取，task的依赖关系也已经生成，如果此时需要做一些事情，可以写在afterEvaluate
```
project.afterEvaluate {
            // Set application id
            def manifest = new XmlParser().parse(project.android.sourceSets.main.manifestFile)
            project.android.defaultConfig.applicationId = manifest.@package
        }
```
>每个task都可以定义doFirst，doLast，用于定义在此task执行之前或之后执行的代码

```
project.assemble.doLast {
                    println "assemble finish"
                }
project.assemble.doFirst {
                    println "assemble start"
                }
```




# 全面理解Gradle - 执行时序
[https://blog.csdn.net/singwhatiwanna/article/details/78797506](https://blog.csdn.net/singwhatiwanna/article/details/78797506)
Gradle脚本的执行分为三个过程：
## 初始化 
分析有哪些module将要被构建，为每个module创建对应的 project实例。这个时候settings.gradle文件会被解析。
## 配置
处理所有的模块的 build 脚本，处理依赖，属性等。这个时候每个模块的build.gradle文件会被解析并配置，这个时候会构建整个task的链表（这里的链表仅仅指存在依赖关系的task的集合，不是数据结构的链表）。
## 执行
根据task链表来执行某一个特定的task，这个task所依赖的其他task都将会被提前执行。
# Gradle插件
[https://benweizhu.gitbooks.io/gradle-best-practice/content/plugin.html](https://benweizhu.gitbooks.io/gradle-best-practice/content/plugin.html)

**脚本插件**，其实就是存在另一个脚本文件（other.gradle）的一段脚本代码，通常情况下存放在同一个构建项目下，主要作用是抽取逻辑，让关注点分离（separate of concern）。
**二进制插件**，则是编译后的class文件，它们通过实现Gradle API中的Plugin接口，通过编程的方式操作构建过程。二进制插件可以在直接在构建脚本（build.gradle）中写，也可以是一个单独的project中（独立的项目模块），又或者来自于一个Jar文件（最常用的做法）。
**插件的使用**
```
apply from: 'other.gradle' // 使用脚本插件
apply plugin: 'java' // 使用二进制插件
```
使用插件的方式是调用Project对象的Project.apply(java.util.Map)方法。在二进制插件的使用中，传入的参数（比如‘java’），叫做plugin id。这个id必须是唯一的，一些核心的插件，Gradle给他们提供了简短的名字，比如：java。而社区的插件，名字则会采用完整名字，比如：me.zeph.database。


# Android Gradle复制打包的apk到固定目录
[https://blog.csdn.net/guijiaoba/article/details/42655437](https://blog.csdn.net/guijiaoba/article/details/42655437)

apply plugin: 'com.android.application'
 
android {
    // ***
}
 
dependencies {
    // *****
}
/// **********************下面的代码才是最主要的*************************
build {
    doLast {
        def fileName = "app-release.apk"
        def fromFile = "./build/outputs/apk/" + fileName
        def intoFile = "./outapks/"
 
 
        def applicationId = android.defaultConfig.applicationId
        def versionName = android.defaultConfig.versionName
        def time = new java.text.SimpleDateFormat("yyyyMMddHHmmss").format(new Date())
        def buildType = "realse"
        def channel = "site"
 
 
        def appName = "${applicationId}_v${versionName}_${time}_${buildType}_${channel}.apk"
 
 
        // copy --> rename
        copy {
            from fromFile
            into intoFile
 
 
            rename {
                appName
            }
        }
 
 
        println("=====================build.doLast success.=========================")
    }
}


# **基本概念(Project 和 Task)**
[http://www.blogjava.net/wldandan/archive/2012/06/27/381605.html](http://www.blogjava.net/wldandan/archive/2012/06/27/381605.html)
doLast意思是定义一个行为(映射Gradle中的Action类)，放在当前task的最后，类似的，还有doFirst, 表示将定义的行为放在当前task最前面，例如
task hello {
  doLast{
      println "Hello world"
    }   
  doFirst{
      println "I am xxx"
    }   
}
执行gradle hello, 将输出
"I am xxx"
"Hello world"

另外，你也可以使用如下更简洁的方式来定义task：
 
task hello <<  {
     println "hello world"
}

