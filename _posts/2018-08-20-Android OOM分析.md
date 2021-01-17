#Android-OOM(OutOfMemoryError）分析

**OOM 发生的原因有以下几类：**
    1.文件描述符 (fd) 数目超限，即 proc/pid/fd 下文件数目突破 /proc/pid/limits 中的限制。可能的发生场景有短时间内大量请求导致 socket 的 fd 数激增，大量（重复）打开文件等 ；
    2.线程数超限，即proc/pid/status中记录的线程数（threads 项）突破 /proc/sys/kernel/threads-max 中规定的最大线程数。可能的发生场景有app 内多线程使用不合理，如多个不共享线程池的 OKhttpclient 等等 ；
    3.传统的 java 堆内存超限，即申请堆内存大小超过了Runtime.getRuntime().maxMemory()；
    4.32 为系统进程逻辑空间被占满导致 OOM（低概率）;
    5.其他。
## 1. 现状
对于每一个移动开发者，内存是都需要小心使用的资源，而线上出现的 OOM（OutOfMemoryError）都会让开发者抓狂，因为我们通常仰仗的直观的堆栈信息对于定位这种问题通常帮助不大。我们可以紧衣缩食的利用宝贵的堆内存（比如，使用小图片，bitmap 复用等），可是:
	1. 线上的 OOM 真的全是由于堆内存紧张导致的吗？
	2. 有没有 App 堆内存宽裕，设备物理内存也宽裕的情况下发生 OOM 的可能？
	3. 内存充裕的时候出现 OOM 崩溃？
通过公司的 APM 平台发现，一部分 OOM 有这样的特征，即：OOM 崩溃时，java 堆内存远远低于 Android 虚拟机设定的上限，并且物理内存充足，SD 卡空间充足
既然内存充足，这时候为什么会有 OOM 崩溃呢？

## 2. 问题描述
在详细描述问题之前，先弄清楚一个问题：
什么导致了 OOM 的产生？
下面是几个关于 Android 官方声明内存限制阈值的 API：
	1. AcitivityManager.getMemoryClass()  虚拟机JAVA堆大小的上限，分配对象时突破这个大小就会OOM
	2. AcitivityManager.getLargeMemeoryClass() menifest中设置largeheap = true时，虚拟机堆上限
	3. Runtime.getRuntime().maxMemory() 当前虚拟机实例的内存使用上限，为上述两者之一
	4. Runtime.getRuntime().totalMemory() 当前已经申请的内存，包括已经使用的和还没有使用的
	5. Runtime.getRuntime().freeMemory() 上一条已经申请但是未使用的那部分，那么已经申请并正在在使用的部分，used = totalMemory()- freeMemeory()
	6. ActivityManager.MemoryInfo.totalMem 设备总内存
	7. ActivityManager.MemoryInfo.availMem 设备当前可用内存
	8. /proc/memeinfo 记录设备内存信息
通常认为 OOM 发生是由于 java 堆内存不够用了，即
Runtime.getRuntime().maxMemory() 这个指标满足不了申请堆内存大小时

这种 OOM 可以非常方便的验证（比如: 通过 new byte[] 的方式尝试申请超过阈值maxMemory() 的堆内存），通常这种 OOM 的错误信息通常如下：
````
java.lang.Out.OfMemoryError: Failed to allocate a XXX byte allocation with XXX free bytes and XXXKB until OOM
````

前面已经提到发现的 OOM 案例中堆内存充裕（Runtime.getRuntime().maxMemory() 大小的堆内存还剩余很大一部分），设备当前内存也很充裕（ActivityManager.MemoryInfo.availMem 还有很多）。这些 OOM 的错误信息大致有下面两种：
### 2.1  这种 OOM 在 Android6.0，Android7.0 上各个机型均有发生，文中简称为 OOM 一，错误信息如下：
```
java.lang.OutOfMemoryError: Could not allocate JNI ENV

```
### 2.2 集中发生在 Android7.0 及以上的华为手机（EmotionUI_5.0 及以上）的 OOM，简称为 OOM 二，对应错误信息如下：
```
java.lang.OutOfMemoryError: pthread_create(1040KB stack) failed: Out of memory
```

## 3. 问题分析及解决
### 3.1代码分析
Android 系统中，OutOfMemoryError 这个错误是怎么被系统抛出的？下面基于 Android6.0 的代码进行简单分析：
 Android 虚拟机最终抛出OutOfMemoryError 的代码位于/art/runtime/thread.cc
