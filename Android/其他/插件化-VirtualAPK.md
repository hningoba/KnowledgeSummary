加载插件Activity时遇到下述问题：java.lang.IllegalStateException: You need to use a Theme.AppCompat theme (or descendant) with this activity

解决方式：构建插件请使用gradle assemblePlugin，而不能直接通过AndroidStudio run出来一个插件apk。


### 1. 解析插件APK

使用PackageParser分析APK，主要是利用XMLPullParser解析AndroidMenifest.xml文件，获取包名（package）、Activity等信息。


### 2. 支持插件Activity

我们都知道，无法启动一个没有在Manifest中注册的Activity。插件对于宿主而言是个黑盒，也就是说常规方法无法启动插件中的Activity，那么VirtualAPK具体是怎么做到的呢？

简单来说，要启动目标Activity时，比如PluginDemo中的BookManagerActivity，先hook Instrumentation，将Intent中的目标Activity替换为占坑Activity，其中占坑Activity是在宿主中注册的，所以不会出现问题了。

PS: Activity有效检查是放在AMS侧，比如启动一个没有在Manifest中注册的Activity会报错。具体实现可以参考ActivityManagerService.startActivity()和ActivityStarter.startActivity()。

##### 2.1 替换Activity

宿主APP，AndroidManifest.xml中预先埋下一些占坑Activity

```
<application>
        <!-- Stub Activities -->
        <activity android:name=".A$1" android:launchMode="standard"/>
        <activity android:name=".A$2" android:launchMode="standard"
            android:theme="@android:style/Theme.Translucent" />

        <!-- Stub Activities -->
        <activity android:name=".B$1" android:launchMode="singleTop"/>
        <activity android:name=".B$2" android:launchMode="singleTop"/>

        <!-- Stub Activities -->
        <activity android:name=".C$1" android:launchMode="singleTask"/>
        <activity android:name=".C$2" android:launchMode="singleTask"/>

        <!-- Stub Activities -->
        <activity android:name=".D$1" android:launchMode="singleInstance"/>
        <activity android:name=".D$2" android:launchMode="singleInstance"/>

		...
		
    </application>
```

Activity替换逻辑：

```
ComponentsHandler.class

private void dispatchStubActivity(Intent intent) {
		// 此处component还是BookManagerActivity
        ComponentName component = intent.getComponent();
        String targetClassName = intent.getComponent().getClassName();
        LoadedPlugin loadedPlugin = mPluginManager.getLoadedPlugin(intent);
        ActivityInfo info = loadedPlugin.getActivityInfo(component);
        if (info == null) {
            throw new RuntimeException("can not find " + component);
        }
        int launchMode = info.launchMode;
        Resources.Theme themeObj = loadedPlugin.getResources().newTheme();
        themeObj.applyStyle(info.theme, true);
        
        // 根据launchMode获取占坑Activity，占坑Activity格式类似于“com.didi.virtualapk.core.A$1”
        String stubActivity = mStubActivityInfo.getStubActivity(targetClassName, launchMode, themeObj);
        Log.i(TAG, String.format("dispatchStubActivity,[%s -> %s]", targetClassName, stubActivity));
        
        // 将BookManagerActivity替换成占坑Activity
        intent.setClassName(mContext, stubActivity);
    }
```

Activity有效性检查：

```
Instrumentation.class

public ActivityResult execStartActivity() {
	...
	// AMS内部进行检查
	int result = ActivityManager.getService().startActivity();
	// 根据检查结果，如果Activity异常，抛出对应Exception
	checkStartActivityResult(result, intent);
	...
}
```

##### 2.2 还原Activity

虽然将目标Activity替换成占坑Activity，避免了启动一个没在Manifest中注册的Activity的错误。但是最终需要启动的不可能是占坑Activity，而是目标Activity。

Activity启动流程，最后会走到Instrumentation.newActivity()，看看hook后的VAInstrumentation的实现：

```
@Override
    public Activity newActivity(ClassLoader cl, String className, Intent intent) throws InstantiationException, IllegalAccessException, ClassNotFoundException {
        try {
            cl.loadClass(className);
        } catch (ClassNotFoundException e) {
            ComponentName component = PluginUtil.getComponent(intent);
            LoadedPlugin plugin = this.mPluginManager.getLoadedPlugin(component);
            String targetClassName = component.getClassName();

            if (plugin != null) {
                Activity activity = mBase.newActivity(plugin.getClassLoader(), targetClassName, intent);
                activity.setIntent(intent);

                ...
                
                return activity;
            }
        }

        return mBase.newActivity(cl, className, intent);
    }

```

先根据Intent获取目标Activity，即BookManagerActivity，然后利用ClassLoader（plugin.getClassLoader()）实例化目标Activity。从而实现反向替换，即可启动目标Activity。

### 3. 支持插件Service



参考：

[滴滴插件化方案 VirtualApk 源码解析](https://blog.csdn.net/lmj623565791/article/details/75000580)

[Activity启动流程(基于Android26)](https://juejin.im/entry/5abdcdea51882555784e114d)