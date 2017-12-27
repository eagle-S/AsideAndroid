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
2. 初始化gps hal接口

#### gps HAL接口介绍

##### gps_device_t
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

### LocationManagerService初始化
```java
```


### 设置监听



### 上报数据


gps框架及简析
http://blog.csdn.net/fantasyhujian/article/details/9262463

Android GPS架构分析
http://blog.csdn.net/g_linuxer_/article/details/50979861

Android 系统中 Location Service 的实现与架构
https://www.ibm.com/developerworks/cn/opensource/os-cn-android-location/index.html