```
void Thread:ThrowOutOfMemoryError (const char * msg)  参数msg携带了OOM时的错误信息
```
 搜索代码可以发现以下几个地方调用了上述方法抛出 OutOfMemoryError 错误
#### 3.1.1 第一个地方是堆操作时
```
系统源码文件：
	/art/runtime/gc/heap.cc
函数：
	void Heap::ThrowOutOfMemoryError(Thread* self, size_t byte_count, AllocatorType allocator_type)
抛出时的错误信息：
	oss << "Failed to allocate a " << byte_cout << " byte allocation withe " << total_bytes_free << " free bytes and " << PrettySize(GetFreeMemoryUntilOOME()) " unitl OOM "
```
这种抛出的其实就是堆内存不够用的时候，即前面提到的申请堆内存大小超过了Runtime.getRuntime().maxMemory()
#### 3.1.2 第二个地方是创建线程时
```
系统源码文件：
    /art/runtime/thread.cc
函数：
	void Thread::CreateNativeThread(JNIENV* evn, jobject java_peer, size_t stack_size, bool is_daemon)
抛出时错误信息：
	“could not allocate JNI Env"
    或者
    StringPrintf("pthread_create(%s stack) failed: %s, PrettySize(stack_size).c_star(), strerror(pthread_create_result)));
```
对比错误信息，可以知道我们遇到的 OOM 崩溃就是这个时机，即创建线程的时候（Thread::CreateNativeThread）产生的。
#### 3.1.3 还有其他的一些错误信息如“[XXXClassName] of length XXX would overflow”可能是系统限制String/Array 的长度所致，本文不做深入讨论
那么，我们关心的就是Thread::CreateNativeThread 时抛出的 OOM 错误，创建线程为什么会导致 OOM 呢？
##4. 推断&验证
既然抛出来 OOM，一定是线程创建过程中触发了某些我们不知道的限制，既然不是 Art 虚拟机为我们设置的堆上限，那么可能是更底层的限制。Android 系统基于 linux，所以 linux 的限制对于 Android 同样适用，这些限制有：
#### 4.1 /proc/pid/limits 描述着 linux 系统对对应进程的限制，下面是一个样例：
![](14847428-0787ef687868d4ae.jpeg)
用排除法筛选上面样例中的 limits：
- 	 Max stack size，Max processes 的限制是整个系统的，不是针对某个进程的，排除
- 	 Max locked memory ，排除，后面会分析，线程创建过程中分配线程私有 stack 使用的 mmap 调用没有设置 MAP_LOCKED，所以这个限制与线程创建过程无关 
- 	 Max pending signals，c 层信号个数阈值，无关，排除 
- 	 Max msgqueue size，Android IPC 机制不支持消息队列，排除

剩下的 limits 项中，Max open files 这一项限制最可疑Max open files 表示 每个进程最大打开文件的数目，进程 每打开一个文件就会产生一个文件描述符 fd（记录在 /proc/pid/fd 下面），这个限制表明 fd 的数目不能超过 Max open files 规定的数目。
后面分析线程创建过程中会发现过程中涉有及到文件描述符。
#### 4.2  /proc/sys/kernel 中描述的限制
这些限制中与线程相关的是 /proc/sys/kernel/threads-max，规定了每个进程创建线程数目的上限，所以线程创建导致 OOM 的原因也有可能与这个限制相关。
### 4.3 验证
下面对上述的推断进行验证，分两步：本地验证和线上验收。
- 	 本地验证：在本地验证推断，试图复现与图 [2-4]OOM 一与图 [2-5]OOM 二所示错误消息一致的 OOM
- 	 线上验收：下发插件，验收线上用户 OOM 时确实是由于上面的推断的原因导致的。

