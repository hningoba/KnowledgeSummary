### 整体结构图



![Fresco主要模块结构图](http://ww2.sinaimg.cn/large/006tNc79ly1g4p3ulbhluj30u015n0y7.jpg)



### 主要环节代码流程

```
-> SimpleDraweeView.setImageUri()
-> AbstractDraweeControllerBuilder.build()
-> AbstractDraweeControllerBuilder.buildController()
-> PipelineDraweeControllerBuilder.obtainController()
-> AbstractDraweeControllerBuilder.obtainDataSourceSupplier()
-> AbstractDraweeControllerBuilder.getDataSourceSupplierForRequest()
-> PipelineDraweeControllerBuilder.getDataSourceForRequest()
-> ImagePipeline.fetchDecodedImage()
-> ImagePipeline.submitFetchRequest()
-> CloseableProducerToDataSourceAdapter.create()
-> AbstractProducerToDataSourceAdapter.construtor()

case1: 首次冷启动APP加载图片，没有BitmapCache，但有磁盘缓存
-> BitmapMemoryCacheGetProducer.produceResults() //后面是拿数据过程，有内存/磁盘/网络多种Producer
-> ThreadHandoffProducer.produceResults()
-> StatefulRunnable.onSuccess()
-> BitmapMemoryCacheKeyMultiplexProducer.produceResults()
-> MultiplexProducer.Multiplexer.startInputProducerIfHasAttachedConsumers()
-> BitmapMemoryCacheProducer.produceResults() //内存缓存
-> DecodeProducer.produceResults()
-> ResizeAndRotateProducer.produceResults()
-> AddImageTransformMetaDataProducer.produceResults()
-> EncodedCacheKeyMultiplexProducer.produceResults()
-> MultiplexProducer.Multiplexer.startInputProducerIfHasAttachedConsumers()
-> EncodedMemoryCacheProducer.produceResults() //未解码缓存
-> DiskCacheReadProducer.produceResults() // 磁盘缓存
-> SmallCacheIfRequestedDiskCachePolicy.createAndStartCacheReadTask()
-> BufferedDiskCache.get()
-> BufferedDiskCache.getAsync(). Callable.call()
	Q: disk cache staging area???
-> BufferedDiskCache.readFromDiskCache() // 读取磁盘文件
-> DiskStorageCache.getResource() // 获取本地资源，如果修改缓存策略，可以在这里处理
-> DefaultDiskStorage.getResource()
-> DiskCacheReadProducer.onFinishDiskReads()
-> DiskCacheReadProducer.Continuation.then()
	ps: EncodedImage cachedReference = task.getResult() 不为空说明拿到磁盘缓存结果，
	否则，走网络请求，即执行MediaVariationsFallbackProducer.produceResults()
	下面都是回调过程。
-> EncodedMemoryCacheProducer.EncodedMemoryCacheConsumer.onNewResultImpl()
-> InstrumentedMemoryCache.cache() // 将拿到的磁盘缓存结果存储到内存缓存中
-> CountingMemoryCache.cache()
-> MultiplexProducer.Multiplexer.ForwardingConsumer.onNewResultImpl()
-> AddImageTransformMetaDataProducer.onNewResultImpl()

case2: 没有BitmapCache、磁盘缓存，走网络请求
前面流程和拿磁盘缓存一样。
-> DiskCacheReadProducer.onFinishDiskReads()
-> DiskCacheReadProducer.Continuation.then()
-> MediaVariationsFallbackProducer.produceResults()
-> MediaVariationsFallbackProducer.startInputProducerWithExistingConsumer()
-> DiskCacheWriteProducer.produceResults()
-> DiskCacheWriteProducer.maybeStartInputProducer()
-> NetworkFetchProducer.produceResults()
-> HttpUrlConnectionNetworkFetcher.fetch() // 配置了OkHttp，此处走OkHttpNetworkFetcher.fetch()
-> HttpUrlConnectionNetworkFetcher.fetchSync() // 网络请求图片
-> HttpUrlConnectionNetworkFetcher.downloadFrom()
-> NetworkFetchProducer.onResponse() // 请求到网络图片，回调
```



SimpleDraweeView attach到window时，执行如下流程：

```
-> DraweeView.onAttachedwindow()
-> DraweeHolder.onAttach()
-> DraweeHolder.attachController()
-> AbstrackDraweeController.onAttach()
-> AbstrackDraweeController.submitRequest()
-> AbstractDataSource.subscribe()
```



### 主要类

##### ImagePipline

负责从网络、本地文件、本地资源加载图片，并将其解码到内存中供系统使用。

Fresco 中设计有一个叫做 Image Pipeline 的模块。它负责从网络，从本地文件系统，本地资源加载图片。为了最大限度节省空间和CPU时间，它含有3级缓存设计（2级内存，1级磁盘）。

Image pipeline 负责完成加载图像，变成Android设备可呈现的形式所要做的每个事情。

大致流程如下:

- 检查内存缓存，如有，返回
- 后台线程开始后续工作
- 检查是否在未解码内存缓存中。如有，解码，变换，返回，然后缓存到内存缓存中。
- 检查是否在磁盘缓存中，如果有，变换，返回。缓存到未解码缓存和内存缓存中。
- 从网络或者本地加载。加载完成后，解码，变换，返回。存到各个缓存中。

既然本身就是一个图片加载组件，那么一图胜千言。
![](https://www.fresco-cn.org/static/imagepipeline.png)



##### Drawees

Fresco 中设计有一个叫做 Drawees 模块，它会在图片加载完成前显示占位图，加载成功后自动替换为目标图片。当图片不再显示在屏幕上时，它会及时地释放内存和空间占用。


```

ImageRequest, 

Supplier
DataSource
CloseableReference
CloseableImage

DraweeController:

SimpleDraweeControllerBuilder(I):
DraweeController的构造器，主要实现类是PipelineDraweeControllerBuilder。
继承结构：
PipelineDraweeControllerBuilder -> 
AbstractDraweeControllerBuilder -> 
SimpleDraweeControllerBuilder


**AbstractDataSource:**
封装了Producer，
具体实现类 AbstractProducerToDataSourceAdapter

AbstractDataSource
	->	VolleyDataSource
	->	AbstractProducerToDataSourceAdapter
		->	CloseableProducerToDataSourceAdapter

```

##### Producer

```
使用生产者消费者模式(Producer-Consumer)，获取不同来源的图片资源(cache, file, url)。

继承关系：
-> Producer (I)
	-> BitmapMemoryCacheProducer
	-> EncodedMemoryCacheProducer
	-> NetworkFetchProducer

BitmapMemoryCacheProducer：
在已解码的内存缓存中获取数据；若未找到，则在nextProducer中获取数据，并在获取到数据的同时将其缓存

BitmapMemoryCacheKeyMultiplexProducer：
是MultiplexProducer的子类，nextProducer为BitmapMemoryCacheProducer，将多个拥有相同已解码内存缓存键的ImageRequest进行“合并”，若缓存命中，它们都会获取到该数据

PostprocessedBitmapMemoryCacheProducer：
在已解码的内存缓存中寻找PostProcessor处理过的图片。它的nextProducer都是PostProcessorProducer，因为如果没有获取到被PostProcess的缓存，就需要对获取的图片进行PostProcess。；若未找到，则在nextProducer中获取数据

EncodedMemoryCacheProducer：
在未解码的内存缓存中寻找数据，如果找到则返回，使用结束后释放资源；若未找到，则在nextProducer中获取数据，并在获取到数据的同时将其缓存

DiskCacheProducer：
在文件内存缓存中获取数据；若未找到，则在nextProducer中获取数据，并在获取到数据的同时将其缓存到disk cache中

MultiplexProducer：
将多个拥有相同CacheKey的ImageRequest进行“合并”，让他们从都从nextProducer中获取数据

ThreadHandoffProducer：
将nextProducer的produceResult方法放在后台线程中执行（线程池容量为1）

SwallowResultProducer:
将nextProducer的获取的数据“吞”掉，回在Consumer的onNewResult中传入null值

ResizeAndRotateProducer:
将nextProducer产生的EncodedImage根据EXIF的旋转、缩放属性进行变换（如果对象不是JPEG格式图像，则不会发生变换）

PostProcessorProducer:
将nextProducer产生的EncodedImage根据PostProcessor进行修改，关于PostProcessor详见修改图片；

DecodeProducer：
将nextProducer产生的EncodedImage解码。解码在后台线程中执行，可以在ImagePipelineConfig中通过setExecutorSupplier来设置线程池数量，默认为最大可用的处理器数；

WebpTranscodeProducer：
若nextProducer产生的EncodedImage为WebP格式，则将其解码成DecodeProducer能够处理的EncodedImage。解码在后代进程中进行。

```

从主要环节代码流程就可以看到，Producer是一种从上到下的层次结构，类似于责任链模式。图片请求过程从上层Producer逐层向下，每层Producer无法处理结果就把请求丢到下层Producer，即其内部的mInputProducer。到达某一层Producer拿到结果后，就通过内部成员Consumer将结果回调给外部。Consumer回调的顺序和Producer相反，从下到上。

Producer上下层级接口可参考"整体架构图"。



Producer流的入口在ImagePipeline.fetchDecodedImage()

```
ImagePipeline.class

public DataSource<CloseableReference<CloseableImage>> fetchDecodedImage(
      ImageRequest imageRequest,
      Object callerContext,
      ImageRequest.RequestLevel lowestPermittedRequestLevelOnSubmit) {
    try {
    	// 初次使用Producer
      Producer<CloseableReference<CloseableImage>> producerSequence =
          mProducerSequenceFactory.getDecodedImageProducerSequence(imageRequest);
          
      return submitFetchRequest(
          producerSequence,
          imageRequest,
          lowestPermittedRequestLevelOnSubmit,
          callerContext);
    } catch (Exception exception) {
      return DataSources.immediateFailedDataSource(exception);
    }
  }
```

Producer的创建都在ProducerSequenceFactory。每创建一个Producer时，都会优先创建其下层Producer作为构造参数，从而形成从上到下的链式结构。

ProducerSequenceFactory.class

```
/**
   * swallow result if prefetch -> bitmap cache get ->
   * background thread hand-off -> multiplex -> bitmap cache -> decode -> multiplex ->
   * encoded cache -> disk cache -> (webp transcode) -> network fetch.
   */
  private synchronized Producer<CloseableReference<CloseableImage>> getNetworkFetchSequence() {
    if (mNetworkFetchSequence == null) {
      mNetworkFetchSequence =
          newBitmapCacheGetToDecodeSequence(getCommonNetworkFetchToEncodedMemorySequence());
    }
    return mNetworkFetchSequence;
  }
  
   /**
   * multiplex -> encoded cache -> disk cache -> (webp transcode) -> network fetch.
   */
  private synchronized Producer<EncodedImage> getCommonNetworkFetchToEncodedMemorySequence() {
    if (mCommonNetworkFetchToEncodedMemorySequence == null) {
      Producer<EncodedImage> inputProducer =
          newEncodedCacheMultiplexToTranscodeSequence(
              mProducerFactory.newNetworkFetchProducer(mNetworkFetcher));
      mCommonNetworkFetchToEncodedMemorySequence =
          ProducerFactory.newAddImageTransformMetaDataProducer(inputProducer);

      mCommonNetworkFetchToEncodedMemorySequence =
          mProducerFactory.newResizeAndRotateProducer(
              mCommonNetworkFetchToEncodedMemorySequence,
              mResizeAndRotateEnabledForNetwork,
              mUseDownsamplingRatio);
    }
    return mCommonNetworkFetchToEncodedMemorySequence;
  }
```



##### NetworkFetcher

```
网络请求实现类，只在NetworkFetchProducer中使用。

继承关系：
-> NetworkFetcher (I)
	-> BaseNetworkFetcher
		-> HttpUrlConnectionNetworkFetcher (default)
		-> OkHttpNetworkFetcher
```




### 缓存结构
三级缓存：Bitmap Cache,  Encoded Memory Cache , Disk Cache

![freso_cache_structure](http://ww1.sinaimg.cn/large/006tNc79ly1g4tdgrd96fj30u013sakh.jpg)



##### 1. 内存缓存

```
**1. Bitmap缓存**
Bitmap缓存存储Bitmap对象，这些Bitmap对象可以立刻用来显示或者用于后处理

在5.0以下系统，Bitmap缓存位于Ashmem，这样Bitmap对象的创建和释放将不会引发GC，更少的GC会使你的APP运行得更加流畅。在5.0以下，频繁GC将会显著地引发界面卡顿。

Ashmem存储区域：它是一个不在Java堆区的一片存储内存空间，它的管理由Linux内核驱动管理，不必深究，只要知道这块存储区域是别于堆内存之外的一块空间就行了，且这块空间是可以多进程共享的，GC的活动不会影响到它。

5.0及其以上系统，相比之下，内存管理有了很大改进，所以Bitmap缓存直接位于Java的heap上。

当应用在后台运行时，该内存会被清空。

**2. 未解码图片的内存缓存**
这个缓存存储的是原始压缩格式的图片。从该缓存取到的图片在使用之前，需要先进行解码。

如果有调整大小，旋转，或者WebP编码转换工作需要完成，这些工作会在解码之前进行。

**3. 磁盘缓存**
和未解码的内存缓存相似，磁盘缓存存储的是未解码的原始压缩格式的图片，在使用之前同样需要经过解码等处理。

和Bitmap缓存不一样，APP在后台时，内容是不会被清空的。即使关机也不会。用户可以随时用系统的设置菜单中进行清空缓存操作。
```



其中前两个就是内存缓存，Bitmap缓存根据系统版本不同放在了不同内存区域中，而未解码图片的缓存只在堆内存中，Fresco分了两步做内存缓存，这样做有什么好处呢？好处是加快图片的加载速度。

Fresco的加载图片的流程为：
1）查找Bitmap缓存中是否存在，存在则直接返回Bitmap直接使用；
2）不存在则查找未解码图片的缓存，如果存在则进行Decode成Bitmap然后直接使用并加入Bitmap缓存中；
3）如果未解码图片缓存中查找不到，则进行硬盘缓存的检查，如有，则进行IO、转化、解码等一系列操作，最后成Bitmap供我们直接使用，并把未解码（Encode）的图片加入未解码图片缓存，把Bitmap加入Bitmap缓存中，如硬盘缓存中没有，则进行Network操作下载图片，然后加入到各个缓存中。

既然Fresco使用了三级缓存，而有两级是内存缓存，所以当我们的App在后台时或者在内存低的情况下在onLowMemory()方法中，我们应该手动清除应用的内存缓存，我们可以使用下面的方式：

```
ImagePipeline imagePipeline = Fresco.getImagePipeline();
//清空内存缓存（包括Bitmap缓存和未解码图片的缓存）
imagePipeline.clearMemoryCaches();
//清空硬盘缓存，一般在设置界面供用户手动清理
imagePipeline.clearDiskCaches();

//同时清理内存缓存和硬盘缓存
imagePipeline.clearCaches();
```

##### 2. 未解码内存缓存

##### 3.硬盘缓存

### 视图结构
```
DraweeView			--	View
DraweeController	-- Controller
DraweeHierarchy		-- Model
```

作用：
DraweeView用来显示顶层视图（getTopLevelDrawable()）。
DraweeController控制加载图片的配置、顶层显示哪个视图以及控制事件的分发。 
DraweeHierarchy意为视图的层次结构，用来存储和描述图片的信息，同时也封装了一些图片的显示和视图层级的方法。

DraweeHolder：

DraweeHolder是协调DraweeView、DraweeHierarchy、DraweeController这三个类交互工作的核心类。



### 项目结构

```
demo
	compile project(':drawee-backends:drawee-pipeline')

drawee-pipeline (包含在drawee-backends目录中，同级还有drawee-volly)
	compile project(':drawee')
    compile project(':fbcore')
    compile project(':imagepipeline')
    
drawee
	compile project(':fbcore')
	
imagepipeline
	compile project(':fbcore')
	
fbcore
	最底层module，提供file, log, datasource, memory等基础功能
    
```



###图片编解码

### 其他问题

##### 1. 前后台切换或页面转场动画时，DraweeView闪烁

原因是，Fresco为了内存考虑，当页面不可见时，会释放图片资源，页面可见时再重新加载。官方文档已经做了解释，[参见](https://frescolib.org/docs/writing-custom-views.html)

```
There is no point in images staying in memory when Android is no longer displaying the view - it may have scrolled off-screen, or otherwise not be drawing. Drawees listen for detaches and release memory when they occur. They will automatically restore the image when it comes back on-screen.
```

看下页面visibility变化时，代码流程。

DraweeHolder.class

```
/**
   * Callback used to notify about top-level-drawable's visibility changes.
   */
  @Override
  public void onVisibilityChange(boolean isVisible) {
    ...
    attachOrDetachController();
  }
  
  private void attachOrDetachController() {
    if (mIsHolderAttached && mIsVisible) {
      attachController();
    } else {
      detachController();
    }
  }
```



```
AbstractDraweeController.class

@Override
  public void release() {
    mEventTracker.recordEvent(Event.ON_RELEASE_CONTROLLER);
    if (mRetryManager != null) {
      mRetryManager.reset();
    }
    if (mGestureDetector != null) {
      mGestureDetector.reset();
    }
    if (mSettableDraweeHierarchy != null) {
      mSettableDraweeHierarchy.reset();
    }
    releaseFetch();
  }
  
```

  GenericDraweeHierarchy.class

```
 @Override
  public void reset() {
    resetActualImages();
    resetFade();
  }
  
  private void resetActualImages() {
    mActualImageWrapper.setDrawable(mEmptyActualImageDrawable);
  }
```





### 待研究

1. 下面三个Drawable是什么区别？

   ```
   GenericDraweeHierarchy.class {
       private final RootDrawable mTopLevelDrawable;
       private final FadeDrawable mFadeDrawable;
       private final ForwardingDrawable mActualImageWrapper;
   }
   ```

   

###参考

* [Fresco Github](https://github.com/facebook/fresco)

* [Fresco官方文档](https://www.fresco-cn.org/)


* [掘金-Android开源框架源码鉴赏：Fresco](https://juejin.im/post/5a7568825188257a7a2d9ddb#heading-3)













