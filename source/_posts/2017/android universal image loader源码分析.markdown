layout: "post"
title: "android universal image loader源码分析"
date: "2017-01-01 10:21"
---
## android universal image loader 源码分析
  UIL是在android中4大主流图片加载框架之一,它为图片加载提供了一套功能强大,灵活和高可定制性的解决方案,具有非常重要的学习意义,虽然该项目目前已经不维护了,但是很多程序依然在使用着,所以本期重点来学习UIL的内部实现,看看作者的实现思路和我们平时实现的图片加载有什么区别和共同点.

  说道图片加载我们常用的3级缓存,先从内存中找对应url的图片,如果没有找到就从手机存储中找,如果还没找到就从网络上去下载.这里的内存,磁盘,网络称为图片加载3级缓存.我么以前的实现也就是这么个大概流程,唯一需要注意的是从本地磁盘加载和从服务器下载是需要在子线程中工作,否者在主线程中耗时太长会阻塞主线程,导致ANR等等问题.

  UIL的实现流程也是这样的,唯一不用的是对三层的定义通过接口来完成灵活的定制,同时内部通过线程池来统一调度任务,还有一些对图片的Bitmap对象的二次处理等等,这些东西我们在下面都会讲到,总之非常值得学习.

  下面这张图是作者画的UIL的流程图,从图中可以看出和我们上面分析的流程类似
![  UIL流程图](images/2017/01/UIL_Flow.png)

### UIL源码分析
#### ImageLoader

首先因为全局只有一个ImageLoader对象来管理加载,so,肯定是个单例:
```java
private volatile static ImageLoader instance;

public static ImageLoader getInstance() {
		if (instance == null) {
			synchronized (ImageLoader.class) {
				if (instance == null) {
					instance = new ImageLoader();
				}
			}
		}
		return instance;
	}
```
然后把各种配置作为一个对象也就是下面提到的 *ImageLoaderConfiguration* 传给ImageLoader对象来完成配置,这里调用了init方法
```java
public synchronized void init(ImageLoaderConfiguration configuration) {
		if (configuration == null) {
			throw new IllegalArgumentException(ERROR_INIT_CONFIG_WITH_NULL);
		}
		if (this.configuration == null) {
			L.d(LOG_INIT_CONFIG);
			engine = new ImageLoaderEngine(configuration);
			this.configuration = configuration;
		} else {
			L.w(WARNING_RE_INIT_CONFIG);
		}
	}
```
注意这里,通过代码发现如果你要修改配置,直接二次调用init方法是无效的,应该先ImageLoader的destroy方法,然后再重新初始化

>Try to initialize ImageLoader which had already been initialized before. " + "To re-init ImageLoader with new configuration call >ImageLoader.destroy() at first.";

```java
public void destroy() {
		if (configuration != null) L.d(LOG_DESTROY);
		stop();
		configuration.diskCache.close();
		engine = null;
		configuration = null;
	}
```
可以看到这里在destroy的时候把config对象置空了.

##### 核心displayImage方法

上面的init方法初始化了一个 *ImageLoaderEngine* 对象,并把config对象传入了,这里暂时先不讲该对象的实现,先从外部是如何调用ImageLoader显示图片的方法来看起,用的最多的就是 *displayImage* 这个对象.我们常用到的方法是这个:
```java
public void displayImage(String uri, ImageView imageView, DisplayImageOptions options) {
		displayImage(uri, new ImageViewAware(imageView), options, null, null);
	}
```
等一系列的重载方法,最终,这些方法会进入下面的这个 *displayImage* 方法中.

```java
public void displayImage(String uri, ImageAware imageAware, DisplayImageOptions options,
			ImageSize targetSize, ImageLoadingListener listener, ImageLoadingProgressListener progressListener) {
		checkConfiguration();
		if (imageAware == null) {
			throw new IllegalArgumentException(ERROR_WRONG_ARGUMENTS);
		}
		if (listener == null) {
			listener = defaultListener;
		}
		if (options == null) {
			options = configuration.defaultDisplayImageOptions;
		}

		if (TextUtils.isEmpty(uri)) {
			engine.cancelDisplayTaskFor(imageAware);
			listener.onLoadingStarted(uri, imageAware.getWrappedView());
			if (options.shouldShowImageForEmptyUri()) {
				imageAware.setImageDrawable(options.getImageForEmptyUri(configuration.resources));
			} else {
				imageAware.setImageDrawable(null);
			}
			listener.onLoadingComplete(uri, imageAware.getWrappedView(), null);
			return;
		}

		if (targetSize == null) {
			targetSize = ImageSizeUtils.defineTargetSizeForView(imageAware, configuration.getMaxImageSize());
		}
		String memoryCacheKey = MemoryCacheUtils.generateKey(uri, targetSize);
		engine.prepareDisplayTaskFor(imageAware, memoryCacheKey);

		listener.onLoadingStarted(uri, imageAware.getWrappedView());

		Bitmap bmp = configuration.memoryCache.get(memoryCacheKey);
		if (bmp != null && !bmp.isRecycled()) {
			L.d(LOG_LOAD_IMAGE_FROM_MEMORY_CACHE, memoryCacheKey);

			if (options.shouldPostProcess()) {
				ImageLoadingInfo imageLoadingInfo = new ImageLoadingInfo(uri, imageAware, targetSize, memoryCacheKey,
						options, listener, progressListener, engine.getLockForUri(uri));
				ProcessAndDisplayImageTask displayTask = new ProcessAndDisplayImageTask(engine, bmp, imageLoadingInfo,
						defineHandler(options));
				if (options.isSyncLoading()) {
					displayTask.run();
				} else {
					engine.submit(displayTask);
				}
			} else {
				options.getDisplayer().display(bmp, imageAware, LoadedFrom.MEMORY_CACHE);
				listener.onLoadingComplete(uri, imageAware.getWrappedView(), bmp);
			}
		} else {
			if (options.shouldShowImageOnLoading()) {
				imageAware.setImageDrawable(options.getImageOnLoading(configuration.resources));
			} else if (options.isResetViewBeforeLoading()) {
				imageAware.setImageDrawable(null);
			}

			ImageLoadingInfo imageLoadingInfo = new ImageLoadingInfo(uri, imageAware, targetSize, memoryCacheKey,
					options, listener, progressListener, engine.getLockForUri(uri));
			LoadAndDisplayImageTask displayTask = new LoadAndDisplayImageTask(engine, imageLoadingInfo,
					defineHandler(options));
			if (options.isSyncLoading()) {
				displayTask.run();
			} else {
				engine.submit(displayTask);
			}
		}
	}
```
这个方法是ImageLoader的核心方法,所有的显示图片的逻辑都是在这里完成,包括我们上面提到的3级缓存的调用,都是在这里完成初始化然后交给 *ImageLoaderEngine* 来控制完成的.
我们把这个方法分为3个部分:参数检查和加载前listener回调准备,从内存中获取到了想要的Bitmap对象和最后没有从内存中获取到Bitmap的逻辑.

先看第一部分,参数检查和加载前listener回调准备
```java
    checkConfiguration();
		if (imageAware == null) {
			throw new IllegalArgumentException(ERROR_WRONG_ARGUMENTS);
		}
		if (listener == null) {
			listener = defaultListener;
		}
		if (options == null) {
			options = configuration.defaultDisplayImageOptions;
		}

		if (TextUtils.isEmpty(uri)) {
			engine.cancelDisplayTaskFor(imageAware);
			listener.onLoadingStarted(uri, imageAware.getWrappedView());
			if (options.shouldShowImageForEmptyUri()) {
				imageAware.setImageDrawable(options.getImageForEmptyUri(configuration.resources));
			} else {
				imageAware.setImageDrawable(null);
			}
			listener.onLoadingComplete(uri, imageAware.getWrappedView(), null);
			return;
		}

		if (targetSize == null) {
			targetSize = ImageSizeUtils.defineTargetSizeForView(imageAware, configuration.getMaxImageSize());
		}
		String memoryCacheKey = MemoryCacheUtils.generateKey(uri, targetSize);
		engine.prepareDisplayTaskFor(imageAware, memoryCacheKey);

		listener.onLoadingStarted(uri, imageAware.getWrappedView());

```
```java
private void checkConfiguration() {
		if (configuration == null) {
			throw new IllegalStateException(ERROR_NOT_INIT);
		}
	}
```
检查配置对象是否加载成功,如果没有的throw一个异常出去.
这里的 *imageAware* 对象实际上是对加载图片的view的一个包装,下面来看ImageAware的实现
##### ImageAware对象
```java
public interface ImageAware {

	int getWidth();

	ViewScaleType getScaleType();

	View getWrappedView();


	boolean isCollected();


	int getId();


	boolean setImageDrawable(Drawable drawable);


	boolean setImageBitmap(Bitmap bitmap);
}
```
通过代码我们发现 *ImageAware* 居然是一个接口,为什么要这样做呢,是因为不仅仅是ImageView可以加载图片,其他的view也是可以的,所以这里做了一层封装,只要是实现了ImageAware接口的view或者对象都是可以完成加载图片的操作的.

