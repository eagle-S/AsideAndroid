
# APK安装

```java
        PackageManager pm = getPackageManager();
        pm.installPackage(Uri.fromFile(new File("/sdcard/NaviOne/apk/CldNavi_1612CC_CHANGJIANG_EV.apk")),
                new PackageInstallObserver(), PackageManager.INSTALL_REPLACE_EXISTING, null);
```