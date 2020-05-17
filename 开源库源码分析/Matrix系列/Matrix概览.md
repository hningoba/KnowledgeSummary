---
title: Matrix概览
categories: 技术
---

Matrix是WeChat研发的一款APM工具，可以对应用安装包大小，帧率变化，启动耗时，卡顿，慢方法，SQLite 操作优化，文件读写，内存泄漏等等做检测。该库的主要贡献者同时也是[《Android开发高手课》](https://time.geekbang.org/column/intro/100021101)的作者，这个课程对APP性能涉及到的技术讲的比较广泛也很有深度，可以作为性能优化的理论指导课，建议对性能优化感兴趣的同学也深入学习下该课程。

<!--more-->

笔者近期在组里承担了一些性能优化的工作，写Matrix的分析文章目的其实是为了从原理层面了解业界对各种性能指标的检测方法，从而更系统的学习性能优化这部分知识。

Matrix-android 当前包含五块内容：

- APK Checker: 针对 APK 安装包的分析检测工具，根据一系列设定好的规则，检测 APK 是否存在特定的问题，并输出较为详细的检测结果报告，用于分析排查问题以及版本追踪
- Resource Canary: 基于 WeakReference 的特性和 [Square Haha](https://github.com/square/haha) 库开发的 Activity 泄漏和 Bitmap 重复创建检测工具
- Trace Canary: 监控界面流畅性、启动耗时、页面切换耗时、慢函数及卡顿等问题
- SQLite Lint: 按官方最佳实践自动化检测 SQLite 语句的使用质量
- IO Canary: 检测文件 IO 问题，包括：文件 IO 监控和 Closeable Leak 监控



### Matrix初始化

使用Matrix之前，看下Matrix的初始化逻辑：

```
sample.tencent.matrix.MatrixApplication

@Override
    public void onCreate() {
        super.onCreate();
        // Matrix配置，项目实际使用时可以动态下发
        DynamicConfigImplDemo dynamicConfig = new DynamicConfigImplDemo();
        boolean matrixEnable = dynamicConfig.isMatrixEnable();
        boolean fpsEnable = dynamicConfig.isFPSEnable();
        boolean traceEnable = dynamicConfig.isTraceEnable();

				// 初始化Builder
        Matrix.Builder builder = new Matrix.Builder(this);
        
        // 添加自定义PluginListener
        builder.patchListener(new TestPluginListener(this));

        //trace 配置
        TraceConfig traceConfig = new TraceConfig.Builder()
                .dynamicConfig(dynamicConfig)
                .enableFPS(fpsEnable) // 监控FPS帧率
                .enableEvilMethodTrace(traceEnable) // 慢函数
                .enableAnrTrace(traceEnable) // ANR检测
                .enableStartup(traceEnable) // 启动耗时
                .splashActivities("sample.tencent.matrix.SplashActivity;")
                .isDebug(true)
                .isDevEnv(false)
                .build();

				// Trace模块的插件，内部管理FrameTracer/EvilMethodTracer/AnrTracer/StartupTracer
        TracePlugin tracePlugin = (new TracePlugin(traceConfig));
        builder.plugin(tracePlugin);

        if (matrixEnable) {
            //Resource监控初始化
            Intent intent = new Intent();
            ResourceConfig.DumpMode mode = ResourceConfig.DumpMode.AUTO_DUMP;
            intent.setClassName(this.getPackageName(), "com.tencent.mm.ui.matrix.ManualDumpActivity");
            ResourceConfig resourceConfig = new ResourceConfig.Builder()
                    .dynamicConfig(dynamicConfig)
                    .setAutoDumpHprofMode(mode)
                    .setNotificationContentIntent(intent)
                    .build();
            builder.plugin(new ResourcePlugin(resourceConfig));
            ResourcePlugin.activityLeakFixer(this);

            //文件IO监控初始化
            IOCanaryPlugin ioCanaryPlugin = new IOCanaryPlugin(new IOConfig.Builder()
                    .dynamicConfig(dynamicConfig)
                    .build());
            builder.plugin(ioCanaryPlugin);

            //Sqlite监控初始化
            SQLiteLintConfig sqlLiteConfig = new SQLiteLintConfig(SQLiteLint.SqlExecutionCallbackMode.CUSTOM_NOTIFY);
            builder.plugin(new SQLiteLintPlugin(sqlLiteConfig));
        }

				// 初始化Matrix
        Matrix.init(builder.build());

				// 启动Trace
        //start only startup tracer, close other tracer.
        tracePlugin.start();
    }
```

可以看出，主要是对开篇介绍的Trace Canary、Resource Canary、IO Canary、Sqlite Lint的初始化。后续通过分析这些模块看看各种性能监控项的实现原理。



### Trace Canary

监控界面流畅性、启动耗时、页面切换耗时、慢函数及卡顿等问题。主要特性如下：

- 编译期动态修改字节码, 高性能记录执行耗时与调用堆栈
- 准确的定位到发生卡顿的函数，提供执行堆栈、执行耗时、执行次数等信息，帮助快速解决卡顿问题
- 自动涵盖卡顿、启动耗时、页面切换、慢函数检测等多个流畅性指标

具体实现分析参考：[Matrix - TraceCanary源码分析](https://hningoba.github.io/2020/04/28/Matrix-TraceCanary实现分析/)



### Resource Canary

基于 WeakReference 的特性和 [Square Haha](https://github.com/square/haha) 库开发的 Activity 泄漏和 Bitmap 重复创建检测工具。主要特性如下：

- 分离了检测和分析部分，便于在不打断自动化测试的前提下持续输出分析后的检测结果
- 对检测部分生成的 Hprof 文件进行了裁剪，移除了大部分无用数据，降低了传输 Hprof 文件的开销
- 增加了重复 Bitmap 对象检测，方便通过减少冗余 Bitmap 数量，降低内存消耗

具体实现分析参考：[]()



其他模块后续补充中。



### 参考

[Matrix](https://github.com/Tencent/matrix)

[Matrix Wiki](https://github.com/Tencent/matrix/wiki/Matrix-Android-TraceCanary)