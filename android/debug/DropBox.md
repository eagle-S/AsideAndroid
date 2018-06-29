从android API8开始，别个引进了个大神，就是dropboxmangaer，干嘛呢？持续化存储系统数据的机制，主要是用来记录安卓在运行过程中，内核，系统进程，用户进程出现严重问题时候的log，英语来说：Enqueues chunks of data (from various sources – application crashes, kernel log records, etc.). The queue is size bounded and will drop old data if the enqueued data exceeds the maximum size. You can think of this as a persistent, system-wide, blob-oriented “logcat”.

记录那些错误呢：

1.crash：当java层遇到没被catch的Exception时，就会记录一次crash到dropbox中
2015-10-08 17:12:17 data_app_crash (text, 1222 bytes)

2.anr：当应用的主线程长时间没得到任何响应时，就会记录一次anr，关于anr的详细分析和提供的log我会再开贴说明

3.wtf（what a terrible failure） :在代码中主动报告一个不应当发生的情况，这个一旦发生很可能会终止当前的应用程序进程

4.lowmem：低内存，当内存不足时，安卓就会去终止后台应用程序来释放内存，如果已经没有后台应用可以被释放时，就会记录一次lowmem，这个其实一般没人会管。。。

5.system serve系统服务启动后的一些检测，首先要明白，系统服务在启动完成后会进行一系列的自检，包含：

 SYSTEM_BOOT：开机，每次开机都会增加一条SYSTEM_BOOT记录
 
 SYSTEM_RESTART：表明系统重启，就是system server不是开机后的第一次启动，这里启动2次，就是bug

 Kernel Panic (内核错误)：在发生内核崩溃时，内核会记录一些log信息道文件系统，因为内核挂了，这时候没有其他机会去记录错误信息了，唯一能检测内核崩溃的办法就是在手机重新启动后去检查这些log文件是不是还在，如果在，就说明上一次手机的重启就是因为内核崩溃导致，这些具体的日志分别是：
   SYSTEM_LAST_KMSG  关于它的说明又可以写一篇文章了，这个稍候
   APANIC_CONSOLE
   APANIC_THREADS

 System Recovery:系统恢复，因为系统恢复而重启，增加一条SYSTEM_RECOVERY_LOG

6.SYSTEM_TOMBSTONE (Native 进程的崩溃)墓碑是用来记录Native进程崩溃的日志的，这里科普下Native层以及安卓各个分层：

  1.java应用层：就是java语言搞得各个app
  2.java框架层：就是framework层：你写的小app之所以能够被识别和操作，主要是因为这一层的支持，
  3.Native层：本地服务和链接库，lib啊之类的大本营，这一层由C和她姐C++统领，主要作用，承上启下，上接java下接kernel硬件交互~，所以这一层的bug，长虹这帮人是几乎无解的，求助三方或者mtk把
  4.kernel，所以看看jd中动不动要求常用的linux命令啊什么的，明白了吧，最终和底层玩的好的，还是linux，

7.watchdog ,如果 WatchDog 监测到系统进程(system_server)出现问题, 会增加一条 watchdog 记录到 DropBoxManager 中, 并终止系统进程的执行.有一种重启就是看门狗引起的~~

8.netstats_error: NetworkStatsService 负责收集并持久化存储网络状态的统计数据, 当遇到明显的网络状态错误时, 它会增加一条 netstats_error 记录到 DropBoxManager.


http://www.51testing.com/index.php?uid/164996/action/viewspace/itemid/3686961/php/1