通过ImageAware所在的目录我们找到了下面的 *ImageViewAware* 对象,这个对象就是我们最常用的ImageView的包装对象了,来看它的实现.
```java
public abstract class ViewAware implements ImageAware {

	public static final String WARN_CANT_SET_DRAWABLE = "Can't set a drawable into view. You should call ImageLoader on UI thread for it.";
	public static final String WARN_CANT_SET_BITMAP = "Can't set a bitmap into view. You should call ImageLoader on UI thread for it.";

	protected Reference<View> viewRef;
	protected boolean checkActualViewSize;


	public ViewAware(View view) {
		this(view, true);
	}


	public ViewAware(View view, boolean checkActualViewSize) {
		if (view == null) throw new IllegalArgumentException("view must not be null");

		this.viewRef = new WeakReference<View>(view);
		this.checkActualViewSize = checkActualViewSize;
	}


	@Override
	public int getWidth() {
		View view = viewRef.get();
		if (view != null) {
			final ViewGroup.LayoutParams params = view.getLayoutParams();
			int width = 0;
			if (checkActualViewSize && params != null && params.width != ViewGroup.LayoutParams.WRAP_CONTENT) {
				width = view.getWidth(); // Get actual image width
			}
			if (width <= 0 && params != null) width = params.width; // Get layout width parameter
			return width;
		}
		return 0;
	}


	@Override
	public int getHeight() {
		View view = viewRef.get();
		if (view != null) {
			final ViewGroup.LayoutParams params = view.getLayoutParams();
			int height = 0;
			if (checkActualViewSize && params != null && params.height != ViewGroup.LayoutParams.WRAP_CONTENT) {
				height = view.getHeight(); // Get actual image height
			}
			if (height <= 0 && params != null) height = params.height; // Get layout height parameter
			return height;
		}
		return 0;
	}

	@Override
	public ViewScaleType getScaleType() {
		return ViewScaleType.CROP;
	}

	@Override
	public View getWrappedView() {
		return viewRef.get();
	}

	@Override
	public boolean isCollected() {
		return viewRef.get() == null;
	}

	@Override
	public int getId() {
		View view = viewRef.get();
		return view == null ? super.hashCode() : view.hashCode();
	}

	@Override
	public boolean setImageDrawable(Drawable drawable) {
		if (Looper.myLooper() == Looper.getMainLooper()) {
			View view = viewRef.get();
			if (view != null) {
				setImageDrawableInto(drawable, view);
				return true;
			}
		} else {
			L.w(WARN_CANT_SET_DRAWABLE);
		}
		return false;
	}

	@Override
	public boolean setImageBitmap(Bitmap bitmap) {
		if (Looper.myLooper() == Looper.getMainLooper()) {
			View view = viewRef.get();
			if (view != null) {
				setImageBitmapInto(bitmap, view);
				return true;
			}
		} else {
			L.w(WARN_CANT_SET_BITMAP);
		}
		return false;
	}

	/**
	 * Should set drawable into incoming view. Incoming view is guaranteed not null.<br />
	 * This method is called on UI thread.
	 */
	protected abstract void setImageDrawableInto(Drawable drawable, View view);

	/**
	 * Should set Bitmap into incoming view. Incoming view is guaranteed not null.< br />
	 * This method is called on UI thread.
	 */
	protected abstract void setImageBitmapInto(Bitmap bitmap, View view);
}
```
为什么要对View做包装呢,这里有下面几个作用:

首先是要显示的图片View可能在加载的过程中被回收了或者移除了,这个时候再往下加载就没有意义了,所以这里我们看到通过弱引用来把view对象包装起来了.

```java
public ViewAware(View view, boolean checkActualViewSize) {
		if (view == null) throw new IllegalArgumentException("view must not be null");

		this.viewRef = new WeakReference<View>(view);
		this.checkActualViewSize = checkActualViewSize;
	}
```
还有一个作用就是给view对象附一个唯一的ID,为什么要这么做,这里也要两个原因:一个是为了防止一个view重复的去请求图片,如果一个view在重复的请求图片的时候,ImageLoader会取消掉旧的请求并重新开始新的加载逻辑;二是为了解决异步加载图片错位的问题,我们都知道在android中的listView或者gridView是推荐使用缓存机制的,也就是说只是生成当前屏幕那么多的view对象,然后再滚动的时候来重复利用已经有的对象来节约内存,这就导致一个问题,比如刚开始去异步加载的view在图片加载回来的时候可能已经不是刚开始的那个view了,如果这个时候把图片附上就会产生图片不对应的问题.而这个ID的作用就在这里,通过比较刚开始和后来图片回来的view的唯一ID,如果一致的话就加载图片,不一致就什么都不做,这样就可以解决图片错位的问题了.
下面是具体的ImageView的包装对象的实现:

另外一个作用是判断当前是否是在主线程中,如果在主线程中就可以顺利的加载图片,否者不予执行.防止在子线程中更新UI报错.

最后的一个作用就是可以提前获取到view的宽度和高度,这样可以在加载图片的时候根据view的宽高来对图片进行缩放来节约内存空间.

```java
public class ImageViewAware extends ViewAware {


	public ImageViewAware(ImageView imageView) {
		super(imageView);
	}

	public ImageViewAware(ImageView imageView, boolean checkActualViewSize) {
		super(imageView, checkActualViewSize);
	}

	@Override
	public int getWidth() {
		int width = super.getWidth();
		if (width <= 0) {
			ImageView imageView = (ImageView) viewRef.get();
			if (imageView != null) {
				width = getImageViewFieldValue(imageView, "mMaxWidth"); // Check maxWidth parameter
			}
		}
		return width;
	}


	@Override
	public int getHeight() {
		int height = super.getHeight();
		if (height <= 0) {
			ImageView imageView = (ImageView) viewRef.get();
			if (imageView != null) {
				height = getImageViewFieldValue(imageView, "mMaxHeight"); // Check maxHeight parameter
			}
		}
		return height;
	}

	@Override
	public ViewScaleType getScaleType() {
		ImageView imageView = (ImageView) viewRef.get();
		if (imageView != null) {
			return ViewScaleType.fromImageView(imageView);
		}
		return super.getScaleType();
	}

	@Override
	public ImageView getWrappedView() {
		return (ImageView) super.getWrappedView();
	}

	@Override
	protected void setImageDrawableInto(Drawable drawable, View view) {
		((ImageView) view).setImageDrawable(drawable);
		if (drawable instanceof AnimationDrawable) {
			((AnimationDrawable)drawable).start();
		}
	}

	@Override
	protected void setImageBitmapInto(Bitmap bitmap, View view) {
		((ImageView) view).setImageBitmap(bitmap);
	}

	private static int getImageViewFieldValue(Object object, String fieldName) {
		int value = 0;
		try {
			Field field = ImageView.class.getDeclaredField(fieldName);
			field.setAccessible(true);
			int fieldValue = (Integer) field.get(object);
			if (fieldValue > 0 && fieldValue < Integer.MAX_VALUE) {
				value = fieldValue;
			}
		} catch (Exception e) {
			L.e(e);
		}
		return value;
	}
}
```
ImageViewAware就是ImageView的包装对象了,它可以获取到imageView的宽高和缩放类型,然后调用原声api来完成图片的加载到ImageView的操作.

##### ImageSize对象
```java
public class ImageSize {

	private static final int TO_STRING_MAX_LENGHT = 9; // "9999x9999".length()
	private static final String SEPARATOR = "x";

	private final int width;
	private final int height;

	public ImageSize(int width, int height) {
		this.width = width;
		this.height = height;
	}

	public ImageSize(int width, int height, int rotation) {
		if (rotation % 180 == 0) {
			this.width = width;
			this.height = height;
		} else {
			this.width = height;
			this.height = width;
		}
	}

	public int getWidth() {
		return width;
	}

	public int getHeight() {
		return height;
	}

	/** Scales down dimensions in <b>sampleSize</b> times. Returns new object. */
	public ImageSize scaleDown(int sampleSize) {
		return new ImageSize(width / sampleSize, height / sampleSize);
	}

	/** Scales dimensions according to incoming scale. Returns new object. */
	public ImageSize scale(float scale) {
		return new ImageSize((int) (width * scale), (int) (height * scale));
	}

	@Override
	public String toString() {
		return new StringBuilder(TO_STRING_MAX_LENGHT).append(width).append(SEPARATOR).append(height).toString();
	}
}
```
这里的 *ImageSize* 对象是存储图片的宽高旋转信息的载体对象,为了方便后期对图片做处理的时候使用.这里如果取不到具体的宽高的时候使用config对象中配置的默认宽高:
```java
public static ImageSize defineTargetSizeForView(ImageAware imageAware, ImageSize maxImageSize) {
		int width = imageAware.getWidth();
		if (width <= 0) width = maxImageSize.getWidth();

		int height = imageAware.getHeight();
		if (height <= 0) height = maxImageSize.getHeight();

		return new ImageSize(width, height);
}
```
对url图片为空的处理可能会造成默认图片的显示和正常图片的显示不一样,因为url为空的时候没有走图片处理的流程,比如都要圆角图片,但是没有url的时候是无法实现的,因为没有Bitmap对象可以给它处理

```java
if (TextUtils.isEmpty(uri)) {
			engine.cancelDisplayTaskFor(imageAware);
			listener.onLoadingStarted(uri, imageAware.getWrappedView());
			if (options.shouldShowImageForEmptyUri()) {
				imageAware.setImageDrawable(options.getImageForEmptyUri(configuration.resources));
			} else {
				imageAware.setImageDrawable(null);
			}
			listener.onLoadingComplete(uri, imageAware.getWrappedView(), null);
			return;
		}
```

最后关于内存缓存的key值得生成:
```java
String memoryCacheKey = MemoryCacheUtils.generateKey(uri, targetSize);
engine.prepareDisplayTaskFor(imageAware, memoryCacheKey);
```
```java
public static String generateKey(String imageUri, ImageSize targetSize) {
		return new StringBuilder(imageUri).append(URI_AND_SIZE_SEPARATOR).append(targetSize.getWidth()).append(WIDTH_AND_HEIGHT_SEPARATOR).append(targetSize.getHeight()).toString();
}
```
这里采用的是URL+"\_" + size信息的方式来作为缓存key的.

