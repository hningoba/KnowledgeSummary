## 整体结构

把fresco拆分成展示层和图片加载层来理解。

展示层主要包含DraweeView、DraweeHolder、DraweeController和DraweeHierachy。四个模块的持有关系就是箭头的指向，主要维护View相关内容和多层Drawable。比如解析XML中相关配置属性、根据图片加载状态显示对应Drawable等。

图片展示层是由ImagePipeline发起，核心是一系列的producer。Producer的执行顺序就是下图中从上到下的顺序，即先读内存缓存、切换线程、图片解码、读编码缓存、发起网络等等。每个Producer内部都有自己的Consumer，用来接收下层Producer的处理数据并做当前层相应的工作。Consumer层次间的回调顺序则是和其Producer反向。

后面会分别对展示层、加载流程、Producer sequence、缓存架构等进行细节理解。

<img src="https://raw.githubusercontent.com/hningoba/KnowledgeSummary/master/开源库源码分析/Fresco系列/img/Fresco总体结构.png"/>





## 展示层

**DraweeView：**

继承自ImageView，是SimpleDraweeView的基类，主要是做为DraweeHierachy的展示层。持有DraweeHolder实例。

实现类GenericDraweeView在初始化过程中``GenericDraweeView.inflateHierarchy()``将我们在XML文件配置的styleable inflate出来，比如placeholderImage、fadeDuration、viewAspectRatio等等，具体各种效果配置可以参考[官网](https://www.fresco-cn.org/docs/drawee-branches.html)。styleable具体解析在``GenericDraweeHierarchyInflater.updateBuilder()``。GenericDraweeHierarchyBuilder构建出的DraweeHierachy实例会传给DraweeHolder和DraweeController持有。

看下最常用的SimpleDraweeView.setImageURI()方法。

```
SimpleDraweeView.java

public void setImageURI(Uri uri, @Nullable Object callerContext) {
    DraweeController controller =
        mControllerBuilder
            .setCallerContext(callerContext)
            .setUri(uri)
            .setOldController(getController())
            .build();
		// 实际执行基类DraweeView的setController()，看下面代码块
    setController(controller);
  }
```

```
DraweeView.java

public void setController(@Nullable DraweeController draweeController) {
	// DraweeView持有DraweeHolder实例
  mDraweeHolder.setController(draweeController);
  // DraweeView展示的是DraweeHolder中的TopLevelDrawable，但Drawable本质是DraweeHierachy维护的，看下面代码块
  super.setImageDrawable(mDraweeHolder.getTopLevelDrawable());
}
```

```
DraweeHolder.java

// DraweeHolder持有DraeeHierachy的实例，返回给DraweeView的TopLevelDrawable是DraeeHierachy维护的
public Drawable getTopLevelDrawable() {
    return mHierarchy == null ? null : mHierarchy.getTopLevelDrawable();
  }
```

```
GenericDraweeHierachy.java

 @Override
  public Drawable getTopLevelDrawable() {
    return mTopLevelDrawable;
  }
```

所以从上面的代码流程可以看出，是DraweeView -> DraweeHolder -> DraweeHierachy的调用关系。

**DraweeHolder:** 

做为DraweeView和DraweeHierarchy的纽带，内部持有DraweeHierarchy和DraweeController的实例。

同时维护着attach/detach逻辑，以及控制发起、取消ImagePipeline图片请求过程(具体图片请求逻辑后面主流程代码会讲到)。比如，当DraweeView attachToWindow()时，会执行下面代码

```
DraweeHolder.java

private void attachController() {
		...
    if (mController != null &&
        mController.getHierarchy() != null) {
      mController.onAttach();
    }
  }
```

```
AbstractDraweeController.java

@Override
  public void onAttach() {
		...
    mIsAttached = true;
    if (!mIsRequestSubmitted) {
      submitRequest();
    }
  }
  
  protected void submitRequest() {
    final T closeableImage = getCachedImage();
    // ImagePipeline.mBitmapMemoryCache中已经有当前请求的图片，则直接返回。比如前后台切换等场景，会涉及attach/detach，没必要再从三级缓存拿图片
    if (closeableImage != null) {
      ...
      onImageLoadedFromCacheImmediately(mId, closeableImage);
      onNewResultInternal(mId, mDataSource, closeableImage, 1.0f, true, true);
      return;
    }
    ...

		// 获取DataSource，内部执行PipelineDraweeControllerBuilder.getDataSourceForRequest()，进而执行ImagePipeline.fetchDecodeImage()，开启非常重要的三级缓存Producer sequence过程
    mDataSource = getDataSource();
    
    final String id = mId;
    final boolean wasImmediate = mDataSource.hasResult();
    final DataSubscriber<T> dataSubscriber =
        new BaseDataSubscriber<T>() {
          ...
        };
    // 状态订阅
    mDataSource.subscribe(dataSubscriber, mUiThreadImmediateExecutor);
  }
```

**DraweeHierarchy：**

具体实现在GenericDraweeHierarchy，内部组合了多层Drawable。从下面变量名称就可以看到具体Drawable层级内容。细节内容可以看下GenericDraweeHierarchy这个类。

```
GenericDraweeHierarchy.java

private static final int BACKGROUND_IMAGE_INDEX = 0;
private static final int PLACEHOLDER_IMAGE_INDEX = 1;
private static final int ACTUAL_IMAGE_INDEX = 2;
private static final int PROGRESS_BAR_IMAGE_INDEX = 3;
private static final int RETRY_IMAGE_INDEX = 4;
private static final int FAILURE_IMAGE_INDEX = 5;
private static final int OVERLAY_IMAGES_INDEX = 6;
```

**DraweeController:**

响应DraweeView，通过ImagePipeline向producer sequence发起图片加载流程，代码在上面介绍DraweeHolder时也提到了。同时根据各种状态来控制不同layer的Drawable的展示。内部持有DraweeHierachy实例。

下面展示一些根据图片加载状态，通过DraweeHierachy修改图层内容的示例。

```
AbstractDraweeController.java

private void onProgressUpdateInternal(
      String id,
      DataSource<T> dataSource,
      float progress,
      boolean isFinished) {
      // 根据图片加载进度，刷新DraweeHierachy.PROGRESS_BAR_IMAGE_INDEX层Drawable
      mSettableDraweeHierarchy.setProgress(progress, false);
}

private void onFailureInternal(
      String id,
      DataSource<T> dataSource,
      Throwable throwable,
      boolean isFinished) {
      // 图片加载失败，Retry和Failure图层的处理
      // Set the previously available image if available.
      if (mRetainImageOnFailure && mDrawable != null) {
        mSettableDraweeHierarchy.setImage(mDrawable, 1f, true);
      } else if (shouldRetryOnTap()) {
        mSettableDraweeHierarchy.setRetry(throwable);
      } else {
        mSettableDraweeHierarchy.setFailure(throwable);
      }
}
```

再简单回顾下：

* DraweeView继承自ImageView，承载View的工作，持有DraweeHolder实例。
* DraweeHolder作为DraweeView和DraweeHierachy的纽带，持有DraweeController和DraweeHierachy实例。
* DraweeHierachy管理Drawable层级，提供各种状态的Drawable。
* DraweeController主要负责逻辑控制，并根据图片加载状态更新DraweeHierachy的Drawable显示内容。

具体的持有关系看下图，持有关系就是箭头的指向。

<img src="https://raw.githubusercontent.com/hningoba/KnowledgeSummary/master/开源库源码分析/Fresco系列/img/fresco-view.png"/>



## 图片加载流程概述

上面描述了fresco和开发接触最多的view层内容，view层之下是图片加载实现层。其实官网中对图片加载流程已经描述的很详细，我们先看下文字流程，对图片加载逻辑有个直观感受，再看后面的代码流程。

Fresco 中设计有一个叫做 Image Pipeline 的模块。它负责从网络，从本地文件系统，本地资源加载图片。为了最大限度节省空间和CPU时间，它含有3级缓存设计（2级内存，1级磁盘），分别是Bitmap Memory Cache、Encoded Memory Cache、Disk Cache。

大致流程如下:

- 检查内存缓存，如有，返回
- 后台线程开始后续工作
- 检查是否在未解码内存缓存中。如有，解码，变换，返回，然后缓存到内存缓存中。
- 检查是否在磁盘缓存中，如果有，变换，返回。缓存到未解码缓存和内存缓存中。
- 从网络或者本地加载。加载完成后，解码，变换，返回。存到各个缓存中。

下图非常直观的展示了整个加载流程以及线程模块。
![](https://www.fresco-cn.org/static/imagepipeline.png)



## 主体代码流程

有了上面的描述，我们跟下图片请求的主体代码流程，包括发起加载网络图片（http schema）、三层内存处理、最终使用网络请求图片。

从最常用的请求图片接口SimpleDraweeView.setImageUri()开始，忽略不重要的代码分支。代码执行顺序就是下面代码块从上到下的顺序。

```
------------- 初始化PipelineDraweeController -------------
-> AbstractDraweeControllerBuilder.build()
-> AbstractDraweeControllerBuilder.buildController()
-> PipelineDraweeControllerBuilder.obtainController()

------------- 获取DataSource<CloseableReference<CloseableImage>> -------------
-> AbstractDraweeControllerBuilder.obtainDataSourceSupplier()
-> AbstractDraweeControllerBuilder.getDataSourceSupplierForRequest()
-> PipelineDraweeControllerBuilder.getDataSourceForRequest()
-> ImagePipeline.fetchDecodedImage() // 获取BitmapMemoryCacheGetProducer实例
-> ImagePipeline.submitFetchRequest()
-> CloseableProducerToDataSourceAdapter.create()
-> AbstractProducerToDataSourceAdapter.construtor() // 构造方法中执行producer.produceResults()

------------- 下面开始执行一系列Producer过程，从内存缓存/未解码内存/磁盘/网络逐级获取图片 ------------

--- 从内存缓存(Bitmap Memory Cache) 获取图片---
-> BitmapMemoryCacheGetProducer.produceResults() // 本质调用父类BitmapMemoryCacheProducer中方法
-> InstrumentedMemoryCache.get() // 只是做了一层封装，真正获取逻辑在CountingMemoryCache
-> CountingMemoryCache.get()
-> CountingLruCache.get() // 用LruCache做了一层内存缓存

--- 内存缓存没有命中，准备从编码内存取图片 ---
-> ThreadHandoffProducer.produceResults() //后续涉及解码、磁盘、网络等操作，切换到非UI线程
-> StatefulRunnable.onSuccess()
-> BitmapMemoryCacheKeyMultiplexProducer.produceResults() // fresco会将多个相同BitmapCacheKey键的请求合并为一个请求，键结构是Pair<BitmapMemoryCacheKey, ImageRequest>
-> MultiplexProducer.Multiplexer.startInputProducerIfHasAttachedConsumers()
-> BitmapMemoryCacheProducer.produceResults()
-> DecodeProducer.produceResults() //封装了JPG渐进式解码逻辑
-> ResizeAndRotateProducer.produceResults() //对图片进行尺寸调整和旋转，比如通过ImageRequestBuilder.setResizeOptions()修改图片尺寸，实现逻辑就在这一层
-> AddImageTransformMetaDataProducer.produceResults() // 获取图片MetaData，传给上层Producer
-> EncodedCacheKeyMultiplexProducer.produceResults() // 编码层重复键请求合并，和解码层逻辑类似，都走到父类MultiplexProducer.produceResults()
-> MultiplexProducer.Multiplexer.startInputProducerIfHasAttachedConsumers()

--- 从编码内存缓存(Encoded Bitmap Memory Cache) 获取图片---
-> EncodedMemoryCacheProducer.produceResults() //从编码内存获取图片，封装成EncodedImage给上层

--- 编码内存缓存中没有对应图片，准备从磁盘缓存取图片 --- 
-> DiskCacheReadProducer.produceResults() //获取磁盘缓存，主要做为BufferedDiskCache的代理
-> SmallCacheIfRequestedDiskCachePolicy.createAndStartCacheReadTask()
-> BufferedDiskCache.get()
-> StagingArea.get()
-> BufferedDiskCache.getAsync()
-> BufferedDiskCache.readFromDiskCache() // 读取磁盘文件
-> DiskStorageCache.getResource() // 获取本地资源，如果修改磁盘缓存命中策略，可以在这里处理
-> DefaultDiskStorage.getResource()
-> DiskCacheReadProducer.onFinishDiskReads() // 磁盘缓存命中则通过Consumer返回上层，否则往下走

--- 从网络获取图片 ---
-> DiskCacheReadProducer.onFinishDiskReads() // cachedReference为null，即磁盘没有缓存时，往下走
-> MediaVariationsFallbackProducer.produceResults()
-> MediaVariationsFallbackProducer.startInputProducerWithExistingConsumer()
-> DiskCacheWriteProducer.produceResults() //用于将网络结果写磁盘，BufferDiskCache.put()
-> DiskCacheWriteProducer.maybeStartInputProducer()
-> NetworkFetchProducer.produceResults()
-> HttpUrlConnectionNetworkFetcher.fetch() // 如果网络部分配置成OkHttp，此处走OkHttpNetworkFetcher.fetch()，此处用到的线程池默认coorPool和maxPool都是3
-> HttpUrlConnectionNetworkFetcher.fetchSync() // 网络请求图片
-> HttpUrlConnectionNetworkFetcher.downloadFrom()
-> NetworkFetchProducer.onResponse() // 请求到网络图片对上层回调
```

整个流程就讲完了，虽然代码量很多，但是逻辑一环扣一环还是非常清晰的。再回顾一下主要流程：

1. SimpleDraweeView.setImageUri()发起图片请求
2. 初始化PipelineDraweeController，走到PipelineDraweeControllerBuilder.obtainController()
3. 在2中PipelineDraweeController初始化方法中获取``Supplier<DataSource>``，Supplier接口的get()实现中使用ImagePipeline获取图片，调用ImagePipeline.fetchDecodedImage()
4. 第3步之后，会启动一系列Producer过程，即调用Producer.produceResults()
5. 先从内存缓存BitmapMemoryCacheGetProducer获取图片
6. 内存缓存没有命中后，从编码内存EncodedMemoryCacheProducer中获取图片，这中间包括线程切换、图片resize和rotate、jpe渐进解码、获取图片metadata等逻辑封装
7. 编码内存没有命中，从磁盘DiskCacheReadProducer获取图片
8. 磁盘缓存没有命中，从网络获取图片。网络库默认走HTTPURLConnection，如果配置了OKHTTP，则走OkHttpNetworkFetcher。



## Producer/Consumer

在主体代码流程中就能看到，从三级缓存/网络获取图片以及对上层的回调逻辑中，用到了大量的Producer/Consumer。官网中介绍[ImagePipeline](https://www.fresco-cn.org/docs/intro-image-pipeline.html)的工作流程，其本质就是producer sequence实现的。

Producer: 在image pipeline流程中构建一个处理图片的block。可以处理例如从三级缓存/网络中请求图片、解码、图片转换等。每一个Producer代表一个单独的图片处理任务。返回值类型由其泛型决定。

```
/**
 * Building block for image processing in the image pipeline.
 *
 * <p> Execution of image request consists of multiple different tasks such as network fetch,
 * disk caching, memory caching, decoding, applying transformations etc. Producer<T> represents
 * single task whose result is an instance of T. Breaking entire request into sequence of
 * Producers allows us to construct different requests while reusing the same blocks.
 *
 * <p> Producer supports multiple values and streaming.
 */
public interface Producer<T> {

  void produceResults(Consumer<T> consumer, ProducerContext context);
}
```

Consumer：主要作为对应每一层Producer产生数据的消费方和相关状态回调。

```
/**
 * Consumes data produced by {@link Producer}.<T>
 *
 * <p> The producer uses this interface to notify its client when new data is ready or an error
 * occurs. Execution of the image request is structured as a sequence of Producers. Each one
 * consumes data produced by producer preceding it in the sequence.
 *
 * <p>For example decode is a producer that consumes data produced by the disk cache get producer.
 *
 * <p> The consumer is passed new intermediate results via onNewResult(isLast = false) method. Each
 * consumer should expect that one of the following methods will be called exactly once, as the very
 * last producer call:
 * <ul>
 *   <li> onNewResult(isLast = true) if producer finishes successfully with a final result </li>
 *   <li> onFailure if producer failed to produce a final result </li>
 *   <li> onCancellation if producer was cancelled before a final result could be created </li>
 * </ul>
 *
 * <p> Implementations of this interface must be thread safe, as callback methods might be called
 * on different threads.
 *
 * @param <T>
 */
public interface Consumer<T> {
	void onNewResult(T newResult, @Status int status);
	void onFailure(Throwable t);
	void onCancellation();
	void onProgressUpdate(float progress);
}
```

### Producer责任链逻辑

通过下面例子简单讲下链式Producer和Consumer如何处理数据流的。

从上到下的链式producer中，从上到下依次是ProducerA/ConsumerA、ProducerB/ConsumerB、ProducerC/ConsumerC。

```

class ProducerB {
	// @param inputProducer是下层producer，即ProducerC
	public ProducerB(Producer inputProducer) {
		mInputProducer = inputProducer
	}

	// @param consumer 是ProducerA中创建的Consumer对象，即ConsumerA
	// @param producerContext 包含了Listener、ImageRequest等内容
  @Override
  public void produceResults(Consumer consumer, ProducerContext producerContext) {
      val result = process data // producerB处理数据部分，比如从缓存中拿图片
      if (result != null) {
      	// 如果本层Producer能够处理数据，则通过上层传入的Consumer对上回调
      	consumer.onNewResult();
      	return;
      }
      
      // 构建本层的Consumer，wrappedConsumer其实就是ConsumerB
      Consumer wrappedConsumer = wrapConsumer(consumer, cacheKey);
      // 调用下层Producer，让其处理，同时传入本层Consumer
      mInputProducer.produceResults(wrappedConsumer, producerContext);
	}
}
```

简单来说就是，每层Producer都会持有上层Consumer实例和下层Producer实例。本层能处理或不需要下层Producer处理时，通过上层Consumer实例向上返回，否则通过下层Producer实例往下走。即责任链的设计模式。

### 常用Producer解释

部分Producer的作用已经在主体代码流程中提到了，再具体讲下常用的一些Producer的作用。

```
BitmapMemoryCacheProducer：
在已解码的内存缓存中获取数据；若未找到，则在nextProducer中获取数据，并在获取到数据的同时将其缓存。

BitmapMemoryCacheKeyMultiplexProducer：
是MultiplexProducer的子类，nextProducer为BitmapMemoryCacheProducer，将多个拥有相同已解码内存缓存键的ImageRequest进行“合并”，若缓存命中，它们都会获取到该数据

PostprocessedBitmapMemoryCacheProducer：
在已解码的内存缓存中寻找PostProcessor处理过的图片。它的nextProducer都是PostProcessorProducer，因为如果没有获取到被PostProcess的缓存，就需要对获取的图片进行PostProcess。；若未找到，则在nextProducer中获取数据。

EncodedMemoryCacheProducer：
在未解码的内存缓存中寻找数据，如果找到则返回，使用结束后释放资源；若未找到，则在nextProducer中获取数据，并在获取到数据的同时将其缓存

MultiplexProducer：
将多个拥有相同CacheKey的ImageRequest进行“合并”，让他们从都从nextProducer中获取数据

ThreadHandoffProducer：
将nextProducer的produceResult方法放在后台线程中执行。默认用Executors.newFixedThreadPool()构建了一个单线程池，NUM_LIGHTWEIGHT_BACKGROUND_THREADS = 1，将包含后续Producer操作的Runnable放到了这个线程池。默认的几个线程池（IO/Decode/Background/LightWeightBackground）的初始化都在DefaultExecutorSupplier中。

ResizeAndRotateProducer:
将nextProducer产生的EncodedImage根据EXIF的旋转、缩放属性进行变换（如果对象不是JPEG格式图像，则不会发生变换）。通过ImageRequestBuilder.setResizeOptions()修改图片尺寸，实现逻辑就在这一层。

DecodeProducer：
将nextProducer产生的EncodedImage解码。解码在后台线程中执行，可以在ImagePipelineConfig中通过setExecutorSupplier来设置线程池数量，默认为最大可用的处理器数；

WebpTranscodeProducer：
若nextProducer产生的EncodedImage为WebP格式，则将其解码成DecodeProducer能够处理的EncodedImage。解码在后代进程中进行。

DiskCacheProducer：
在文件内存缓存中获取数据；若未找到，则在nextProducer中获取数据，并在获取到数据的同时将其缓存到disk cache中

DiskCacheReadProducer/DiskCacheWriteProducer: 磁盘读写相关。

NetworkFetchProducer: 发起网络请求。
```

### 构造Producer流链式结构

Producer流的入口在ImagePipeline.fetchDecodedImage()

```
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

```
ProducerSequenceFactory.java

public Producer<CloseableReference<CloseableImage>> getDecodedImageProducerSequence(
      ImageRequest imageRequest) {
		...
		//获取默认的Producer
    Producer<CloseableReference<CloseableImage>> pipelineSequence =
        getBasicDecodedImageSequence(imageRequest);

		//配置了自定义的PostProcessor
    if (imageRequest.getPostprocessor() != null) {
      pipelineSequence = getPostprocessorSequence(pipelineSequence);
    }
		...
    return pipelineSequence;
  }


//获取默认的Producer
  private Producer<CloseableReference<CloseableImage>> getBasicDecodedImageSequence(
      ImageRequest imageRequest) {
    ...

      Uri uri = imageRequest.getSourceUri();
      Preconditions.checkNotNull(uri, "Uri is null.");

      switch (imageRequest.getSourceUriType()) {
      	// Http schema的图片链接走到这里
        case SOURCE_TYPE_NETWORK:
          return getNetworkFetchSequence();
        ...
      }
    } finally {
			...
    }
  }
  

// 方法注释中也给到Producer流程
  /**
   * swallow result if prefetch -> bitmap cache get -> background thread hand-off -> multiplex ->
   * bitmap cache -> decode -> multiplex -> encoded cache -> disk cache -> (webp transcode) ->
   * network fetch.
   */
  private synchronized Producer<CloseableReference<CloseableImage>> getNetworkFetchSequence() {
	  ...
    if (mNetworkFetchSequence == null) {
    // 这里主要是组装一系列Producer，形成从上到下的链式结构，细节可以自己跟代码
      mNetworkFetchSequence =
          newBitmapCacheGetToDecodeSequence(getCommonNetworkFetchToEncodedMemorySequence());
    }
    return mNetworkFetchSequence;
  }
```

### 自定义PostProcessor

业务开发中，如果我们想配置一个自定义的图片后处理器，比如对图片做高斯模糊、添加logo等特殊效果，可以通过ImageRequestBuilder.setPostprocessor()配置一个PostProcessor的Producer，这是最上层的Producer，从内存缓存中获取图片后会执行到这个Producer。

代码调用流程，在ImagePipeline.submitFetchRequest()之前，先通过ProducerSequenceFactory.getDecodedImageProducerSequence()获取sequence最顶层的Producer。

```
val imageRequest = ImageRequestBuilder
	.newBuilderWithSource(mUri)
	.postprocessor(yourProcessor)
	.build()
	
SimpleDraweeView.getControllerBuilder()
	.setImageRequest(imageRequest)
```



## 缓存架构

### 内存缓存

内存缓存，即Bitmap Memory Cache，本质是BitmapMemoryCacheProducer实现的，看下代码：

```
public class BitmapMemoryCacheProducer implements Producer<CloseableReference<CloseableImage>> {

  @Override
  public void produceResults(
      final Consumer<CloseableReference<CloseableImage>> consumer,
      final ProducerContext producerContext) {

    // 1. 从内存缓存拿图片，mMemoryCache是InstrumentedMemoryCache实例，但其只是封装了统计相关内存，正在实现在CountingMemoryCache
    CloseableReference<CloseableImage> cachedReference = mMemoryCache.get(cacheKey);

	// 2. 拿到缓存，通过上层Consumer回调出去
    if (cachedReference != null) {
      ...
        consumer.onProgressUpdate(1f);
      }
      consumer.onNewResult(cachedReference, BaseConsumer.simpleStatusForIsLast(isFinal));
      cachedReference.close();
      if (isFinal) {
        return;
      }
    }

    // 3.封装本层Producer的Consumer
    Consumer<CloseableReference<CloseableImage>> wrappedConsumer = wrapConsumer(consumer, cacheKey);
    
    // 4.没有命中内存缓存，则交给下层Producer处理
    mInputProducer.produceResults(wrappedConsumer, producerContext);
  }

	// 构建本层Consumer
  protected Consumer<CloseableReference<CloseableImage>> wrapConsumer(
      final Consumer<CloseableReference<CloseableImage>> consumer,
      final CacheKey cacheKey) {
      
    return new DelegatingConsumer(consumer) {
      @Override
      public void onNewResultImpl(
          CloseableReference<CloseableImage> newResult,
          @Status int status) {
        ...
        
        // 5.将下层Producer处理好的图片存到内存缓存
        CloseableReference<CloseableImage> newCachedResult =
            mMemoryCache.cache(cacheKey, newResult);
            
        // 6.通过上层Consumer回调
        try {
          if (isLast) {
            getConsumer().onProgressUpdate(1f);
          }
          getConsumer().onNewResult(
              (newCachedResult != null) ? newCachedResult : newResult, status);
        } finally {
          CloseableReference.closeSafely(newCachedResult);
        }
      }
    };
  }
}
```

从内存缓存中取图片的实现逻辑在CountingMemoryCache。

```
CountingMemoryCache.java

public CloseableReference<V> get(final K key) {
    Preconditions.checkNotNull(key);
    Entry<K, V> oldExclusive;
    CloseableReference<V> clientRef = null;
    synchronized (this) {
    	// mExclusiveEntries保存没有被使用的图片，如果这里有，先移走
      oldExclusive = mExclusiveEntries.remove(key);
      // 获取图片的逻辑，下面会讲到
      Entry<K, V> entry = mCachedEntries.get(key);
      if (entry != null) {
        clientRef = newClientReference(entry);
      }
    }
    maybeNotifyExclusiveEntryRemoval(oldExclusive);
    maybeUpdateCacheParams();
    maybeEvictEntries();
    return clientRef;
  }
  
  // 将缓存结果封装成CloseableReference
  private synchronized CloseableReference<V> newClientReference(final Entry<K, V> entry) {
    increaseClientCount(entry);
    return CloseableReference.of(
        entry.valueRef.get(),
        new ResourceReleaser<V>() {
          @Override
          public void release(V unused) {
            releaseClientReference(entry);
          }
        });
  }
```

从mCachedEntries中获取到缓存后，用CloseableReference做了一层封装，主要是为了管理Bitmap内存，设计逻辑不复杂，但挺有意思，这一块后面单独会讲。

看下``mCachedEntries.get(key)``，其实是一个LRUCache。

```
CountingLruMap.java

private final LinkedHashMap<K, V> mMap = new LinkedHashMap<>();

public synchronized V get(K key) {
    return mMap.get(key);
  }
```

大家都知道fresco内存缓存使用的LruCache策略，到这里，也就有了解释。

### 编码内存缓存

编码内存缓存（Encoded Memory Cache）的逻辑和Bitmap Memory Cache类似，区别只是返回的图片包装类型是CloseableReference<PooledByteBuffer>，而前者是CloseableReference<CloseableImage>。

```
EncodedMemoryCacheProducer.java

  @Override
  public void produceResults(
      final Consumer<EncodedImage> consumer,
      final ProducerContext producerContext) {
      ...
  	//获取缓存
	  CloseableReference<PooledByteBuffer> cachedReference = mMemoryCache.get(cacheKey);
	  if (cachedReference != null) {
	  	//封装成EncodedImage
        EncodedImage cachedEncodedImage = new EncodedImage(cachedReference);
		}
	}
```

### 磁盘缓存

获取磁盘缓存（Disk Cache）的实现在DiskCacheReadProducer中。

```
DiskCacheReadProducer.java

public void produceResults(
      final Consumer<EncodedImage> consumer,
      final ProducerContext producerContext) {
      
      // 获取磁盘缓存，preferredCache默认实现是BufferdDiskCache
      final Task<EncodedImage> diskLookupTask = preferredCache.get(cacheKey, isCancelled);
}
```

BufferdDiskCache的读操作中，优先从disk cache的staging area中读取，没读到，切换到background thread，从磁盘里读：

```
BufferdDiskCache.java

public Task<EncodedImage> get(CacheKey key, AtomicBoolean isCancelled) {
  final EncodedImage pinnedImage = mStagingArea.get(key);
  if (pinnedImage != null) {
    return foundPinnedImage(key, pinnedImage);
  }
  // 切换到background thread
  return getAsync(key, isCancelled);
}

private Task<EncodedImage> getAsync(final CacheKey key, final AtomicBoolean isCancelled) {
    try {
      return Task.call(
          new Callable<EncodedImage>() {
            @Override
            public EncodedImage call()
                throws Exception {
              ...

                try {
	                //从磁盘读
                  final PooledByteBuffer buffer = readFromDiskCache(key);
                  CloseableReference<PooledByteBuffer> ref = CloseableReference.of(buffer);
                  try {
                    result = new EncodedImage(ref);
                  } finally {
                    CloseableReference.closeSafely(ref);
                  }
                } catch (Exception exception) {
                  return null;
                }
              }

             ...
            }
          },
          mReadExecutor);
    } catch (Exception exception) {
      ...
      return Task.forError(exception);
    }
  }
  
  // 读磁盘
  private PooledByteBuffer readFromDiskCache(final CacheKey key) throws IOException {
		// 读磁盘具体实现
      final BinaryResource diskCacheResource = mFileCache.getResource(key);
      ...
		// 结果转换成PooledByteBuffer
      PooledByteBuffer byteBuffer;
      final InputStream is = diskCacheResource.openStream();
        byteBuffer = mPooledByteBufferFactory.newByteBuffer(is, (int) diskCacheResource.size());
      ...
  }
