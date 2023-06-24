# SharedPreferences

此文主要讲解 SharedPreferences 的具体原理。
搞 Android 的相信应该都使用过 SharedPreferences，相信也都知道此工具主要用于保存的相对较小键值集合，但是具体原因，大家可能不太清楚，这也造成一些乱用。
源码说明一切，下面我们来分析一下源码。

## 使用
```
SharedPreferences sp = getApplicationContext().getSharedPreferences("keyzone", Context.MODE_PRIVATE);
SharedPreferences.Editor editor = sp.edit();
editor.putString("key", "value");
editor.commit();
```
如上，这是我们平时的使用的代码，不啰嗦。

## SharedPreferences
从上边代码，我们看到的只涉及 SharedPreferences.java，那我们就先看一下这个类，源码如下：
```
public interface SharedPreferences {

    public interface OnSharedPreferenceChangeListener {
        void onSharedPreferenceChanged(SharedPreferences sharedPreferences, String key);
    }

    // 各种 set 方法
    public interface Editor {
        Editor putString(String key, @Nullable String value);
        Editor putStringSet(String key, @Nullable Set<String> values);
        Editor putInt(String key, int value);
        Editor putLong(String key, long value);
        Editor putFloat(String key, float value);
        Editor putBoolean(String key, boolean value);
        Editor remove(String key);
        Editor clear();

        boolean commit();
        void apply();
    }

    // 各种 get 方法
    Map<String, ?> getAll();
    String getString(String key, @Nullable String defValue);
    Set<String> getStringSet(String key, @Nullable Set<String> defValues);
    int getInt(String key, int defValue);
    long getLong(String key, long defValue);
    float getFloat(String key, float defValue);
    boolean getBoolean(String key, boolean defValue);
    boolean contains(String key);
    
    Editor edit();
    
    void registerOnSharedPreferenceChangeListener(OnSharedPreferenceChangeListener listener);
    void unregisterOnSharedPreferenceChangeListener(OnSharedPreferenceChangeListener listener);
}
```
由此可知，SharedPreferences 也好，SharedPreferences.Editor 也罢，都是 interface，不涉及核心逻辑。
通过 getString、getInt 等各种 get 方法根据 key 值获取 value。Editor 则包含各种 set 方法，用来设置数据。
那接下来，我们看是谁实现了这个 interface。

## SharedPreferencesImpl
Android 里边一些实现类都是类似命名，在其后添加 impl（implements 缩写），例如 Context（ContextImpl）等，至于我们是怎么找到这个 class 的，我们后边再说。接下来我们看一下 SharedPreferencesImpl 的实现。为了展示主线，删除了很多具体的逻辑，如果想知道具体的操作，可以自己查阅源码。
```
final class SharedPreferencesImpl implements SharedPreferences {

  private Map<String, Object> mMap;
  private int mDiskWritesInFlight = 0;
  private boolean mLoaded = false;

  private static final Object mContent = new Object();
  private final WeakHashMap<OnSharedPreferenceChangeListener, Object> mListeners =
    new WeakHashMap<OnSharedPreferenceChangeListener, Object>();

  SharedPreferencesImpl(File file, int mode) {
    ...
    startLoadFromDisk();
  }

  // 从本地文件中 load 数据，该方法是异步
  private void startLoadFromDisk() {
    synchronized (this) {
      mLoaded = false;
    }
    new Thread("SharedPreferencesImpl-load") {
      public void run() {
      	// 具体从文件中读取数据的函数
        loadFromDisk();
      }
    }.start();
  }

  private void awaitLoadedLocked() {
    if (!mLoaded) {
      BlockGuard.getThreadPolicy().onReadFromDisk();
    }
    while (!mLoaded) {
      try {
        wait();
      } catch (InterruptedException unused) {
      }
    }
  }

  // 根据 key 值获取数据，这里就需要注意了，从代码可以看出这是一个同步函数，多线程时会阻塞，另外，如果存储数据过多，则会造成 awaitLoadedLocked 阻塞。
  public String getString(String key, @Nullable String defValue) {
    synchronized (this) {
      awaitLoadedLocked();
      String v = (String)mMap.get(key);
      return v != null ? v : defValue;
    }
  }

  ...

  public Editor edit() {
    synchronized (this) {
      awaitLoadedLocked();
    }
    return new EditorImpl();
  }

  public final class EditorImpl implements Editor {
    private final Map<String, Object> mModified = Maps.newHashMap();

    ...

    // 将改变的数据放入 mModified 中
    public Editor putString(String key, @Nullable String value) {
      synchronized (this) {
        mModified.put(key, value);
        return this;
      }
    }

    // 与 commit 逻辑相同， 只不过是一个异步函数
    public void apply() { ... }

    // 先将 mModified 数据存入内存，然后同步到文件中
    // 具体的操作就是在 enqueueDiskWrite 中调用 writeToFile 函数，会以 xml 的形式写入文件，当数据过多时，这里就会造成 bolck 了
    public boolean commit() {
      MemoryCommitResult mcr = commitToMemory();
      SharedPreferencesImpl.this.enqueueDiskWrite(mcr, null);
      try {
        mcr.writtenToDiskLatch.await();
      } catch (InterruptedException e) {
        return false;
      }
      notifyListeners(mcr);
      return mcr.writeToDiskResult;
    }
    ...
  }
}
```
由此可知，SharedPreferencesImpl 中在内存中的数据就是 mMap 这个变量，读取则是从 mMap 中读取，通过 EditorImpl 设置时则会先同步到 mMap，然后在序列化到本地文件中。
并且 SharedPreferences 中这的不能放太多数据，本身 xml 格式就是一个冗余严重的格式，如果数据过多，每次 commit 耗费的时间就非常可观了。

