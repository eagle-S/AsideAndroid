
### framework接口
代码：frameworks/base/media/java/android/media/
编译：mmm -B frameworks/base
输出：/system/framework/framework.jar
包含：android.media.* API 与音频硬件进行交互

### JNI
代码：frameworks/base/core/jni
编译：mmm -B frameworks/base/core/jni
输出：/system/lib/libandroid_runtime.so

frameworks/base/media/jni
编译：mmm -B frameworks/base/media/jni
输出：/system/lib/libaudioeffect_jni.so

### native原生框架/Binder IPC
代码：frameworks/av/media/libmedia
编译：mmm -B frameworks/av/media/libmedia
输出：/system/lib/libmedia.so

### native媒体服务
代码：frameworks/av/services/audioflinger
编译：mmm -B frameworks/av/services/audioflinger/
输出：/system/lib/libaudioflinger.so

### HAL
代码：hardware/libhardware/include/hardware
编译：mmm -B hardware/libhardware
输出：/system/lib/libhardware.so
     /system/lib/hw/audio.primary.default.so
     /system/lib/hw/audio_policy.stub.so

代码：hardware/libhardware_legacy/include/hardware_legacy/
编译：mmm -B hardware/libhardware_legacy
输出：/system/lib/libhardware_legacy.so
     /system/lib/hw/audio_policy.default.so

### 内核驱动程序
代码：external/tinyalsa
编译：mmm -B external/tinyalsa
输出：/system/lib/libtinyalsa.so

![image](/Image/audio/ape_fwk_audio.png)
来源:androidhttps://source.android.com/devices/audio/?hl=zh-cn

![image](/Image/audio/audio_fwk1.png)
![image](/Image/audio/audio_fwk2.png)
来源：https://source.android.com/devices/audio/?hl=zh-cn


Android系统Audio框架介绍
https://source.android.com/devices/audio/?hl=zh-cn

Android系统Audio框架介绍
http://blog.csdn.net/yangwen123/article/details/39502689