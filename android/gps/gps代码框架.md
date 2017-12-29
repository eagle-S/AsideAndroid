[TOC]

### 相关代码

frameworks/base/location/java/android/location/LocationManager.java

frameworks/base/location/java/android/location/IGpsStatusListener.aidl
frameworks/base/location/java/android/location/IGpsStatusProvider.aidl
frameworks/base/services/java/com/android/server/LocationManagerService.java
frameworks/base/services/java/com/android/server/location/GpsLocationProvider.java
frameworks/base/services/jni/com_android_server_location_GpsLocationProvider.cpp

hardware/libhardware/include/hardware/gps.h
device/softwinner/t3-common/hardware/gps/gps.c

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

LocationManagerService在SystemServer启动时加入到ServiceManager中，服务名称为location。

### GPS模块初始化

LocationManagerService.java中导入了类GpsLocationProvider，GpsLocationProvider的static代码会被调用

```java
// frameworks/base/services/java/com/android/server/location/GpsLocationProvider.java
static { class_init_native(); }

private static native void class_init_native();
```

看下JNI调用实现：

```c++
// frameworks/base/services/jni/com_android_server_location_GpsLocationProvider.cpp
static void android_location_GpsLocationProvider_class_init_native(JNIEnv* env, jclass clazz) {
    int err;
    hw_module_t* module;

    method_reportLocation = env->GetMethodID(clazz, "reportLocation", "(IDDDFFFJ)V");
    method_reportStatus = env->GetMethodID(clazz, "reportStatus", "(I)V");
    method_reportSvStatus = env->GetMethodID(clazz, "reportSvStatus", "()V");
    method_reportAGpsStatus = env->GetMethodID(clazz, "reportAGpsStatus", "(III)V");
    method_reportNmea = env->GetMethodID(clazz, "reportNmea", "(J)V");
    method_setEngineCapabilities = env->GetMethodID(clazz, "setEngineCapabilities", "(I)V");
    method_xtraDownloadRequest = env->GetMethodID(clazz, "xtraDownloadRequest", "()V");
    method_reportNiNotification = env->GetMethodID(clazz, "reportNiNotification",
            "(IIIIILjava/lang/String;Ljava/lang/String;IILjava/lang/String;)V");
    method_requestRefLocation = env->GetMethodID(clazz,"requestRefLocation","(I)V");
    method_requestSetID = env->GetMethodID(clazz,"requestSetID","(I)V");
    method_requestUtcTime = env->GetMethodID(clazz,"requestUtcTime","()V");
    method_reportGeofenceTransition = env->GetMethodID(clazz,"reportGeofenceTransition",
            "(IIDDDFFFJIJ)V");
    method_reportGeofenceStatus = env->GetMethodID(clazz,"reportGeofenceStatus",
            "(IIDDDFFFJ)V");
    method_reportGeofenceAddStatus = env->GetMethodID(clazz,"reportGeofenceAddStatus",
            "(II)V");
    method_reportGeofenceRemoveStatus = env->GetMethodID(clazz,"reportGeofenceRemoveStatus",
            "(II)V");
    method_reportGeofenceResumeStatus = env->GetMethodID(clazz,"reportGeofenceResumeStatus",
            "(II)V");
    method_reportGeofencePauseStatus = env->GetMethodID(clazz,"reportGeofencePauseStatus",
            "(II)V");

    err = hw_get_module(GPS_HARDWARE_MODULE_ID, (hw_module_t const**)&module);
    if (err == 0) {
        hw_device_t* device;
        err = module->methods->open(module, GPS_HARDWARE_MODULE_ID, &device);
        if (err == 0) {
            gps_device_t* gps_device = (gps_device_t *)device;
            sGpsInterface = gps_device->get_gps_interface(gps_device);
        }
    }
    if (sGpsInterface) {
        sGpsXtraInterface =
            (const GpsXtraInterface*)sGpsInterface->get_extension(GPS_XTRA_INTERFACE);
        sAGpsInterface =
            (const AGpsInterface*)sGpsInterface->get_extension(AGPS_INTERFACE);
        sGpsNiInterface =
            (const GpsNiInterface*)sGpsInterface->get_extension(GPS_NI_INTERFACE);
        sGpsDebugInterface =
            (const GpsDebugInterface*)sGpsInterface->get_extension(GPS_DEBUG_INTERFACE);
        sAGpsRilInterface =
            (const AGpsRilInterface*)sGpsInterface->get_extension(AGPS_RIL_INTERFACE);
        sGpsGeofencingInterface =
            (const GpsGeofencingInterface*)sGpsInterface->get_extension(GPS_GEOFENCING_INTERFACE);
    }
}
```

初始化函数主要做了两件事：

1. 获取java层函数，供后续回调
2. 初始化gps hal模块，获取GpsInterface

#### gps HAL接口介绍

##### hw_module_t/gps_device_t初始化

```c
// hardware/libhardware/include/hardware/gps.h
struct gps_device_t {
    struct hw_device_t common;

    /**
     * Set the provided lights to the provided values.
     *
     * Returns: 0 on succes, error code on failure.
     */
    const GpsInterface* (*get_gps_interface)(struct gps_device_t* dev);
};
```

