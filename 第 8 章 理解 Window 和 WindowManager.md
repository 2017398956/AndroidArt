# 第 8 章 理解 Window 和 Window Manager #
Window 表示一个窗口的概念，在日常开发中直接接触 Window 的场景不多；如果需要展示一个悬浮窗，这时就用到了 Window 。Window 是一个抽象类，具体实现是 PhoneWindow ，创建一个 Window 可以通过 WindowManager 来实现。WindowManager 是外界访问 Window 的入口，Window 的具体实现位于 WindowManagerService 中，WindowManager 和 WindowManagerService 的交互是一个 IPC 过程。Android 中的所有视图都是通过 Window 来呈现的，不管是 Activity 、 Dialog 还是 Toast ，它们的视图实际上都是附加在 Window 上的，因此 Window 实际是 View 的直接管理者。View 事件分发时，点击事件也是由 Window 传递给 DecorView ，然后再由 DecorView 传递给我们的 View ，就连 Activity 的设置视图的方法 setContentView 在底层也是通过 Window 来完成的。

## 8.1 Window 和 WindowManager ##
为了方便地理解 Window 的工作机制，下面我们通过 WindowManager 来演示一个添加 Window 的过程：

	private void addWindow() {
			// 准备好一个 Button
	        buttonTemp = new Button(this);
	        buttonTemp.setText("Button");
			// window 的展示的参数
	        windowManagerLayoutParams = new WindowManager.LayoutParams(ViewGroup.LayoutParams.WRAP_CONTENT,
	                ViewGroup.LayoutParams.WRAP_CONTENT,
	                WindowManager.LayoutParams.TYPE_APPLICATION , // Type
					0, // Flags ，这里赋值和下面赋值效果等同
					PixelFormat.TRANSPARENT);
			// flag 用于标注 Window 的特性
	        windowManagerLayoutParams.flags = WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL
	                | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
	                | WindowManager.LayoutParams.FLAG_SHOW_WHEN_LOCKED;
			// Window 相对于屏幕的布局方式
	        windowManagerLayoutParams.gravity = Gravity.LEFT | Gravity.TOP;
			// window 在上面布局的基础上出现在屏幕的位置
	        windowManagerLayoutParams.x = 100;
	        windowManagerLayoutParams.y = 300;
	        windowManager = (WindowManager) getSystemService(WINDOW_SERVICE);
	        windowManager.addView(buttonTemp, windowManagerLayoutParams);
	}

Flags 参数表示 Window 的特性，它有很多选项，这里介绍几个比较常用的：

**FLAG_NOT_FOCUSABLE**

表示 Window 不需要获取焦点，也不需要接收各种输入事件，此标记会同时启用 FLAG_NOT_TOUCH_MODAL ，最终事件会直接传递给下层的具有焦点的 Window 。

**FLAG_NOT_TOUCH_MODAL**

在此模式下，系统会将当前 Window 区域以外的点击事件传递给底层 Window ，当前 Window 区域内的点击事件则自己处理。这个标记很重要，一般来说都需要开启此标记，否则其它 Window 将不能接收点击事件。

**FLAG_SHOW_WHEN_LOCKED**

开启此模式可以将 Window 显示在锁屏界面上。

**Type** 参数表示 Window 的类型，Window 有多种类型，如：软键盘、状态栏等。这里主要介绍三种常用的类型：应用 Window、子Window和系统 Window。应用类 Window 对应着一个 Activity 。子 Window 不能单独存在，它需要附属在特定的父 Window 之中，比如常见的 Dialog 就是一个子 Window 。系统 Window 是需要声明系统权限才能创建的 Window ，比如 Toast 和 系统状态栏这些都是系统 Window 。

Window 是分层的，每个 Window 都有对应的 ordered ，层级大的会覆盖在层级小的 Window 的上面，这和 HTML 中 z-index 的概念是完全一致的。在这三类 Window 中，应用 Window 的层级范围是 1~99 ，子 Window 的层级范围是 1000~1999，系统 Window 的层级范围是 2000~2999，这些层级范围对应着 WindowManager.LayoutParams 的 Type 参数。**注意**：当使用系统 Window 时，需要声明权限 <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" /> ,如果不声明权限，app 将会发生 crash 。

WindowManager 所提供的功能很简单，常用的只有三个方法： addView(...) 、 updateViewLayout(...) 、 removeView(...)，这三个方法定义在 ViewManager 中，而 WindowManager 继承了 ViewManager 。如果想删除一个 Window 只需要删除它里面的 View 即可。由此可见 WindowManager 操作 Window 更像是操作 Window 中的 View 。

## 8.2 Window 的内部机制 ##

Window 是一个抽象概念，每个 Window 都对应着一个 View 和一个 ViewRootImpl ，Window 和 View 通过 ViewRootImpl 来建立联系，因此 Window 并不是实际存在的，它是以 View 的形式存在。在实际使用中无法直接访问 Window ，对 Window 的访问必须通过 WindowManager 。

