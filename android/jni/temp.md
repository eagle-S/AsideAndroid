安卓JNI提高
http://blog.csdn.net/u010657219/article/details/43053493


JNI 实战全面解析
http://blog.csdn.net/banketree/article/details/40535325


/device/softwinner/t3-common/hardware/gps/gps.c

gps_state_init函数中state->thread = callbacks->create_thread_cb( "gps_state_thread", gps_state_thread, state );

Y:\kitkat_T3\android\frameworks\base\services\jni/com_android_server_location_GpsLocationProvider.cpp
调用android_location_GpsLocationProvider_init  然后调用到 gps_state_init( GpsState*  state, GpsCallbacks* callbacks )