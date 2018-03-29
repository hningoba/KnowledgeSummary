# Tinker

[Tinker Wiki](https://github.com/Tencent/tinker/wiki)
[优秀博客](http://w4lle.com/2016/12/16/tinker/)

### 技术选型
当前比较流行的热补丁方案有微信Tinker、阿里AndFix、美团Robust、QZone超级补丁。每个平台都有自己的局限，大家在技术选型时可以根据业务需要做选择。

|		| Tinker |	QZone |	AndFix | Robust |
| ------|:-------|:------:| -----:|
| 类替换 | yes | yes | no | no | 
| So替换	| yes	| no	| no| 	no| 
| 资源替换| 	yes| 	yes| 	no| 	no| 
| 全平台支持| 	yes| 	yes| 	yes| 	yes| 
| 即时生效| 	no| 	no| 	yes| 	yes| 
| 性能损耗| 	较小| 	较大| 	较小| 	较小| 
| 补丁包大小| 	较小| 	较大| 	一般| 	一般| 
| 开发透明| 	yes| 	yes| 	no| 	no| 
| 复杂度| 	较低| 	较低| 	复杂| 	复杂| 
| gradle支持| 	yes| 	no| 	no| 	no| 
| Rom体积| 	较大| 	较小| 	较小| 	较小| 
| 成功率| 	较高| 	较高| 	一般| 	最高| 


总的来说:

1. AndFix作为native解决方案，首先面临的是稳定性与兼容性问题，更重要的是它无法实现类替换，它是需要大量额外的开发成本的；
2. Robust兼容性与成功率较高，但是它与AndFix一样，无法新增变量与类只能用做的bugFix方案；
3. Qzone方案可以做到发布产品功能，但是它主要问题是插桩带来Dalvik的性能问题，以及为了解决Art下内存地址问题而导致补丁包急速增大的。


**Tinker的已知问题**

由于原理与系统限制，Tinker有以下已知问题：

Tinker不支持修改AndroidManifest.xml，Tinker不支持新增四大组件(1.9.0支持新增非export的Activity)；
由于Google Play的开发者条款限制，不建议在GP渠道动态更新代码；
在Android N上，补丁对应用启动时间有轻微的影响；
不支持部分三星android-21机型，加载补丁时会主动抛出"TinkerRuntimeException:checkDexInstall failed"；
对于资源替换，不支持修改remoteView。例如transition动画，notification icon以及桌面图标。

### 衍生问题
1. Dalvik VS ART
2. oat mode
3. 如何加载dex、so、资源等文件

###### 4. 如何打差分包，即补丁文件
使用DexDiff算法，参考[link](https://www.zybuluo.com/dodola/note/554061)


###### 5. 不同API level，哪些环节需要分别处理
加载dex文件会区分API level。<br>

Level 24之前：<br>
采用类似于MultiDex加载dex文件的原理，将补丁dex在dexElements中前置。因为谁在前面用谁，所以系统会使用补丁代码，从而达到热修复功能。

Level 24及其之后：<br>


```
补充知识：
API Level 26 : Android Oreo, 8.0
API Level 24 : Android Nougat, 7.0
API Level 23 : Android Marshmallow, 6.0
API Level 21 : Android Lollipop, 5.0
API Level 19 : Android Kitkat, 4.4
```

###### 6. MultiDex原理？Tinker参考了MultiDex哪些技术？

###### 7. InstantRun原理？Tinker参考了InstantRun哪些技术？
更新资源文件时，借鉴了InstantRun的技术。




#### 重要点
1. Tinker的补丁方案，Tinker采用的是下发差分包，然后在手机端合成全量的dex文件进行加载。

###### 加载补丁dex文件

###### 加载补丁so文件

###### 加载补丁资源文件
采用InstantRun的加载补丁资源方式，全量替换资源。实现方式是hook AssetManager.addAssetPath()，将补丁的资源目录传进去，以此达到替换老资源的方式。

InstantRun的资源更新方式最简便而且兼容性也最好，市面上大多数的热补丁框架都采用这套方案。Tinker的这套方案虽然也采用全量的替换，但是在下发patch中依然采用差量资源的方式获取差分包，下发到手机后再合成全量的资源文件，有效的控制了补丁文件的大小。