```c
// device/softwinner/t3-common/hardware/gps/gps.c
static int open_gps(const struct hw_module_t* module, char const* name, struct hw_device_t** device)
{
    D("GPS dev open_gps");
    struct gps_device_t *dev = malloc(sizeof(struct gps_device_t));
    memset(dev, 0, sizeof(*dev));

    dev->common.tag = HARDWARE_DEVICE_TAG;
    dev->common.version = 0;
    dev->common.module = (struct hw_module_t*)module;
    dev->get_gps_interface = gps_get_hardware_interface;

    *device = &dev->common;
    return 0;
}

static struct hw_module_methods_t gps_module_methods = {
    .open = open_gps
};


struct hw_module_t HAL_MODULE_INFO_SYM = {
    .tag = HARDWARE_MODULE_TAG,
    .version_major = 1,
    .version_minor = 0,
    .id = GPS_HARDWARE_MODULE_ID,
    .name = "Serial GPS Module",
    .author = "Keith Conger",
    .methods = &gps_module_methods,
};
```

在结构体gps_device_t中提供了获取对外接口GpsInterface的方法get_gps_interface

##### GpsInterface声明及初始化

GpsInterface声明

```c
// hardware/libhardware/include/hardware/gps.h
typedef struct {
    size_t          size;

    int   (*init)( GpsCallbacks* callbacks );

    int   (*start)( void );

    int   (*stop)( void );

    void  (*cleanup)( void );

    int   (*inject_time)(GpsUtcTime time, int64_t timeReference,
                         int uncertainty);

    int  (*inject_location)(double latitude, double longitude, float accuracy);

    void  (*delete_aiding_data)(GpsAidingData flags);

    int   (*set_position_mode)(GpsPositionMode mode, GpsPositionRecurrence recurrence,
            uint32_t min_interval, uint32_t preferred_accuracy, uint32_t preferred_time);

    const void* (*get_extension)(const char* name);
} GpsInterface
```

GpsInterface初始化

```c
// device/softwinner/t3-common/hardware/gps/gps.c
static const GpsInterface  serialGpsInterface = {
    sizeof(GpsInterface),
    serial_gps_init,
    serial_gps_start,
    serial_gps_stop,
    serial_gps_cleanup,
    serial_gps_inject_time,
    serial_gps_inject_location,
    serial_gps_delete_aiding_data,
    serial_gps_set_position_mode,
    serial_gps_get_extension,
};

const GpsInterface* gps_get_hardware_interface()
{
    D("GPS dev get_hardware_interface");
    return &serialGpsInterface;
}
```

serialGpsInterface将GpsInterface接口与类中的函数对应起来了。

从以上逻辑看出，class_init_native函数初始化gps hal模块，并通过函数gps_get_hardware_interface获取到了gps接口GpsInterface

### 设置监听

在ActivityManagerService.self().systemReady回调中，将会调用LocationManagerService.systemRunning()初始化一些状态

```java
// frameworks/base/services/java/com/android/server/SystemServer.java
    try {
        if (locationF != null) locationF.systemRunning();
    } catch (Throwable e) {
        reportWtf("Notifying Location Service running", e);
    }
```

LocationManagerService.systemRunning()实现：

```java
// frameworks/base/services/java/com/android/server/LocationManagerService.java
    public void systemRunning() {
        synchronized (mLock) {
            if (D) Log.d(TAG, "systemReady()");

            // fetch package manager
            mPackageManager = mContext.getPackageManager();

            // fetch power manager
            mPowerManager = (PowerManager) mContext.getSystemService(Context.POWER_SERVICE);

            // prepare worker thread
            mLocationHandler = new LocationWorkerHandler(BackgroundThread.get().getLooper());

            // prepare mLocationHandler's dependents
            mLocationFudger = new LocationFudger(mContext, mLocationHandler);
            mBlacklist = new LocationBlacklist(mContext, mLocationHandler);
            mBlacklist.init();
            mGeofenceManager = new GeofenceManager(mContext, mBlacklist);

            // Monitor for app ops mode changes.
            AppOpsManager.OnOpChangedListener callback
                    = new AppOpsManager.OnOpChangedInternalListener() {
                public void onOpChanged(int op, String packageName) {
                    synchronized (mLock) {
                        for (Receiver receiver : mReceivers.values()) {
                            receiver.updateMonitoring(true);
                        }
                        applyAllProviderRequirementsLocked();
                    }
                }
            };
            mAppOps.startWatchingMode(AppOpsManager.OP_COARSE_LOCATION, null, callback);

            // prepare providers
            loadProvidersLocked();
            updateProvidersLocked();
        }

        // listen for settings changes
        mContext.getContentResolver().registerContentObserver(
                Settings.Secure.getUriFor(Settings.Secure.LOCATION_PROVIDERS_ALLOWED), true,
                new ContentObserver(mLocationHandler) {
                    @Override
                    public void onChange(boolean selfChange) {
                        synchronized (mLock) {
                            updateProvidersLocked();
                        }
                    }
                }, UserHandle.USER_ALL);
        mPackageMonitor.register(mContext, mLocationHandler.getLooper(), true);

        // listen for user change
        IntentFilter intentFilter = new IntentFilter();
        intentFilter.addAction(Intent.ACTION_USER_SWITCHED);

        mContext.registerReceiverAsUser(new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                String action = intent.getAction();
                if (Intent.ACTION_USER_SWITCHED.equals(action)) {
                    switchUser(intent.getIntExtra(Intent.EXTRA_USER_HANDLE, 0));
                }
            }
        }, UserHandle.ALL, intentFilter, null, mLocationHandler);
    }
```