到这里, *displayImage* 的第一部分就完成了.下面进入那个来看剩下的两部分,也就是从内存中找到和没找到图片对象的处理.
先看如果在内存中找到了想要的图片对象的处理:
```java
Bitmap bmp = configuration.memoryCache.get(memoryCacheKey);
if (bmp != null && !bmp.isRecycled()) {
			L.d(LOG_LOAD_IMAGE_FROM_MEMORY_CACHE, memoryCacheKey);

			if (options.shouldPostProcess()) {
				ImageLoadingInfo imageLoadingInfo = new ImageLoadingInfo(uri, imageAware, targetSize, memoryCacheKey,
						options, listener, progressListener, engine.getLockForUri(uri));
				ProcessAndDisplayImageTask displayTask = new ProcessAndDisplayImageTask(engine, bmp, imageLoadingInfo,
						defineHandler(options));
				if (options.isSyncLoading()) {
					displayTask.run();
				} else {
					engine.submit(displayTask);
				}
			} else {
				options.getDisplayer().display(bmp, imageAware, LoadedFrom.MEMORY_CACHE);
				listener.onLoadingComplete(uri, imageAware.getWrappedView(), bmp);
			}
}
```
首先生成一个 *ImageLoadingInfo对象* 来保存图片加载过程中需要的信息,看下面的代码基本上所有的对象都包含了:

##### ImageLoadingInfo对象
```java
final class ImageLoadingInfo {

	final String uri;
	final String memoryCacheKey;
	final ImageAware imageAware;
	final ImageSize targetSize;
	final DisplayImageOptions options;
	final ImageLoadingListener listener;
	final ImageLoadingProgressListener progressListener;
	final ReentrantLock loadFromUriLock;

	public ImageLoadingInfo(String uri, ImageAware imageAware, ImageSize targetSize, String memoryCacheKey,
			DisplayImageOptions options, ImageLoadingListener listener,
			ImageLoadingProgressListener progressListener, ReentrantLock loadFromUriLock) {
		this.uri = uri;
		this.imageAware = imageAware;
		this.targetSize = targetSize;
		this.options = options;
		this.listener = listener;
		this.progressListener = progressListener;
		this.loadFromUriLock = loadFromUriLock;
		this.memoryCacheKey = memoryCacheKey;
	}
}
```
可以看到如果需要对图片对象做处理的时候就把创建的 *ProcessAndDisplayImageTask* 这个对象交给 *ImageLoaderEngine* 来完成,如果是 *isSyncLoading* 的话就直接运行 *ProcessAndDisplayImageTask* 对象(实际上就是一个线程).
如果不需要处理,就直接显示交给BitmapDisplayer来显示图片:
```java
public interface BitmapDisplayer {
  //This method is called on UI thread so it's strongly recommended not to do any heavy work in it.
	void display(Bitmap bitmap, ImageAware imageAware, LoadedFrom loadedFrom);
}
```
默认使用的这个:
```java
public final class SimpleBitmapDisplayer implements BitmapDisplayer {
	@Override
	public void display(Bitmap bitmap, ImageAware imageAware, LoadedFrom loadedFrom) {
		imageAware.setImageBitmap(bitmap);
	}
}
```
到这里,成功从内存中加载图片的逻辑就完成了,下面在看如果从内存中没有找到对应的图片对象的处理:
```java
else {
			if (options.shouldShowImageOnLoading()) {
				imageAware.setImageDrawable(options.getImageOnLoading(configuration.resources));
			} else if (options.isResetViewBeforeLoading()) {
				imageAware.setImageDrawable(null);
			}

			ImageLoadingInfo imageLoadingInfo = new ImageLoadingInfo(uri, imageAware, targetSize, memoryCacheKey,
					options, listener, progressListener, engine.getLockForUri(uri));
			LoadAndDisplayImageTask displayTask = new LoadAndDisplayImageTask(engine, imageLoadingInfo,
					defineHandler(options));
			if (options.isSyncLoading()) {
				displayTask.run();
			} else {
				engine.submit(displayTask);
			}
		}
```
可以看到这里也是很清晰的,直接创建一个 *LoadAndDisplayImageTask* 对象交给engin来处理.如果是 *isSyncLoading* 的话就直接运行 *LoadAndDisplayImageTask* ,这个对象实际上也是要给线程对象,这个后面我们会讲到.

到这里displayImage的实现就讲完了,可以看到非常的简单,下面开始 *ImageLoderEngine* 的任务调度处理逻辑.

#### ImageLoaderEngine
```java
class ImageLoaderEngine {

	final ImageLoaderConfiguration configuration;

	private Executor taskExecutor;
	private Executor taskExecutorForCachedImages;
	private Executor taskDistributor;

	private final Map<Integer, String> cacheKeysForImageAwares = Collections
			.synchronizedMap(new HashMap<Integer, String>());
	private final Map<String, ReentrantLock> uriLocks = new WeakHashMap<String, ReentrantLock>();

	private final AtomicBoolean paused = new AtomicBoolean(false);
	private final AtomicBoolean networkDenied = new AtomicBoolean(false);
	private final AtomicBoolean slowNetwork = new AtomicBoolean(false);

	private final Object pauseLock = new Object();

	ImageLoaderEngine(ImageLoaderConfiguration configuration) {
		this.configuration = configuration;

		taskExecutor = configuration.taskExecutor;
		taskExecutorForCachedImages = configuration.taskExecutorForCachedImages;

		taskDistributor = DefaultConfigurationFactory.createTaskDistributor();
	}

	/** Submits task to execution pool */
	void submit(final LoadAndDisplayImageTask task) {
		taskDistributor.execute(new Runnable() {
			@Override
			public void run() {
				File image = configuration.diskCache.get(task.getLoadingUri());
				boolean isImageCachedOnDisk = image != null && image.exists();
				initExecutorsIfNeed();
				if (isImageCachedOnDisk) {
					taskExecutorForCachedImages.execute(task);
				} else {
					taskExecutor.execute(task);
				}
			}
		});
	}

	/** Submits task to execution pool */
	void submit(ProcessAndDisplayImageTask task) {
		initExecutorsIfNeed();
		taskExecutorForCachedImages.execute(task);
	}

	private void initExecutorsIfNeed() {
		if (!configuration.customExecutor && ((ExecutorService) taskExecutor).isShutdown()) {
			taskExecutor = createTaskExecutor();
		}
		if (!configuration.customExecutorForCachedImages && ((ExecutorService) taskExecutorForCachedImages)
				.isShutdown()) {
			taskExecutorForCachedImages = createTaskExecutor();
		}
	}

	private Executor createTaskExecutor() {
		return DefaultConfigurationFactory
				.createExecutor(configuration.threadPoolSize, configuration.threadPriority,
				configuration.tasksProcessingType);
	}

	/**
	 * Returns URI of image which is loading at this moment into passed {@link com.nostra13.universalimageloader.core.imageaware.ImageAware}
	 */
	String getLoadingUriForView(ImageAware imageAware) {
		return cacheKeysForImageAwares.get(imageAware.getId());
	}

	/**
	 * Associates <b>memoryCacheKey</b> with <b>imageAware</b>. Then it helps to define image URI is loaded into View at
	 * exact moment.
	 */
	void prepareDisplayTaskFor(ImageAware imageAware, String memoryCacheKey) {
		cacheKeysForImageAwares.put(imageAware.getId(), memoryCacheKey);
	}

	/**
	 * Cancels the task of loading and displaying image for incoming <b>imageAware</b>.
	 *
	 * @param imageAware {@link com.nostra13.universalimageloader.core.imageaware.ImageAware} for which display task
	 *                   will be cancelled
	 */
	void cancelDisplayTaskFor(ImageAware imageAware) {
		cacheKeysForImageAwares.remove(imageAware.getId());
	}

	/**
	 * Denies or allows engine to download images from the network.<br /> <br /> If downloads are denied and if image
	 * isn't cached then {@link ImageLoadingListener#onLoadingFailed(String, View, FailReason)} callback will be fired
	 * with {@link FailReason.FailType#NETWORK_DENIED}
	 *
	 * @param denyNetworkDownloads pass <b>true</b> - to deny engine to download images from the network; <b>false</b> -
	 *                             to allow engine to download images from network.
	 */
	void denyNetworkDownloads(boolean denyNetworkDownloads) {
		networkDenied.set(denyNetworkDownloads);
	}

	/**
	 * Sets option whether ImageLoader will use {@link FlushedInputStream} for network downloads to handle <a
	 * href="http://code.google.com/p/android/issues/detail?id=6066">this known problem</a> or not.
	 *
	 * @param handleSlowNetwork pass <b>true</b> - to use {@link FlushedInputStream} for network downloads; <b>false</b>
	 *                          - otherwise.
	 */
	void handleSlowNetwork(boolean handleSlowNetwork) {
		slowNetwork.set(handleSlowNetwork);
	}

	/**
	 * Pauses engine. All new "load&display" tasks won't be executed until ImageLoader is {@link #resume() resumed}.<br
	 * /> Already running tasks are not paused.
	 */
	void pause() {
		paused.set(true);
	}

	/** Resumes engine work. Paused "load&display" tasks will continue its work. */
	void resume() {
		paused.set(false);
		synchronized (pauseLock) {
			pauseLock.notifyAll();
		}
	}

	/**
	 * Stops engine, cancels all running and scheduled display image tasks. Clears internal data.
	 * <br />
	 * <b>NOTE:</b> This method doesn't shutdown
	 * {@linkplain com.nostra13.universalimageloader.core.ImageLoaderConfiguration.Builder#taskExecutor(java.util.concurrent.Executor)
	 * custom task executors} if you set them.
	 */
	void stop() {
		if (!configuration.customExecutor) {
			((ExecutorService) taskExecutor).shutdownNow();
		}
		if (!configuration.customExecutorForCachedImages) {
			((ExecutorService) taskExecutorForCachedImages).shutdownNow();
		}

		cacheKeysForImageAwares.clear();
		uriLocks.clear();
	}

	void fireCallback(Runnable r) {
		taskDistributor.execute(r);
	}

	ReentrantLock getLockForUri(String uri) {
		ReentrantLock lock = uriLocks.get(uri);
		if (lock == null) {
			lock = new ReentrantLock();
			uriLocks.put(uri, lock);
		}
		return lock;
	}

	AtomicBoolean getPause() {
		return paused;
	}

	Object getPauseLock() {
		return pauseLock;
	}

	boolean isNetworkDenied() {
		return networkDenied.get();
	}

	boolean isSlowNetwork() {
		return slowNetwork.get();
	}
}
```
可以看到在 *ImageLoaderEngine* 对象中主要有3个线程池来做任务调度,核心的方法就是两个 *submit* 方法.
```java
private Executor taskExecutor;
private Executor taskExecutorForCachedImages;
private Executor taskDistributor;
```
*taskDistributor* 用来统一管理所有的任务,根据任务类型把任务放入不同的线程池中,如果是内存中或者本地存储中加载图片的话交给 *taskExecutorForCachedImages* 否者就交给 *taskExecutor* 来处理.

