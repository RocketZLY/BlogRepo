# View绘制源码浅析(一）setContentView()和View显示到Window

### 概述

View的绘制流程大致可以分为两大块，一块是setContentView()和View显示在屏幕上这个整体流程的梳理，另外一块是measure、layout、draw细节的实现，由于内容比较多所以我准备分两篇博客讲述。

那么本篇先从整体入手，分析下setContentView()和View显示到屏幕这个流程，掌握一个大体流程。测量、布局、绘制的话将在第二篇介绍。

本文的源码基于API27。



### setContentView()

`setContentView()`见名知义，就是将我们的布局设置到id为content的布局上并不包含View的绘制流程，接下来看下实现细节。

```java
//MainActivity.java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        
    }
}

//AppCompatActivity.java
@Override
public void setContentView(@LayoutRes int layoutResID) {
    getDelegate().setContentView(layoutResID);
}

//AppCompatActivity.java
@NonNull
public AppCompatDelegate getDelegate() {
    if (mDelegate == null) {
        mDelegate = AppCompatDelegate.create(this, this);
    }
    return mDelegate;
}

//AppCompatDelegate.java
public static AppCompatDelegate create(Activity activity, AppCompatCallback callback) {
    return create(activity, activity.getWindow(), callback);//这里需要注意他是把activity.getWindow()传入到了后面，也就是PhoneWindow
}

//AppCompatDelegate.java
private static AppCompatDelegate create(Context context, Window window,
                                        AppCompatCallback callback) {
    if (Build.VERSION.SDK_INT >= 24) {
        return new AppCompatDelegateImplN(context, window, callback);
    } else if (Build.VERSION.SDK_INT >= 23) {
        return new AppCompatDelegateImplV23(context, window, callback);
    } else {
        return new AppCompatDelegateImplV14(context, window, callback);
    }
}

//AppCompatDelegateImplV9.java 
//最终是调到这个类的setContentView方法
@Override
public void setContentView(int resId) {
    ensureSubDecor();//确保DecorView相关布局初始化完成
    ViewGroup contentParent = (ViewGroup) mSubDecor.findViewById(android.R.id.content);//找到id为content的view
    contentParent.removeAllViews();
    LayoutInflater.from(mContext).inflate(resId, contentParent);//通过LayoutInflater直接把我们的布局添加到id为content的布局上
    mOriginalWindowCallback.onContentChanged();
}
```

主要的流程就是找到`AppCompatDelegate`的实现类，然后最终调到了`AppCompatDelegateImplV9`的`setContentView()`方法，接下来分为两步。

1. 通过`ensureSubDecor()`方法确保`DecorView`相关布局初始化完成。
2. 找到`DecorView`中id为content的布局，将我们自己的布局inflater到content上。

这里先说第一步`ensureSubDecor()`

```java
//AppCompatDelegateImplV9.java
private void ensureSubDecor() {
    if (!mSubDecorInstalled) {
        mSubDecor = createSubDecor();
        ...
    }
}

private ViewGroup createSubDecor() {
    TypedArray a = mContext.obtainStyledAttributes(R.styleable.AppCompatTheme);
	//拿到各种Feature进行相应的标记处理
    if (a.getBoolean(R.styleable.AppCompatTheme_windowNoTitle, false)) {
        requestWindowFeature(Window.FEATURE_NO_TITLE);//这个标记熟悉吧no_title
    } else if (a.getBoolean(R.styleable.AppCompatTheme_windowActionBar, false)) {
        // Don't allow an action bar if there is no title.
        requestWindowFeature(FEATURE_SUPPORT_ACTION_BAR);
    }
    if (a.getBoolean(R.styleable.AppCompatTheme_windowActionBarOverlay, false)) {
        requestWindowFeature(FEATURE_SUPPORT_ACTION_BAR_OVERLAY);
    }
    if (a.getBoolean(R.styleable.AppCompatTheme_windowActionModeOverlay, false)) {
        requestWindowFeature(FEATURE_ACTION_MODE_OVERLAY);
    }
    mIsFloating = a.getBoolean(R.styleable.AppCompatTheme_android_windowIsFloating, false);
    a.recycle();

    // Now let's make sure that the Window has installed its decor by retrieving it
    mWindow.getDecorView();//确保decorview创建了
    final LayoutInflater inflater = LayoutInflater.from(mContext);
    ViewGroup subDecor = null;
    //省略了很多条件判断，其实是根据各种Feature条件创建不同的subDecor
    subDecor = (ViewGroup) inflater.inflate(R.layout.xxx, null);
    // 将subDecor设置到Decorview上
    mWindow.setContentView(subDecor);
    return subDecor;
}
```

