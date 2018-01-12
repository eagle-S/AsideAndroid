http://blog.csdn.net/luoshengyang/article/details/7691321

http://windrunnerlihuan.com/2017/05/02/Android-SurfaceFlinger-%E5%AD%A6%E4%B9%A0%E4%B9%8B%E8%B7%AF-%E4%B8%89-Android%E5%BC%80%E6%9C%BA%E5%8A%A8%E7%94%BB%E6%B5%81%E7%A8%8B%E7%AE%80%E8%BF%B0/


开机速度受launcher启动速度影响

在启动的第一个activity处于Idle状态时才会去结束开机动画，调用栈：
```
java.lang.RuntimeException: here
        at com.android.server.wm.WindowManagerService.enableScreenAfterBoot(WindowManagerService.java:5231)
        at com.android.server.am.ActivityManagerService.enableScreenAfterBoot(ActivityManagerService.java:5363)
        at com.android.server.am.ActivityStackSupervisor.activityIdleInternalLocked(ActivityStackSupervisor.java:1932)
        at com.android.server.am.ActivityManagerService.activityIdle(ActivityManagerService.java:5336)
        at android.app.ActivityManagerNative.onTransact(ActivityManagerNative.java:409)
        at com.android.server.am.ActivityManagerService.onTransact(ActivityManagerService.java:2129)
        at android.os.Binder.execTransact(Binder.java:404)
        at dalvik.system.NativeStart.run(Native Method)
```