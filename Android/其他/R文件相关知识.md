​		Android通过AAPT(Android Asset Packing Tool)将各种资源集成打包并生成索引文件，即R文件。做模块化或编译优化相关工作时，需要对R文件有更多的了解，比如library module中的资源需要加resourcePrefix，做增量编译时可以只编译application module中的R文件等等。

### application和library module中R文件的区别

结论：

* application module中的R文件的资源声明字段都是以**常量**形式存在，编译后使用到的资源字段以**常量值**的方式编译进class中。
* library module中的R文件的资源声明都是以**变量**形式存在，编译后使用到的字段会以**代码引用方式**编译进class中。

​	

举个例子：

​		在application module中新建MainActivity，布局文件是activity_main.xml。编译后，application module的R文件中可以看到：

```
public final class R {
	public static final class layout {
		public static final int activity_main=0x7f09001c;
	}
}
```

activity_main被final修饰，以常量形式存在。且编译后，这个field在class中的引用会被替换成常量值。

MainActivity.class中R.layout.activity_main会被替换成常量值0x7f09001c，十进制为2131296284。

```
public class MainActivity extends AppCompatActivity {
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		this.setContentView(2131296284);
	}
}
```



​		在library module中新建SubModuleActivity，布局文件是activity_sub_module.xml。编译后，library module中的R文件是：

```
public final class R {
	public static final class layout {
		public static int activity_sub_module = 0x7f0f001d;
	}
}
```

activity_sub_module没有被final修饰，以变量形式存在。

编译后，SubModuleActivity.class中R.layout.activity_sub_module仍然以变量形式存在，以代码引用方式编译进class中。

```
public class SubModuleActivity extends AppCompatActivity {
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        this.setContentView(layout.activity_sub_module);
    }
}
```



试想下，如果library module中的R文件的资源声明也是以常量形式声明的话，代码中的资源引用会被替换成常量值。library module和application module中如果出现同名资源，根据R文件合并规则，该资源最终会以application中的资源为准，这就导致library module中代码里资源的常量值对应不到资源，报Resource NotFoundException。



### module中R文件个数

​		一个module被编译时，会生成一个当前module的R文件，同时，该module依赖的module或aar也会在当前module生成R文件。这种依赖关系也包括跨层的传递依赖。所以，一个module中R文件的个数 = 其依赖的module/aar数量 + 1(自身的R文件)。

​		同理，application module中包含了整个app所有模块的R.java文件。

举个例子：

demo项目依赖结构：

```
 - app (pkg: com.tools.demo)
  - library (pkg: com.tools.second)
   - third (pkg: com.tools.third)
```

其中library引用了fresco。

编译后，library模块中不仅包含了自身的R文件(com.tools.second.R)，还包含了依赖的module(third模块)和依赖的aar(fresco)的R文件。

<img src="https://github.com/hningoba/KnowledgeSummary/blob/master/img/R%E6%96%87%E4%BB%B6%E7%BB%93%E6%9E%84_library.png?raw=true" style="zoom:70" />

app模块则包含了整个app所有模块的R文件，包括直接依赖(library)和间接依赖(third, fresco)的所有module/aar的R文件。

<img src="https://github.com/hningoba/KnowledgeSummary/blob/master/img/R%E6%96%87%E4%BB%B6%E7%BB%93%E6%9E%84_app.png?raw=true" style="zoom:70" />





### 资源合并规则

APK中的资源主要有3个来源：

* The main source set (generally located in  src/main/res/)
* Build variant source sets
* Android libraries (AARs)

​		一个资源文件的文件名在resource type (anim/drawable/layout/menu..) 和 resource qualifier (比如drawable中的hdpi、value中的语言) 目录内是唯一的话，这个资源就被认为是唯一的。

​		资源文件冲突时，按照下面的优先级进行合并，低优的资源会被覆盖。

```
build variant > build type > product flavor > main source set > library dependencies
```

比如：main source set包含以下资源：

- res/layout/foo.xml
- res/layout-land/foo.xml

debug build type包含：

* res/layout/foo.xml

最终APK文件中的`res/layout/foo.xml`来自debug build type，`res/layout-land/foo.xml`来自main source set。



##### build variant：

​	build variant可以认为是build type和product flavor的结合。Gradle 会根据build type和product flavor自动创建build variant，并按照 `<product-flavor><Build-Type>` 的格式命名这些变体。

```
android {
    ...
    defaultConfig {...}
    
    buildTypes {
        debug{...}
        release{...}
    }
    
    productFlavors {
        internal {...}
        googlePlay {...}
    }
}
```

​	根据上述配置，gradle会构建以下build variant：

* internalDebug
* internalRelease
* googlePlayDebug
* googlePlayRelease



参考：

[Android Resource merging](https://developer.android.com/studio/write/add-resources.html#resource_merging)