### 8.2。1 Window 的添加过程 ###
Window 的添加过程需要通过 WindowManager 的 addView 来实现，WindowManager 是一个接口，它的真正实现是 WindowManagerImpl 类。在 WindowManagerImpl 中 Window 的三大操作的实现如下：

	@Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }

    @Override
    public void updateViewLayout(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.updateViewLayout(view, params);
    }

	@Override
    public void removeView(View view) {
        mGlobal.removeView(view, false);
    }

可以发现，WindowManagerImpl 并没有直接实现 Window 的三大操作，而是全部交给了 WindowManagerGlobal 来处理，WindowManagerGlobal 以工厂的形式向外提供自己的实例，在 WindowManagerImpl 中有如下一段代码： private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance(); WindowManagerImpl 这种工作模式是典型的 **桥接模式** ，将所有的操作全部委托给 WindowManagerGlobal 来实现。WindowManagerGlobal 的 addView 方法主要分为如下几步：

1. 检查参数是否合法，如果是子 Window 那么还需要调整一些布局参数

		if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        if (display == null) {
            throw new IllegalArgumentException("display must not be null");
        }
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }

        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
        if (parentWindow != null) {
            parentWindow.adjustLayoutParamsForSubWindow(wparams);
        }

2. 创建 ViewRootImpl 并将 View 添加到列表中

  在 WindowManagerGlobal 内部有如下几个列表比较重要：


    private final ArrayList<View> mViews = new ArrayList<View>();
    private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
    private final ArrayList<WindowManager.LayoutParams> mParams =
            new ArrayList<WindowManager.LayoutParams>();
    private final ArraySet<View> mDyingViews = new ArraySet<View>();

在上面的声明中，mViews 存储的是所有 Window 对应的 View ； mRoots 存储的是所有 Window 所对应的 WindowManagerImpl ； mParams 存储的是所有 Window 所对应的布局参数；而 mDyingViews 则存储了那些正被删除的 View 对象，或者说是那些已经调用 removeView 方法但是删除操作还未完成的 Window 对象。在 addView 中通过如下方式将 Window 的一系列对象添加到列表中：

	root = new WindowManagerImpl(view.getContext() , display) ;
	view.setLayoutParams(wparams) ;
	
	mView.add(view) ;
	mRoot.add(root) ;
	mParams.add(wparams) ;

3. 通过 ViewRootImpl 来更新界面并完成 Window 的添加过程
   
  这个步骤由 ViewRootImpl 的 setView 方法来完成，从第四章可以知道，View 的绘制过程是由 ViewRootImpl 来完成的，这里当然也不例外，在 setView 内部会通过 requestLayout 来完成异步刷新请求。在下面的代码中，scheduleTraversals 实际是 View 的绘制入口：

	public void requestLayout(){
		if(!mHandlingLayoutInLayoutReques){
			checkThread() ;
			mLayoutRequested = true ;
			scheduleTraversals() ;
		}
	}

接着会通过 WindowSession 最终来完成 Window 的添加过程。在下面的代码中， mWindowSession 的类型是 IWindowSession ，它是一个 Binder 对象，真正的实现类是 Session ，也就是 Window 的添加过程是一次 IPC 调用 。

	try{
		mOrigWindowType = mWindowAttributes.type ;
		mAttachInfo.mRecomputeGlobalAttributes = true ;
		collectViewAttributes() ;
		res = mWindowSession.addToDisplay(mWindow , mSeq , mWindowAttributes , getHostVisibility() , mDisplay.getDisplayId() ,
			mAttachInfo.mContentInsets , mInputChannel) ;
	}catch(RemoteException e){
		mAdded = false ;
		mView = null ;
		mAttachInfo.mRootView = null ;
		mInputChannel = null ;
		mFallbackEventHandler.setView(null) ;
		unsheduleTraversals() ;
		setAccessibilityFocus(null , null) ;
		throw new RuntimeException("Adding window failed" , e ) ;
	}

在 Session 内部会通过 WindowManagerService 来实现 Window 的添加，代码如下：

	public int addToDisplay(IWindow window , int seq , 	WindowManager.LayoutParams attrs , int viewVisbility , int displayId ,Rect outContentInsets , InputChannel outInputChannel){
		return mService.addWindow(this , window , seq , attrs , viewVisbility , displayId , outConntentInsets , outInputChannel) ;
	}

如此一来，Window 的添加请求就交给 WindowManagerService 去处理了，在 WindowManagerService 内部会为每一个应用保留一个单独的 Session 。

### 8.2.2 Window 的删除过程 ###

Window 的删除过程和添加过程一样，都是先通过 WindowManagerImpl 后，再进一步通过 WindowManagerGlobal 来实现的。代码如下：

    public void removeView(View view, boolean immediate) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }

        synchronized (mLock) {
            int index = findViewLocked(view, true);
            View curView = mRoots.get(index).getView();
            removeViewLocked(index, immediate);
            if (curView == view) {
                return;
            }

            throw new IllegalStateException("Calling with view " + view
                    	+ " but the ViewAncestor is attached to " + curView);
        }
    }