###4.3.1 本地验证
实验一： 触发大量网络连接（每个连接处于独立的线程中）并保持，每打开一个 socket 都会增加一个 fd（/proc/pid/fd 下多一项）
注：不只有这一种增加 fd 数的方式，也可以用其他方法，比如打开文件，创建 handlerthread 等等
- 实验预期：当进程 fd 数（可以通过 ls /proc/pid/fd | wc -l 获得）突破 /proc/pid/limits 中规定的 Max open files 时，产生 OOM；
- 实验结果：当 fd 数目到达 /proc/pid/limits 中规定的 Max open files 时，继续开线程确实会导致 OOM 的产生。
错误信息及堆栈如下：
![](14847428-f03d35827ec8ec80.jpeg)
可以看出，此 OOM 发生时的错误信息确与线上发现的 OOM 一的“Could not allocate JNI Env” 吻合，因此线上上报的 OOM 一 可能 就是由 FD 数超限导致的，不过最终确定需要到线上进行验证 (下一小节)。此外从 ART 虚拟机的 Log 中看出，还有一个关键的信息 “ art: ashmem_create_region failed for 'indirect ref table': Too many open files”，后面会用于问题定位及解释。
实验二：创建大量的空线程（不做任何事情，直接 sleep）
	* 实验预期：当线程数（可以在/proc/pid/status 中的threads项实时查看）超过/proc/sys/kernel/threads-max 中规定的上限时产生 OOM 崩溃。
	* 实验结果：在 Android7.0 及以上的华为手机（EmotionUI_5.0 及以上）的手机产生 OOM，这些手机的线程数限制都很小 (应该是华为 rom 特意修改的 limits)，每个进程只允许最大同时开 500 个线程，因此很容易复现了。
OOM 时错误信息如下：
![](14847428-d53d49a468d6b3a3.jpeg)
可以看出 错误信息与我们线上遇到的 OOM 二吻合："pthread_create (1040KB stack) failed: Out of memory" 另外 ART 虚拟机还有一个关键 Log：“pthread_create failed: clone failed: Out of memory”，后面会用于问题定位及解释。
其他 Rom 的手机线程数的上限都比较大，不容易复现上述问题。但是，对于 32 位的系统，当进程的逻辑地址空间不够的时候也会产生 OOM，每个线程通常需要 mapp 1MB 左右的 stack 空间（stack 大小可以自行设置），32 为系统进程逻辑地址 4GB，用户空间少于 3GB。逻辑地址空间不够（已用逻辑空间地址可以查看 /proc/pid/status 中的 VmPeak/VmSize 记录），此时创建线程产生的 OOM 具有如下信息：
![](14847428-262a6f7dc0eb901f.jpeg)

### 3.3.2 线上验收及问题解决
本地尝试复现的 OOM 错误信息中图 [3-5] 与线上 OOM 一情况比较吻合，图 [3-6] 与线上 OOM 二的情况比较吻合，但线上的 OOM 一真的时 FD 数目超限，OOM 二真的是由于华为手机线程数超限的原因导致的吗？最终确定还需要取线上设备的数据进行验证。
验证方法：
下发插件到线上用户，当 Thread.UncaughtExceptionHandler 捕获到OutOfMemoryError 时记录 /proc/pid 目录下的如下信息：
1. /proc/pid/fd 目录下文件数 (fd 数)
2. /proc/pid/status 中 threads 项（当前线程数目）
3. OOM 的日志信息（出了堆栈信息还包含其他的一些 warning 信息
线上 OOM 一验证，发生 OOM 一的线上设备中采集到的信息：
1. /proc/pid/fd 目录下文件数与 /proc/pid/limits 中的 Max open files 数目持平，证明 FD 数目已经满了；
2. 崩溃时日志信息与图基本一致；
由此，证明 线上的 OOM 一确实是由于 FD 数目过多导致的 OOM，推断验证成功。
OOM 一的定位与解决：
最终原因是 App 中使用的长连接库再某些时候会有瞬时发出大量 http 请求的 bug(导致 FD 数激增)，已修复。


线上 OOM 二验证 集中在华为系统的 OOM 二崩溃时收集到的信息样例如下，（收集的样例中包含的 devicemodel 有 VKY-AL00，TRT-AL00A，BLN-AL20，BLN-AL10，DLI-AL10，TRT-TL10，WAS-AL00 等）：
1. /proc/pid/status 中 threads 记录全部到达上限：Threads： 500；
2. 崩溃时日志信息与图 [3-6] 基本一致；
推断验证成功，即 线程数受限导致创建线程时 clone failed 导致了线上的 OOM 二。
OOM 二的定位与解决：
关于 App 业务代码中的问题还在定位修复中。
## 3.4 解释
下面从代码分析本文描述的 OOM 是怎么发生的，首先线程创建的简易版流程图如下所示：
![](14847428-0331ac7d58251d85.jpeg)
上图中，线程创建大概有两个关键的步骤：
	* 第一列中的 创建线程私有的结构体 JNIENV(JNI 执行环境，用于 C 层调用 Java 层代码)
	* 第二列中的 调用 posix C 库的函数 pthread_create 进行线程创建工作

