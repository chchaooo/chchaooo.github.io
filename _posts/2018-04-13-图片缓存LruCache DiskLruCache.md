---
layout:     post
title:      "(七)Bitmap缓存"
subtitle:   ""
date:       2018-04-13 20:30:00
author:     "chchaooo"
header-img: "img/main_banner.jpg"
header-mask: 0.3
catalog:    true
tags:
    - 内存 Bitmap
---

### 图片缓存

在Android开发中，加载一个图片到界面很容易，但如果一次加载大量图片就复杂多了。在很多情况下（比如：ListView,GridView或ViewPager），能够滚动的组件需要加载的图片几乎是无限多的。

有些组件的child view在不显示时会回收，并循环使用，如果没有任何对bitmap的持久引用的话，垃圾回收器会释放你加载的bitmap。当这些图片再次显示的时候，要想避免重复处理这些图片而达到加载流畅的效果，就要使用内存缓存和本地缓存了，这些缓存可以让你快速加载处理过的图片。

这些缓存，就是本章要讨论的内容。

### 使用内存缓存

内存缓存以牺牲内存的代价，带来快速的图片访问。**LruCache**类非常适合处理图片缓存任务，在一个LinkedHashMap中保存着对Bitmap的强引用，当缓存数量超过容器容量时，删除最近最少使用的成员（LRU）。

注意：
1. **在过去，非常流行用SoftReference或WeakReference来实现图片的内存缓存，但现在不再推荐使用这个方法了。因为从Android 2.3 （API Level 9）之后，垃圾回收器会更积极的回收soft/weak的引用，这将导致使用soft/weak引用的缓存几乎没有缓存效果**。
2. **在Android3.0（API Level 11）以前，bitmap是储存在native 内存中的**

为了给LruCache一个合适的容量，需要考虑很多因素，比如：

* 你其它的Activity 和/或 Application是怎样使用内存的？
* 屏幕一次显示多少图片？需要多少图片为显示到屏幕做准备？
* 屏幕的大小(size)和密度(density)是多少？像Galaxy Nexus这样高密度（xhdpi）的屏幕在缓存相同数量的图片时，就需要比低密度屏幕Nexus S（hdpi）更大的内存。
* 每个图片的尺寸多大，相关配置怎样的，占用多大内存？
* 图片的访问频率高不高？不同图片的访问频率是否不一样？如果是，你可能会把某些图片一直缓存在内存中，或需要多种不同缓存策略的LruCache。
* 你能平衡图片的质量和数量吗？有时候，缓存多个质量低的图片是很有用的，而质量高的图片应该（像下载文件一样）在后台任务中加载。

这里没有适应所有应用的特定大小或公式，只能通过分析具体的使用方法，来得出合适的解决方案。缓存太小的话没有实际用处，还会增加额外开销；缓存太大的话，会再一次造成OutOfMemory异常，并给应用的其他部分留下很少的内存。

下面是一个图片的LruCache的配置例子：

``` java
private LruCache<String, Bitmap> mMemoryCache;

@Override
protected void onCreate(Bundle savedInstanceState) {
    ...
    // Get max available VM memory, exceeding this amount will throw an
    // OutOfMemory exception. Stored in kilobytes as LruCache takes an
    // int in its constructor.转换单位
    final int maxMemory = (int) (Runtime.getRuntime().maxMemory() / 1024);

    // Use 1/8th of the available memory for this memory cache.
    final int cacheSize = maxMemory / 8;

    mMemoryCache = new LruCache<String, Bitmap>(cacheSize) {
        @Override
        protected int sizeOf(String key, Bitmap bitmap) {
            // The cache size will be measured in kilobytes rather than
            // number of items.转换单位
            return bitmap.getByteCount() / 1024;
        }
    };
    ...
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

注意：在这个例子中，将应用最大可使用内存的八分之一分配给了缓存，在一个普通的或者hdpi设备上，这个值大约至少是4MB(32/8)。在一个800x480分辨率的设备上全屏显示一个填满图片的GridView大约要1.5MB(800*480*4 bytes)，因此，这个LruCache至少能够缓存2.5页的图片。

当往ImageView中加载图片时，先检查以下LruCache中是否已经缓存。如果已经缓存，则直接取出并更新ImageView，否则要启动个后台任务来处理图片：

```java
public void loadBitmap(int resId, ImageView imageView) {
    final String imageKey = String.valueOf(resId);

    final Bitmap bitmap = getBitmapFromMemCache(imageKey);
    if (bitmap != null) {
        mImageView.setImageBitmap(bitmap);
    } else {
        mImageView.setImageResource(R.drawable.image_placeholder);
        BitmapWorkerTask task = new BitmapWorkerTask(mImageView);
        task.execute(resId);
    }
}
```

这个BitmapWorkerTask也需要将新处理完的图片添加进缓存：

```java
class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {
    ...
    // Decode image in background.
    @Override
    protected Bitmap doInBackground(Integer... params) {
        final Bitmap bitmap = decodeSampledBitmapFromResource(
                getResources(), params[0], 100, 100));
        addBitmapToMemoryCache(String.valueOf(params[0]), bitmap);
        return bitmap;
    }
    ...
}
```
 

### 使用磁盘缓存

内存缓存能够加快对最近显示过的图片的访问速度，然而你不能认为缓存中的图片全是有效的。像GridView这样需要大量数据的组件是很容易填满内存缓存的。你的应用可能会被别的任务打断（比如一个来电），它可能会在后台被杀掉，其内存缓存当然也被销毁了。当用户恢复你的应用时，应用将重新处理之前缓存的每一张图片。

在这个情形中，使用磁盘缓存可以持久的储存处理过的图片，并且缩短加载内存缓存中无效的图片的时间。当然从磁盘加载图片比从内存中加载图片要慢的多，并且由于磁盘读取的时间是不确定的，所以要在后台线程进行磁盘加载。

>>注意：如果以更高的频率访问图片，比如图片墙应用，使用ContentProvider可能更适合储存图片缓存。

下面这个例子除了之前的内存缓存，还添加了一个磁盘缓存，这个磁盘缓存实现自Android源码中的DiskLruCache：

```java
private DiskLruCache mDiskLruCache;
private final Object mDiskCacheLock = new Object();
private boolean mDiskCacheStarting = true;
private static final int DISK_CACHE_SIZE = 1024 * 1024 * 10; // 10MB
private static final String DISK_CACHE_SUBDIR = "thumbnails";