这里先拿到各种Feature其中有个我们比较熟悉的`FEATURE_NO_TITLE`。平时调用`requestWindowFeature()`的时候我们都是在`setContentView()`前面，看到这里你也应该知道为啥了，因为`setContentView()`方法中会处理相关逻辑，所以需要在他之前调用。

这里的`mWindow`是之前通过`activity.getWindow()`方法传进来的，具体实现类是`PhoneWindow`，调用`getDecorView()`确保`DecorView`初始化了，然后根据各种Feature条件创建不同的subDecor通过`setContentView()`方法添加到`DecorView`上。

接下来第二步`inflate`我们的布局到id位content的view上。

```java
    //LayoutInflater.java
    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
            return inflate(resource, root, root != null);
    }

    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
		...
        final XmlResourceParser parser = res.getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }

    public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
           	...
            final Context inflaterContext = mContext;
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context) mConstructorArgs[0];
            mConstructorArgs[0] = inflaterContext;
            View result = root;

            try {
                // Look for the root node.
                int type;
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {//拿到第一个节点
                    // Empty
                }

                if (type != XmlPullParser.START_TAG) {//如果不是开始标签报错
                    throw new InflateException(parser.getPositionDescription()
                            + ": No start tag found!");
                }

                final String name = parser.getName();//拿到节点名字
                if (TAG_MERGE.equals(name)) {//merge单独处理
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }

                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
                    // Temp is the root view that was found in the xml
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);//创建xml中根布局
                    ViewGroup.LayoutParams params = null;

                    if (root != null) {
                        if (DEBUG) {
                            System.out.println("Creating params from root: " +
                                    root);
                        }
                        // Create layout params that match root, if supplied
                        params = root.generateLayoutParams(attrs);//根据父布局创建xml中根布局的lp，因为自己的lp是跟父布局有关的。
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }

                    if (DEBUG) {
                        System.out.println("-----> start inflating children");
                    }

                    // Inflate all children under temp against its context.
                    rInflateChildren(parser, temp, attrs, true);//inflate temp的子布局，具体实现就是递归的把view添加到temp上

                    if (DEBUG) {
                        System.out.println("-----> done inflating children");
                    }

                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }

                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }

            } catch (XmlPullParserException e) {
                final InflateException ie = new InflateException(e.getMessage(), e);
                ie.setStackTrace(EMPTY_STACK_TRACE);
                throw ie;
            } catch (Exception e) {
                final InflateException ie = new InflateException(parser.getPositionDescription()
                        + ": " + e.getMessage(), e);
                ie.setStackTrace(EMPTY_STACK_TRACE);
                throw ie;
            } finally {
                // Don't retain static reference on context.
                mConstructorArgs[0] = lastContext;
                mConstructorArgs[1] = null;

                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
            }

            return result;
        }
    }


```

inflate比较简单，根据xml解析拿到xml中根布局`temp`，然后通过`root.generateLayoutParams(attrs)`创建lp，之所以要这样是因为每个view的lp的创建都是跟父布局有关的，比如root是`LinearLayout`那么创建的就是`LinearLayout.LayoutParams`并初始化`weight`和`gravity`属性，而如果root是`RelativeLayout`那么创建的是`RelativeLayout.LayoutParams`并初始化跟各个布局的约束关系。接下来再通过`rInflateChildren(parser, temp, attrs, true)`将temp的子布局inflate到temp上，最后我们把temp添加到root上。

这里我们可以看下`LinearLayout.generateLayoutParams()`方法