两个submit方法用来把对应类型的task放入不同的线程池中来执行任务,非常的清晰.
```java
/** Submits task to execution pool */
void submit(final LoadAndDisplayImageTask task) {
  taskDistributor.execute(new Runnable() {
    @Override
    public void run() {
      File image = configuration.diskCache.get(task.getLoadingUri());
      boolean isImageCachedOnDisk = image != null && image.exists();
      initExecutorsIfNeed();
      if (isImageCachedOnDisk) {
        taskExecutorForCachedImages.execute(task);
      } else {
        taskExecutor.execute(task);
      }
    }
  });
}

/** Submits task to execution pool */
void submit(ProcessAndDisplayImageTask task) {
  initExecutorsIfNeed();
  taskExecutorForCachedImages.execute(task);
}
```
关于Engine对象还有要注意的地方就是UIL对每个url提供了一个ReentrantLock锁,来解决对重复URL加载的线程同步问题.
```java
ReentrantLock getLockForUri(String uri) {
		ReentrantLock lock = uriLocks.get(uri);
		if (lock == null) {
			lock = new ReentrantLock();
			uriLocks.put(uri, lock);
		}
		return lock;
	}

```
这个锁在后面要讲到的两个Task中会用到,稍后我们再讲他们的具体实现.
Engine对象同时还提供了暂停,停止,开始等操作,通过线程安全的AtomicBoolean对象来控制任务调度状态,这里就不具体讲了,都是线程同步的简单实现.
下面开始讲两个具体的Task线程对象:

#### ProcessAndDisplayImageTask
```java
final class ProcessAndDisplayImageTask implements Runnable {

	private static final String LOG_POSTPROCESS_IMAGE = "PostProcess image before displaying [%s]";

	private final ImageLoaderEngine engine;
	private final Bitmap bitmap;
	private final ImageLoadingInfo imageLoadingInfo;
	private final Handler handler;

	public ProcessAndDisplayImageTask(ImageLoaderEngine engine, Bitmap bitmap, ImageLoadingInfo imageLoadingInfo,
			Handler handler) {
		this.engine = engine;
		this.bitmap = bitmap;
		this.imageLoadingInfo = imageLoadingInfo;
		this.handler = handler;
	}

	@Override
	public void run() {
		L.d(LOG_POSTPROCESS_IMAGE, imageLoadingInfo.memoryCacheKey);

		BitmapProcessor processor = imageLoadingInfo.options.getPostProcessor();
		Bitmap processedBitmap = processor.process(bitmap);
		DisplayBitmapTask displayBitmapTask = new DisplayBitmapTask(processedBitmap, imageLoadingInfo, engine,
				LoadedFrom.MEMORY_CACHE);
		LoadAndDisplayImageTask.runTask(displayBitmapTask, imageLoadingInfo.options.isSyncLoading(), handler, engine);
	}
}
```
可以看到 *ProcessAndDisplayImageTask* 很简单,如果需要对图片做处理的话就交给 *BitmapProcessor* 把图片处理一下,然后把处理完的Bitmap对象封装成一个 *DisplayBitmapTask* 交给 *LoadAndDisplayImageTask* 的 *runTask* 方法来处理.
```java
static void runTask(Runnable r, boolean sync, Handler handler, ImageLoaderEngine engine) {
		if (sync) {
			r.run();
		} else if (handler == null) {
			engine.fireCallback(r);
		} else {
			handler.post(r);
		}
	}

```
可以看到通过具体的情况来完成回调处理.如果是同步的话直接运行该线程,如果handler为空的话交给总调度线程池 *taskDistributor* 来调用,否者通过handler来执行该线程.
```java
final class DisplayBitmapTask implements Runnable {

	private static final String LOG_DISPLAY_IMAGE_IN_IMAGEAWARE = "Display image in ImageAware (loaded from %1$s) [%2$s]";
	private static final String LOG_TASK_CANCELLED_IMAGEAWARE_REUSED = "ImageAware is reused for another image. Task is cancelled. [%s]";
	private static final String LOG_TASK_CANCELLED_IMAGEAWARE_COLLECTED = "ImageAware was collected by GC. Task is cancelled. [%s]";

	private final Bitmap bitmap;
	private final String imageUri;
	private final ImageAware imageAware;
	private final String memoryCacheKey;
	private final BitmapDisplayer displayer;
	private final ImageLoadingListener listener;
	private final ImageLoaderEngine engine;
	private final LoadedFrom loadedFrom;

	public DisplayBitmapTask(Bitmap bitmap, ImageLoadingInfo imageLoadingInfo, ImageLoaderEngine engine,
			LoadedFrom loadedFrom) {
		this.bitmap = bitmap;
		imageUri = imageLoadingInfo.uri;
		imageAware = imageLoadingInfo.imageAware;
		memoryCacheKey = imageLoadingInfo.memoryCacheKey;
		displayer = imageLoadingInfo.options.getDisplayer();
		listener = imageLoadingInfo.listener;
		this.engine = engine;
		this.loadedFrom = loadedFrom;
	}

	@Override
	public void run() {
		if (imageAware.isCollected()) {
			L.d(LOG_TASK_CANCELLED_IMAGEAWARE_COLLECTED, memoryCacheKey);
			listener.onLoadingCancelled(imageUri, imageAware.getWrappedView());
		} else if (isViewWasReused()) {
			L.d(LOG_TASK_CANCELLED_IMAGEAWARE_REUSED, memoryCacheKey);
			listener.onLoadingCancelled(imageUri, imageAware.getWrappedView());
		} else {
			L.d(LOG_DISPLAY_IMAGE_IN_IMAGEAWARE, loadedFrom, memoryCacheKey);
			displayer.display(bitmap, imageAware, loadedFrom);
			engine.cancelDisplayTaskFor(imageAware);
			listener.onLoadingComplete(imageUri, imageAware.getWrappedView(), bitmap);
		}
	}

	/** Checks whether memory cache key (image URI) for current ImageAware is actual */
	private boolean isViewWasReused() {
		String currentCacheKey = engine.getLoadingUriForView(imageAware);
		return !memoryCacheKey.equals(currentCacheKey);
	}
}
```
可以看到 *DisplayBitmapTask* 的run方法的钱两个判断,正好对应前面讲到的关于view的回收或删除和重复利用view错位的问题的判断,如果两种情况对应上就任务本次任务被取消.否则直接通知 *displayer* 显示图片.这里不用担心是在子线程中更新UI,因为 *displayer* 内部的display方法是调用的 *ImageAware* 接口的实现类中 的setImageBitmap方法,在这个方法的具体实现中已经判断了是否是在主线程中了.
```java
  ViewAware对象的setImageView方法

	@Override
	public boolean setImageBitmap(Bitmap bitmap) {
		if (Looper.myLooper() == Looper.getMainLooper()) {
			View view = viewRef.get();
			if (view != null) {
				setImageBitmapInto(bitmap, view);
				return true;
			}
		} else {
			L.w(WARN_CANT_SET_BITMAP);
		}
		return false;
	}
```

#### LoadAndDisplayImageTask

下面进入比较复杂的任务对象 *LoadAndDisplayImageTask* 中,改对象之所以复杂,主要是因为要判断硬盘中是否有需要的图片的对象,如果没有就去下载,然后保存图片,最后才把图片对象返回,代码较多,我们一步步的来分析