removeView 的逻辑很清晰，首先通过 findViewLocked 来查找待删除的 View 的索引，这个查找过程就是建立的数组遍历，然后在调用 removeViewLocked 来做进一步删除，代码如下：

    private void removeViewLocked(int index, boolean immediate) {
        ViewRootImpl root = mRoots.get(index);
        View view = root.getView();

        if (view != null) {
            InputMethodManager imm = InputMethodManager.getInstance();
            if (imm != null) {
                imm.windowDismissed(mViews.get(index).getWindowToken());
            }
        }
        boolean deferred = root.die(immediate);
        if (view != null) {
            view.assignParent(null);
            if (deferred) {
                mDyingViews.add(view);
            }
        }
    }

removeViewLocked 是通过 ViewRootImpl 来完成删除操作的。在 WindowManager 中提供了两种删除接口 removeView 和 removeViewImmediate ，它们分别表示异步删除和同步删除。其中 removeViewImmediate 使用时要特别注意，一般不需要使用该方法来删除 Window 以免发生意外错误。而 removeView 的具体删除操作是由 ViewRootImpl 的 die 方法来完成；在异步删除的情况下，die 方法只是发送了一个请求删除的消息就立刻返回了，这时 View 并没有完成删除造作，所以最后会将其添加到 mDyingViews 中，mDyingViews 表示待删除的 View 列表。 ViewRootImpl 的 die 方法如下：

    /**
     	* @param immediate True, do now if not in traversal. False, put on queue and do later.
     	* @return True, request has been queued. False, request has been completed.
     */
    boolean die(boolean immediate) {
        // Make sure we do execute immediately if we are in the middle of a traversal or the damage
        // done by dispatchDetachedFromWindow will cause havoc on return.
        if (immediate && !mIsInTraversal) {
            doDie();
            return false;
        }

        if (!mIsDrawing) {
            destroyHardwareRenderer();
        } else {
            Log.e(mTag, "Attempting to destroy the window while drawing!\n" +
                    "  window=" + this + ", title=" + mWindowAttributes.getTitle());
        }
        mHandler.sendEmptyMessage(MSG_DIE);
        return true;
    }

在 die 方法内部只是做了简单的判断，如果是异步删除，那么就发送一个 MSG_DIE 的消息，ViewRootImpl 中的 Handler 会处理此消息并调用 doDie 方法，如果是同步删除（立即删除）那么就不发送消息直接调用 doDie 方法，这就是两种删除方式的区别。在 doDie 内部会调用 dispatchDetachedFromWindow 方法，真正的删除 View 的逻辑在 dispatchDetachedFromWindow 方法的内部实现。dispatchDetachedFromWindow 方法主要做四件事：

1. 垃圾回收的相关工作，比如清除数据和消息、移除回调。
2. 通过 Session 的 remove 方法删除 Window ： mWindowSession.remove(mWindow) ,这同样是一个 IPC 过程，最终会调用 WindowManagerService 的 removeWindow 方法。
3. 调用 View 的 dispatchDetachedFromWindow 方法，在内部会调用 View 的onDetachedFromWindow 以及 onDetachedFromWindowInternal 。onDetachedFromWindow 方法会在 View 被删除时调用，并做一些资源回收的工作，比如终止动画，停止线程等。
4. 调用 WindowManagerGlobal 的 doRemoveView 方法刷新数据，包括 mRoots 、 mParams 、 以及 mDyingViews ，需要将当前 Window 所关联的这三类对象从列表中删除。

### 8.2.3 Window 的更新过程 ###

Window 的更新方法为 WindowManagerGlobal 中的 updateViewLayout ,代码如下：

    public void updateViewLayout(View view, ViewGroup.LayoutParams params) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }

        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;

        view.setLayoutParams(wparams);

        synchronized (mLock) {
            int index = findViewLocked(view, true);
            ViewRootImpl root = mRoots.get(index);
            mParams.remove(index);
            mParams.add(index, wparams);
            root.setLayoutParams(wparams, false);
        }
    }

updateViewLayout 做的事情比较少，首先它需要更新 View 的 LayoutParams 并替换掉老的 LayoutParams ，接着更新 ViewRootImpl 中的 LayoutParams ，这一步是通过 ViewRootImpl 的 setLayoutParams 方法来实现的。在 ViewRootImpl 中会通过 scheduleTraversals 方法对 View 重新布局，包括测量、布局、重绘这三个过程。除了 View 本身的重绘外 ，ViewRootImpl 还通过 WindowSession 来更新 Window 的视图，这个过程最终是由 WindowManagerService 的 relayoutWindow() 来具体实现的，它同样是一个 IPC 过程。

## 8.3 Window 的创建过程 ##

通过上面的分析可以看出，View 是 Android 中的视图的呈现方式，但是 View 不能单独存在，






















