在loadProvidersLocked()函数中会初始化GpsLocationProvider，并将其缓存到mProviders中，updateProvidersLocked()函数会根据设置启用对应的LocationProvider。最终会通过GpsLocationProvider的enable()函数调用到native_init()。

native_init()在JNI实现中对应android_location_GpsLocationProvider_init函数：

```c
// frameworks/base/services/jni/com_android_server_location_GpsLocationProvider.cpp

GpsCallbacks sGpsCallbacks = {
    sizeof(GpsCallbacks),
    location_callback,
    status_callback,
    sv_status_callback,
    nmea_callback,
    set_capabilities_callback,
    acquire_wakelock_callback,
    release_wakelock_callback,
    create_thread_callback,
    request_utc_time_callback,
};

static jboolean android_location_GpsLocationProvider_init(JNIEnv* env, jobject obj)
{
    // this must be set before calling into the HAL library
    if (!mCallbacksObj)
        mCallbacksObj = env->NewGlobalRef(obj);

    // fail if the main interface fails to initialize
    if (!sGpsInterface || sGpsInterface->init(&sGpsCallbacks) != 0)
        return false;

    // if XTRA initialization fails we will disable it by sGpsXtraInterface to NULL,
    // but continue to allow the rest of the GPS interface to work.
    if (sGpsXtraInterface && sGpsXtraInterface->init(&sGpsXtraCallbacks) != 0)
        sGpsXtraInterface = NULL;
    if (sAGpsInterface)
        sAGpsInterface->init(&sAGpsCallbacks);
    if (sGpsNiInterface)
        sGpsNiInterface->init(&sGpsNiCallbacks);
    if (sAGpsRilInterface)
        sAGpsRilInterface->init(&sAGpsRilCallbacks);
    if (sGpsGeofencingInterface)
        sGpsGeofencingInterface->init(&sGpsGeofenceCallbacks);

    return true;
}
```

在上述实现中调用了GpsInterface->init函数，将GpsCallbacks赋值给GpsInterface，这样gps数据就能通过回调返回给上层应用。

|  HAL   |     JNI                 |   java  |
|--------|-------------------------|---------|
|        |location_callback        |reportLocation|
|        |status_callback          |reportStatus|
|        |sv_status_callback       |reportSvStatus|
|        |nmea_callback            |reportNmea|
|        |set_capabilities_callback|setEngineCapabilities|
|        |acquire_wakelock_callback|NA|
|        |release_wakelock_callback|NA|
|        |create_thread_callback   |NA|
|        |request_utc_time_callback|requestUtcTime|

### 上报数据

```c
// device/softwinner/t3-common/hardware/gps/gps.c
static int
serial_gps_init(GpsCallbacks* callbacks)
{
    D("serial_gps_init");
    GpsState*  s = _gps_state;

    if (!s->init)
        gps_state_init(s, callbacks);

    if (s->fd < 0)
        return -1;

    return 0;
}

static void
gps_state_init( GpsState*  state, GpsCallbacks* callbacks )
{
    char   prop[PROPERTY_VALUE_MAX];
    char   baud[PROPERTY_VALUE_MAX];
    char   device[256];
    int    ret;
    int    done = 0;

    struct sigevent tmr_event;

    state->init       = 1;
    state->control[0] = -1;
    state->control[1] = -1;
    state->fd         = -1;
    state->callbacks  = callbacks;
    D("gps_state_init");

    // Look for a kernel-provided device name
    if (property_get("ro.kernel.android.gps",prop,"ttyS2") == 0) {
        D("no kernel-provided gps device name");
        return;
    }

	_gps_mLocation_p=&_gps_mLocation;
    snprintf(device, sizeof(device), "/dev/%s",prop);
    do {
        state->fd = open( device, O_RDWR );
    } while (state->fd < 0 && errno == EINTR);

    if (state->fd < 0) {
        ALOGE("could not open gps serial device %s: %s", device, strerror(errno) );
        return;
    }

   ..........

    state->thread = callbacks->create_thread_cb( "gps_state_thread", gps_state_thread, state );

    if ( !state->thread ) {
        ALOGE("Could not create GPS thread: %s", strerror(errno));
        goto Fail;
    }

    D("GPS state initialized");

    return;

Fail:
    gps_state_done( state );
}
```





gps框架及简析
http://blog.csdn.net/fantasyhujian/article/details/9262463

Android GPS架构分析
http://blog.csdn.net/g_linuxer_/article/details/50979861

Android 系统中 Location Service 的实现与架构
https://www.ibm.com/developerworks/cn/opensource/os-cn-android-location/index.html