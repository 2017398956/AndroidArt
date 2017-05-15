
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

在 Android 开发中，Binder 主要用在 Service 中，包括 AIDL 和 Messenger ，其中普通Service 中的 Binder 不涉及进程间通信，所以较为简单，无法触及 Binder 核心，而 Messenger 的底层其实是 AIDL ，所以可以选择用 AIDL 来分析 Binder 的工作机制。

所有可在 Binder 中传输的接口都需要继承 IInterface 接口； AIDL 对应的类（这里记为 A ）会为每个方法声明一个唯一的整型 id 用于标识这些方法，以便在 tranact 过程中区分客户端请求了那个方法； A 中还有一个内部类 Stub ，这个 Stub 就是一个 Binder 类，当客户端和服务端都位于一个进程时，方法调用不会走跨进程的 transact 过程，而当两者位于不同进程时，方法调用需走 transact 过程，这个逻辑是由 Stub 的内部代理类 Proxy 来完成；下面将介绍 Stub 和 Proxy 中一些参数和方法的意义。

**DESCRIPTOR**

Binder 的唯一标识，一般用当前 Binder 的类名表示。

**asInterface(android.os.IBinder obj)**

用于将服务端的 Binder 对象转换为客户端所需的 AIDL 接口类型的对象，这种转换过程是区分进程的，若两者在同一进程，那么此方法返回的就是服务端的 Stub 对象本事，否则返回的是系统封装后的 Stub.proxy 对象。

**asBinder**

返回当前 Binder 对象。

**onTransact**

此方法运行在服务端中的 Binder 线程池中，当客户端发起跨进程请求时，远程请求会通过系统底层封装后交由此方法来处理。该方法的原形为：
	
	public boolean onTransact(int code, android.os.Parcel data, 
										android.os.Parcel reply, int flags)

服务端可以通过 code （即对应上面类 A 中方法的 id）来确定客户端请求的方法时什么，接着从 data 中取出目标方法所需的参数（如果目标方法有参数的话），然后执行目标方法，当目标方法执行完毕后，就向 reply 写入返回值（如果此方法有返回值的话），到此 onTransact 方法执行完毕。需要注意的是此方法的返回值是 boolean ，那么当客户端请求失败时，就会返回 false ，因此我们可以利用这个特性做权限验证，毕竟我们也不希望随便一个进程都能远程调用我们的服务。

**Proxy#methodXXX(...)**

这些方法都运行在客户端，当客户端远程调用时，它们内部实现是这样的：创建该方法所需的输入型 Parcel 对象 _data 、输出型 Parcel 对象 _reply 和返回值对象，然后把该方法的参数信息写入 _data 中（如果有参数的话）；接着调用 transact 方法来发起 RPC（远程过程调用）请求，同时当前线程挂起；然后服务端的 onTransact 方法会被调用，直到 RPC 过程返回后，当前线程继续执行，并从 _reply 中取出 RPC 过程的返回结果（如果有返回值的话），最后返回 _reply 中的数据。

**注意**

1. 当客户端发起远程请求时，由于当前线程会被挂起直到服务端进程返回数据，所以如果一个远程方法是很耗时的，那么不能再 UI 线程中发起此远程请求；
2. 由于服务端的 Binder 方法运行在 Binder 线程池中，所以 Binder 方法不管是否耗时都应该采用同步的方式实现，因为它已经运行在一个线程中了。
3. 由于 Binder 运行在服务端进程中，如果服务端进程由于某种原因异常终止，这个时候我们到服务端的 Binder 连接断裂（称之为 Binder 死亡），会导致我们的远程调用失败。更为关键的是，如果我们不知道 Binder 连接已经断裂，那么客户端的功能将受影响，为了解决这个问题，Binder 提供了两个配对的方法 linkToDeath 和 unlinkToDeath ，通过 linkToDeath 我们可以为 Binder 设置一个死亡代理，当 Binder 死亡时，我们就会收到通知，这时候我们就可以重新发起连接请求从而恢复连接，关于如何使用暂时不做介绍；另外，我们也可以通过 Binder.isBinderAlive 判读 Binder 是否死亡。

## 2.4 Android 中的 IPC 方式 ##

### 2.4.1 使用 Bundle ###
四大组件中的三大组件（Activity 、 Service 、 Receiver） 都是支持在 Intent 中传递 Bundle 数据的，由于 Bundle 实现了 Parcelable 接口，所以它可以方便地在不同进程中传递数据。
### 2.4.2 使用文件共享 ###
文件共享方式适合在对数据同步要求不高的进程之间进行通信，并且要妥善处理并发读/写的问题。另外，SharedPreference 也是一种文件共享的方式，虽然系统为它的读/写提供了一定缓存策略，但当面对高并发或多进程时，也有很大几率会丢失数据，因此不建议在多进程中对其操作。
### 2.4.3 使用 Messenger ###

Messager 可以翻译为信使，他可以在不同的进程中传递 Message 对象，在 Message 中放入我们需要传递的数据，就可以轻松实现进程间数据传递。Messenger 是一种轻量级的 IPC 解决方案，它的底层实现是 AIDL。另外，由于 Messenger 每次只处理一个请求，不存在并发情形，因此在服务端我们不需要考虑同步问题。
未完待续

### 2.4.4 使用 AIDL ###
由于 Messenger 是以串行方式来处理客户端消息的，所以当有大量客户端请求时，Messenger 就不合适了；同时，Messenger 的主要作用是传递消息，很多时候我们需要跨进程调用服务器方法，此时 Messenger 也无法做到，因此我们需要使用其他方式来处理这种复杂的情况，而 AIDL 就是其中一种。
在 AIDL 中不是所有的数据类型都可以使用的，其使用规则如下：

1. 基本数据类型（int 、 long 、 char 、 boolean 、 double 等）；
2. String 和 CharSequence ；
3. List ： 只支持 ArrayList ，且里面的元素都被 AIDL 支持 ；
4. Map ： 只支持 HashMap ， 且里面的元素都被 AIDL 支持 ；
5. Parcelable ： 所有实现了 Parcelable 接口的对象；
6. AIDL ： 所有的 AIDL 接口本身也可以在 AIDL 文件中使用；

**注意** 

1. 自定义的 Parcelable 对象和 AIDL 对象必须要显示 import 进来，不来它们是否和当前 AIDL 在一个包中；
2. 如果 AIDL 使用了自定义的 Parcelable 对象，那么必须新建一个和它同名的 AIDL 文件，并在其中声明它是 Parcelable 类型，例如声明一个 Book.aidl ：

		package xxx ;
		parcelable Book ;
3. AIDL 中除了基本数据类型，其它类型参数必须标上方向： in 、 out 或 inout ，in 表示输入型参数，out 表示输出型参数，inout 表示输入输出型参数；我们不能一概使用 out 或 inout ，要根据实际需求选择合适的类型，因为这在底层是有开销的；
4. AIDL 中只支持方法，不支持声明静态常量；

未完待续

### 2.4.5 使用 ContentProvider ###
ContentProvider 是 Android 中提供的专门用于不同应用间进行数据共享的方式，从这点来看，它天生适合进程间通信。和 Messenger 一样，ContentProvider 的底层实现同样也是 Binder 。