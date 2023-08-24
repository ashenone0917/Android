## 1. ViewModel的作用
ViewModel是以生命周期的方式存储和管理界面相关的数据。当系统销毁或重新创建Activity/Fragment的时候，那么存储在其中的数据都会消失，对于简单的数据，Activity可以通过onSaveInstanceState()方法从 onCreate() 中的捆绑包恢复其数据，但此方法仅适合可以序列化再反序列化的少量数据，而不适合数量可能较大的数据,ViewModel的出现，正好弥补了这一个不足。
## 2. 本文所使用相关依赖
```
implementation 'androidx.lifecycle:lifecycle-runtime-ktx:2.6.1'
implementation 'androidx.activity:activity-compose:1.7.0'
```

## 测试代码，使用ViewModel实现一个简单的计数器
```kotlin
package com.example.myapplication

import android.os.Handler
import android.os.Looper
import androidx.lifecycle.ViewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.update

class MyViewModel : ViewModel() {
    val count = MutableStateFlow(0)
    private val handler = Handler(Looper.getMainLooper())
    private val runnable = object : Runnable {
        override fun run() {
            count.update {
                it.inc()
            }
            handler.postDelayed(this, 1000L)
        }
    }

    fun run() {
        runnable.run()
    }

    override fun onCleared() {
        super.onCleared()
        handler.removeCallbacksAndMessages(null)
    }
}
```

```kotlin
package com.example.myapplication

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.viewModels
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.tooling.preview.Preview
import com.example.myapplication.ui.theme.MyApplicationTheme
import kotlinx.coroutines.flow.asStateFlow

class MainActivity : ComponentActivity() {
    private val myViewModel: MyViewModel by viewModels()
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MyApplicationTheme {
                // A surface container using the 'background' color from the theme
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = MaterialTheme.colorScheme.background
                ) {
                    val count by myViewModel.count.collectAsState()
                    Greeting("count = $count")
                }
            }
        }
        myViewModel.run()
    }
}

@Composable
fun Greeting(name: String, modifier: Modifier = Modifier) {
    Text(
        text = "Hello $name!",
        modifier = modifier
    )
}
```
上面的代码在旋转销毁activity后仍会保存count的数据，从而使得旋转前后计数的延续

## 3.ViewModel是如何创建的？

### 3.1 先创建ViewModelProvider
```kotlin
private val myViewModel: MyViewModel by viewModels()//委托给viewModels()创建的辅助对象

//viewModel()内部使用Lazy的方式创建ViewModelLazy继承自MyViewModel，只有当使用到的时候才会创建
@MainThread
public inline fun <reified VM : ViewModel> ComponentActivity.viewModels(
    noinline extrasProducer: (() -> CreationExtras)? = null,
    noinline factoryProducer: (() -> Factory)? = null
): Lazy<VM> {
    val factoryPromise = factoryProducer ?: {
        defaultViewModelProviderFactory
    }

    return ViewModelLazy(
        VM::class,
        { viewModelStore },
        factoryPromise,
        { extrasProducer?.invoke() ?: this.defaultViewModelCreationExtras }
    )
}
//--------------------------------------------------------
public class ViewModelLazy<VM : ViewModel> @JvmOverloads constructor(
    private val viewModelClass: KClass<VM>,
    private val storeProducer: () -> ViewModelStore,
    private val factoryProducer: () -> ViewModelProvider.Factory,
    private val extrasProducer: () -> CreationExtras = { CreationExtras.Empty }
) : Lazy<VM> {
    private var cached: VM? = null

    override val value: VM
        get() {
            val viewModel = cached
            return if (viewModel == null) {
                val factory = factoryProducer()
                val store = storeProducer()
                ViewModelProvider( //ViewModelProvider生成ViewModel
                    store,
                    factory,
                    extrasProducer()
                ).get(viewModelClass.java).also { //此get内部创建ViewModel
                    cached = it
                }
            } else {
                viewModel
            }
        }

    override fun isInitialized(): Boolean = cached != null
}
//--------------------------------------------------------
public open class ViewModelProvider
/**
 * Creates a ViewModelProvider
 *
 * @param store `ViewModelStore` where ViewModels will be stored.
 * @param factory factory a `Factory` which will be used to instantiate new `ViewModels`
 * @param defaultCreationExtras extras to pass to a factory
 */
@JvmOverloads
constructor(
    private val store: ViewModelStore,
    private val factory: Factory,
    private val defaultCreationExtras: CreationExtras = CreationExtras.Empty,
) {
  //...
}
```
从上可以看出是由ViewModelProvider创建的ViewModel，而ViewModelProvider包含4个参数分别是
- KClass<VM> kotlin class类型
- ViewModelStore //用于存储ViewModel
- ViewModelProvider.Factory //factory用于创建ViewModel
- CreationExtras//额外参数，默认为空
所以看一看两个重要的参数ViewModelStore和ViewModelProvider.Factory