```java
	//LinearLayout.java
	@Override
    public LayoutParams generateLayoutParams(AttributeSet attrs) {
        return new LinearLayout.LayoutParams(getContext(), attrs);
    }

	//LinearLayout.LayoutParams 内部类
    public LayoutParams(Context c, AttributeSet attrs) {
        super(c, attrs);//这里还调用了父类MarginLayoutParams的构造方
        TypedArray a =
            c.obtainStyledAttributes(attrs, com.android.internal.R.styleable.LinearLayout_Layout);

        weight = a.getFloat(com.android.internal.R.styleable.LinearLayout_Layout_layout_weight, 0);//获取xml中weight属性
        gravity = a.getInt(com.android.internal.R.styleable.LinearLayout_Layout_layout_gravity, -1);//获取layout_gravity属性

        a.recycle();
    }

    //ViewGroup.MarginLayoutParams 内部类 初始化margin属性
    public MarginLayoutParams(Context c, AttributeSet attrs) {
        super();

        TypedArray a = c.obtainStyledAttributes(attrs, R.styleable.ViewGroup_MarginLayout);
        setBaseAttributes(a,
                          R.styleable.ViewGroup_MarginLayout_layout_width,
                          R.styleable.ViewGroup_MarginLayout_layout_height);//调用父类ViewGroup.LayoutParams的setBaseAttributes()读取最基础的宽高属性
        //下面就是读取xml各种margin属性
        int margin = a.getDimensionPixelSize(
            com.android.internal.R.styleable.ViewGroup_MarginLayout_layout_margin, -1);
        if (margin >= 0) {
            leftMargin = margin;
            topMargin = margin;
            rightMargin= margin;
            bottomMargin = margin;
        } else {
            int horizontalMargin = a.getDimensionPixelSize(
                R.styleable.ViewGroup_MarginLayout_layout_marginHorizontal, -1);
            int verticalMargin = a.getDimensionPixelSize(
                R.styleable.ViewGroup_MarginLayout_layout_marginVertical, -1);

            if (horizontalMargin >= 0) {
                leftMargin = horizontalMargin;
                rightMargin = horizontalMargin;
            } else {
                leftMargin = a.getDimensionPixelSize(
                    R.styleable.ViewGroup_MarginLayout_layout_marginLeft,
                    UNDEFINED_MARGIN);
                if (leftMargin == UNDEFINED_MARGIN) {
                    mMarginFlags |= LEFT_MARGIN_UNDEFINED_MASK;
                    leftMargin = DEFAULT_MARGIN_RESOLVED;
                }
                rightMargin = a.getDimensionPixelSize(
                    R.styleable.ViewGroup_MarginLayout_layout_marginRight,
                    UNDEFINED_MARGIN);
                if (rightMargin == UNDEFINED_MARGIN) {
                    mMarginFlags |= RIGHT_MARGIN_UNDEFINED_MASK;
                    rightMargin = DEFAULT_MARGIN_RESOLVED;
                }
            }

            startMargin = a.getDimensionPixelSize(
                R.styleable.ViewGroup_MarginLayout_layout_marginStart,
                DEFAULT_MARGIN_RELATIVE);
            endMargin = a.getDimensionPixelSize(
                R.styleable.ViewGroup_MarginLayout_layout_marginEnd,
                DEFAULT_MARGIN_RELATIVE);

            if (verticalMargin >= 0) {
                topMargin = verticalMargin;
                bottomMargin = verticalMargin;
            } else {
                topMargin = a.getDimensionPixelSize(
                    R.styleable.ViewGroup_MarginLayout_layout_marginTop,
                    DEFAULT_MARGIN_RESOLVED);
                bottomMargin = a.getDimensionPixelSize(
                    R.styleable.ViewGroup_MarginLayout_layout_marginBottom,
                    DEFAULT_MARGIN_RESOLVED);
            }

            if (isMarginRelative()) {
                mMarginFlags |= NEED_RESOLUTION_MASK;
            }
        }

        final boolean hasRtlSupport = c.getApplicationInfo().hasRtlSupport();
        final int targetSdkVersion = c.getApplicationInfo().targetSdkVersion;
        if (targetSdkVersion < JELLY_BEAN_MR1 || !hasRtlSupport) {
            mMarginFlags |= RTL_COMPATIBILITY_MODE_MASK;
        }

        // Layout direction is LTR by default
        mMarginFlags |= LAYOUT_DIRECTION_LTR;

        a.recycle();
    }

	//ViewGroup.LayoutParams 内部类
    protected void setBaseAttributes(TypedArray a, int widthAttr, int heightAttr) {//读取宽高
        width = a.getLayoutDimension(widthAttr, "layout_width");
        height = a.getLayoutDimension(heightAttr, "layout_height");
    }

		

```