## ContextImpl
我们看到构造函数 SharedPreferencesImpl(File file, int mode) 是根据 File 来构造的，但是我们使用时 getSharedPreferences("keyzone", Context.MODE_PRIVATE) 却是根据 keyzone 来获取的，那这里是怎么关联起来的呢？这里就涉及到 ContextImpl 了。一些关键代码如下：
```
class ContextImpl extends Context {

    private static ArrayMap<String, ArrayMap<File, SharedPreferencesImpl>> sSharedPrefsCache;

    private ArrayMap<String, File> mSharedPrefsPaths;

     @Override
    public SharedPreferences getSharedPreferences(String name, int mode) {
        if (mPackageInfo.getApplicationInfo().targetSdkVersion <
                Build.VERSION_CODES.KITKAT) {
            if (name == null) {
                name = "null";
            }
        }

        File file;
        synchronized (ContextImpl.class) {
            if (mSharedPrefsPaths == null) {
                mSharedPrefsPaths = new ArrayMap<>();
            }
            file = mSharedPrefsPaths.get(name);
            if (file == null) {
                file = getSharedPreferencesPath(name);
                mSharedPrefsPaths.put(name, file);
            }
        }
        return getSharedPreferences(file, mode);
    }

    @Override
    public SharedPreferences getSharedPreferences(File file, int mode) {
        checkMode(mode);
        SharedPreferencesImpl sp;
        synchronized (ContextImpl.class) {
            final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();
            sp = cache.get(file);
            if (sp == null) {
                sp = new SharedPreferencesImpl(file, mode);
                cache.put(file, sp);
                return sp;
            }
        }
        if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
            getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
            // If somebody else (some other process) changed the prefs
            // file behind our back, we reload it.  This has been the
            // historical (if undocumented) behavior.
            sp.startReloadIfChangedUnexpectedly();
        }
        return sp;
    }

    private ArrayMap<File, SharedPreferencesImpl> getSharedPreferencesCacheLocked() {
        if (sSharedPrefsCache == null) {
            sSharedPrefsCache = new ArrayMap<>();
        }

        final String packageName = getPackageName();
        ArrayMap<File, SharedPreferencesImpl> packagePrefs = sSharedPrefsCache.get(packageName);
        if (packagePrefs == null) {
            packagePrefs = new ArrayMap<>();
            sSharedPrefsCache.put(packageName, packagePrefs);
        }

        return packagePrefs;
    }

    @Override
    public boolean moveSharedPreferencesFrom(Context sourceContext, String name) {
        synchronized (ContextImpl.class) {
            ...
        }
    }

    @Override
    public boolean deleteSharedPreferences(String name) {
        synchronized (ContextImpl.class) {
            ...
        }
    }

    private File getPreferencesDir() {
        synchronized (mSync) {
            if (mPreferencesDir == null) {
                mPreferencesDir = new File(getDataDir(), "shared_prefs");
            }
            return ensurePrivateDirExists(mPreferencesDir);
        }
    }

    @Override
    public File getSharedPreferencesPath(String name) {
        return makeFilename(getPreferencesDir(), name + ".xml");
    }
}
```
可以看出，ContextImpl 会根据 name 从 mSharedPrefsPaths 获取 File，然后根据 File 去实例化 SharedPreferencesImpl，那上边的问题就算是串起来了。而所有的 SharedPreferencesImpl 都会缓存到 sSharedPrefsCache 中。虽然 SharedPreferencesImpl 是懒加载，及只有在使用到的时候才会去实例化，但是实例化后就会一直缓存在内存中了，如果数据过多，则会造成内存一直占用，由此也可以得出，不要在 SharedPreferences 中存过多的数据。一些高频、但是占用空间小的则适合存放与 SharedPreferences。
