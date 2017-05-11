# 第 2 章 IPC 机制 #
## 2.1 Android IPC 简介 ##
IPC 是 Inter-Process Communication 的缩写，含义为进程间通讯或者跨进程通讯，是指两个进程间数据交换的过程。在 Android 上最有特色的进程间通信方式是 Binder ， 通过 Binder 可以轻松实现进程间通信。除了 Binder ，Android 还支持 Socket ，通过 Socket 可以实现任意两个端间的通信，当然，一个设备上的两个进程也可以通过 Socket 通信。由于 Android 为每个进程可分配的最大内存做了限制，所以，在单个应用中使用多进程也可以帮我们获取更多的内存资源。另外，系统提供的 ContentProvider 也是一种进程间通讯方式，只是细节被系统屏蔽了，我们无法感知。
## 2.2 Android 中的多进程模式 ##
在介绍多进程间的通讯前，我们首先要开启多进程模式，Android 开启多进程很简单，通过在 AndroidManifest.xml 中给四大组件指定 android:process 即可；除此之外我们还有一种特殊的方法：通过 JNI 在 native 层去 fork 一个新的进程，由于这种方法不常用，这里暂不考虑。
### 2.2.1 开启多进程模式 ###
一般在 Android 中多进程是指在一个应用中存在多个进程，对于两个应用间的多进程问题，这里不作考虑。下面是一个开启多进程的示例：

	<activity
		...
		android:name="com.test.MainActivity"
	>
	// 这里是程序入口，省略 intent-filter
	</activity>

	<activity
		...
		android:name="com.test.SecondActivity"
		android:process=":remote"
	>
	</activity>

	<activity
		...
		android:name="com.test.ThirdActivity"
		android:process="test.com.remote"
	>
	</activity>

上面 MainActivity 没有指定进程名，系统会以包名作为进程名，SecondActivity 的进程名为 包名 + “:remote” ，ThirdActivity 的进程名就是指定的名称 “com.test.ThirdActivity” 。当相对应的 Activity 启动后，我们可以再 DDMS 中看到其所在的进程；另外，也可以使用 adb shell ps [| grep 进程名] 来查看。

**注意**：上面 SecondActivity 和 ThirdActivity 开启多进程的模式是有区别的：

1. “:” 的含义是指当前进程名附加上当前的包名才是完整的进程名；
2. 以 “:” 开头的进程属于当前应用的私有进程，其它应用的组件不能与它跑在同一个进程中，而不以 “:” 开头的进程属于全局进程，其它应用可以通过 shareUID 和它跑在同一个进程中。

Android 会为每一个应用分配一个唯一的 shareUID ，只有 shareUID 相同的应用才能共享数据。另外，两个应用通过 shareUID 跑在同一个进程是有要求的，需要两个应用 shareUID 相同且签名相同才可以。在这种情况下，它们可以互相访问对方的私有数据，如 data 目录、组件信息等，不管它们是否跑在同一个进程中。当然如果它们跑在同一个进程，它们不仅可以访问对方的私有数据，还可以访问对方的内存数据，就像一个应用的两个部分。

### 2.2.2 多进程模式的运行机制 ###
由于每个进程都分配一个独立的虚拟机，不同的虚拟机在内存分配上有不同的地址空间，这就导致在不同的虚拟机访问同一个类的对象会产生多个副本。例如：当应用启动第二个或第 N 个进程时，这个新创建的进程会从应用的 Application 开始重新创建并初始化一遍所涉及类，而不会使用先前进程已经创建的对象，所以运行在不同进程中的四大组件之间不能通过内存来共享数据。因此多进程一般会造成下面几种问题：

1. 静态成员和单例模式完全失效；
2. 线程同步机制完全失效；
3. SharePreference 的可靠性下降；
4. Application 会多次创建；

一个应用的多进程模式就像两个不同的应用采用了 SharedUID 模式。

## 2.3 IPC 基础概念介绍 ##
### 2.3.1 Serializable 接口 ###
1. 一个类序列化和反序列化后不是一个对象 ；
2. 不使用 serialVersionUID 时，系统会自动计算类的 hash 值并赋于 serialVersionUID ，当类修改后 serialVersionUID 会改变导致无法正常反序列化；
3. 静态成员变量属于类不属于对象，所以不会参与序列化过程；
4. 使用 transient 关键字标记的成员变量不参与序列化过程；
5. 系统默认的序列化过程是可以改变的，可以通过重写 writeObject 和 readObject 方法修改系统默认的序列化过程，不过一般不需要；

### 2.3.2 Parcelable 接口###
只要实现这个接口，一个类的对象就可以实现序列化并可通过 Intent 和 Binder 传递。系统已经为我们提供了许多实现了 Parcelable 接口的类，它们都是可以直接序列化的，如： Intent 、 Bundle 、 Bitmap 等，同时 List 和 Map 也是可序列化的，前提是它们中的每个元素都是可序列化的。

既然 Parcelable 和 Serializable 都可以实现序列化并可以通过 Intent 传递，那么二者应该如何选取呢？ Serializable 是 Java 中的序列化接口，使用起来简单但是开销大，序列化和反序列化都需要大量的 I/O 操作；而 Parcelable 是 Android 平台的序列化方式，缺点是使用稍微麻烦，但效率高，这也是 Android 的推荐序列化方式，所以我们首选 Parcelable 。 Parcelable 注意用于内存序列化上，当需要将对象序列化到存储设备上或序列化后通过网络传输，这时使用 Parcelable 会稍显复杂，此时建议使用 Serializable 。

### 2.3.3 Binder ###
直观来说，Binder 是 Android 中的一个类，它实现了 IBinder 接口。从 IPC 角度来说，Binder 是 Android 中的一种跨进程通信方式，Binder 还可以理解为一种虚拟的物理设备，它的设备驱动是 /dev/binder ，该通信方式在 Linux 中没有；从 Android FrameWork 角度来说，Binder 是 ServiceManager 连接各种 Manager（ActivityManager 、 WindowManager，等等）和 ManagerService 的桥梁；从 Android 应用层来说，Binder 是客户端和服务端进行通信的媒介，当 bindService 的时候，服务端会返回一个包含了服务端业务调用的 Binder 对象，通过这个 Binder 对象，客户端就可以获取服务端提供的服务或者数据，这里的服务包括普通服务和基于 AIDL 的服务。