可以看到`LinearLayout.LayoutParams`有两层继承关系，`LinearLayout.LayoutParams`负责拿到`LinearLayout`的特有属性，`ViewGroup.MarginLayoutParams`负责拿到`margin`属性，`ViewGroup.LayoutParams`负责拿到宽高。一般的`ViewGroup`基本都是实现自己的lp然后继承`ViewGroup.MarginLayoutParams`。

那么对`activity.setContentView()`总结成一句话就是完成DecorView相关布局初始化，然后将我们的布局`inflater`到DecorView中id为content的ViewGroup上并初始化好对应的`LayoutParams`，到此布局的添加和xml中相关属性的初始化完成，接下来在看他是如何显示到屏幕上的。



### View显示到屏幕上

View的显示是Activity收到Resume事件以后，这个事件其实是AMS发送给客户端的，在收到后会依次执行测量、布局、绘制流程，并将PhoneWindow添加到屏幕上。

不过再说`Resume`事件前，我们先看下是如何监听的。是在`ActivityThread.attach()`方法中添加的。

```java
final ApplicationThread mAppThread = new ApplicationThread();
private void attach(boolean system) {
    ···
    sCurrentActivityThread = this;
    mSystemThread = system;
    final IActivityManager mgr = ActivityManager.getService();//拿到ams
    try {
        mgr.attachApplication(mAppThread);//将mAppThread设置给了ams 类似于setOnClickListener设置监听
    } catch (RemoteException ex) {
        throw ex.rethrowFromSystemServer();
    }
    ···
}
```

`mAppThread`其实就是个binder对象，通过`ActivityManager.getService()`拿到系统进程的AMS然后调用`attachApplication()`把客户端的监听`mAppThread`设置给他。

这里我们看下`ApplicationThread`这个接受`AMS`事件的监听

```java
private class ApplicationThread extends IApplicationThread.Stub {
    ...//省略多个schedulexxxActivity()方法    
    public final void scheduleResumeActivity(IBinder token, int processState,
                                             boolean isForward, Bundle resumeArgs) {//收到AMS的Resume事件
        int seq = getLifecycleSeq();
        if (DEBUG_ORDER) Slog.d(TAG, "resumeActivity " + ActivityThread.this
                                + " operation received seq: " + seq);
        updateProcessState(processState, false);
        sendMessage(H.RESUME_ACTIVITY, token, isForward ? 1 : 0, 0, seq);//发送事件
    }
    
    private void sendMessage(int what, Object obj, int arg1, int arg2, int seq) {
        if (DEBUG_MESSAGES) Slog.v(
            TAG, "SCHEDULE " + mH.codeToString(what) + " arg1=" + arg1 + " arg2=" + arg2 +
            "seq= " + seq);
        Message msg = Message.obtain();
        msg.what = what;
        SomeArgs args = SomeArgs.obtain();
        args.arg1 = obj;
        args.argi1 = arg1;
        args.argi2 = arg2;
        args.argi3 = seq;
        msg.obj = args;
        mH.sendMessage(msg);//发送到主线程Handler
    }
    ...  
    
}
```

是有个`scheduleResumeActivity()`方法接受Resume事件，然后由于是进程间通信所以该方法运行在binder线程池上，因此通过`sendMessage()`到主线程`Handler`执行相应逻辑。

```java
private class H extends Handler {
    ...
    case RESUME_ACTIVITY:
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityResume");
    SomeArgs args = (SomeArgs) msg.obj;
    handleResumeActivity((IBinder) args.arg1, true, args.argi1 != 0, true,
                         args.argi3, "RESUME_ACTIVITY");//真正处理Resume事件的方法
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    break;
    ...
}

```

所以Resume事件实际上调到了`ActivityThread.handleResumeActivity()`方法

```java
    final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
        ···
        ActivityClientRecord r = mActivities.get(token);
        r = performResumeActivity(token, clearHide, reason);//执行activity的onResume方法
        if (r != null) {
            if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                if (a.mVisibleFromClient) {
                    if (!a.mWindowAdded) {
                        a.mWindowAdded = true;
                        wm.addView(decor, l);//将decorview添加到wm 记住哦后面的view在这个流程中都是DecorView
                    }
                }
            }
        }
        ···
    }

```