```java
final class LoadAndDisplayImageTask implements Runnable, IoUtils.CopyListener {

  ....

	private final ImageLoaderEngine engine;
	private final ImageLoadingInfo imageLoadingInfo;
	private final Handler handler;

	// Helper references
	private final ImageLoaderConfiguration configuration;
	private final ImageDownloader downloader;
	private final ImageDownloader networkDeniedDownloader;
	private final ImageDownloader slowNetworkDownloader;
	private final ImageDecoder decoder;
	final String uri;
	private final String memoryCacheKey;
	final ImageAware imageAware;
	private final ImageSize targetSize;
	final DisplayImageOptions options;
	final ImageLoadingListener listener;
	final ImageLoadingProgressListener progressListener;
	private final boolean syncLoading;

	// State vars
	private LoadedFrom loadedFrom = LoadedFrom.NETWORK;

	public LoadAndDisplayImageTask(ImageLoaderEngine engine, ImageLoadingInfo imageLoadingInfo, Handler handler) {
		this.engine = engine;
		this.imageLoadingInfo = imageLoadingInfo;
		this.handler = handler;

		configuration = engine.configuration;
		downloader = configuration.downloader;
		networkDeniedDownloader = configuration.networkDeniedDownloader;
		slowNetworkDownloader = configuration.slowNetworkDownloader;
		decoder = configuration.decoder;
		uri = imageLoadingInfo.uri;
		memoryCacheKey = imageLoadingInfo.memoryCacheKey;
		imageAware = imageLoadingInfo.imageAware;
		targetSize = imageLoadingInfo.targetSize;
		options = imageLoadingInfo.options;
		listener = imageLoadingInfo.listener;
		progressListener = imageLoadingInfo.progressListener;
		syncLoading = options.isSyncLoading();
	}

	@Override
	public void run() {
		if (waitIfPaused()) return;
		if (delayIfNeed()) return;

		ReentrantLock loadFromUriLock = imageLoadingInfo.loadFromUriLock;
		L.d(LOG_START_DISPLAY_IMAGE_TASK, memoryCacheKey);
		if (loadFromUriLock.isLocked()) {
			L.d(LOG_WAITING_FOR_IMAGE_LOADED, memoryCacheKey);
		}

		loadFromUriLock.lock();
		Bitmap bmp;
		try {
			checkTaskNotActual();

			bmp = configuration.memoryCache.get(memoryCacheKey);
			if (bmp == null || bmp.isRecycled()) {
				bmp = tryLoadBitmap();
				if (bmp == null) return; // listener callback already was fired

				checkTaskNotActual();
				checkTaskInterrupted();

				if (options.shouldPreProcess()) {
					L.d(LOG_PREPROCESS_IMAGE, memoryCacheKey);
					bmp = options.getPreProcessor().process(bmp);
					if (bmp == null) {
						L.e(ERROR_PRE_PROCESSOR_NULL, memoryCacheKey);
					}
				}

				if (bmp != null && options.isCacheInMemory()) {
					L.d(LOG_CACHE_IMAGE_IN_MEMORY, memoryCacheKey);
					configuration.memoryCache.put(memoryCacheKey, bmp);
				}
			} else {
				loadedFrom = LoadedFrom.MEMORY_CACHE;
				L.d(LOG_GET_IMAGE_FROM_MEMORY_CACHE_AFTER_WAITING, memoryCacheKey);
			}

			if (bmp != null && options.shouldPostProcess()) {
				L.d(LOG_POSTPROCESS_IMAGE, memoryCacheKey);
				bmp = options.getPostProcessor().process(bmp);
				if (bmp == null) {
					L.e(ERROR_POST_PROCESSOR_NULL, memoryCacheKey);
				}
			}
			checkTaskNotActual();
			checkTaskInterrupted();
		} catch (TaskCancelledException e) {
			fireCancelEvent();
			return;
		} finally {
			loadFromUriLock.unlock();
		}

		DisplayBitmapTask displayBitmapTask = new DisplayBitmapTask(bmp, imageLoadingInfo, engine, loadedFrom);
		runTask(displayBitmapTask, syncLoading, handler, engine);
	}

	/** @return <b>true</b> - if task should be interrupted; <b>false</b> - otherwise */
	private boolean waitIfPaused() {
		AtomicBoolean pause = engine.getPause();
		if (pause.get()) {
			synchronized (engine.getPauseLock()) {
				if (pause.get()) {
					L.d(LOG_WAITING_FOR_RESUME, memoryCacheKey);
					try {
						engine.getPauseLock().wait();
					} catch (InterruptedException e) {
						L.e(LOG_TASK_INTERRUPTED, memoryCacheKey);
						return true;
					}
					L.d(LOG_RESUME_AFTER_PAUSE, memoryCacheKey);
				}
			}
		}
		return isTaskNotActual();
	}

	/** @return <b>true</b> - if task should be interrupted; <b>false</b> - otherwise */
	private boolean delayIfNeed() {
		if (options.shouldDelayBeforeLoading()) {
			L.d(LOG_DELAY_BEFORE_LOADING, options.getDelayBeforeLoading(), memoryCacheKey);
			try {
				Thread.sleep(options.getDelayBeforeLoading());
			} catch (InterruptedException e) {
				L.e(LOG_TASK_INTERRUPTED, memoryCacheKey);
				return true;
			}
			return isTaskNotActual();
		}
		return false;
	}

	private Bitmap tryLoadBitmap() throws TaskCancelledException {
		Bitmap bitmap = null;
		try {
			File imageFile = configuration.diskCache.get(uri);
			if (imageFile != null && imageFile.exists() && imageFile.length() > 0) {
				L.d(LOG_LOAD_IMAGE_FROM_DISK_CACHE, memoryCacheKey);
				loadedFrom = LoadedFrom.DISC_CACHE;

				checkTaskNotActual();
				bitmap = decodeImage(Scheme.FILE.wrap(imageFile.getAbsolutePath()));
			}
			if (bitmap == null || bitmap.getWidth() <= 0 || bitmap.getHeight() <= 0) {
				L.d(LOG_LOAD_IMAGE_FROM_NETWORK, memoryCacheKey);
				loadedFrom = LoadedFrom.NETWORK;

				String imageUriForDecoding = uri;
				if (options.isCacheOnDisk() && tryCacheImageOnDisk()) {
					imageFile = configuration.diskCache.get(uri);
					if (imageFile != null) {
						imageUriForDecoding = Scheme.FILE.wrap(imageFile.getAbsolutePath());
					}
				}

				checkTaskNotActual();
				bitmap = decodeImage(imageUriForDecoding);

				if (bitmap == null || bitmap.getWidth() <= 0 || bitmap.getHeight() <= 0) {
					fireFailEvent(FailType.DECODING_ERROR, null);
				}
			}
		} catch (IllegalStateException e) {
			fireFailEvent(FailType.NETWORK_DENIED, null);
		} catch (TaskCancelledException e) {
			throw e;
		} catch (IOException e) {
			L.e(e);
			fireFailEvent(FailType.IO_ERROR, e);
		} catch (OutOfMemoryError e) {
			L.e(e);
			fireFailEvent(FailType.OUT_OF_MEMORY, e);
		} catch (Throwable e) {
			L.e(e);
			fireFailEvent(FailType.UNKNOWN, e);
		}
		return bitmap;
	}

	private Bitmap decodeImage(String imageUri) throws IOException {
		ViewScaleType viewScaleType = imageAware.getScaleType();
		ImageDecodingInfo decodingInfo = new ImageDecodingInfo(memoryCacheKey, imageUri, uri, targetSize, viewScaleType,
				getDownloader(), options);
		return decoder.decode(decodingInfo);
	}

	/** @return <b>true</b> - if image was downloaded successfully; <b>false</b> - otherwise */
	private boolean tryCacheImageOnDisk() throws TaskCancelledException {
		L.d(LOG_CACHE_IMAGE_ON_DISK, memoryCacheKey);

		boolean loaded;
		try {
			loaded = downloadImage();
			if (loaded) {
				int width = configuration.maxImageWidthForDiskCache;
				int height = configuration.maxImageHeightForDiskCache;
				if (width > 0 || height > 0) {
					L.d(LOG_RESIZE_CACHED_IMAGE_FILE, memoryCacheKey);
					resizeAndSaveImage(width, height); // TODO : process boolean result
				}
			}
		} catch (IOException e) {
			L.e(e);
			loaded = false;
		}
		return loaded;
	}

	private boolean downloadImage() throws IOException {
		InputStream is = getDownloader().getStream(uri, options.getExtraForDownloader());
		if (is == null) {
			L.e(ERROR_NO_IMAGE_STREAM, memoryCacheKey);
			return false;
		} else {
			try {
				return configuration.diskCache.save(uri, is, this);
			} finally {
				IoUtils.closeSilently(is);
			}
		}
	}

	/** Decodes image file into Bitmap, resize it and save it back */
	private boolean resizeAndSaveImage(int maxWidth, int maxHeight) throws IOException {
		// Decode image file, compress and re-save it
		boolean saved = false;
		File targetFile = configuration.diskCache.get(uri);
		if (targetFile != null && targetFile.exists()) {
			ImageSize targetImageSize = new ImageSize(maxWidth, maxHeight);
			DisplayImageOptions specialOptions = new DisplayImageOptions.Builder().cloneFrom(options)
					.imageScaleType(ImageScaleType.IN_SAMPLE_INT).build();
			ImageDecodingInfo decodingInfo = new ImageDecodingInfo(memoryCacheKey,
					Scheme.FILE.wrap(targetFile.getAbsolutePath()), uri, targetImageSize, ViewScaleType.FIT_INSIDE,
					getDownloader(), specialOptions);
			Bitmap bmp = decoder.decode(decodingInfo);
			if (bmp != null && configuration.processorForDiskCache != null) {
				L.d(LOG_PROCESS_IMAGE_BEFORE_CACHE_ON_DISK, memoryCacheKey);
				bmp = configuration.processorForDiskCache.process(bmp);
				if (bmp == null) {
					L.e(ERROR_PROCESSOR_FOR_DISK_CACHE_NULL, memoryCacheKey);
				}
			}
			if (bmp != null) {
				saved = configuration.diskCache.save(uri, bmp);
				bmp.recycle();
			}
		}
		return saved;
	}

	@Override
	public boolean onBytesCopied(int current, int total) {
		return syncLoading || fireProgressEvent(current, total);
	}

	/** @return <b>true</b> - if loading should be continued; <b>false</b> - if loading should be interrupted */
	private boolean fireProgressEvent(final int current, final int total) {
		if (isTaskInterrupted() || isTaskNotActual()) return false;
		if (progressListener != null) {
			Runnable r = new Runnable() {
				@Override
				public void run() {
					progressListener.onProgressUpdate(uri, imageAware.getWrappedView(), current, total);
				}
			};
			runTask(r, false, handler, engine);
		}
		return true;
	}

	private void fireFailEvent(final FailType failType, final Throwable failCause) {
		if (syncLoading || isTaskInterrupted() || isTaskNotActual()) return;
		Runnable r = new Runnable() {
			@Override
			public void run() {
				if (options.shouldShowImageOnFail()) {
					imageAware.setImageDrawable(options.getImageOnFail(configuration.resources));
				}
				listener.onLoadingFailed(uri, imageAware.getWrappedView(), new FailReason(failType, failCause));
			}
		};
		runTask(r, false, handler, engine);
	}

	private void fireCancelEvent() {
		if (syncLoading || isTaskInterrupted()) return;
		Runnable r = new Runnable() {
			@Override
			public void run() {
				listener.onLoadingCancelled(uri, imageAware.getWrappedView());
			}
		};
		runTask(r, false, handler, engine);
	}

	private ImageDownloader getDownloader() {
		ImageDownloader d;
		if (engine.isNetworkDenied()) {
			d = networkDeniedDownloader;
		} else if (engine.isSlowNetwork()) {
			d = slowNetworkDownloader;
		} else {
			d = downloader;
		}
		return d;
	}

	/**
	 * @throws TaskCancelledException if task is not actual (target ImageAware is collected by GC or the image URI of
	 *                                this task doesn't match to image URI which is actual for current ImageAware at
	 *                                this moment)
	 */
	private void checkTaskNotActual() throws TaskCancelledException {
		checkViewCollected();
		checkViewReused();
	}

	/**
	 * @return <b>true</b> - if task is not actual (target ImageAware is collected by GC or the image URI of this task
	 * doesn't match to image URI which is actual for current ImageAware at this moment)); <b>false</b> - otherwise
	 */
	private boolean isTaskNotActual() {
		return isViewCollected() || isViewReused();
	}

	/** @throws TaskCancelledException if target ImageAware is collected */
	private void checkViewCollected() throws TaskCancelledException {
		if (isViewCollected()) {
			throw new TaskCancelledException();
		}
	}

	/** @return <b>true</b> - if target ImageAware is collected by GC; <b>false</b> - otherwise */
	private boolean isViewCollected() {
		if (imageAware.isCollected()) {
			L.d(LOG_TASK_CANCELLED_IMAGEAWARE_COLLECTED, memoryCacheKey);
			return true;
		}
		return false;
	}

	/** @throws TaskCancelledException if target ImageAware is collected by GC */
	private void checkViewReused() throws TaskCancelledException {
		if (isViewReused()) {
			throw new TaskCancelledException();
		}
	}

	/** @return <b>true</b> - if current ImageAware is reused for displaying another image; <b>false</b> - otherwise */
	private boolean isViewReused() {
		String currentCacheKey = engine.getLoadingUriForView(imageAware);
		// Check whether memory cache key (image URI) for current ImageAware is actual.
		// If ImageAware is reused for another task then current task should be cancelled.
		boolean imageAwareWasReused = !memoryCacheKey.equals(currentCacheKey);
		if (imageAwareWasReused) {
			L.d(LOG_TASK_CANCELLED_IMAGEAWARE_REUSED, memoryCacheKey);
			return true;
		}
		return false;
	}

	/** @throws TaskCancelledException if current task was interrupted */
	private void checkTaskInterrupted() throws TaskCancelledException {
		if (isTaskInterrupted()) {
			throw new TaskCancelledException();
		}
	}

	/** @return <b>true</b> - if current task was interrupted; <b>false</b> - otherwise */
	private boolean isTaskInterrupted() {
		if (Thread.interrupted()) {
			L.d(LOG_TASK_INTERRUPTED, memoryCacheKey);
			return true;
		}
		return false;
	}

	String getLoadingUri() {
		return uri;
	}

	static void runTask(Runnable r, boolean sync, Handler handler, ImageLoaderEngine engine) {
		if (sync) {
			r.run();
		} else if (handler == null) {
			engine.fireCallback(r);
		} else {
			handler.post(r);
		}
	}

}
```
核心代码就是在run方法中,先看开始的两个判断 *waitIfPaused* 和 *delayIfNeed*
```java
private boolean waitIfPaused() {
		AtomicBoolean pause = engine.getPause();
		if (pause.get()) {
			synchronized (engine.getPauseLock()) {
				if (pause.get()) {
					L.d(LOG_WAITING_FOR_RESUME, memoryCacheKey);
					try {
						engine.getPauseLock().wait();
					} catch (InterruptedException e) {
						L.e(LOG_TASK_INTERRUPTED, memoryCacheKey);
						return true;
					}
					L.d(LOG_RESUME_AFTER_PAUSE, memoryCacheKey);
				}
			}
		}
		return isTaskNotActual();
	}

```
如果engine对象处于暂停状态的时候,会让阻塞当前运行的线程,等待engine再次调用 *resume* 方法
```java
/** @return <b>true</b> - if task should be interrupted; <b>false</b> - otherwise */
	private boolean delayIfNeed() {
		if (options.shouldDelayBeforeLoading()) {
			L.d(LOG_DELAY_BEFORE_LOADING, options.getDelayBeforeLoading(), memoryCacheKey);
			try {
				Thread.sleep(options.getDelayBeforeLoading());
			} catch (InterruptedException e) {
				L.e(LOG_TASK_INTERRUPTED, memoryCacheKey);
				return true;
			}
			return isTaskNotActual();
		}
		return false;
	}

```
如果在配置中设定了延迟显示的属性的话,休眠设置的时间然后继续执行.