下面对流程图中关键节点（图中有标号的）进行说明：
1. 图中节点①，/art/runtime/thread.cc 中的函数Thread:CreateNativeThread部分节选代码如下：
![](14847428-3ea4a86d26199c49.jpeg)
可知：
* JNIENV 创建不成功时产生 OOM 的错误信息为 "Could not allocate JNI Env"，与文中 OOM 一一致
pthread_create失败时抛出 OOM 的错误信息为"pthread_create (%s stack) failed: %s"．其中详细的错误信息由 pthread_create 的返回值（错误码）给出。错误码与错误描述的对应关系可以参见 bionic/libc/include/sys/_errdefs.h中的定义。文中 OOM 二的具体错误信息为"Out of memory"，就说明 pthread_create 的返回值为 12。
![](14847428-689ad98d836934d0.jpeg)
2. 图中节点②和③是创建 JNIENV 过程的关键节点，节点②/art/runtime/mem_map.cc 中 函数 MemMap:MapAnonymous 的作用是为 JNIENV 结构体中Indirect_Reference_table（C 层用于存储 JNI 局部 / 全局变量）申请内存，申请内存的方法是节点③所示的函数ashmem_create_region（创建一块 ashmen 匿名共享内存, 并返回一个文件描述符）。节点②代码节选如下：
![](14847428-e301d11b23f40872.jpeg)
我们线上的OOM 一的错误信息＂ashmem_create_region failed for 'indirect ref table': Too many open files＂，与此处打印的信息吻合。＂Too many open files＂的错误描述说明此处的 errno（系统全局错误标识）为 24(见图 [3-10] 系统错误定义 _errdefs.h)。由此看出我们线上的 OOM 一是由于文件描述符数目已满，ashmem_create_region 无法返回新的 FD 而导致的。

3. 图中节点④和⑤是调用 C 库创建线程时的环节，创建线程首先 调用 __allocate_thread 函数申请线程私有的栈内存 (stack) 等，然后 调用 clone 方法进行线程创建．申请 stack 采用的时 mmap 的方式，节点⑤代码节选如下：
![](14847428-f4e447c20717be2e.jpeg)
打印的错误信息与图 [3-7] 中进程逻辑地址占满导致的 OOM 错误信息吻合，图 [3-7] 中错误信息＂ Try again＂说明系统全局错误标识 errno 为 11(见图 [3-10] 系统错误定义_errdefs.h). pthread_create 过程中，节点４相关代码如下：
![](14847428-3292f4df5e5e2121.jpeg)
此处输出的错误日志"pthread_create failed: clone failed: %s"与我们线上发现的 OOM 二吻合，图 [3-6] 中的错误描述＂ Out of memory＂说明系统全局错误标识 errno 为 12(见图 [3-10] 系统错误定义 _errdefs.h)。 由此线上的 OOM 二就是由于线程数的限制而在节点 5 clone 失败导致 OOM。
## 4. 结论及监控
###4.1导致OOM发生的原因
综上，可以导致 OOM 的原因有以下几种：
	1. 文件描述符 (fd) 数目超限，即 proc/pid/fd 下文件数目突破 /proc/pid/limits 中的限制。可能的发生场景有：短时间内大量请求导致 socket 的 fd 数激增，大量（重复）打开文件等 ；
	2. 线程数超限，即proc/pid/status中记录的线程数（threads 项）突破 /proc/sys/kernel/threads-max 中规定的最大线程数。可能的发生场景有：app 内多线程使用不合理，如多个不共享线程池的 OKhttpclient 等等 ；
	3. 传统的 java 堆内存超限，即申请堆内存大小超过了Runtime.getRuntime().maxMemory()；
	4. （低概率）32 为系统进程逻辑空间被占满导致 OOM；
	5. 其他。
###4.2 监控措施
可以利用 linux 的 inotify 机制进行监控：
	* watch /proc/pid/fd来监控 app 打开文件的情况，
	* watch /proc/pid/task来监控线程使用情况。

