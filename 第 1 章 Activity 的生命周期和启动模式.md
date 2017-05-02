# 第 1 章 Activity 的生命周期和启动模式 #
## 1.1 Activity 生命周期的全面分析 ##
### 1.1.1 典型情况下的生命周期分析 ###
1. onCreate : 生命周期第一个方法，可做一些初始化工作；
2. onRestart ： Activity 重新启动，由不可见变为可见；
3. onStart ： Activity 正在启动，此时 Activity 已经可见，但没有出现在前台，还无法和用户交互；
4. onResume ： Activity 可见，出现在前台，并开始活动；
5. onPause ： Activity 正在停止，正常情况下 onStop 会紧接着被调用，但是该方法中不宜做耗时操作，因为当该方法执行完毕后，新 Activity 的 onResume 方法才会执行；打开新的 Activity 时，当前 Activity 的 onPause 方法先执行完毕，新 Activity 的 onCreate 方法才会执行 ；
6. onStop ： Activity 即将停止，可以做一些耗时操作，但不能太耗时；
7. onDestroy ： Activity 即将被销毁，这是 Activity 生命周期中最后一个回调，在这里我们可以做一些回收工作和资源释放；

**注意**
当用户打开新的 Activity 或者切换到桌面的时候，回调如下： onPause -> onStop ，这里有一种特殊的情况，如果新的 Activity 采用了透明主题，那么当前的 Activity 不会回调 onStop ；如果新的 Activity 采用了透明主题呢 ？
### 1.1.2 异常情况下的生命周期分析 ###
