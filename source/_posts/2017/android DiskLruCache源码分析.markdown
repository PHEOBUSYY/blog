layout: "post"
title: "android DiskLruCache源码分析"
date: "2017-02-17 10:02"
---

## adroid DiskLruCache源码分析

  在学习Universal image loader源码的时候,看到了它在用到本地文件缓存的时候使用的是 *DiskLruCache* ,所有抽时间来分析下 *DiskLruCache* 的源码,来看下为什么它支持类似于 *LruCache* 功能,内部是怎么实现的.顺便再查看源码过程中了解下它的大概用法.

<!--more-->

### journal文件
  先来介绍下这个 *journal* 文件是什么东西.首先我们知道缓存必须对应是key-value结构,而文件缓存肯定是通过一定规则生成的key,然后对应找到制定路径上的文件来达到一个缓存的目的.那么在 *DiskLruCache* 中所有的缓存key的信息都是通过一个叫做 *journal* 的文件保存的.在使用的 *DiskLruCache* 的主文件目录中可以找到这个文件. 文件的格式内容大概像这样的:

  ```
    *     libcore.io.DiskLruCache
    *     1
    *     100
    *     2
    *
    *     CLEAN 3400330d1dfc7f3f7f4b8d4d803dfcf6 832 21054
    *     DIRTY 335c4c6028171cfddfbaae1a9c313c52
    *     CLEAN 335c4c6028171cfddfbaae1a9c313c52 3934 2342
    *     REMOVE 335c4c6028171cfddfbaae1a9c313c52
    *     DIRTY 1ab96a171faeeee38496d8b330771a7a
    *     CLEAN 1ab96a171faeeee38496d8b330771a7a 1600 234
    *     READ 335c4c6028171cfddfbaae1a9c313c52
    *     READ 3400330d1dfc7f3f7f4b8d4d803dfcf6
    *
  ```
  这段格式摘录自源码中的顶部注释说明中.我们可以把这个文件内容分成两部分.
  首先是顶部header,共有5行,包含了:
  1. 标志位,说明使用的DiskLruCache
  2. LruCache的版本号
  3. app的版本号
  4. 一个key对应多少个value值个数,在 *open* 方法中会说明
  5. 一个空行

  然后是下面的就是用到的缓存信息了,前面的关键字表明了缓存文件的状态,主要有 *DIRTY* *CLEAN* *REMOVE* *READ* 4中状态.
  1. DIRTY 每一条缓存在被创建或者更新的时候,状态都是 DIRTY 状态,下面紧跟着一条状态 *CLEAN* 或者 *REMOVE* 的信息,如果一条DIRTY信息下面没有找到 *CLEAN* 或者 *REMOVE* 的信息,说明是一条 "脏数据" ,需要被删除掉
  2. CLEAN 表明这是一条可以被读取的数据 ,后面跟着缓存文件的文件size
  3. REMOVE 表明这是一条已经被删除的数据
  4. READ 记录访问过该条缓存数据

  上面两部分组成了 *journal* 文件的所有内容.