@Override
protected void onCreate(Bundle savedInstanceState) {
    ...
    // Initialize memory cache
    ...
    // Initialize disk cache on background thread
    File cacheDir = getDiskCacheDir(this, DISK_CACHE_SUBDIR);
    new InitDiskCacheTask().execute(cacheDir);
    ...
}

class InitDiskCacheTask extends AsyncTask<File, Void, Void> {
    @Override
    protected Void doInBackground(File... params) {
        synchronized (mDiskCacheLock) {
            File cacheDir = params[0];
            mDiskLruCache = DiskLruCache.open(cacheDir, DISK_CACHE_SIZE);
            mDiskCacheStarting = false; // Finished initialization
            mDiskCacheLock.notifyAll(); // Wake any waiting threads
        }
        return null;
    }
}

class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {
    ...
    // Decode image in background.
    @Override
    protected Bitmap doInBackground(Integer... params) {
        final String imageKey = String.valueOf(params[0]);

        // Check disk cache in background thread
        Bitmap bitmap = getBitmapFromDiskCache(imageKey);

        if (bitmap == null) { // Not found in disk cache
            // Process as normal
            final Bitmap bitmap = decodeSampledBitmapFromResource(
                    getResources(), params[0], 100, 100));
        }

        // Add final bitmap to caches
        addBitmapToCache(imageKey, bitmap);

        return bitmap;
    }
    ...
}


public void addBitmapToCache(String key, Bitmap bitmap) {
    // Add to memory cache as before
    if (getBitmapFromMemCache(key) == null) {
        mMemoryCache.put(key, bitmap);
    }

    // Also add to disk cache
    synchronized (mDiskCacheLock) {
        if (mDiskLruCache != null && mDiskLruCache.get(key) == null) {
            mDiskLruCache.put(key, bitmap);
        }
    }
}

public Bitmap getBitmapFromDiskCache(String key) {
    synchronized (mDiskCacheLock) {
        // Wait while disk cache is started from background thread
        while (mDiskCacheStarting) {
            try {
                mDiskCacheLock.wait();
            } catch (InterruptedException e) {}
        }
        if (mDiskLruCache != null) {
            return mDiskLruCache.get(key);
        }
    }
    return null;
}

// Creates a unique subdirectory of the designated app cache directory. Tries to use external
// but if not mounted, falls back on internal storage.
public static File getDiskCacheDir(Context context, String uniqueName) {
    // Check if media is mounted or storage is built-in, if so, try and use external cache dir
    // otherwise use internal cache dir
    final String cachePath =
            Environment.MEDIA_MOUNTED.equals(Environment.getExternalStorageState()) ||
                    !isExternalStorageRemovable() ? getExternalCacheDir(context).getPath() :
                            context.getCacheDir().getPath();

    return new File(cachePath + File.separator + uniqueName);
}
```

注意：磁盘缓存的初始化会涉及磁盘操作，因此不应该在UI线程做初始化操作。然而，这样做可能会出现初始化完成前，就访问缓存的情况。为了解决这个问题，上面的例子使用了一个锁对象，确保在磁盘缓存初始化完成前，不会有其他对象使用它。

检查内存缓存可以在UI线程，检查磁盘缓存最好在后台线程。永远不要在UI线程做磁盘操作。当图片处理完成，应该将其添加进内存缓存和磁盘缓存中，以备将来不时之需。

### Handle Configuration Changes
Runtime Configuration Changes,例如屏幕方向变化导致android系统销毁和重建当前Activity。在此过程中如果想要用户有一个流畅的交互体验,需要避免重新将所有图片都从头重新初始化一遍。

在Use a memory cache这一节中，我们已经介绍了一个好用的内存缓存机制。我们可以通过一个fragment将这个cache从被销毁的activity传递给重建之后的activity（这个fragment需要调用**setRetainInstance(true)** ），在Activity重建之后，这个fragment将会被关联到新的activity上。

```java
private LruCache<String, Bitmap> mMemoryCache;

@Override
protected void onCreate(Bundle savedInstanceState) {
    ...
    RetainFragment retainFragment =
            RetainFragment.findOrCreateRetainFragment(getFragmentManager());
    mMemoryCache = retainFragment.mRetainedCache;
    if (mMemoryCache == null) {
        mMemoryCache = new LruCache<String, Bitmap>(cacheSize) {
            ... // Initialize cache here as usual
        }
        retainFragment.mRetainedCache = mMemoryCache;
    }
    ...
}

class RetainFragment extends Fragment {
    private static final String TAG = "RetainFragment";
    public LruCache<String, Bitmap> mRetainedCache;

    public RetainFragment() {}

    public static RetainFragment findOrCreateRetainFragment(FragmentManager fm) {
        RetainFragment fragment = (RetainFragment) fm.findFragmentByTag(TAG);
        if (fragment == null) {
            fragment = new RetainFragment();
            fm.beginTransaction().add(fragment, TAG).commit();
        }
        return fragment;
    }

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setRetainInstance(true);
    }
}
```

