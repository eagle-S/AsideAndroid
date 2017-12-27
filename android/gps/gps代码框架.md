
### 代码路径



### LocationManagerService启动
```java
// frameworks/base/services/java/com/android/server/SystemServer.java
    try {
        Slog.i(TAG, "Location Manager");
        location = new LocationManagerService(context);
        ServiceManager.addService(Context.LOCATION_SERVICE, location);
    } catch (Throwable e) {
        reportWtf("starting Location Manager", e);
    }
```