### open方法
  当我们要开始使用DiskLruCache时,是不能直接通过new来创建对象,要通过它提供的静态方法 *open* 方法.通过open方法来返回一个DiskLruCache对象.
  ```java
  public static DiskLruCache open(File directory, int appVersion, int valueCount, long maxSize)
            throws IOException {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        if (valueCount <= 0) {
            throw new IllegalArgumentException("valueCount <= 0");
        }

        // prefer to pick up where we left off
        DiskLruCache cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
        if (cache.journalFile.exists()) {
            try {
                cache.readJournal();
                cache.processJournal();
                cache.journalWriter = new BufferedWriter(new FileWriter(cache.journalFile, true),
                        IO_BUFFER_SIZE);
                return cache;
            } catch (IOException journalIsCorrupt) {
//                System.logW("DiskLruCache " + directory + " is corrupt: "
//                        + journalIsCorrupt.getMessage() + ", removing");
                cache.delete();
            }
        }

        // create a new empty cache
        directory.mkdirs();
        cache = new DiskLruCache(directory, appVersion, valueCount, maxSize);
        cache.rebuildJournal();
        return cache;
  }
  ```
  在 *open* 方法中先来看DiskLruCache的构造函数:
  ```java
  private DiskLruCache(File directory, int appVersion, int valueCount, long maxSize) {
        this.directory = directory;
        this.appVersion = appVersion;
        this.journalFile = new File(directory, JOURNAL_FILE);
        this.journalFileTmp = new File(directory, JOURNAL_FILE_TMP);
        this.valueCount = valueCount;
        this.maxSize = maxSize;
    }
  ```
  构造函数中就是简单的赋值操作,其中
  1. directory 表明缓存要方法手机的什么位置上,一般放入 */sdcard/Android/data/<application package>/cache* 系统默认这个文件是app专用缓存文件,当app要被卸载的时候,会顺便把这个文件下的内容删除. 我们可以通过API来获取这个文件的位置:
  ```java
  cachePath = context.getExternalCacheDir().getPath();
  ```
  不过我们可能会遇到这种情况,就是SD卡不可用被挂起的状态下,我们只能使用手机本身的app缓存目录了,这个目录对应 */data/data/<application package>/cache* ,通过API来获取就是:
  ```java
  cachePath = context.getCacheDir().getPath();
  ```
  那么在使用DiskLruCache时,我们可以通过下面的代码来判断缓存目录:
  ```java
  public File getDiskCacheDir(Context context, String uniqueName) {  
    String cachePath;  
    if (Environment.MEDIA_MOUNTED.equals(Environment.getExternalStorageState())  
            || !Environment.isExternalStorageRemovable()) {  
        cachePath = context.getExternalCacheDir().getPath();  
    } else {  
        cachePath = context.getCacheDir().getPath();  
    }  
    return new File(cachePath + File.separator + uniqueName);  
   }  
  ```
  2. appVersion 表示app的version,我们可以通过PackageManager来获取
  ```java
  public int getAppVersion(Context context) {  
    try {  
        PackageInfo info = context.getPackageManager().getPackageInfo(context.getPackageName(), 0);  
        return info.versionCode;  
    } catch (NameNotFoundException e) {  
        e.printStackTrace();  
    }  
    return 1;  
    }
  ```
  3. valueCount 表示一个key可以获取多少个文件个数.默认情况设置为1
  4. maxSize 表示允许DIskLruCache最大存储空间是多少.如果超过了该设置空间,DIskLruCache会优先删除之前的缓存数据来腾出空间.

  回到open方法中,接下来的if判断是否已经存在了 *journal* 文件了.如果存在的话调用了 *readJournal* 方法:
  ```java
  private void readJournal() throws IOException {
        InputStream in = new BufferedInputStream(new FileInputStream(journalFile), IO_BUFFER_SIZE);
        try {
            String magic = readAsciiLine(in);
            String version = readAsciiLine(in);
            String appVersionString = readAsciiLine(in);
            String valueCountString = readAsciiLine(in);
            String blank = readAsciiLine(in);
            if (!MAGIC.equals(magic)
                    || !VERSION_1.equals(version)
                    || !Integer.toString(appVersion).equals(appVersionString)
                    || !Integer.toString(valueCount).equals(valueCountString)
                    || !"".equals(blank)) {
                throw new IOException("unexpected journal header: ["
                        + magic + ", " + version + ", " + valueCountString + ", " + blank + "]");
            }

            while (true) {
                try {
                    readJournalLine(readAsciiLine(in));
                } catch (EOFException endOfJournal) {
                    break;
                }
            }
        } finally {
            closeQuietly(in);
        }
    }

  ```
  这里通过 *readAsciiLine* 逐行读取已经存在的 *journal* 文件内容.然后把文件header中的属性与传入open方法的属性做对比,如果发现不匹配抛出异常.如果抛出异常就会回到open方法中的try catch中的catch部分,通过调用 *delete* 方法把缓存目录中所有文件清空.
  如果匹配就通过 *readJournalLine* 逐行读出文件的后续内容.

  ```java
  private void readJournalLine(String line) throws IOException {
        String[] parts = line.split(" ");
        if (parts.length < 2) {
            throw new IOException("unexpected journal line: " + line);
        }

        String key = parts[1];
        if (parts[0].equals(REMOVE) && parts.length == 2) {
            lruEntries.remove(key);
            return;
        }

        Entry entry = lruEntries.get(key);
        if (entry == null) {
            entry = new Entry(key);
            lruEntries.put(key, entry);
        }

        if (parts[0].equals(CLEAN) && parts.length == 2 + valueCount) {
            entry.readable = true;
            entry.currentEditor = null;
            entry.setLengths(copyOfRange(parts, 2, parts.length));
        } else if (parts[0].equals(DIRTY) && parts.length == 2) {
            entry.currentEditor = new Editor(entry);
        } else if (parts[0].equals(READ) && parts.length == 2) {
            // this work was already done by calling lruEntries.get()
        } else {
            throw new IOException("unexpected journal line: " + line);
        }
    }
  ```
  这里通过一个LinkedHashMap lruEntries 来存放所有的缓存记录.这所以使用 LinkedHashMap 这种数据结构是为了方便修改和删除,提高效率.
  如果在lruEntries中没有找到对应的记录就创建一个新的 *entry* 对象,反之,如果找到了就根据前面的状态来调整 *entry* 的属性.到这里就相当于把历史的缓存记录加载到了内存中了.
  读取完成之后接着调用 *processJournal* 来对缓存数据做一个预处理.
  ```java
  private void processJournal() throws IOException {
       deleteIfExists(journalFileTmp);
       for (Iterator<Entry> i = lruEntries.values().iterator(); i.hasNext(); ) {
           Entry entry = i.next();
           if (entry.currentEditor == null) {
               for (int t = 0; t < valueCount; t++) {
                   size += entry.lengths[t];
               }
           } else {
               entry.currentEditor = null;
               for (int t = 0; t < valueCount; t++) {
                   deleteIfExists(entry.getCleanFile(t));
                   deleteIfExists(entry.getDirtyFile(t));
               }
               i.remove();
           }
       }
   }
  ```
  如果记录是 *CLEAN* 状态,把文件大小计入总大小中,反之如果记录是 *脏数据* 就删除掉记录文件.
  走完 *readJournal* 和 *processJournal* 方法之后,就直接返回cache对象了.如果在这个过程中抛出异常就会把缓存文件全部删除.

  最后回到open方法的剩下部分,创建一个新的DiskLruCache对象,然后通过 *rebuildJournal* 创建一个新的 *journal* 文件:
  ```java
  private synchronized void rebuildJournal() throws IOException {
        if (journalWriter != null) {
            journalWriter.close();
        }

        Writer writer = new BufferedWriter(new FileWriter(journalFileTmp), IO_BUFFER_SIZE);
        writer.write(MAGIC);
        writer.write("\n");
        writer.write(VERSION_1);
        writer.write("\n");
        writer.write(Integer.toString(appVersion));
        writer.write("\n");
        writer.write(Integer.toString(valueCount));
        writer.write("\n");
        writer.write("\n");

        for (Entry entry : lruEntries.values()) {
            if (entry.currentEditor != null) {
                writer.write(DIRTY + ' ' + entry.key + '\n');
            } else {
                writer.write(CLEAN + ' ' + entry.key + entry.getLengths() + '\n');
            }
        }

        writer.close();
        journalFileTmp.renameTo(journalFile);
        journalWriter = new BufferedWriter(new FileWriter(journalFile, true), IO_BUFFER_SIZE);
    }
  ```
  可以看到还是创建两部分,header和缓存信息.非常的清晰.
  到这里,open方法就讲完了,主要是做一个缓存的创建和预处理.核心就在维护这个 *journal* 文件中.下面讲到的写入缓存和读取缓存都是对这个文件的操作.

