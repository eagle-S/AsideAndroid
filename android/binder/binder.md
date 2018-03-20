# Binder

## /dev/binder 创建

/dev/binder在system/core/rootdir/ueventd.rc中创建

ueventd.rc在 /system/core/init/ueventd.c中加载

ueventd在/system/core/rootdir/init.rc启动

init.rc在/system/core/init/init.c中加载




frameworks/base/core/java/com/android/internal/os/BinderInternal.java
对应JNI类frameworks/base/core/jni/android_util_Binder.cpp
关注GC计数



https://www.jianshu.com/p/33c96606d604
http://gityuan.com/tags/#binder