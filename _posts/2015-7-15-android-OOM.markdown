---
layout: post
title:  "Android OOM(OUT OF MEMORY)"
date:   2015-7-15
---
> 注：本文参考了郭霖的blog [Android高效加载大图、多图解决方案，有效避免程序OOM](http://blog.csdn.net/guolin_blog/article/details/9316683 "Android OOM")

<p class="intro"><span class="dropcap">O</span>OM的中文意思是内存溢出，通俗理解就是软件（应用）运行需要的内存，超出了它可用的最大内存。在android中，最常出现OOM的时候就是加载图片的时候，因此也就主要讲的是怎么处理图片加载时避免OOM.</p>
<p>我们知道我们的app在运行的时候是由内存限制的，而加载一张高分辨率的图片就会占用程序许多的内存，但是我们只是需要它显示在一张小的imageView上，根本没必要这么大分辨率的图片，那么怎么做呢？</p>
##SOLUTION:
***
###使用BitmapFactory.Options参数
将这个参数的inJustDecodeBounds属性设置为true就可以让解析方法禁止为bitmap分配内存，返回值也不再是一个Bitmap对象，而是null。虽然Bitmap是null了，但是BitmapFactory.Options的outWidth、outHeight和outMimeType属性都会被赋值。这个技巧让我们可以在加载图片之前就获取到图片的长宽值和MIME类型，从而根据情况对图片进行压缩。

```java

	BitmapFactory.Options options = new BitmapFactory.Options();
	options.inJustDecodeBounds = true;
	BitmapFactory.decodeResource(getResources(), R.id.myimage, options);
	int imageHeight = options.outHeight;
	int imageWidth = options.outWidth;
	String imageType = options.outMimeType;
```
现在我们知道了图片的大小，现在要做的就是对图片进行压缩了，怎么压缩呢？这时我们依然要用到options,通过设置BitmapFactory.Options中inSampleSize的值就可以实现.

```java

	public static int calculateInSampleSize(BitmapFactory.Options options,
		int reqWidth, int reqHeight) {
	// 源图片的高度和宽度
	final int height = options.outHeight;
	final int width = options.outWidth;
	int inSampleSize = 1;
	if (height > reqHeight || width > reqWidth) {
		// 计算出实际宽高和目标宽高的比率
		final int heightRatio = Math.round((float) height / (float) reqHeight);
		final int widthRatio = Math.round((float) width / (float) reqWidth);
		// 选择宽和高中最小的比率作为inSampleSize的值，这样可以保证最终图片的宽和高
		// 一定都会大于等于目标的宽和高。
		inSampleSize = heightRatio < widthRatio ? heightRatio : widthRatio;
	}
	return inSampleSize;
	}	
```
这样之后,如果我们有一张2048*1536像素的图片，将inSampleSize的值设置为4，就可以把这张图片压缩成512*384像素。原本加载这张图片需要占用13M的内存，压缩后就只需要占用0.75M了(假设图片是ARGB_8888类型，即每个像素点占用4个字节)，就可以大大的节约我们的内存。

###使用LRUCache图片缓存技术
LRU是least Recently Used，中文意思就是最近最少使用的意思，相信学习过计算机操作系统的同学很容易就会理解，当然没学习过得同学也很容易从字面意思理解到就是当我们要加载新的图片的时候，如果这张图片不在我们的缓存中，就将最近最少使用的的图片替换出来，很容易理解吧。那怎么做呢？

为了能够选择一个合适的缓存大小给LruCache, 有以下多个因素应该放入考虑范围内，例如：

- 你的设备可以为每个应用程序分配多大的内存？
- 设备屏幕上一次最多能显示多少张图片？有多少图片需要进行预加载，因为有可能很快也会显示在屏幕 上？
- 你的设备的屏幕大小和分辨率分别是多少？一个超高分辨率的设备（例如 Galaxy Nexus) 比起一个- 较低分辨率的设备（例如 Nexus S），在持有相同数量图片的时候，需要更大的缓存空间。
- 图片的尺寸和大小，还有每张图片会占据多少内存空间。
- 图片被访问的频率有多高？会不会有一些图片的访问频率比其它图片要高？如果有的话，你也许应该让一些图片常驻在内存当中，或者使用多个LruCache 对象来区分不同组的图片。
- 你能维持好数量和质量之间的平衡吗？有些时候，存储多个低像素的图片，而在后台去开线程加载高像素的图片会更加的有效。

一个简单的LRUCache

```java

	private LruCache<String, Bitmap> mMemoryCache;

	@Override
	protected void onCreate(Bundle savedInstanceState) {
	// 获取到可用内存的最大值，使用内存超出这个值会引起OutOfMemory异常。
	// LruCache通过构造函数传入缓存值，以KB为单位。
	int maxMemory = (int) (Runtime.getRuntime().maxMemory() / 1024);
	// 使用最大可用内存值的1/8作为缓存的大小。
	int cacheSize = maxMemory / 8;
	mMemoryCache = new LruCache<String, Bitmap>(cacheSize) {
		@Override
		protected int sizeOf(String key, Bitmap bitmap) {
			// 重写此方法来衡量每张图片的大小，默认返回图片数量。
			return bitmap.getByteCount() / 1024;
			}
		};
	}

	public void addBitmapToMemoryCache(String key, Bitmap bitmap) {
	if (getBitmapFromMemCache(key) == null) {
		mMemoryCache.put(key, bitmap);
		}
	}

	public Bitmap getBitmapFromMemCache(String key) {
		return mMemoryCache.get(key);
	}

```

当向 ImageView 中加载一张图片时,首先会在 LruCache 的缓存中进行检查。如果找到了相应的键值，则会立刻更新ImageView ，否则开启一个后台线程来加载这张图片。

```java

	public void loadBitmap(int resId, ImageView imageView) {
	final String imageKey = String.valueOf(resId);
	final Bitmap bitmap = getBitmapFromMemCache(imageKey);
	if (bitmap != null) {
		imageView.setImageBitmap(bitmap);
	} else {
		imageView.setImageResource(R.drawable.image_placeholder);
		BitmapWorkerTask task = new BitmapWorkerTask(imageView);
		task.execute(resId);
	}
}

```
BitmapWorkerTask 还要把新加载的图片的键值对放到缓存中。

```java

	class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {
	// 在后台加载图片。
	@Override
	protected Bitmap doInBackground(Integer... params) {
		final Bitmap bitmap = decodeSampledBitmapFromResource(
				getResources(), params[0], 100, 100);
		addBitmapToMemoryCache(String.valueOf(params[0]), bitmap);
		return bitmap;
		}
	}
```
***
OOM的内容也就到这里来，掌握了这两种方法就可以有效避免在加载image的时候出现OOM了，当然感兴趣的同学还可以去了解一下

- [Android-Universal-Image-Loader](https://github.com/nostra13/Android-Universal-Image-Loader)，是现在用的最广的图片加载控件，可以让你轻松避免OOM.

如果你想了解网络上的图片加载，也可以看下

- volley,推荐郭神的[Volley系列教程](http://blog.csdn.net/guolin_blog/article/details/17482095)，讲的很详细。