### 写入缓存 editor
  当要写入缓存的时候,首先通过 *edit* 方法获取到 *Editor* 对象,之后通过editor对象的 *newOutputStream* 方法传入一个输入流,传入完成之后调用 *commit* 方法,完成写入缓存.
  ```java
  private synchronized Editor edit(String key, long expectedSequenceNumber) throws IOException {
       checkNotClosed();
       validateKey(key);
       Entry entry = lruEntries.get(key);
       if (expectedSequenceNumber != ANY_SEQUENCE_NUMBER
               && (entry == null || entry.sequenceNumber != expectedSequenceNumber)) {
           return null; // snapshot is stale
       }
       if (entry == null) {
           entry = new Entry(key);
           lruEntries.put(key, entry);
       } else if (entry.currentEditor != null) {
           return null; // another edit is in progress
       }

       Editor editor = new Editor(entry);
       entry.currentEditor = editor;

       // flush the journal before creating files to prevent file leaks
       journalWriter.write(DIRTY + ' ' + key + '\n');
       journalWriter.flush();
       return editor;
   }
  ```
  这里通过edit方法生成一个新的editor对象,然后在 *journal* 文件中加入一条 *DIRTY* 类型的数据.获取到editor对象之后就可以通过editor的对象的 *newOutputStream* 拿到一个输出流,然后把要保存的文件输入到这个流中.来看下editor的源码:
  ```java
  public OutputStream newOutputStream(int index) throws IOException {
           synchronized (DiskLruCache.this) {
               if (entry.currentEditor != this) {
                   throw new IllegalStateException();
               }
               return new FaultHidingOutputStream(new FileOutputStream(entry.getDirtyFile(index)));
           }
       }
  ```
  返回一个 *FaultHidingOutputStream* 这个对象只是对outputStream的异常做了一个封装处理,,没有其他的区别.
  注意这里的流文件的路径用的是 *getDirtyFile* 这相当于一个中间文件,当本次写入最终成功的时候会把这个 *getDirtyFile* 覆盖到 *getCleanFile* 中.
  ```java
  private class FaultHidingOutputStream extends FilterOutputStream {
           private FaultHidingOutputStream(OutputStream out) {
               super(out);
           }

           @Override public void write(int oneByte) {
               try {
                   out.write(oneByte);
               } catch (IOException e) {
                   hasErrors = true;
               }
           }

           @Override public void write(byte[] buffer, int offset, int length) {
               try {
                   out.write(buffer, offset, length);
               } catch (IOException e) {
                   hasErrors = true;
               }
           }

           @Override public void close() {
               try {
                   out.close();
               } catch (IOException e) {
                   hasErrors = true;
               }
           }

           @Override public void flush() {
               try {
                   out.flush();
               } catch (IOException e) {
                   hasErrors = true;
               }
           }
       }
  ```
  拿到这个对象之后,我们就可以把下载的数据流放入这个流中,比如通过下面的示例代码放入:
  ```java
  new Thread(new Runnable() {  
    @Override  
    public void run() {  
        try {  
            String imageUrl = "http://img.my.csdn.net/uploads/201309/01/1378037235_7476.jpg";  
            String key = hashKeyForDisk(imageUrl);  
            DiskLruCache.Editor editor = mDiskLruCache.edit(key);  
            if (editor != null) {  
                OutputStream outputStream = editor.newOutputStream(0);  
                if (downloadUrlToStream(imageUrl, outputStream)) {  
                    editor.commit();  
                } else {  
                    editor.abort();  
                }  
            }  
            mDiskLruCache.flush();  
        } catch (IOException e) {  
            e.printStackTrace();  
        }  
    }  
  }).start();  
  ```
  当下载完成后,调用 *commit* 方法修改缓存 *journal* 文件:
  ```java
  public void commit() throws IOException {
           if (hasErrors) {
               completeEdit(this, false);
               remove(entry.key); // the previous entry is stale
           } else {
               completeEdit(this, true);
           }
       }
  ```
  ```java
  public void abort() throws IOException {
          completeEdit(this, false);
      }
  ```
  这里可以看到 *commit* 和 *abort* 方法都是调用的 *completeEdit* 方法.那什么情况下满足 *hasErrors* 呢?还记得上面说的那个 *FaultHidingOutputStream* 对象吗?里面的try catch发生异常的时候就会把 *hasErrors* 这个变量设置为true.

  ```java
  private synchronized void completeEdit(Editor editor, boolean success) throws IOException {
        Entry entry = editor.entry;
        if (entry.currentEditor != editor) {
            throw new IllegalStateException();
        }

        // if this edit is creating the entry for the first time, every index must have a value
        if (success && !entry.readable) {
            for (int i = 0; i < valueCount; i++) {
                if (!entry.getDirtyFile(i).exists()) {
                    editor.abort();
                    throw new IllegalStateException("edit didn't create file " + i);
                }
            }
        }

        for (int i = 0; i < valueCount; i++) {
            File dirty = entry.getDirtyFile(i);
            if (success) {
                if (dirty.exists()) {
                    File clean = entry.getCleanFile(i);
                    dirty.renameTo(clean);
                    long oldLength = entry.lengths[i];
                    long newLength = clean.length();
                    entry.lengths[i] = newLength;
                    size = size - oldLength + newLength;
                }
            } else {
                deleteIfExists(dirty);
            }
        }

        redundantOpCount++;
        entry.currentEditor = null;
        if (entry.readable | success) {
            entry.readable = true;
            journalWriter.write(CLEAN + ' ' + entry.key + entry.getLengths() + '\n');
            if (success) {
                entry.sequenceNumber = nextSequenceNumber++;
            }
        } else {
            lruEntries.remove(entry.key);
            journalWriter.write(REMOVE + ' ' + entry.key + '\n');
        }

        if (size > maxSize || journalRebuildRequired()) {
            executorService.submit(cleanupCallable);
        }
    }
  ```
  下面来分析 *completeEdit* 这个方法.首先如果成功的话要判断是不是对应缓存的所有文件都是存在的,如果有一个不存在就取消这次写入.
  然后遍历所有的 "DirtyFile" 写入到 "CleanFile" 中,同时更新DiskLruCache中size.看看是否要触发越界处理.如果全部写入成功就在 *journal* 文件中写入一条 *CLEAN* 记录.
  感觉Editor整体就像一个事物管理器,为了保证这次写入的完整性,当最终写入完成的时候Editor才算执行完成,否者就会发生回滚.

  当写入文件的容量超过了总容量限制的时候,就会触发减容操作,也就是删除之前一些缓存文件来保证后面的文件的写入.这正好也是DiskLruCache的特性.下面来看下减容操作的实现:
  ```java
  private final Callable<Void> cleanupCallable = new Callable<Void>() {
      @Override public Void call() throws Exception {
          synchronized (DiskLruCache.this) {
              if (journalWriter == null) {
                  return null; // closed
              }
              trimToSize();
              if (journalRebuildRequired()) {
                  rebuildJournal();
                  redundantOpCount = 0;
              }
          }
          return null;
      }
  };

  ```
  这里把 *cleanupCallable* 交给线程池来调用.内部调用的 *trimToSize* 方法.

  ```java
  private void trimToSize() throws IOException {
        while (size > maxSize) {
//            Map.Entry<String, Entry> toEvict = lruEntries.eldest();
            final Map.Entry<String, Entry> toEvict = lruEntries.entrySet().iterator().next();
            remove(toEvict.getKey());
        }
    }
  ```
  这里变量所有的缓存,从头开始删除,直到总容量小于容量限制为止.这也就是为什么要使用 *LinkedHashMap* 的原因,有序并且便于删除.这里遍历是从头开始,相应的开始的文件应该是最早的文件,使用可能性要远低于后来的.

  这里进入了 *remove* 方法,正好对应删除缓存操作.
  ```java
  public synchronized boolean remove(String key) throws IOException {
       checkNotClosed();
       validateKey(key);
       Entry entry = lruEntries.get(key);
       if (entry == null || entry.currentEditor != null) {
           return false;
       }

       for (int i = 0; i < valueCount; i++) {
           File file = entry.getCleanFile(i);
           if (!file.delete()) {
               throw new IOException("failed to delete " + file);
           }
           size -= entry.lengths[i];
           entry.lengths[i] = 0;
       }

       redundantOpCount++;
       journalWriter.append(REMOVE + ' ' + key + '\n');
       lruEntries.remove(key);

       if (journalRebuildRequired()) {
           executorService.submit(cleanupCallable);
       }

       return true;
   }
  ```
  这里删除掉所有的缓存文件,同时修改 *journal* 文件中的记录.同时有个隐藏触发条件判断 *journalRebuildRequired* :
  ```java
  private boolean journalRebuildRequired() {
      final int REDUNDANT_OP_COMPACT_THRESHOLD = 2000;
      return redundantOpCount >= REDUNDANT_OP_COMPACT_THRESHOLD
              && redundantOpCount >= lruEntries.size();
  }
  ```
  这里冗余数 *redundantOpCount* 超过了限制,并且冗余数大于缓存记录个数的话就会触发再一次的减容操作,并且会重新生成一个 *journal* 文件,里面只包含 *CLEAN* 和 *DIRTY* 类型的记录.这里对应上面讲到的 *rebuildJournal* 的中间部分:
  ```java
  for (Entry entry : lruEntries.values()) {
             if (entry.currentEditor != null) {
                 writer.write(DIRTY + ' ' + entry.key + '\n');
             } else {
                 writer.write(CLEAN + ' ' + entry.key + entry.getLengths() + '\n');
             }
         }
  ```
  这里有个疑问要解答一下,就是说如果重构了 *journal* 文件,那么那些标记为 *REMOVE* 的文件会不会一直都留在了文件夹中变成了无头文件.因为在 *journal* 文件中没有它们的记录,下次重新启动后没人知道那些死文件的存在了,但是文件夹的容量会一直不断的变大.后来想了下应该不会发生这种情况,因为一条记录被标记为 *REMOVE* 类型的时候,对应的文件已经被删除了.不管是通过减容操作还是通过外部调用 *remove* 方法,文件和记录应该还是一一对应的.
  也就是说如果一条记录已经被标记为 *REMOVE* 那么对应的文件应该已经不存在了.