最终是调到`WindowManagerGlobal.addView()`

```java
    public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
            // If this is a panel window, then find the window it is being
            // attached to for future reference.
            if (wparams.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
                    wparams.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
                final int count = mViews.size();
                for (int i = 0; i < count; i++) {
                    if (mRoots.get(i).mWindow.asBinder() == wparams.token) {
                        panelParentView = mViews.get(i);
                    }
                }
            }

            root = new ViewRootImpl(view.getContext(), display);//创建ViewRootImpl

            view.setLayoutParams(wparams);

            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);

            // do this last because it fires off messages to start doing things
            try {
                root.setView(view, wparams, panelParentView);//调用ViewRootImpl.setView()传入DecorView
            } catch (RuntimeException e) {
                // BadTokenException or InvalidDisplayException, clean up.
                if (index >= 0) {
                    removeViewLocked(index, true);
                }
                throw e;
            }
        }
    }
```

然后创建了一个`ViewRootImpl`调到`ViewRootImpl.setView()`

```java
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                mView = view;

                // Schedule the first layout -before- adding to the window
                // manager, to make sure we do the relayout before receiving
                // any other events from the system.
                requestLayout();//进行measure、layout、draw
                try {
                    mOrigWindowType = mWindowAttributes.type;
                    mAttachInfo.mRecomputeGlobalAttributes = true;
                    collectViewAttributes();
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mInputChannel);//将view添加到屏幕上显示
                } catch (RemoteException e) {
                    mAdded = false;
                    mView = null;
                    mAttachInfo.mRootView = null;
                    mInputChannel = null;
                    mFallbackEventHandler.setView(null);
                    unscheduleTraversals();
                    setAccessibilityFocus(null, null);
                    throw new RuntimeException("Adding window failed", e);
                } finally {
                    if (restore) {
                        attrs.restore();
                    }
                }
                view.assignParent(this);//给decorView添加父对象ViewRootImpl
            }   
        }
    }
```

先调用了`requestLayout()`对布局进行测量、布局、绘制，然后通过`mWindowSession.addToDisplay()`将View显示到window上完成显示。

这里我们看下`requestLayout()`

```java
    @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();//检查是不是主线程
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }

    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);//运行mTraversalRunnable
        }
    }

    final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }
    final TraversalRunnable mTraversalRunnable = new TraversalRunnable();

    void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

            if (mProfile) {
                Debug.startMethodTracing("ViewAncestor");
            }

            performTraversals();//真正的traversals方法

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
    }
```

最终走到了`performTraversals()`

```java
private void performTraversals() {
    measureHierarchy(host, lp, res,
                    desiredWindowWidth, desiredWindowHeight)//实际内部也是performMeasure()
    performLayout(lp, mWidth, mHeight);
    performDraw();
}

    private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,
                                     final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {
        int childWidthMeasureSpec;
        int childHeightMeasureSpec;
        childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
        childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
        return windowSizeMayChange;
    }


```

看到这里可以发现其实`requestLayout()`实际上是分别调用了`performMeasure()`、`performLayout()`、`performDraw()`去执行的测量、布局、绘制，具体实现细节将会在第二篇分析。

这里我们总结下view显示到屏幕上的调用链

`ActivityThread.handleResumeActivity()`->`WindowManager.addView()`->`WindowManagerImpl.addView()`->`WindowManagerGlobal.addView()`->`ViewRootImpl.setView()`->`IWindowSession.addToDisplay()`

在`ViewRootImpl.setView()`中完成了View的测量、布局、绘制过程，然后通过`IWindowSession.addToDisplay()`使VIew显示出来。



### 总结

梳理下整体流程

1. `setContentView`完成`DecorView`相关布局初始化并将我们的布局通过`LayoutInflater.inflate()`方法添加到id为Content的ViewGroup上。
2. 在收到AMS的Resume事件，最终调到`ViewRootImpl.setView()`当中通过`ViewRootImpl.requestLayout()`完成View的测量、布局、绘制，然后在通过`IWindowSession.addToDisplay()`显示到屏幕上。

