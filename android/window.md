<h1 style="text-align: center; width: 100%">Android Window</h1>
<p style="font-size: small; text-align: center; width: 100%"><span>2020-11-02 | Haidy</span></p>

---
## 简介
Android手机屏幕的内容，是由一个个Window组成的，比如状态栏，应用窗口、导航栏等等，Window有自己的优先级，所以组合起来可以看着一层叠着一层。

一般一个应用常见的几种Window形式有`Activity`、`Dialog`和`PopupWindow`。

Window是由`WMS`(WindowManagerService)进行管理的。

Window是一个抽象类，它的具体实现为`PhoneWindow`。
Window是由`WindowManager`管理的，`WindowManager`是一个接口，它的实现类为`WindowManagerImpl`，具体地逻辑操作又实现在`WindowManagerGlobal`中。

#### Window层级

Window的显示层级是通过`Z-order`的概念来管理的，`Z-order`大的会覆盖比较小的上面。
在Android中，可以通过设置`WindowManager.LayoutParams`的`type`的值来设置Windows的值，目前系统预设了一部分的Z-order的值，可以在[WindowManager.LayoutParams](https://developer.android.google.cn/reference/android/view/WindowManager.LayoutParams?hl=en#TYPE_APPLICATION)。
下表为层级与应用的关系

|  Window                               | 层级(Z-order) |
|  ----                                 | ----         |
| normal application windows (普通应用)   | 1~99         |
| sub-windows (子Window)                 | 1000~1999    |
| system-specific windows (系统特定窗口)   | 1000~1999    |

### Window管理服务（WindowManagerService WMS）
Window管理服务（WindowManagerService）`WMS`是Android Framework层的一个用来管理Window的服务，由系统Zygote进程创建。应用程序通过`IWindowManager`这个Binder与`WMS`通信。

+ **Session**
`Session`是应用与WMS通信所需要的一个对象，Session类继承自`IWindowSession.Stub`，每个进程都最多有一个`Session`对象与`WMS`通信。
在客户端中，`Session`的Binder对象为`IWindowSession`，通过`WindowManagerGlobal.getWindowSession()`获取。

#### Window在Activity中的创建

通过跟踪`ActivityThread`的代码，我们发现在`performLaunchActivity`中，调用了`Activity`的`attach`方法，然后在这个方法中，创建了Window对象：
    
    Activity.java
    
    final void attach(Context context, ActivityThread aThread,
                Instrumentation instr, IBinder token, int ident,
                Application application, Intent intent, ActivityInfo info,
                CharSequence title, Activity parent, String id,
                NonConfigurationInstances lastNonConfigurationInstances,
                Configuration config, String referrer, IVoiceInteractor voiceInteractor,
                Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken) {
            attachBaseContext(context);
    
            mFragments.attachHost(null /*parent*/);
    
            // 通过activity和window对象，实例化一个PhoneWindow，正常情况下window这个对象都是空
            mWindow = new PhoneWindow(this, window, activityConfigCallback);
            mWindow.setWindowControllerCallback(mWindowControllerCallback);
            mWindow.setCallback(this);
            mWindow.setOnWindowDismissedCallback(this);
            mWindow.getLayoutInflater().setPrivateFactory(this);
            if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
                mWindow.setSoftInputMode(info.softInputMode);
            }
            if (info.uiOptions != 0) {
                mWindow.setUiOptions(info.uiOptions);
            }
            mUiThread = Thread.currentThread();
            ...
    }
    
    PhoneWindow.java
    
        public PhoneWindow(Context context, Window preservedWindow,
                ActivityConfigCallback activityConfigCallback) {
            this(context);
            // Only main activity windows use decor context, all the other windows depend on whatever
            // context that was given to them.
            mUseDecorContext = true;
            if (preservedWindow != null) {
                mDecor = (DecorView) preservedWindow.getDecorView();
                mElevation = preservedWindow.getElevation();
                mLoadElevation = false;
                mForceDecorInstall = true;
                // If we're preserving window, carry over the app token from the preserved
                // window, as we'll be skipping the addView in handleResumeActivity(), and
                // the token will not be updated as for a new window.
                getAttributes().token = preservedWindow.getAttributes().token;
            }
            // Even though the device doesn't support picture-in-picture mode,
            // an user can force using it through developer options.
            boolean forceResizable = Settings.Global.getInt(context.getContentResolver(),
                    DEVELOPMENT_FORCE_RESIZABLE_ACTIVITIES, 0) != 0;
            mSupportsPictureInPicture = forceResizable || context.getPackageManager().hasSystemFeature(
                    PackageManager.FEATURE_PICTURE_IN_PICTURE);
            mActivityConfigCallback = activityConfigCallback;
        }

