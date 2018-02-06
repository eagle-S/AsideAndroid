http://www.cnblogs.com/helloworldtoyou/p/8378604.html

http://blog.csdn.net/jingxia2008/article/details/26701899
http://www.ailab.com.cn/article-1037-161174-1.html


android/device/softwinner/common/hardware/audio/audio_hw.c
路径：AUDIO_XML_PATH "/system/etc/codec_paths.xml"
 AUDIO_XML_PATH "/system/etc/ac100_paths.xml"

在以下函数中会调用到audio_route_init函数
select_output_device 
select_input_device
start_output_stream 
adev_open 

android/system/media/audio_route/audio_route.c
audio_route_init

路径

#define AUDIO_XML_PATH "/system/etc/codec_paths.xml"
#define AUDIO_XML_PATH "/system/etc/ac100_paths.xml"