

## 自定义Gradle插件流程

编译后的文件路径：

app/build/intermediates/javac/debug/compileDebugJavaWithJavac/...



## 构建方式

### 1.BuildSrc

2. 

### 4.includeBuild

​	BuildSrc本质上也是一种includeBuild



### 如何测试插件

命令：

```
./gradlew assembleDebug --debug --info -Dorg.gradle.daemon=false -Dorg.gradle.debug=true
```





Multi Project Build & Composite Build



## Transform介绍



## Javassist介绍

[Javassist Tutorial](https://www.javassist.org/tutorial/tutorial.html)







## 常见问题：

下面的几个问题可能对于其他同学来说并不算常见问题，只是在我开发插件过程中遇到的问题。

##### 1. groovy.lang.MissingPropertyException: No such property

自定义类中缺少对使用的类的引用时就会报错。这点比较坑，因为IDE并不会提示所有应用的类需要import，groovy又是一种动态语言，只会在运行时报错。



##### 2. javassist.CannotCompileException: [source error] no such class

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



##### 3. 处理Android相关类

因为Android SDK中的类并不是Javassist默认支持的类库，需要特殊处理。处理方式就是通过ClassPool手动添加SDK jar文件路径。

```
String androidJar = "${android.sdkDirectory.absolutePath}/platforms/${android.compileSdkVersion}/android.jar"
ClassPool.getDefault().insertClassPath(androidJar)
```



### 参考

[Developing Custom Gradle Plugins](https://docs.gradle.org/current/userguide/custom_plugins.html#custom_plugins)

[Using Gradle Plugins](https://docs.gradle.org/current/userguide/plugins.html)