```

看下``mFileCache.getResource(key)``：

```
DiskStorageCache.java

public BinaryResource getResource(final CacheKey key) {
    String resourceId = null;
    SettableCacheEvent cacheEvent = SettableCacheEvent.obtain()
        .setCacheKey(key);

      synchronized (mLock) {
        BinaryResource resource = null;
        // 将CacheKey转换成resourceId，后面会提到实现逻辑
        List<String> resourceIds = CacheKeyUtil.getResourceIds(key);
        for (int i = 0; i < resourceIds.size(); i++) {
          // 磁盘读文件
          resource = mStorage.getResource(resourceId, key);
        }
        ...
        return resource;
     ...
  }
```

看下获取resourceId的过程``CacheKeyUtil.getResourceIds(key)``：

```
CacheKeyUtil.java

// 将CacheKey转换成ResourceId
public static List<String> getResourceIds(final CacheKey key) {
    final List<String> ids;
    ...
    ids = new ArrayList<>(1);
    ids.add(secureHashKey(key));
    return ids;
  }
  
  // 编码和加密
  private static String secureHashKey(final CacheKey key) throws UnsupportedEncodingException {
    return SecureHashUtil.makeSHA1HashBase64(key.getUriString().getBytes("UTF-8"));
  }
```

```
SecureHashUtil.java

public static String makeSHA1HashBase64(byte[] bytes) {
  try {
    MessageDigest md = MessageDigest.getInstance("SHA-1");
    md.update(bytes, 0, bytes.length);
    byte[] sha1hash = md.digest();
    return Base64.encodeToString(sha1hash, Base64.URL_SAFE | Base64.NO_PADDING | Base64.NO_WRAP);
  } catch (NoSuchAlgorithmException e) {
    throw new RuntimeException(e);
  }
}
```

所以，整体转换逻辑是将CacheKey先utf-8编码，然后在SHA-1加密，再base64编码。

磁盘读取文件``Storage.getResource(resourceId, key)``内部实现主要是组装文件绝对路径的过程，感兴趣可以看下``DynamicDefaultDiskStorage.get()``的实现，这里就不展开了。

以下面一个磁盘文件绝对路径名示例讲下整个路径的组成结构：

``/data/user/0/your_package_name/cache/image_cache/v2.ols100.1/58/YvKmnI_toMgXiCXuQ5XdkEQDv7A.cnt``

* /data/user/0/your_package_name/cache: 应用的内部存储空间cache目录

* image_cache：DiskCacheConfig配置的mBaseDirectoryName

* v2.ols100.1： VersionSubdirectoryName

* 58：根据分区取得subdirectory

* YvKmnI_toMgXiCXuQ5XdkEQDv7A：resourceId

* .cnt：content文件类型后缀



## 自定义Fresco配置

下面提供一段fresco自定义配置，平时开发中可能并不需要那么多自定义项，仅做参考。

```
// 大图磁盘缓冲区配置
        val mainDiskCacheConfig = DiskCacheConfig.newBuilder(context)
            .setBaseDirectoryName("your_fresco_cache")
            .setBaseDirectoryPath(context.applicationContext.cacheDir) // 磁盘缓存目录建议使用内存存储的应用空间内，跟随应用卸载自动移除，对用户友好
            .build()

// 小图磁盘缓冲区配置
        val smallDiskCacheConfig = DiskCacheConfig.newBuilder(context)
            .setBaseDirectoryPath(context.applicationContext.cacheDir)
            .setBaseDirectoryName("your_fresco_small_cache")
            .setMaxCacheSize(20L * Constants.MB)
            .setMaxCacheSizeOnLowDiskSpace(1L * Constants.MB)
            .build()

        val imagePipelineConfigBuilder = ImagePipelineConfig.newBuilder(context)
        imagePipelineConfigBuilder.setBitmapsConfig(Bitmap.Config.RGB_565) //默认ARGB_8888
            .setDownsampleEnabled(true) // 在解码时改变图片的大小，与ResizeOptions配合使用
            .setCacheKeyFactory(YourImageCacheKeyFactory())
	.setFileCacheFactory(YourDiskStorageCacheFactory(YourDynamicDefaultDiskStorageFactory())) // 定制磁盘缓存命中逻辑，比如希望宽高比相同、宽高差距不大的图片认为命中，提高显示效率
            .setNetworkFetcher(YourImageFetcher()) //可以配置自己的网络库
            .setBitmapMemoryCacheParamsSupplier(YourBitmapMemoryCacheParamsSupplier())// 设置内存配置
            .setMainDiskCacheConfig(mainDiskCacheConfig) // 设置主磁盘配置
            .setSmallImageDiskCacheConfig(smallDiskCacheConfig) // 设置小图的磁盘配置
            .setExecutorSupplier(this)
            .build()

        Fresco.initialize(context, imagePipelineConfigBuilder.build())
```





## 可关闭的引用 ClosableReference

在内存缓存部分，从CountingMemoryCache获取到图片后使用CloseableReference进行一层包装，目的是在View不可用状态下，比如切到后台时（View detachFromWindow）释放图片，降低APP内存占用率。从LowMemoryKiller的角度考虑fresco的这种做法，也可以尽量减少当前APP非前台情况下被系统回收的概率。

但是这种做法也有一些缺陷，比如App切到后台，View会detachFromWindow，导致Drawable释放，等APP切回前台时，需要重新attach加载Drawable，能够看到View闪烁的不和谐情况。

 [官网](https://www.fresco-cn.org/docs/closeable-references.html)对ClosableReference也做了一些解释，可以去看看。

ClosableReference内部使用引用计数的方式记录活跃的引用数，当引用数降为0时，ClosableReference就会释放持有的资源。下面通过CountingMemoryCache中使用CloseableReference对缓存图片包装的代码看下引用和释放的逻辑：

```
CountingMemoryCache.java

private synchronized CloseableReference<V> newClientReference(final Entry<K, V> entry) {
  increaseClientCount(entry);
  // 使用CloseableReference.of()方式对引用对象构建一个CloseableReference
  return CloseableReference.of(
      entry.valueRef.get(),
      new ResourceReleaser<V>() {
        @Override
        public void release(V unused) {
          releaseClientReference(entry);
        }
      });
}
```

CloseableReference.of()入参有真正持有的对象t和ResourceReleaser（后面会提到）。

```
CloseableReference.java

public static <T> CloseableReference<T> of(
    @PropagatesNullable T t, ResourceReleaser<T> resourceReleaser) {
  if (t == null) {
    return null;
  } else {
  	// 构造器
    return new CloseableReference<T>(t, resourceReleaser);
  }
}
```

CloseableReference构造方法中实例化了一个SharedReference，这是fresco自己的类，不同于android.content.SharedPreferences。

```
private CloseableReference(T t, ResourceReleaser<T> resourceReleaser) {
    mSharedReference = new SharedReference<T>(t, resourceReleaser);
  }
```

SharedReference构造方法中引用计数mRefCount置为1。对对象多一次持有，引用计数便会加1（addaddReference）。减少持有时(decreaseRefCount())，引用计数便会减1，当引用计数为0时，即``decreaseRefCount() == 0``，引用对象便会释放``mResourceReleaser.release(deleted)``，其中mResourceReleaser是CloseableReference.of()第二个入参。

```
SharedReference.java

public SharedReference(T value, ResourceReleaser<T> resourceReleaser) {
  mValue = Preconditions.checkNotNull(value);
  mResourceReleaser = Preconditions.checkNotNull(resourceReleaser);
  // 默认引用数为1
  mRefCount = 1;
  addLiveReference(value);
}
	
	// 增加持有数
  public synchronized void addReference() {
    ensureValid();
    mRefCount++;
  }
  
  // 减少持有数
  public void deleteReference() {
    if (decreaseRefCount() == 0) {
      T deleted;
      synchronized (this) {
        deleted = mValue;
        mValue = null;
      }
      mResourceReleaser.release(deleted);
      removeLiveReference(deleted);
    }
  }
  
  private synchronized int decreaseRefCount() {
    ensureValid();
    Preconditions.checkArgument(mRefCount > 0);

    mRefCount--;
    return mRefCount;
  }
```

前面提到，App切到后台时，View会执行onDetachFromWindow()，fresco会释放图片，看下具体代码。

```
DraweeHolder.java

// DraweeView的onDetachFromWindow()，后面会执行到下面方法
private void detachController() {
    ...
    if (isControllerValid()) {
      mController.onDetach();
    }
  }
```

看下Controller的onDetach()，

```
AbstractDraweeController.java

@Override
  public void onDetach() {
		...
    mIsAttached = false;
    mDeferredReleaser.scheduleDeferredRelease(this);
  }
```

mDeferredReleaser.scheduleDeferredRelease()最终会执行到下面方法，其中mFetchedImage就是从Producer中获取到的图片。``releaseImage(mFetchedImage)``内部使用CloseableReference减少引用数，当引用数为0时，释放资源。

```
AbstractDraweeController.java

private void releaseFetch() {
    boolean wasRequestSubmitted = mIsRequestSubmitted;
    mIsRequestSubmitted = false;
    mHasFetchFailed = false;
    if (mDataSource != null) {
      mDataSource.close();
      mDataSource = null;
    }
    if (mDrawable != null) {
      releaseDrawable(mDrawable);
    }
    if (mContentDescription != null) {
      mContentDescription = null;
    }
    mDrawable = null;
    if (mFetchedImage != null) {
      logMessageAndImage("release", mFetchedImage);
      // 释放图片
      releaseImage(mFetchedImage);
      mFetchedImage = null;
    }
    if (wasRequestSubmitted) {
      getControllerListener().onRelease(mId);
    }
  }
```

```
PipelineDraweeController.java

@Override
protected void releaseImage(@Nullable CloseableReference<CloseableImage> image) {
  CloseableReference.closeSafely(image);
}
```

```
CloseableReference.java

public static void closeSafely(@Nullable CloseableReference<?> ref) {
  if (ref != null) {
    ref.close();
  }
}

// 关闭当前引用，减少引用数
@Override
  public void close() {
    synchronized (this) {
      if (mIsClosed) {
        return;
      }
      mIsClosed = true;
    }
	// 前面讲SharedReference时提到过deleteReference()，如果引用数为0，则释放图片资源
    mSharedReference.deleteReference();
  }
```

回顾一下：

图片的内存缓存使用CloseableReference为的是更好的管理Bitmap的内存占用，当View不可用时及时释放Bitmap内存，毕竟APP中Bitmap占用的内存才是大户。实现逻辑在SharedReference，内部采用活动引用计数的方式判断对象是否可以释放。



##参考

* [Fresco Github](https://github.com/facebook/fresco)

* [Fresco官方文档](https://www.fresco-cn.org/)