看到这里我们已经知道了Window是在什么时候创建的了，但是现在还是很好奇我们在调用`setContentView`的时候，我们自己的界面是怎么显示出来的。
我们接下来继续跟代码：
    
    Activity.java
        
        public void setContentView(@LayoutRes int layoutResID) {
            getWindow().setContentView(layoutResID);
            initWindowDecorActionBar();
        }
        
        public void setContentView(View view) {
            getWindow().setContentView(view);
            initWindowDecorActionBar();
        }
        
        ...        

这时候发现其实`Activity`的`setContentView`其实没有具体实现，而是有调用了`Window`的`setContentView`方法

    PhoneWindow.java
    
        public void setContentView(int layoutResID) {
            // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
            // decor, when theme attributes and the like are crystalized. Do not check the feature
            // before this happens.
            // mContentParent在第一次调setContentView肯定为null
            if (mContentParent == null) {
                // 这里比较重要，DecorView的初始化
                installDecor();
            } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
                mContentParent.removeAllViews();
            }
    
            if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
                final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                        getContext());
                transitionTo(newScene);
            } else {
                mLayoutInflater.inflate(layoutResID, mContentParent);
            }
            mContentParent.requestApplyInsets();
            final Callback cb = getCallback();
            if (cb != null && !isDestroyed()) {
                cb.onContentChanged();
            }
            mContentParentExplicitlySet = true;
        }
    
        private void installDecor() {
            mForceDecorInstall = false;
            if (mDecor == null) {
                // 获取DocorView对象
                mDecor = generateDecor(-1);
                mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
                mDecor.setIsRootNamespace(true);
                if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                    mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
                }
            } else {
                mDecor.setWindow(this);
            }
            if (mContentParent == null) {
                // 从mDecor中找到ContentParent布局并初始化，mContentParent就是ContentView的容器View
                mContentParent = generateLayout(mDecor);
                ...
        }

终于，我们看到了Window中有一个`mDecor`的View将我们创建的View加进去了，但是现在又很好奇`mDecor`是怎么产生以及它的作用，不过在此之前先看一下Window在Dialog中是怎么产生的

#### Window在Dialog中的创建

        Dialog(@NonNull Context context, @StyleRes int themeResId, boolean createContextThemeWrapper) {
            if (createContextThemeWrapper) {
                if (themeResId == Resources.ID_NULL) {
                    final TypedValue outValue = new TypedValue();
                    context.getTheme().resolveAttribute(R.attr.dialogTheme, outValue, true);
                    themeResId = outValue.resourceId;
                }
                mContext = new ContextThemeWrapper(context, themeResId);
            } else {
                mContext = context;
            }
            
            // 这里拿到了WindowManager的客户端代理对象
            mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
    
            // 新建一个Window
            final Window w = new PhoneWindow(mContext);
            mWindow = w;
            w.setCallback(this);
            w.setOnWindowDismissedCallback(this);
            w.setOnWindowSwipeDismissedCallback(() -> {
                if (mCancelable) {
                    cancel();
                }
            });
            w.setWindowManager(mWindowManager, null, null);
            w.setGravity(Gravity.CENTER);
    
            mListenersHandler = new ListenersHandler(this);
        }

