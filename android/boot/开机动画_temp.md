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

开机广播，调用栈：
```
finishBooting
java.lang.RuntimeException: here
      at com.android.server.am.ActivityManagerService.finishBooting(ActivityManagerService.java:5458)
      at com.android.server.am.ActivityStackSupervisor.activityIdleInternalLocked(ActivityStackSupervisor.java:1920)
      at com.android.server.am.ActivityManagerService.activityIdle(ActivityManagerService.java:5336)
      at android.app.ActivityManagerNative.onTransact(ActivityManagerNative.java:409)
      at com.android.server.am.ActivityManagerService.onTransact(ActivityManagerService.java:2129)
      at android.os.Binder.execTransact(Binder.java:404)
      at dalvik.system.NativeStart.run(Native Method)
```

```java
//  frameworks/base/services/java/com/android/server/am/ActivityStackSupervisor.java
    final ActivityRecord activityIdleInternalLocked(final IBinder token, boolean fromTimeout,
            Configuration config) {
        .............

        if (booting) {
            mService.finishBooting();
        } else if (startingUsers != null) {
            for (int i = 0; i < startingUsers.size(); i++) {
                mService.finishUserSwitch(startingUsers.get(i));
            }
        }

        mService.trimApplications();

        if (enableScreen) {
            mService.enableScreenAfterBoot();
        }
```

boot_progress_start: 3401
boot_progress_preload_start: 4214
boot_progress_preload_end: 5601
boot_progress_system_run: 5725
boot_progress_pms_start: 5860
boot_progress_pms_system_scan_start: 6083
boot_progress_pms_data_scan_start: 6973
boot_progress_pms_scan_end: 6974
boot_progress_pms_ready: 7074
boot_progress_ams_ready: 7814
boot_progress_enable_screen: 11408
SurfaceFlinger: Total boot time is (11704 ms)
```

打印代码位置：
boot_progress_start
init.c ->  app_main.main  -> AndroidRuntime.start
LOG_EVENT_LONG(LOG_BOOT_PROGRESS_START, ns2ms(systemTime(SYSTEM_TIME_MONOTONIC)));


boot_progress_preload_start
boot_progress_preload_end
ZygoteInit.main
EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START, SystemClock.uptimeMillis());
preload();			
EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END, SystemClock.uptimeMillis());


boot_progress_system_run
SystemServer.initAndLoop
EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_SYSTEM_RUN,
            SystemClock.uptimeMillis());

boot_progress_pms_start
boot_progress_pms_system_scan_start
boot_progress_pms_data_scan_start
boot_progress_pms_scan_end
boot_progress_pms_ready
PackageManagerService.new
EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_START, SystemClock.uptimeMillis());
EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_SYSTEM_SCAN_START, startTime);
EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_DATA_SCAN_START, SystemClock.uptimeMillis());
EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_SCAN_END, SystemClock.uptimeMillis());
EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_READY, SystemClock.uptimeMillis());

boot_progress_ams_ready
ActivityManagerService.systemReady
EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_AMS_READY, SystemClock.uptimeMillis());

boot_progress_enable_screen
ActivityManagerService.enableScreenAfterBoot
EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_ENABLE_SCREEN, SystemClock.uptimeMillis());

Total boot time is (11704 ms)
SurfaceFlinger.bootFinished
ALOGI("Total boot time is (%ld ms)", long(ns2ms(now)) );