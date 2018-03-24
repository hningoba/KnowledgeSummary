# Tinker

[优秀博客](http://w4lle.com/2016/12/16/tinker/)

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