```java
ReentrantLock loadFromUriLock = imageLoadingInfo.loadFromUriLock;
		L.d(LOG_START_DISPLAY_IMAGE_TASK, memoryCacheKey);
		if (loadFromUriLock.isLocked()) {
			L.d(LOG_WAITING_FOR_IMAGE_LOADED, memoryCacheKey);
		}

		loadFromUriLock.lock();
```
对应前面url锁,一个url对应一个锁对象,这样同一时间内相同的URL只有一个任务在运行.
下面的 *checkTaskNotActual* 在前面讲过了,无非就是判断view的状态是否可用,这里不重复了

然后再次从内存中获取,这是因为加锁的原因,可能前面的线程已经把相同的url加载完成了,这里就先从内存中获取一下,防止重复下载.
如何没有找到bitmap对象就进入核心方法 *tryLoadBitmap* 中,这个方法就完成了上面的本地存储查找,如果没有的就从网上下载的逻辑,最后和上面一样,交给displayer来显示.下面重点讲下 *tryLoadBitmap* 这个方法

```java
private Bitmap tryLoadBitmap() throws TaskCancelledException {
		Bitmap bitmap = null;
		try {
			File imageFile = configuration.diskCache.get(uri);
			if (imageFile != null && imageFile.exists() && imageFile.length() > 0) {
				L.d(LOG_LOAD_IMAGE_FROM_DISK_CACHE, memoryCacheKey);
				loadedFrom = LoadedFrom.DISC_CACHE;

				checkTaskNotActual();
				bitmap = decodeImage(Scheme.FILE.wrap(imageFile.getAbsolutePath()));
			}
			if (bitmap == null || bitmap.getWidth() <= 0 || bitmap.getHeight() <= 0) {
				L.d(LOG_LOAD_IMAGE_FROM_NETWORK, memoryCacheKey);
				loadedFrom = LoadedFrom.NETWORK;

				String imageUriForDecoding = uri;
				if (options.isCacheOnDisk() && tryCacheImageOnDisk()) {
					imageFile = configuration.diskCache.get(uri);
					if (imageFile != null) {
						imageUriForDecoding = Scheme.FILE.wrap(imageFile.getAbsolutePath());
					}
				}

				checkTaskNotActual();
				bitmap = decodeImage(imageUriForDecoding);

				if (bitmap == null || bitmap.getWidth() <= 0 || bitmap.getHeight() <= 0) {
					fireFailEvent(FailType.DECODING_ERROR, null);
				}
			}
		} catch (IllegalStateException e) {
			fireFailEvent(FailType.NETWORK_DENIED, null);
		} catch (TaskCancelledException e) {
			throw e;
		} catch (IOException e) {
			L.e(e);
			fireFailEvent(FailType.IO_ERROR, e);
		} catch (OutOfMemoryError e) {
			L.e(e);
			fireFailEvent(FailType.OUT_OF_MEMORY, e);
		} catch (Throwable e) {
			L.e(e);
			fireFailEvent(FailType.UNKNOWN, e);
		}
		return bitmap;
	}

```
首先是从diskCache中查找有没有对应的图片对象,如果有的话交给 *decodeImage* 方法处理完图片之后直接返回,如果没有的话就进入下面的判断中
```java
.....
if (options.isCacheOnDisk() && tryCacheImageOnDisk()) {
					imageFile = configuration.diskCache.get(uri);
					if (imageFile != null) {
						imageUriForDecoding = Scheme.FILE.wrap(imageFile.getAbsolutePath());
					}
				}
.....
```
是否需要缓存到本地,如果需要的话进入 *tryCacheImageOnDisk* 这个方法用来从网络上下载图片
```java
private boolean tryCacheImageOnDisk() throws TaskCancelledException {
		L.d(LOG_CACHE_IMAGE_ON_DISK, memoryCacheKey);

		boolean loaded;
		try {
			loaded = downloadImage();
			if (loaded) {
				int width = configuration.maxImageWidthForDiskCache;
				int height = configuration.maxImageHeightForDiskCache;
				if (width > 0 || height > 0) {
					L.d(LOG_RESIZE_CACHED_IMAGE_FILE, memoryCacheKey);
					resizeAndSaveImage(width, height); // TODO : process boolean result
				}
			}
		} catch (IOException e) {
			L.e(e);
			loaded = false;
		}
		return loaded;
	}
```
进入 *downloadImage* 方法:
```java
private boolean downloadImage() throws IOException {
  InputStream is = getDownloader().getStream(uri, options.getExtraForDownloader());
  if (is == null) {
    L.e(ERROR_NO_IMAGE_STREAM, memoryCacheKey);
    return false;
  } else {
    try {
      return configuration.diskCache.save(uri, is, this);
    } finally {
      IoUtils.closeSilently(is);
    }
  }
}
```
可以看到如果下载成功后,先把对象放入diskCache中,和最上面那个流程一样,下载先保存,然后从缓存中取.
这里 *getDownloader* 的对象是一个 *ImageDownloader* 接口对象,之所以要这样式因为UIL不仅仅可以处理http和https类型的图片,它也可以处理 "file:///","content://","assert://","drawable://" 这些类型的URI,所以要根据不同的URI类型来找到不通的 *ImageDownloader* 加载图片.
最后当从内存中找到了对应的图片对象之后也是交给 *decode* 方法来处理图片. *decode* 主要是用来处理图片缩放的:
```java
private Bitmap decodeImage(String imageUri) throws IOException {
  ViewScaleType viewScaleType = imageAware.getScaleType();
  ImageDecodingInfo decodingInfo = new ImageDecodingInfo(memoryCacheKey, imageUri, uri, targetSize, viewScaleType,
      getDownloader(), options);
  return decoder.decode(decodingInfo);
}
```
这里ImageDecodingInfo也是一个中间对象,用来把需要的参数都封装成一个对象传给Decoder对象
```java
public interface ImageDecoder {

	/**
	 * Decodes image to {@link Bitmap} according target size and other parameters.
	 *
	 * @param imageDecodingInfo
	 * @return
	 * @throws IOException
	 */
	Bitmap decode(ImageDecodingInfo imageDecodingInfo) throws IOException;
}
```
```java
public class BaseImageDecoder implements ImageDecoder {

	protected static final String LOG_SUBSAMPLE_IMAGE = "Subsample original image (%1$s) to %2$s (scale = %3$d) [%4$s]";
	protected static final String LOG_SCALE_IMAGE = "Scale subsampled image (%1$s) to %2$s (scale = %3$.5f) [%4$s]";
	protected static final String LOG_ROTATE_IMAGE = "Rotate image on %1$d\u00B0 [%2$s]";
	protected static final String LOG_FLIP_IMAGE = "Flip image horizontally [%s]";
	protected static final String ERROR_NO_IMAGE_STREAM = "No stream for image [%s]";
	protected static final String ERROR_CANT_DECODE_IMAGE = "Image can't be decoded [%s]";

	protected final boolean loggingEnabled;


	public BaseImageDecoder(boolean loggingEnabled) {
		this.loggingEnabled = loggingEnabled;
	}

	/**
	 * Decodes image from URI into {@link Bitmap}. Image is scaled close to incoming {@linkplain ImageSize target size}
	 * during decoding (depend on incoming parameters).
	 *
	 * @param decodingInfo Needed data for decoding image
	 * @return Decoded bitmap
	 * @throws IOException                   if some I/O exception occurs during image reading
	 * @throws UnsupportedOperationException if image URI has unsupported scheme(protocol)
	 */
	@Override
	public Bitmap decode(ImageDecodingInfo decodingInfo) throws IOException {
		Bitmap decodedBitmap;
		ImageFileInfo imageInfo;

		InputStream imageStream = getImageStream(decodingInfo);
		if (imageStream == null) {
			L.e(ERROR_NO_IMAGE_STREAM, decodingInfo.getImageKey());
			return null;
		}
		try {
			imageInfo = defineImageSizeAndRotation(imageStream, decodingInfo);
			imageStream = resetStream(imageStream, decodingInfo);
			Options decodingOptions = prepareDecodingOptions(imageInfo.imageSize, decodingInfo);
			decodedBitmap = BitmapFactory.decodeStream(imageStream, null, decodingOptions);
		} finally {
			IoUtils.closeSilently(imageStream);
		}

		if (decodedBitmap == null) {
			L.e(ERROR_CANT_DECODE_IMAGE, decodingInfo.getImageKey());
		} else {
			decodedBitmap = considerExactScaleAndOrientatiton(decodedBitmap, decodingInfo, imageInfo.exif.rotation,
					imageInfo.exif.flipHorizontal);
		}
		return decodedBitmap;
	}

	protected InputStream getImageStream(ImageDecodingInfo decodingInfo) throws IOException {
		return decodingInfo.getDownloader().getStream(decodingInfo.getImageUri(), decodingInfo.getExtraForDownloader());
	}

	protected ImageFileInfo defineImageSizeAndRotation(InputStream imageStream, ImageDecodingInfo decodingInfo)
			throws IOException {
		Options options = new Options();
		options.inJustDecodeBounds = true;
		BitmapFactory.decodeStream(imageStream, null, options);

		ExifInfo exif;
		String imageUri = decodingInfo.getImageUri();
		if (decodingInfo.shouldConsiderExifParams() && canDefineExifParams(imageUri, options.outMimeType)) {
			exif = defineExifOrientation(imageUri);
		} else {
			exif = new ExifInfo();
		}
		return new ImageFileInfo(new ImageSize(options.outWidth, options.outHeight, exif.rotation), exif);
	}

	private boolean canDefineExifParams(String imageUri, String mimeType) {
		return "image/jpeg".equalsIgnoreCase(mimeType) && (Scheme.ofUri(imageUri) == Scheme.FILE);
	}

	protected ExifInfo defineExifOrientation(String imageUri) {
		int rotation = 0;
		boolean flip = false;
		try {
			ExifInterface exif = new ExifInterface(Scheme.FILE.crop(imageUri));
			int exifOrientation = exif.getAttributeInt(ExifInterface.TAG_ORIENTATION, ExifInterface.ORIENTATION_NORMAL);
			switch (exifOrientation) {
				case ExifInterface.ORIENTATION_FLIP_HORIZONTAL:
					flip = true;
				case ExifInterface.ORIENTATION_NORMAL:
					rotation = 0;
					break;
				case ExifInterface.ORIENTATION_TRANSVERSE:
					flip = true;
				case ExifInterface.ORIENTATION_ROTATE_90:
					rotation = 90;
					break;
				case ExifInterface.ORIENTATION_FLIP_VERTICAL:
					flip = true;
				case ExifInterface.ORIENTATION_ROTATE_180:
					rotation = 180;
					break;
				case ExifInterface.ORIENTATION_TRANSPOSE:
					flip = true;
				case ExifInterface.ORIENTATION_ROTATE_270:
					rotation = 270;
					break;
			}
		} catch (IOException e) {
			L.w("Can't read EXIF tags from file [%s]", imageUri);
		}
		return new ExifInfo(rotation, flip);
	}

	protected Options prepareDecodingOptions(ImageSize imageSize, ImageDecodingInfo decodingInfo) {
		ImageScaleType scaleType = decodingInfo.getImageScaleType();
		int scale;
		if (scaleType == ImageScaleType.NONE) {
			scale = 1;
		} else if (scaleType == ImageScaleType.NONE_SAFE) {
			scale = ImageSizeUtils.computeMinImageSampleSize(imageSize);
		} else {
			ImageSize targetSize = decodingInfo.getTargetSize();
			boolean powerOf2 = scaleType == ImageScaleType.IN_SAMPLE_POWER_OF_2;
			scale = ImageSizeUtils.computeImageSampleSize(imageSize, targetSize, decodingInfo.getViewScaleType(), powerOf2);
		}
		if (scale > 1 && loggingEnabled) {
			L.d(LOG_SUBSAMPLE_IMAGE, imageSize, imageSize.scaleDown(scale), scale, decodingInfo.getImageKey());
		}

		Options decodingOptions = decodingInfo.getDecodingOptions();
		decodingOptions.inSampleSize = scale;
		return decodingOptions;
	}

	protected InputStream resetStream(InputStream imageStream, ImageDecodingInfo decodingInfo) throws IOException {
		if (imageStream.markSupported()) {
			try {
				imageStream.reset();
				return imageStream;
			} catch (IOException ignored) {
			}
		}
		IoUtils.closeSilently(imageStream);
		return getImageStream(decodingInfo);
	}

	protected Bitmap considerExactScaleAndOrientatiton(Bitmap subsampledBitmap, ImageDecodingInfo decodingInfo,
			int rotation, boolean flipHorizontal) {
		Matrix m = new Matrix();
		// Scale to exact size if need
		ImageScaleType scaleType = decodingInfo.getImageScaleType();
		if (scaleType == ImageScaleType.EXACTLY || scaleType == ImageScaleType.EXACTLY_STRETCHED) {
			ImageSize srcSize = new ImageSize(subsampledBitmap.getWidth(), subsampledBitmap.getHeight(), rotation);
			float scale = ImageSizeUtils.computeImageScale(srcSize, decodingInfo.getTargetSize(), decodingInfo
					.getViewScaleType(), scaleType == ImageScaleType.EXACTLY_STRETCHED);
			if (Float.compare(scale, 1f) != 0) {
				m.setScale(scale, scale);

				if (loggingEnabled) {
					L.d(LOG_SCALE_IMAGE, srcSize, srcSize.scale(scale), scale, decodingInfo.getImageKey());
				}
			}
		}
		// Flip bitmap if need
		if (flipHorizontal) {
			m.postScale(-1, 1);

			if (loggingEnabled) L.d(LOG_FLIP_IMAGE, decodingInfo.getImageKey());
		}
		// Rotate bitmap if need
		if (rotation != 0) {
			m.postRotate(rotation);

			if (loggingEnabled) L.d(LOG_ROTATE_IMAGE, rotation, decodingInfo.getImageKey());
		}

		Bitmap finalBitmap = Bitmap.createBitmap(subsampledBitmap, 0, 0, subsampledBitmap.getWidth(), subsampledBitmap
				.getHeight(), m, true);
		if (finalBitmap != subsampledBitmap) {
			subsampledBitmap.recycle();
		}
		return finalBitmap;
	}

	protected static class ExifInfo {

		public final int rotation;
		public final boolean flipHorizontal;

		protected ExifInfo() {
			this.rotation = 0;
			this.flipHorizontal = false;
		}

		protected ExifInfo(int rotation, boolean flipHorizontal) {
			this.rotation = rotation;
			this.flipHorizontal = flipHorizontal;
		}
	}

	protected static class ImageFileInfo {

		public final ImageSize imageSize;
		public final ExifInfo exif;

		protected ImageFileInfo(ImageSize imageSize, ExifInfo exif) {
			this.imageSize = imageSize;
			this.exif = exif;
		}
	}
}
```
可以看到decoder类对Bitmap根据需要做了缩放和旋转处理,如果必要的话.这里涉及到Bitmap操作的api,这里就不展开了,后续有时间我们介绍一下Bitmap的常用操作方法.
回到 *LoadAndDisplayImageTask* 的run方法,可以看到,这里在等到获取到Bitmap对象之后,可以调用getPreProcessor来对对象做二次处理,然后放入内存缓存中,最后也可以再对Bitmap做一次处理,如果设置了getPostProcessor的话,最后就交给 *DisplayBitmapTask* 去显示图片了,和上面的 *ProcessAndDisplayImageTask* 一样.

