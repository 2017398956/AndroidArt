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
