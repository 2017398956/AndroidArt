# 第 3 章 View 的事件体系 #
## 3.1 View 基础知识 ##
### 3.1.1 什么是 View ###
**注意** ViewGroup 也是继承自 View 。
### 3.1.2 View 的位置参数 ###
View 的位置主要由四个顶点决定，分别对应于 View 的四个属性 top ， left ，right ，bottom ，要注意的是这四个参数都是 View 相对于父容器来说的。这样我们就可以轻松计算出 View 的 width = right - left ， height = bottom - top ；另外这四个属性可以用下面的方法获得：

	top = getTop() ;
	left = getLeft() ;
	right = getRight() ;
	bottom = getBottom() ;
从 Android 3.0 开始，View 额外增加了几个参数： x 、 y 、 translationX 和 translationY ，这四个属性也是 View 相对于父容器的。其中 translationX 和 translationY 是 View 平移后相对于初始位置的偏移量，其关系如下：

	x = left + translationX ;
	y = top + translationY ;
**注意**在 View 平移过程中 top ， left ，right ，bottom 不会改变，变的是 x 、 y 、 translationX 和 translationY 。
### 3.1.3 MotionEvent 和 TouchSlop ###
#### 3.1.3.1 MotionEvent ####
手指接触到屏幕的一系列事件中，典型的有如下几种：

1. ACTION_DOWN 手指刚接触屏幕；
2. ACTION_MOVE 手指在屏幕上移动；
3. ACTION_UP 手指从屏幕松开的一瞬间；

正常情况下，一次手指触摸屏幕的行为会触法一系列点击事件，例如下面几种情况：

1. 点击屏幕后离开松开，事件序列为 DOWN -> UP ;
2. 点击屏幕滑动一会再松开，事件序列为 DOWN -> MOVE -> ... -> MOVE -> UP ;

触摸时我们可以根据 MotionEvent 对象获得点击事件发生的 x ，y 坐标，系统提供了两组方法： getX / getY 和 getRawX / getRawY , getX / getY 返回的是相对于当前 View 左上角的 x ，y 坐标，getRawX / getRawY 返回的是相对于屏幕左上角的 x ，y 坐标。
#### 3.1.3.2 TouchSlop ####
TouchSlop 系统所能识别出的被认为是滑动的最小距离，这是一个常量，和设备有关，不同的设备可能取值不同，可以通过 ViewConfiguration.get(getContext()).getScaledTouchSlop() 获得。
### 3.1.4 VelocityTracker 、 GestureDetector 和 Scroller ###
#### 3.1.4.1 VelocityTracker ####
速度追踪，用于追踪手指在滑动过程中的速度，包括水平速度和竖直速度，它的使用如下：

	VelocityTracker velocityTracker = VelocityTracker.obtain() ;
	velocityTracker.add(onTouchEvent) ;// 追踪 View 的事件
	velocityTracker.computeCurrentVelocity(1000) ;// 设置间隔时间
	int xVelocity = velocityTracker.getXVelocity() ;
	int yVelocity = velocityTracker.getYVelocity() ;
	// 不使用时重置并回收内存
	velocityTracker.clear() ;
	velocityTracker.recycle() ;

这里的速度是指一段时间内手指所划过的像素数，因为屏幕坐标系的存在，所以速度可以为负。
#### 3.1.4.2 GestureDetector ####
手势检测，用于辅助检测用户的单击、滑动、长按、双击等行为。使用方法如下：

	GestureDetector mGestureDetector = new GestureDetector(this) ;
	// 解决长按屏幕后无法拖动的现象
	mGestureDetector.setIsLongpressEnabled(false) ;

未完 待续
#### 3.1.4.3 Scroller ####
弹性滑动对象，用于实现 View 的弹性滑动。
## 3.2 View 的滑动 ##
### 3.2.1 使用 scrollTo / scrollBy ###
scrollBy 实际上也是调用了 scrollTo 方法，它实现了基于当前位置的相对滑动，而 scrollTo 则实现了基于所传递参数的绝对滑动。View 内部有 2 个属性 mScrollX 和 mScrollY ，可分别通过 getScrollX 和 getScrollY 获得。在滑动过程中 mScrollX 的值总是等于 View 左边缘和 View 内容左边缘在水平方向的距离，而 mSrollY 的值总是等于 View 上边缘和 View 内容上边缘在竖直方向的距离。View 边缘指的是 View 的位置，由四个顶点组成，而 View 内容边缘是指 View 中的内容的边缘，scrollTo 和 scrollBy 只能改变 View 内容的位置而不能改变 View 在布局中的位置。mScrollX 和 mScrollY 的单位是像素，正负代表 View 内容移动的方向，右下为正，左上为负。
### 3.2.2 使用动画 ###
通过动画我们能让一个 View 进行平移，而平移是一种滑动。使用动画来移动 View ，主要是操作 View 的 translationX 和 translationY 属性，既可以采用传统的 View 动画，也可以采用属性动画，如果采用属性动画的话，为了能兼容 3.0 以下的版本，需要采用开源动画库 nineoldandroids （http://nineoldandroids.com/)
View 动画是对 View 的影像做操作，它不能真正改变 View 的位置参数，包括宽/高，并且如果希望动画后的状态得以保留还必须将 fillAfter 属性设置为 true ，否则动画完成后期动画结果会消失。使用属性动画则不为存在这种问题，但是在 Android 3.0 以下无法使用属性动画，这个时候我们可以使用动画兼容库 nineoldandroids 来实现属性动画，尽管如此，在 Android 3.0 以下的手机上通过 nineoldandroids 来实现的属性动画本质上仍然是 View 动画。
**注意**：由于 View 动画无法真正改变 View 的位置，当 View 移动后会造成在新位置无法触发 onClick 等事件。
### 3.2.3 改变布局参数 ###
对于 View 的滑动，我们还可以通过 LayoutParams 的 margin 、 width 等来达到移动 View 的效果。
### 3.2.4 各种滑动方式的对比 ###

| - | scrollTo / scrollBy | 动画 | 改变布局参数 |
|------|---------------------| ---- |------------|
| 优点 | 操作简单，不影响内部元素的点击事件 | 动画 | 改变布局参数 |
| 缺点 | 只能滑动 View 内容，不能滑动 View 本身 | 动画 | 改变布局参数 |