可以看出来，Dialog的创建其实还是比较简单的，就是实例化一个PhoneWindow对象，然后对其初始化。下面看一下在Dialog显示的时候是怎么操作的，我们都知道Dialog是在调用`show`方法后显示出来的

        public void show() {
            if (mShowing) {
                if (mDecor != null) {
                    if (mWindow.hasFeature(Window.FEATURE_ACTION_BAR)) {
                        mWindow.invalidatePanelMenu(Window.FEATURE_ACTION_BAR);
                    }
                    mDecor.setVisibility(View.VISIBLE);
                }
                return;
            }
    
            mCanceled = false;
    
            if (!mCreated) {
                dispatchOnCreate(null);
            } else {
                // Fill the DecorView in on any configuration changes that
                // may have occured while it was removed from the WindowManager.
                final Configuration config = mContext.getResources().getConfiguration();
                mWindow.getDecorView().dispatchConfigurationChanged(config);
            }
    
            // 从Window中获取mDecor
            onStart();
            mDecor = mWindow.getDecorView();
    
            if (mActionBar == null && mWindow.hasFeature(Window.FEATURE_ACTION_BAR)) {
                final ApplicationInfo info = mContext.getApplicationInfo();
                mWindow.setDefaultIcon(info.icon);
                mWindow.setDefaultLogo(info.logo);
                mActionBar = new WindowDecorActionBar(this);
            }
    
            WindowManager.LayoutParams l = mWindow.getAttributes();
            boolean restoreSoftInputMode = false;
            if ((l.softInputMode
                    & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION) == 0) {
                l.softInputMode |=
                        WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION;
                restoreSoftInputMode = true;
            }
    
            // WindowManager调用了addView方法
            mWindowManager.addView(mDecor, l);
            if (restoreSoftInputMode) {
                l.softInputMode &=
                        ~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION;
            }
    
            mShowing = true;
    
            sendShowMessage();
        }

看完`show`方法的代码我们发现了`mWindowManager.addView(mDecor, l);`这一行，看这意思就是要显示`mDecor`了呗

#### WindowManager

WindowManager是一个接口，继承于`ViewManager`，它的实现类是`WindowManagerImpl`。在`WindowManagerImpl`中，有一个重要的全局变量`mGlobal`，它是`WindowManagerGlobal`类的一个实例。

    public interface ViewManager
    {
        public void addView(View view, ViewGroup.LayoutParams params);
        public void updateViewLayout(View view, ViewGroup.LayoutParams params);
        public void removeView(View view);
    }
    
在Activity调用setContentView后，最终会调用WindowManager的addView方法，它的实现在`WindowManagerImpl`中

    WindowManagerImpl.java
    
        @Override
        public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
            applyDefaultToken(params);
            // 调用了mGlobal的addView方法
            mGlobal.addView(view, params, mContext.getDisplayNoVerify(), mParentWindow,
                    mContext.getUserId());
        }
        
    WindowManagerGlobal.java
    
    public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow, int userId) {
        ...

        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
            ...

            root = new ViewRootImpl(view.getContext(), display);

            view.setLayoutParams(wparams);

            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);

            // do this last because it fires off messages to start doing things
            try {
                // ViewRootImpl设置DecorView
                root.setView(view, wparams, panelParentView, userId);
            } catch (RuntimeException e) {
                // BadTokenException or InvalidDisplayException, clean up.
                if (index >= 0) {
                    removeViewLocked(index, true);
                }
                throw e;
            }
        }
    }

这里又引出一个重要的类`ViewRootImpl`，从字面意思上看，它是一个根View的实现。
但其实`ViewRootImpl`并不是一个`View`，它实现了`ViewParent`接口，它的主要就是实现了View和WindowManager之间必须的协议，用作View与WMS的粘合剂

    WindowManagerGlobal.java

    public final class WindowManagerGlobal {
        ...

        // 所有Window对象中的View
        private final ArrayList<View> mViews = new ArrayList<View>();
        // 所有Window对象中的View所对应的ViewRootImpl
        private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
        // 所有Window对象中的View所对应的布局参数
        private final ArrayList<WindowManager.LayoutParams> mParams = new ArrayList<WindowManager.LayoutParams>();
        
        ...
    }

