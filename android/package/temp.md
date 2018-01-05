1. 系统中内置xx.apk，通过adb install -r 安装同版本号apk，重启机器后安装的apk会消失

获取未安装apk包信息：
private String getVersionName(String apkPath) {  
        PackageInfo pi = pm.getPackageArchiveInfo(apkPath,  
                PackageManager.GET_ACTIVITIES);  
        String versionName = null;  
        if (pi != null) {  
            versionName = pi.versionName;  
        }  
        return versionName;  
    }  
  

Android应用APK安装的方式
http://cstsinghua.github.io/2016/06/13/Android%E5%AE%89%E8%A3%85APK%E8%AF%A6%E8%A7%A3/