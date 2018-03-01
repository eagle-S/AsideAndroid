ADB通过WIFI连接Android设备

原文：http://vjson.com/wordpress/adb%E9%80%9A%E8%BF%87wifi%E8%BF%9E%E6%8E%A5android%E8%AE%BE%E5%A4%87.html


通常情况下，我们都通过USB线连接Android设备，以此达到调试的目的，但是我相信你一定遇到过下面的问题。

USB线比较松的时候，ADB经常断开。
USB线容易绊脚，这个时候要么人摔倒，要么手机碎屏。
如果你的开发环境时Windows系统，当连接USB线的时候，QQ,360等程序会自动连接ADB，它们也会导致ADB断开。
那么有什么办法可以解决上面的问题呢？答案是肯定的，ADB支持USB连接模式和TCPIP链接模式。我们可以用TCPIP模式通过WIFI无线连接ADB。设置非常简单。

第一步

确保电脑和Android设备连接在同一个WIFI网络环境。

第二步

用USB线连接Android设备。连接上之后你的电脑就会检查到设备并且ADB将会以USB模式启动。可以通过adb devices命令检查连接上的设备，用adb usb命令确认adb是运行在usb模式下面。

$ adb devices
List of devices attached
04bdc4c9252391b9    device
$ adb usb
restarting in USB mode

第三步

用adb tcpip模式重启adb

$ adb tcpip 5555
restarting in TCP mode port: 5555

第四步

查看Android设备的IP地址，这里有三种方式查看Android设备IP。

设置－关于手机－状态信息－ip地址中查看
设置－WLAN-点击当前链接上的Wi-Fi查看IP
通过ADB命令查看设备IP地址：adb shell netcfg

第五步

知道设备IP地址之后，就可以用adb connect命令通过IP和端口号连接ADB了。


$ adb connect 192.168.1.3:5555
connected to 192.168.1.3:5555
//查看一下连接上的设备，usb连接和wifi连接都存在
adb devices
List of devices attached
04bdc4c9252391b9    device
192.168.1.3:5555    device

拔掉USB线，你会发现设备仍然是连接上的，如果没有连接上，用刚才的命令重现尝试一下。

总结

采用wifi连接ADB和uiautomotor结合起来可以用来在usb线的状态下跑测试脚本，对于测试人员来说也是非常有帮助的。

查看端口号：
getprop service.adb.tcp.port