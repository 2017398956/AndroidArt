# 第 8 章 理解 Window 和 Window Manager #
Window 表示一个窗口的概念，在日常开发中直接接触 Window 的场景不多；如果需要展示一个悬浮窗，这时就用到了 Window 。Window 是一个抽象类，具体实现是 PhoneWindow ，创建一个 Window 可以通过 WindowManager 来实现。WindowManager 是外界访问 Window 的入口，Window 的具体实现位于 WindowManagerService 中，WindowManager 和 WindowManagerService 的交互是一个 IPC 过程。Android 中的所有视图都是通过 Window 来呈现的，不管是 Activity 、 Dialog 还是 Toast ，它们的视图实际上都是附加在 Window 上的，因此 Window 实际是 View 的直接管理者。View 事件分发时，点击事件也是由 Window 传递给 DecorView ，然后再由 DecorView 传递给我们的 View ，就连 Activity 的设置视图的方法 setContentView 在底层也是通过 Window 来完成的。

## 8.1 Window 和 WindowManager ##
为了方便地理解 Window 的工作机制，下面我们通过 WindowManager 来演示一个添加 Window 的过程：