核心加载流程就讲完了,是不是很容易理解,其实也没有上面特别拿点的地方,只要明白加载的流程就很容易跟到代码了.最后简单介绍一下config对象的相关属性:

#### ImageLoaderConfiguration
```java
public final class ImageLoaderConfiguration {

	final Resources resources;

	final int maxImageWidthForMemoryCache;
	final int maxImageHeightForMemoryCache;
	final int maxImageWidthForDiskCache;
	final int maxImageHeightForDiskCache;
	final BitmapProcessor processorForDiskCache;

	final Executor taskExecutor;
	final Executor taskExecutorForCachedImages;
	final boolean customExecutor;
	final boolean customExecutorForCachedImages;

	final int threadPoolSize;
	final int threadPriority;
	final QueueProcessingType tasksProcessingType;

	final MemoryCache memoryCache;
	final DiskCache diskCache;
	final ImageDownloader downloader;
	final ImageDecoder decoder;
	final DisplayImageOptions defaultDisplayImageOptions;

	final ImageDownloader networkDeniedDownloader;
	final ImageDownloader slowNetworkDownloader;

	private ImageLoaderConfiguration(final Builder builder) {
		resources = builder.context.getResources();
		maxImageWidthForMemoryCache = builder.maxImageWidthForMemoryCache;
		maxImageHeightForMemoryCache = builder.maxImageHeightForMemoryCache;
		maxImageWidthForDiskCache = builder.maxImageWidthForDiskCache;
		maxImageHeightForDiskCache = builder.maxImageHeightForDiskCache;
		processorForDiskCache = builder.processorForDiskCache;
		taskExecutor = builder.taskExecutor;
		taskExecutorForCachedImages = builder.taskExecutorForCachedImages;
		threadPoolSize = builder.threadPoolSize;
		threadPriority = builder.threadPriority;
		tasksProcessingType = builder.tasksProcessingType;
		diskCache = builder.diskCache;
		memoryCache = builder.memoryCache;
		defaultDisplayImageOptions = builder.defaultDisplayImageOptions;
		downloader = builder.downloader;
		decoder = builder.decoder;

		customExecutor = builder.customExecutor;
		customExecutorForCachedImages = builder.customExecutorForCachedImages;

		networkDeniedDownloader = new NetworkDeniedImageDownloader(downloader);
		slowNetworkDownloader = new SlowNetworkImageDownloader(downloader);

		L.writeDebugLogs(builder.writeLogs);
	}

	public static ImageLoaderConfiguration createDefault(Context context) {
		return new Builder(context).build();
	}

	ImageSize getMaxImageSize() {
		DisplayMetrics displayMetrics = resources.getDisplayMetrics();

		int width = maxImageWidthForMemoryCache;
		if (width <= 0) {
			width = displayMetrics.widthPixels;
		}
		int height = maxImageHeightForMemoryCache;
		if (height <= 0) {
			height = displayMetrics.heightPixels;
		}
		return new ImageSize(width, height);
	}
  ....
}
```
config对象的属性起名很清晰,基本和字面翻译一致,可以看到很多地方都可以配置,比如线程池,decoder等对象,非常的灵活.方便根据需求去扩展.