### 读取缓存 get
  通过DiskLruCache的 *get* 方法可以获取指定的缓存:
  ```java
  public synchronized Snapshot get(String key) throws IOException {
      checkNotClosed();
      validateKey(key);
      Entry entry = lruEntries.get(key);
      if (entry == null) {
          return null;
      }

      if (!entry.readable) {
          return null;
      }

      /*
       * Open all streams eagerly to guarantee that we see a single published
       * snapshot. If we opened streams lazily then the streams could come
       * from different edits.
       */
      InputStream[] ins = new InputStream[valueCount];
      try {
          for (int i = 0; i < valueCount; i++) {
              ins[i] = new FileInputStream(entry.getCleanFile(i));
          }
      } catch (FileNotFoundException e) {
          // a file must have been deleted manually!
          return null;
      }

      redundantOpCount++;
      journalWriter.append(READ + ' ' + key + '\n');
      if (journalRebuildRequired()) {
          executorService.submit(cleanupCallable);
      }

      return new Snapshot(key, entry.sequenceNumber, ins);
  }
  ```
  get方法非常的简单,就是从 *lruEntries* 获取对应的key,如果找到了就返回一个 *snap* 对象,同时在 *journal* 文件中写入一条 *READ* 类型的数据.
  为啥要包装一个 *snap* 对象呢?这是因为上面讲的open方法的第三个参数,就是一个缓存key可以对应几个文件的个数.因为可能是对应多个,所以需要包装一个对象,可以通过里面的index来获取制定key下面的指定index的文件对象.

  ```java
  public final class Snapshot implements Closeable {
         private final String key;
         private final long sequenceNumber;
         private final InputStream[] ins;

         private Snapshot(String key, long sequenceNumber, InputStream[] ins) {
             this.key = key;
             this.sequenceNumber = sequenceNumber;
             this.ins = ins;
         }

         /**
          * Returns an editor for this snapshot's entry, or null if either the
          * entry has changed since this snapshot was created or if another edit
          * is in progress.
          */
         public Editor edit() throws IOException {
             return DiskLruCache.this.edit(key, sequenceNumber);
         }

         /**
          * Returns the unbuffered stream with the value for {@code index}.
          */
         public InputStream getInputStream(int index) {
             return ins[index];
         }

         /**
          * Returns the string value for {@code index}.
          */
         public String getString(int index) throws IOException {
             return inputStreamToString(getInputStream(index));
         }

         @Override public void close() {
             for (InputStream in : ins) {
                 closeQuietly(in);
             }
         }
     }
  ```
  这里通过snap的 *getInputStream* 可以获取到一个流对象,有了对象你就可以把它写入图片Bitmap对象等等操作都可以啦.
  ```java
  try {  
    String imageUrl = "http://img.my.csdn.net/uploads/201309/01/1378037235_7476.jpg";  
    String key = hashKeyForDisk(imageUrl);  
    DiskLruCache.Snapshot snapShot = mDiskLruCache.get(key);  
    if (snapShot != null) {  
        InputStream is = snapShot.getInputStream(0);  
        Bitmap bitmap = BitmapFactory.decodeStream(is);  
        mImage.setImageBitmap(bitmap);  
    }  
} catch (IOException e) {  
    e.printStackTrace();  
}  
  ```

### 总结
  到这里,所有的操作都讲完了,可以看到就是通过维护 *journal* 文件来配置缓存,找到指定的文件位置,然后通过size属性来控制缓存容量的.这里通过 *Editor* 来模拟了事物的操作感觉还是很值得借鉴的.

### 参考资料

[DiskLruCache源码][cc012721]

  [cc012721]: https://developer.android.com/samples/DisplayingBitmaps/src/com.example.android.displayingbitmaps/util/DiskLruCache.html#l55 "DiskLruCache源码"

[Android DiskLruCache全解析][8d10d14c]

  [8d10d14c]: http://blog.csdn.net/guolin_blog/article/details/28863651 "Android DiskLruCache全解析"