#### 3.1.1 ViewModelStore参数
{ viewModelStore }调用了getViewModelStore()
```java
public ViewModelStore getViewModelStore() {
        if (getApplication() == null) {
            throw new IllegalStateException("Your activity is not yet attached to the "
                    + "Application instance. You can't request ViewModel before onCreate call.");
        }
        ensureViewModelStore();
        return mViewModelStore;
}

@SuppressWarnings("WeakerAccess") /* synthetic access */
void ensureViewModelStore() {
  if (mViewModelStore == null) {
    //
    NonConfigurationInstances nc =
      (NonConfigurationInstances) getLastNonConfigurationInstance();
    if (nc != null) {
      // Restore the ViewModelStore from NonConfigurationInstances
      mViewModelStore = nc.viewModelStore;
    }
    if (mViewModelStore == null) {
      mViewModelStore = new ViewModelStore();
    }
  }
}
```
可以发现getViewModelStore()其实就是返回了ViewModelStore对象，如果存在NonConfigurationInstances对象就使用已有的，没有就new一个新的，接下来看看NonConfigurationInstances是个什么
```java
static final class NonConfigurationInstances {
        Object custom;
        ViewModelStore viewModelStore;
}
```
NonConfigurationInstances是一个静态对象，里面存储了ViewModelStore，接着看看ViewModelStore的具体内容
```java
public class ViewModelStore {

    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }

    final ViewModel get(String key) {
        return mMap.get(key);
    }

    Set<String> keys() {
        return new HashSet<>(mMap.keySet());
    }

    /**
     *  Clears internal storage and notifies ViewModels that they are no longer used.
     */
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.clear();
        }
        mMap.clear();
    }
}
```
ViewModelStore中包含了一个用于存储ViewModel的hashmap，而key值后面可以看到
#### 3.1.2 Factory参数
```java
public inline fun <reified VM : ViewModel> ComponentActivity.viewModels(
    noinline extrasProducer: (() -> CreationExtras)? = null,
    noinline factoryProducer: (() -> Factory)? = null
): Lazy<VM> {
    val factoryPromise = factoryProducer ?: {//当没有传入Factory的时候，使用的就是默认的factory
        defaultViewModelProviderFactory
    }

    return ViewModelLazy(
        VM::class,
        { viewModelStore },
        factoryPromise,
        { extrasProducer?.invoke() ?: this.defaultViewModelCreationExtras }
    )
}
//----------------------------------------------------------------------------
public ViewModelProvider.Factory getDefaultViewModelProviderFactory() {
        if (mDefaultFactory == null) {
            mDefaultFactory = new SavedStateViewModelFactory( //默认类型
                    getApplication(),
                    this,
                    getIntent() != null ? getIntent().getExtras() : null);
        }
        return mDefaultFactory;
}
```
当没有传入Factory的时候，使用的就是默认的factory，而defaultViewModelProviderFactory的类型是SavedStateViewModelFactory，具体不往里细看，但是后续毁掉用factory的create函数用于创建ViewModel
如果需要传入factory，可以自己实现一个fatcory后传入，如下
```kotlin
class TestFactory(private val context: Context) : ViewModelProvider.Factory {
    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        if (modelClass.isAssignableFrom(Test::class.java)) {
            return Test(context) as T
        }
        throw IllegalArgumentException("Unknown ViewModel class")
    }
}
```