#### ViewRootImpl.setView
在`ViewRootImpl`初始化后，调用了`root.setView(view, wparams, panelParentView, userId);`方法将DecorView赋给内部的`mView`变量，
并开始测量、布局并绘制`DecorView`及其子View，下面是相关代码稍微有点长，不过关键的地方基本都有中文注释，并且方法调用都在下面依次贴了出来

    ViewRootImpl.java
    
        public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
                int userId) {
            synchronized (this) {
                if (mView == null) {
                    mView = view;
    
                    ...
    
                    // Schedule the first layout -before- adding to the window
                    // manager, to make sure we do the relayout before receiving
                    // any other events from the system.
                    // 请求布局
                    requestLayout();
                    InputChannel inputChannel = null;
                    if ((mWindowAttributes.inputFeatures
                            & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                        inputChannel = new InputChannel();
                    }
                    mForceDecorViewVisibility = (mWindowAttributes.privateFlags
                            & PRIVATE_FLAG_FORCE_DECOR_VIEW_VISIBILITY) != 0;
                    try {
                        mOrigWindowType = mWindowAttributes.type;
                        mAttachInfo.mRecomputeGlobalAttributes = true;
                        collectViewAttributes();
                        adjustLayoutParamsForCompatibility(mWindowAttributes);
                        // 将mWindow显示出来，这里mWindowSession是一个实现了IWindowSession的接口，它是Session客户端的Binder对象
                        // 调用addToDisplayAsUser之后，WMS将会添加mWindow
                        res = mWindowSession.addToDisplayAsUser(mWindow, mSeq, mWindowAttributes,
                                getHostVisibility(), mDisplay.getDisplayId(), userId, mTmpFrame,
                                mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                                mAttachInfo.mDisplayCutout, inputChannel,
                                mTempInsets, mTempControls);
                        setFrame(mTmpFrame);
                    } catch (RemoteException e) {
                        mAdded = false;
                        mView = null;
                        mAttachInfo.mRootView = null;
                        inputChannel = null;
                        mFallbackEventHandler.setView(null);
                        unscheduleTraversals();
                        setAccessibilityFocus(null, null);
                        throw new RuntimeException("Adding window failed", e);
                    } finally {
                        if (restore) {
                            attrs.restore();
                        }
                    }
    
                    ...
                    
                    // 设置当前View的mParent
                    view.assignParent(this);
                    mAddedTouchMode = (res & WindowManagerGlobal.ADD_FLAG_IN_TOUCH_MODE) != 0;
                    mAppVisible = (res & WindowManagerGlobal.ADD_FLAG_APP_VISIBLE) != 0;
    
                    ...
                }
            }
        }
        
        @Override
        public void requestLayout() {
            if (!mHandlingLayoutInLayoutRequest) {
                checkThread();
                mLayoutRequested = true;
                // 调用了这个方法
                scheduleTraversals();
            }
        }

        void scheduleTraversals() {
            if (!mTraversalScheduled) {
                mTraversalScheduled = true;
                mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
                // 异步方法，最终触发了mTraversalRunnable
                mChoreographer.postCallback(
                        Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
                notifyRendererOfFramePending();
                pokeDrawLockIfNeeded();
            }
        }

        final class TraversalRunnable implements Runnable {
            @Override
            public void run() {
                // 异步回调到了这里
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
                
                // 真正的实现
                performTraversals();
    
                if (mProfile) {
                    Debug.stopMethodTracing();
                    mProfile = false;
                }
            }
        }

    private void performTraversals() {

            ...
    
            if (mFirst || windowShouldResize || viewVisibilityChanged || cutoutChanged || params != null
                    || mForceNextWindowRelayout) {
                mForceNextWindowRelayout = false;
    
                ...
    
                if (!mStopped || mReportNextDraw) {
                    boolean focusChangedDueToTouchMode = ensureTouchModeLocally(
                            (relayoutResult&WindowManagerGlobal.RELAYOUT_RES_IN_TOUCH_MODE) != 0);
                    if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth()
                            || mHeight != host.getMeasuredHeight() || dispatchApplyInsets ||
                            updatedConfiguration) {
                        ...
    
                         // Ask host how big it wants to be
                         // 开始对DecorView测量
                        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    
                        ...
    
                        if (measureAgain) {
                            if (DEBUG_LAYOUT) Log.v(mTag,
                                    "And hey let's measure once more: width=" + width
                                    + " height=" + height);
                            // 需要再次测量
                            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                        }
    
                        layoutRequested = true;
                    }
                }
            } else {
                // Not the first pass and no window/insets/visibility change but the window
                // may have moved and we need check that and if so to update the left and right
                // in the attach info. We translate only the window frame since on window move
                // the window manager tells us only for the new frame but the insets are the
                // same and we do not want to translate them more than once.
                maybeHandleWindowMove(frame);
            }
    
            ...

            if (didLayout) {
                // 开始对DecorView 进行布局
                performLayout(lp, mWidth, mHeight);
    
                ...
            }
    
            ...
    
            boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;
    
            if (!cancelDraw) {
                if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
                    for (int i = 0; i < mPendingTransitions.size(); ++i) {
                        mPendingTransitions.get(i).startChangingAnimations();
                    }
                    mPendingTransitions.clear();
                }
                
                // 开始绘制View
                performDraw();
            } else {
                if (isViewVisible) {
                    // Try again
                    scheduleTraversals();
                } else if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
                    for (int i = 0; i < mPendingTransitions.size(); ++i) {
                        mPendingTransitions.get(i).endChangingAnimations();
                    }
                    mPendingTransitions.clear();
                }
            }
    
            if (mAttachInfo.mContentCaptureEvents != null) {
                notifyContentCatpureEvents();
            }
    
            mIsInTraversal = false;
        }

        private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
            if (mView == null) {
                return;
            }
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
            try {
                // 调用了View的measure方法
                mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
            } finally {
                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
            }
        }
    
        private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
                int desiredWindowHeight) {
            mScrollMayChange = true;
            mInLayout = true;
    
            final View host = mView;
            if (host == null) {
                return;
            }
            if (DEBUG_ORIENTATION || DEBUG_LAYOUT) {
                Log.v(mTag, "Laying out " + host + " to (" +
                        host.getMeasuredWidth() + ", " + host.getMeasuredHeight() + ")");
            }
    
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "layout");
            try {
                // 对view进行布局
                host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
    
                mInLayout = false;
                int numViewsRequestingLayout = mLayoutRequesters.size();
                if (numViewsRequestingLayout > 0) {
                    // requestLayout() was called during layout.
                    // If no layout-request flags are set on the requesting views, there is no problem.
                    // If some requests are still pending, then we need to clear those flags and do
                    // a full request/measure/layout pass to handle this situation.
                    ArrayList<View> validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters,
                            false);
                    if (validLayoutRequesters != null) {
                        ...
                        host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
    
                        mHandlingLayoutInLayoutRequest = false;
    
                        ...
                    }
    
                }
            } finally {
                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
            }
            mInLayout = false;
        }

        private void performDraw() {
            ...
    
            try {
                ...
                
                // 调用draw方法
                boolean canUseAsync = draw(fullRedrawNeeded);
                if (usingAsyncReport && !canUseAsync) {
                    mAttachInfo.mThreadedRenderer.setFrameCompleteCallback(null);
                    usingAsyncReport = false;
                    finishBLASTSync(true /* apply */);
                }
            } finally {
                mIsDrawing = false;
                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
            }
            ...
        }

        private boolean draw(boolean fullRedrawNeeded) {
            Surface surface = mSurface;
            ...
    
            boolean useAsyncReport = false;
            if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
                if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
                    ...
    
                    mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
                } else {
                    ...
    
                    // 调用drawSoftware
                    if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                            scalingRequired, dirty, surfaceInsets)) {
                        return false;
                    }
                }
            }
    
            if (animating) {
                mFullRedrawNeeded = true;
                scheduleTraversals();
            }
            return useAsyncReport;
        }

        private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
                boolean scalingRequired, Rect dirty, Rect surfaceInsets) {
            ...
    
            try {
                ...
    
                // 调用DecorView的draw方法
                mView.draw(canvas);
    
                drawAccessibilityFocusedDrawableIfNeeded(canvas);
            } finally {
                ...
            }
            return true;
        }

最终调了mView的Draw方法，再深入都是关于View的了，就不在这里展开了。

## 参考文档
+ [Android 中的 Window](https://www.jianshu.com/p/4cbcd2a01464)
+ [Android源码-深入理解Window和WindowManager](https://www.jianshu.com/p/1c4059d3865b)
+ [初步理解 Window 体系](https://www.jianshu.com/p/94fa885e6d99)
+ [ViewRootImpl的独白，我不是一个View(布局篇)](https://blog.csdn.net/stven_king/article/details/78775166)
