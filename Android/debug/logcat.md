### logcat 命令
system/core/logcat/logcat.cpp

### event tag
frameworks/base/core/java/android/util/EventLog.java
frameworks/base/core/jni/android_util_EventLog.cpp



python脚本build/tools/java-event-log-tags.py则负责将EventLogTags.logtags以及调用转化为java文件，或者是将java文件中的writeEvent调用转为标准的java调用，以及生成system/etc/event-log-tags文件

以frameworks/base/services/core/java/com/android/server/am/EventLogTags.logtags文件为例，该文件编译过程中通过python脚本生成的对应java文件在out目录中：

out/target/common/obj/JAVA_LIBRARIES/services.core_intermediates/src/com/android/server/am/EventLogTags.java

<http://blog.csdn.net/yaowei514473839/article/details/53513435>