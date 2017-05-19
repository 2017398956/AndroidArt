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

触摸时我们可以根据 MotionEvent 对象获得点击事件发生的 x ，y 坐标，系统提供了两组方法： getX / getY 和 getRawX / getRawY , getX / getY 返回的是相对于当前 View 左上角的 x ，y 坐标，getX / getY 返回的是相对于屏幕左上角的 x ，y 坐标。
#### 3.1.3.2 TouchSlop ####
