layout: "post"
title: "Android LruMemoryCache源码分析"
date: "2017-02-17 14:45"
---
## Android LruMemoryCache源码分析
  在分析Univeral Image Loader的时候看到里面的缓存实现很不错.其中的内存缓存使用的是它自己写的 *LruMemoryCache* ,文件缓存使用的是 *DiskLruCache* ,其中的 *DiskLruCache* 的源码已经分析完了,下面简单讲下 *LruMemoryCache* 的实现.
<!--more-->
  ```java
  public class LruMemoryCache implements MemoryCache {

	private final LinkedHashMap<String, Bitmap> map;

	private final int maxSize;
	/** Size of this cache in bytes */
	private int size;

	/** @param maxSize Maximum sum of the sizes of the Bitmaps in this cache */
	public LruMemoryCache(int maxSize) {
		if (maxSize <= 0) {
			throw new IllegalArgumentException("maxSize <= 0");
		}
		this.maxSize = maxSize;
		this.map = new LinkedHashMap<String, Bitmap>(0, 0.75f, true);
	}

	/**
	 * Returns the Bitmap for {@code key} if it exists in the cache. If a Bitmap was returned, it is moved to the head
	 * of the queue. This returns null if a Bitmap is not cached.
	 */
	@Override
	public final Bitmap get(String key) {
		if (key == null) {
			throw new NullPointerException("key == null");
		}

		synchronized (this) {
			return map.get(key);
		}
	}

	/** Caches {@code Bitmap} for {@code key}. The Bitmap is moved to the head of the queue. */
	@Override
	public final boolean put(String key, Bitmap value) {
		if (key == null || value == null) {
			throw new NullPointerException("key == null || value == null");
		}

		synchronized (this) {
			size += sizeOf(key, value);
			Bitmap previous = map.put(key, value);
			if (previous != null) {
				size -= sizeOf(key, previous);
			}
		}

		trimToSize(maxSize);
		return true;
	}

	/**
	 * Remove the eldest entries until the total of remaining entries is at or below the requested size.
	 *
	 * @param maxSize the maximum size of the cache before returning. May be -1 to evict even 0-sized elements.
	 */
	private void trimToSize(int maxSize) {
		while (true) {
			String key;
			Bitmap value;
			synchronized (this) {
				if (size < 0 || (map.isEmpty() && size != 0)) {
					throw new IllegalStateException(getClass().getName() + ".sizeOf() is reporting inconsistent results!");
				}

				if (size <= maxSize || map.isEmpty()) {
					break;
				}

				Map.Entry<String, Bitmap> toEvict = map.entrySet().iterator().next();
				if (toEvict == null) {
					break;
				}
				key = toEvict.getKey();
				value = toEvict.getValue();
				map.remove(key);
				size -= sizeOf(key, value);
			}
		}
	}

	/** Removes the entry for {@code key} if it exists. */
	@Override
	public final Bitmap remove(String key) {
		if (key == null) {
			throw new NullPointerException("key == null");
		}

		synchronized (this) {
			Bitmap previous = map.remove(key);
			if (previous != null) {
				size -= sizeOf(key, previous);
			}
			return previous;
		}
	}

	@Override
	public Collection<String> keys() {
		synchronized (this) {
			return new HashSet<String>(map.keySet());
		}
	}

	@Override
	public void clear() {
		trimToSize(-1); // -1 will evict 0-sized elements
	}

	/**
	 * Returns the size {@code Bitmap} in bytes.
	 * <p/>
	 * An entry's size must not change while it is in the cache.
	 */
	private int sizeOf(String key, Bitmap value) {
		return value.getRowBytes() * value.getHeight();
	}

	@Override
	public synchronized final String toString() {
		return String.format("LruCache[maxSize=%d]", maxSize);
	}
  }
  ```
  整个代码大概一百多行,主要是通过维护一个LinkedhashMap来保存key对应的bitmap对象,同时内部通过size属性来表明总容量.
  当size超过最大限制的时候,就会触发 *trimToSize* 方法.
  ```java
  private void trimToSize(int maxSize) {
    while (true) {
      String key;
      Bitmap value;
      synchronized (this) {
        if (size < 0 || (map.isEmpty() && size != 0)) {
          throw new IllegalStateException(getClass().getName() + ".sizeOf() is reporting inconsistent results!");
        }

        if (size <= maxSize || map.isEmpty()) {
          break;
        }

        Map.Entry<String, Bitmap> toEvict = map.entrySet().iterator().next();
        if (toEvict == null) {
          break;
        }
        key = toEvict.getKey();
        value = toEvict.getValue();
        map.remove(key);
        size -= sizeOf(key, value);
      }
    }
  }
  ```
  可以看到这里循环遍历map对象,从头到尾的删除对象,每删除一个对象之后计算size容量是否低于最大容量,直到满足小于最大容量限制.

  就是这么简单,over.
  可以看到所有 LRUXX 缓存类的实现都是基于这种思路,交给一个map容器,如果超过容量限制的时候就触发减容操作,直到满足条件.这里的数据结构很重要,都是用的 *LinkedHashMap* ,有序并且容易删除.