最后的最后,来看下两个接口,分别对应图片二次处理和图片显示的
#### processor 和 displayer
```java
public interface BitmapProcessor {

	Bitmap process(Bitmap bitmap);
}

```
```java
public interface BitmapDisplayer {
	void display(Bitmap bitmap, ImageAware imageAware, LoadedFrom loadedFrom);
}
```
这里可以根据需求去实现自己的process和displayer,UIL已经给提供了几种displayer了,比如 *CircleBitmapDisplayer* , *FadeInBitmapDisplayer* 等,位于core包目录下的display包下面,感兴趣的话可以自己看下实现.

### DefaultConfigurationFactory
这个类用来生成默认的config和option对象的,都是一些简单的初始化操作,这里就不讲了,具体的请看代码实现.
```java
public class DefaultConfigurationFactory {

	/** Creates default implementation of task executor */
	public static Executor createExecutor(int threadPoolSize, int threadPriority,
			QueueProcessingType tasksProcessingType) {
		boolean lifo = tasksProcessingType == QueueProcessingType.LIFO;
		BlockingQueue<Runnable> taskQueue =
				lifo ? new LIFOLinkedBlockingDeque<Runnable>() : new LinkedBlockingQueue<Runnable>();
		return new ThreadPoolExecutor(threadPoolSize, threadPoolSize, 0L, TimeUnit.MILLISECONDS, taskQueue,
				createThreadFactory(threadPriority, "uil-pool-"));
	}

	/** Creates default implementation of task distributor */
	public static Executor createTaskDistributor() {
		return Executors.newCachedThreadPool(createThreadFactory(Thread.NORM_PRIORITY, "uil-pool-d-"));
	}

	/** Creates {@linkplain HashCodeFileNameGenerator default implementation} of FileNameGenerator */
	public static FileNameGenerator createFileNameGenerator() {
		return new HashCodeFileNameGenerator();
	}

	/**
	 * Creates default implementation of {@link DiskCache} depends on incoming parameters
	 */
	public static DiskCache createDiskCache(Context context, FileNameGenerator diskCacheFileNameGenerator,
			long diskCacheSize, int diskCacheFileCount) {
		File reserveCacheDir = createReserveDiskCacheDir(context);
		if (diskCacheSize > 0 || diskCacheFileCount > 0) {
			File individualCacheDir = StorageUtils.getIndividualCacheDirectory(context);
			try {
				return new LruDiskCache(individualCacheDir, reserveCacheDir, diskCacheFileNameGenerator, diskCacheSize,
						diskCacheFileCount);
			} catch (IOException e) {
				L.e(e);
				// continue and create unlimited cache
			}
		}
		File cacheDir = StorageUtils.getCacheDirectory(context);
		return new UnlimitedDiskCache(cacheDir, reserveCacheDir, diskCacheFileNameGenerator);
	}

	/** Creates reserve disk cache folder which will be used if primary disk cache folder becomes unavailable */
	private static File createReserveDiskCacheDir(Context context) {
		File cacheDir = StorageUtils.getCacheDirectory(context, false);
		File individualDir = new File(cacheDir, "uil-images");
		if (individualDir.exists() || individualDir.mkdir()) {
			cacheDir = individualDir;
		}
		return cacheDir;
	}

	/**
	 * Creates default implementation of {@link MemoryCache} - {@link LruMemoryCache}<br />
	 * Default cache size = 1/8 of available app memory.
	 */
	public static MemoryCache createMemoryCache(Context context, int memoryCacheSize) {
		if (memoryCacheSize == 0) {
			ActivityManager am = (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
			int memoryClass = am.getMemoryClass();
			if (hasHoneycomb() && isLargeHeap(context)) {
				memoryClass = getLargeMemoryClass(am);
			}
			memoryCacheSize = 1024 * 1024 * memoryClass / 8;
		}
		return new LruMemoryCache(memoryCacheSize);
	}

	private static boolean hasHoneycomb() {
		return Build.VERSION.SDK_INT >= Build.VERSION_CODES.HONEYCOMB;
	}

	@TargetApi(Build.VERSION_CODES.HONEYCOMB)
	private static boolean isLargeHeap(Context context) {
		return (context.getApplicationInfo().flags & ApplicationInfo.FLAG_LARGE_HEAP) != 0;
	}

	@TargetApi(Build.VERSION_CODES.HONEYCOMB)
	private static int getLargeMemoryClass(ActivityManager am) {
		return am.getLargeMemoryClass();
	}

	/** Creates default implementation of {@link ImageDownloader} - {@link BaseImageDownloader} */
	public static ImageDownloader createImageDownloader(Context context) {
		return new BaseImageDownloader(context);
	}

	/** Creates default implementation of {@link ImageDecoder} - {@link BaseImageDecoder} */
	public static ImageDecoder createImageDecoder(boolean loggingEnabled) {
		return new BaseImageDecoder(loggingEnabled);
	}

	/** Creates default implementation of {@link BitmapDisplayer} - {@link SimpleBitmapDisplayer} */
	public static BitmapDisplayer createBitmapDisplayer() {
		return new SimpleBitmapDisplayer();
	}

	/** Creates default implementation of {@linkplain ThreadFactory thread factory} for task executor */
	private static ThreadFactory createThreadFactory(int threadPriority, String threadNamePrefix) {
		return new DefaultThreadFactory(threadPriority, threadNamePrefix);
	}

	private static class DefaultThreadFactory implements ThreadFactory {

		private static final AtomicInteger poolNumber = new AtomicInteger(1);

		private final ThreadGroup group;
		private final AtomicInteger threadNumber = new AtomicInteger(1);
		private final String namePrefix;
		private final int threadPriority;

		DefaultThreadFactory(int threadPriority, String threadNamePrefix) {
			this.threadPriority = threadPriority;
			group = Thread.currentThread().getThreadGroup();
			namePrefix = threadNamePrefix + poolNumber.getAndIncrement() + "-thread-";
		}

		@Override
		public Thread newThread(Runnable r) {
			Thread t = new Thread(group, r, namePrefix + threadNumber.getAndIncrement(), 0);
			if (t.isDaemon()) t.setDaemon(false);
			t.setPriority(threadPriority);
			return t;
		}
	}
}
```

### 总结
到这里,UIL的源码主流程就分析完成了,这是第二个研究的开源项目,第一个是EventBus,以前老是觉得UIL非常的复杂,现在回头看了其实很简单,内部的调用也很清晰,看来还是得勇敢迈出第一步,这样才能提高.
