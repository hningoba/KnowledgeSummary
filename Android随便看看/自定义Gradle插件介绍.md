自定义gradle插件现在用的越来越广泛，比如增量编译、路由、插桩等等。这篇文章比较基础，算是gradle插件入门，部分内容也是直接来自官网。

## 构建方式

[官方文档](https://docs.gradle.org/current/userguide/custom_plugins.html#custom_plugins)提出来三种gradle插件的构建方式。

### Build script 

这种方式是直接在build脚本中编写task，这种方式的好处是自动编译、自动包进build脚本目录中，使用非常方便。坏处是不能被其他脚本引用。

官方实例如下：

```
class GreetingPlugin implements Plugin<Project> {
    void apply(Project project) {
        project.task('hello') {
            doLast {
                println 'Hello from the GreetingPlugin'
            }
        }
    }
}
```

使用插件时直接``apply plugin: GreetingPlugin``即可。这种方式引用插件时不需要引号包住。

### buildSrc

这种构建方式是以一个名称必须为buildSrc的module形式构建插件，可以直接在其他module中引用，编译时也会自动构建该buildSrc，开发测试非常方便。根据我们的开发语言，代码可以放到下面三种中的任意一种：

* rootProjectDir/buildSrc/src/main/groovy
* rootProjectDir/buildSrc/src/main/java
* rootProjectDir/buildSrc/src/main/kotlin

这种方式开发流程大致如下：

1. 在项目中创建名称为buildSrc的module

2. buildSrc/build.gradle中添加如下配置

```
apply plugin: 'groovy'

dependencies {
    implementation gradleApi()
    implementation localGroovy()
    implementation 'com.android.tools.build:gradle:3.5.0'
    implementation 'com.android.tools.build:gradle-api:3.5.0'
}

repositories {
    google()
    jcenter()
    mavenCentral()
}
```

3. 实现插件类

在buildSrc module中创建Plugin实现类，即我们的插件类。根据我们使用的语言创建基础目录，目前支持groovy、kotlin、java三种。三种语言对应目前前面已经提到。以groovy为例，创建package=``com.neptune.buildsrc``， class=``AsmPlugin``插件类。以打印日志作示例：

```
package com.neptune.buildsrc

import org.gradle.api.Plugin
import org.gradle.api.Project

class AsmPlugin implements Plugin<Project> {

    @Override
    void apply(Project project) {
        println("====== AsmPlugin#apply() ======")
    }
}
```

4. META-INF中声明插件

创建``resources/META-INF/gradle-plugins/com.neptune.asm-plugin.properties``文件，文件中添加如下配置：

```
implementation-class=com.neptune.buildsrc.AsmPlugin // Plugin实现类
```

注意，properties文件名``com.neptune.asm-plugin``即为插件id。这个文件的目的主要是让gradle找到Plugin的实现。

5. 使用插件

app/build.gradle中引用插件：

```
apply plugin: 'com.neptune.asm-plugin' // 插件id
```

上述步骤完成后，构建程序时就能看到AsmPlugin中打印的日志。

项目目录结构如下：

<img src="https://raw.githubusercontent.com/hningoba/KnowledgeSummary/master/img/gradle_plugin_project_directory_structure.png"/>



### Standalone project

前面两种构建方式，创建的插件只能在当面项目中使用。standalone projec可以以一个独立项目的形式构建插件，构建成功后可以将插件发布到maven，提供给其他项目使用。

这种方式的开发流程和buildSrc类似，只是创建的module名称不受限制了，具体不再重复描述。下面提供比较通用的插件上传maven仓库的task供参考。

项目中创建上传脚本gradle文件，比如upload.gradle：

```
apply plugin: 'maven'

def VERSION_NAME = '1.2.3'
def POM_ARTIFACT_ID = 'your_project_name'
def GROUP = 'your_package_name'

uploadArchives {
  repositories {
    mavenDeployer {
      pom.groupId = GROUP
        pom.artifactId = POM_ARTIFACT_ID
        pom.version = VERSION_NAME
        // release仓
        repository(url: your_release_repo_url) {
          authentication(userName: your_user_name, password: your_pwd)
        }
        // snapshot仓
      snapshotRepository(url: your_snapshot_repo_url) {
        authentication(userName: your_user_name, password: your_pwd)
      }
    }
  }
}
```

插件编写完成，执行`` ./gradlew uploadArchives  ``上传即可。



## 调试插件

如果能够调试插件，对于提升开发效率是很有帮助的。下面介绍一种开发方式：

1. 添加远程调试

工具栏Edit Configurations -> +号 -> Remote -> 填写远程Host和端口号，参考下图。其中Host填localhost即可。

<img src="https://raw.githubusercontent.com/hningoba/KnowledgeSummary/master/img/android_studio_build_debug.png"/>

2. 项目目录下，控制台执行下面命令，开启debug且不使用daemon进程。

```
./gradlew :app:clean -Dorg.gradle.debug=true  --no-daemon
```

命令执行后，如下图，进程挂起，直到attach到调试进程。

<img src="https://raw.githubusercontent.com/hningoba/KnowledgeSummary/master/img/android_studio_debug_waiting.png"/>

3. 启动调试进程，如下图

<img src="https://raw.githubusercontent.com/hningoba/KnowledgeSummary/master/img/android_studio_debug_start.png"/>

上述步骤完成后，就能成功调试build期间的代码。



## Gradle Build Lifecycle

[build lifecycle](https://docs.gradle.org/4.3/userguide/build_lifecycle.html)



## Transform

[transform](http://google.github.io/android-gradle-dsl/javadoc/2.1/com/android/build/api/transform/Transform.html)



## 常见问题：

下面的几个问题可能对于其他同学来说并不算常见问题，只是在我开发插件过程中遇到的问题。

### groovy.lang.MissingPropertyException: No such property

自定义类中缺少对使用的类的引用时就会报错。这点比较坑，因为IDE并不会提示所有应用的类需要import，groovy又是一种动态语言，只会在运行时报错。



###  javassist.CannotCompileException: [source error] no such class

使用Javassist的ClassPool对class文件做处理时，实例化ClassPool时要用``ClassPool.getDefault()``或``ClassPool(true)``，本质上是要执行``ClassPool.appendSystemPath()``。这个方法的意思是类的搜索路径添加了platform library、 extension libraries和classpath定义环境变量中的类。

```
   /**
     * Appends the system search path to the end of the
     * search path.  The system search path
     * usually includes the platform library, extension
     * libraries, and the search path specified by the
     * <code>-classpath</code> option or the <code>CLASSPATH</code>
     * environment variable.
     *
     * @return the appended class path.
     */
    public ClassPath appendSystemPath() {
        return source.appendSystemPath();
    }
```



### 处理Android相关类

因为Android SDK中的类并不是Javassist默认支持的类库，需要特殊处理。处理方式就是通过ClassPool手动添加SDK jar文件路径。

```
String androidJar = "${android.sdkDirectory.absolutePath}/platforms/${android.compileSdkVersion}/android.jar"
ClassPool.getDefault().insertClassPath(androidJar)
```







编译后的文件路径：

app/build/intermediates/javac/debug/compileDebugJavaWithJavac/...







## 参考

[Developing Custom Gradle Plugins](https://docs.gradle.org/current/userguide/custom_plugins.html#custom_plugins)

[Using Gradle Plugins](https://docs.gradle.org/current/userguide/plugins.html)

[Gradle build lifecycle](https://docs.gradle.org/4.3/userguide/build_lifecycle.html)

[深入理解Transform](https://juejin.im/post/5cbffc7af265da03a97aed41#heading-6)