### 3.2 创建ViewModel
接下来看看get(viewModelClass.java)也就是ViewModelProvider的get函数
```java
public open operator fun <T : ViewModel> get(modelClass: Class<T>): T {
        val canonicalName = modelClass.canonicalName //类的规范名称，其实就是类的完整类名，例如com.example.MyClass包含了包名+类名
            ?: throw IllegalArgumentException("Local and anonymous classes can not be ViewModels")
        return get("$DEFAULT_KEY:$canonicalName", modelClass)
}
//-----------------------------------------------------------------------
public open operator fun <T : ViewModel> get(key: String, modelClass: Class<T>): T {
        val viewModel = store[key]
        if (modelClass.isInstance(viewModel)) {
            (factory as? OnRequeryFactory)?.onRequery(viewModel)
            return viewModel as T//如果存在，直接返回
        } else {
            @Suppress("ControlFlowWithEmptyBody")
            if (viewModel != null) {
                // TODO: log a warning.
            }
        }
        val extras = MutableCreationExtras(defaultCreationExtras)
        extras[VIEW_MODEL_KEY] = key
        // AGP has some desugaring issues associated with compileOnly dependencies so we need to
        // fall back to the other create method to keep from crashing.
        return try {
            factory.create(modelClass, extras)//否则通过factory创建
        } catch (e: AbstractMethodError) {
            factory.create(modelClass)
        }.also { store.put(key, it) }//生成的ViewModel存入ViewModelStore的hashmap
}
```
- val viewModel = store[key]是从ViewModelStore中获取ViewModel
- key就是"$DEFAULT_KEY:$canonicalName"，canonicalName是由传入的类的名称
- factory.create(modelClass, extras)可以看到，如果store中没有对应类名的ViewModel则生成一个
- .also { store.put(key, it) }将生成的ViewModel存入ViewModelStore的hashmap后返回一个ViewModel

