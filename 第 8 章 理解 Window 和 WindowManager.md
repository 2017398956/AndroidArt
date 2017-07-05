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