## 4.ViewModel如何在Activity销毁后仍保持着数据
通过上面可以看出ViewModel是从ViewModelStore中获取的，当getLastNonConfigurationInstance()能够返回NonConfigurationInstances对象时，ViewModelStore是从NonConfigurationInstances对象中获得的，说明可以推测出NonConfigurationInstances保存着上一个activity的ViewModelStore，所以接着看看getLastNonConfigurationInstance()
```java
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback,
        ContentCaptureManager.ContentCaptureClient {

        public Object getLastNonConfigurationInstance() {
            return mLastNonConfigurationInstances != null
                ? mLastNonConfigurationInstances.activity : null;
        }

        final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken,
            IBinder shareableActivityToken) {
        //...
        mLastNonConfigurationInstances = lastNonConfigurationInstances;
        //....
}
```
可以看到是在activityattch的时候重新设置的mLastNonConfigurationInstances，那么调用attach的地方是
```java
//这是启动activity的一步
public final Activity startActivityNow(Activity parent, String id,
            Intent intent, ActivityInfo activityInfo, IBinder token, Bundle state,
            Activity.NonConfigurationInstances lastNonConfigurationInstances, IBinder assistToken,
            IBinder shareableActivityToken) {
        ActivityClientRecord r = new ActivityClientRecord();
            r.token = token;
            r.assistToken = assistToken;
            r.shareableActivityToken = shareableActivityToken;
            r.ident = 0;
            r.intent = intent;
            r.state = state;
            r.parent = parent;
            r.embeddedID = id;
            r.activityInfo = activityInfo;
            r.lastNonConfigurationInstances = lastNonConfigurationInstances;//这里设置了lastNonConfigurationInstances
        if (localLOGV) {
            ComponentName compname = intent.getComponent();
            String name;
            if (compname != null) {
                name = compname.toShortString();
            } else {
                name = "(Intent " + intent + ").getComponent() returned null";
            }
            Slog.v(TAG, "Performing launch: action=" + intent.getAction()
                    + ", comp=" + name
                    + ", token=" + token);
        }
        // TODO(lifecycler): Can't switch to use #handleLaunchActivity() because it will try to
        // call #reportSizeConfigurations(), but the server might not know anything about the
        // activity if it was launched from LocalAcvitivyManager.
        return performLaunchActivity(r, null /* customIntent */);
}


 private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ...
        activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback,
                        r.assistToken);

        ...
        return activity;
}
```
可以NonConfigurationInstances对象保存在ActivityClientRecord中，然后在重启Activity中的attach()方法中将NonConfigurationInstances拿回来，从而实现了数据的不丢失，接下来看看数据是怎么保存的
既然要保存状态，那么必然是在onDestory()中，所以接下来看看performDestroyActivity
```java
//代码来源 https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/android/app/ActivityThread.java;l=5400?q=performDestroyActivity
/** Core implementation of activity destroy call. */
    void performDestroyActivity(ActivityClientRecord r, boolean finishing,//r是ActivityClientRecord类型
            int configChanges, boolean getNonConfigInstance, String reason) {
        Class<? extends Activity> activityClass;
        if (localLOGV) Slog.v(TAG, "Performing finish of " + r);
        activityClass = r.activity.getClass();
        r.activity.mConfigChangeFlags |= configChanges;
        if (finishing) {
            r.activity.mFinished = true;
        }

        performPauseActivityIfNeeded(r, "destroy");

        if (!r.stopped) {
            callActivityOnStop(r, false /* saveState */, "destroy");
        }
        if (getNonConfigInstance) {
            try {
                r.lastNonConfigurationInstances = r.activity.retainNonConfigurationInstances();//关键代码,在ActivityClientRecord中保存了lastNonConfigurationInstances，用于后续创建activity时恢复
            } catch (Exception e) {
                if (!mInstrumentation.onException(r.activity, e)) {
                    throw new RuntimeException("Unable to retain activity "
                            + r.intent.getComponent().toShortString() + ": " + e.toString(), e);
                }
            }
        }
        try {
            r.activity.mCalled = false;
            mInstrumentation.callActivityOnDestroy(r.activity);
            if (!r.activity.mCalled) {
                throw new SuperNotCalledException("Activity " + safeToComponentShortString(r.intent)
                        + " did not call through to super.onDestroy()");
            }
            if (r.window != null) {
                r.window.closeAllPanels();
            }
        } catch (SuperNotCalledException e) {
            throw e;
        } catch (Exception e) {
            if (!mInstrumentation.onException(r.activity, e)) {
                throw new RuntimeException("Unable to destroy activity "
                        + safeToComponentShortString(r.intent) + ": " + e.toString(), e);
            }
        }
        r.setState(ON_DESTROY);
        mLastReportedWindowingMode.remove(r.activity.getActivityToken());
        schedulePurgeIdler();
        synchronized (this) {
            if (mSplashScreenGlobal != null) {
                mSplashScreenGlobal.tokenDestroyed(r.token);
            }
        }
        // updatePendingActivityConfiguration() reads from mActivities to update
        // ActivityClientRecord which runs in a different thread. Protect modifications to
        // mActivities to avoid race.
        synchronized (mResourcesManager) {
            mActivities.remove(r.token);
        }
        StrictMode.decrementExpectedActivityCount(activityClass);
    }
```
关键代码r.lastNonConfigurationInstances = r.activity.retainNonConfigurationInstances();接着看看retainNonConfigurationInstances
```java
NonConfigurationInstances retainNonConfigurationInstances() {
        Object activity = onRetainNonConfigurationInstance();//这里获取到了activity，具体的实现在ComponentActivity
        HashMap<String, Object> children = onRetainNonConfigurationChildInstances();
        FragmentManagerNonConfig fragments = mFragments.retainNestedNonConfig();

        // We're already stopped but we've been asked to retain.
        // Our fragments are taken care of but we need to mark the loaders for retention.
        // In order to do this correctly we need to restart the loaders first before
        // handing them off to the next activity.
        mFragments.doLoaderStart();
        mFragments.doLoaderStop(true);
        ArrayMap<String, LoaderManager> loaders = mFragments.retainLoaderNonConfig();

        if (activity == null && children == null && fragments == null && loaders == null
                && mVoiceInteractor == null) {
            return null;
        }

        NonConfigurationInstances nci = new NonConfigurationInstances();
        nci.activity = activity;
        nci.children = children;
        nci.fragments = fragments;
        nci.loaders = loaders;
        if (mVoiceInteractor != null) {
            mVoiceInteractor.retainInstance();
            nci.voiceInteractor = mVoiceInteractor;
        }
        return nci;
    }
```
onRetainNonConfigurationInstance该方法在ComponentActivity的实现如下:
```java
public final Object onRetainNonConfigurationInstance() {
        // Maintain backward compatibility.
        Object custom = onRetainCustomNonConfigurationInstance();

        ViewModelStore viewModelStore = mViewModelStore;
        if (viewModelStore == null) {
            // No one called getViewModelStore(), so see if there was an existing
            // ViewModelStore from our last NonConfigurationInstance
            NonConfigurationInstances nc =
                    (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                viewModelStore = nc.viewModelStore;
            }
        }

        if (viewModelStore == null && custom == null) {
            return null;
        }

        NonConfigurationInstances nci = new NonConfigurationInstances();//创建了一个NonConfigurationInstances
        nci.custom = custom;
        nci.viewModelStore = viewModelStore;//保存了viewModelStore
        return nci;
    }
```

## 5.ViewModel是何时回收的
我们可以看到在创建ComponentActivity的时候，会添加一个Lifecycle的Observer。如下：
```java
public ComponentActivity() {
       	...
        getLifecycle().addObserver(new LifecycleEventObserver() {
            @Override
            public void onStateChanged(@NonNull LifecycleOwner source,
                    @NonNull Lifecycle.Event event) {
                if (event == Lifecycle.Event.ON_DESTROY) {////旋转屏幕销毁activity时会触发Lifecycle.Event.ON_DESTROY
                    if (!isChangingConfigurations()) {//旋转屏幕销毁activity时不会进入此if内的代码去清理ViewModelStore，从而保存ViewModelStore，原因在下面
                        getViewModelStore().clear();
                    }
                }
            }
        });
		...
}
```
在接收到Lifecycle.Event.ON_DESTROY，onDestroy的事件时候，会去调用 getViewModelStore().clear();方法。getViewModelStore我们知道其实就是ViewModeStore,clear方法如下：
```java
public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.clear();
        }
        mMap.clear();
}
```
其实就是遍历map，把数据清空。
在旋转屏幕销毁activity的时候不会清空ViewModelStore，可以看到是做了这么一个判断if (!isChangingConfigurations()) ，内部代码如下
```java
public boolean isChangingConfigurations() {
        return mChangingConfigurations;
}
```
此时会判断是否重置配置，
而旋转屏幕销毁activity的时候会自动调用handleRelaunchActivity重启启动activity，此时会把mChangingConfigurations设置为true，从而不会清空ViewModelStore，数据得以继续使用
```java
Override
    public void handleRelaunchActivity(ActivityClientRecord tmp,
            PendingTransactionActions pendingActions) {
	//...
	r.activity.mChangingConfigurations = true;
	//...
}
```
如果使用finish等方式销毁activity则不会调用handleRelaunchActivity，那么就无法保存数据，且会清空ViewModelStore

## 6.ViewModel的生命周期
ViewModel 对象存在的时间范围是获取 ViewModel 时传递给 ViewModelProvider 的 Lifecycle。ViewModel 将一直留在内存中，直到限定其存在时间范围的 Lifecycle 永久消失：对于 Activity，是在 Activity 完成时；而对于 Fragment，是在 Fragment 分离时。下面提供一张官网的图，可以很清晰的看到生命周期

![ViewModelCycle](https://github.com/ashenone0917/image/blob/main/ViewModelCycle.jpg)

## 简短总结
ViewModel在activity结束后会有application暂存，在activity重建后

参考： https://juejin.cn/post/6920122401678163976

