原始的android代码，系统默认语言是英文，一般来说需要改成默认中文，修改的方法很多：

1.修改PRODUCT_LOCALES字段，

将要选择的语言放在第一位，如： PRODUCT_LOCALES := en_US zh_CN 默认语言是英语，这个从build/target/product/sdk.mk中拷贝出来，粘贴到device/{硬件平台}/{产品}.mk中，将zh_CN放在第一位即可。或者直接粘贴到build/target/product/core.mk中，所有分支都继承这个设置。



2.修改device/{硬件平台}/{产品}/system.prop或者default.prop，加入：

　　[persist.sys.language]: [zh]

　　[persist.sys.country]: [CN]

　　[persist.sys.localevar]: []

　　[persist.sys.timezone]: [Asia/Shanghai]

　　[ro.product.locale.language]: [zh]

　　[ro.product.locale.region]: [CN]


3.修改init.rc，加入:

　　setprop persist.sys.language zh

　　setprop persist.sys.country CN

　　setprop persist.sys.localevar 

　　setprop persist.sys.timezone Asia/Shanghai

　　setprop ro.product.locale.language zh

　　setprop ro.product.locale.region CN

       这个方法有个问题，因为每次开机都会执行，所以每次开机后语言都是默认语言。



4.修改device/{硬件平台}/{产品}/device.mk，加入:

PRODUCT_PROPERTY_OVERRIDES += \

　　persist.sys.language=zh \

　　persist.sys.country=CN \

　　persist.sys.localevar=  “”\

　　persist.sys.timezone=Asia/Shanghai \

　　ro.product.locale.language=zh \

　　ro.product.locale.region=CN 


我采用的是第4种。注意，上面的引号不能去掉，否则在build.prop中两行会变成一行：

　　persist.sys.localevar=persist.sys.timezone=Asia/Shanghai

这会导致获取不到persist.sys.timezone的值，时区还是不对。



5.修改build/tools/buildinfo.sh：

　　echo "persist.sys.language=zh"

　　echo "persist.sys.country=CN"

　　echo "persist.sys.localevar="

　　echo "persist.sys.timezone=Asia/Shanghai"

　　echo "ro.product.locale.language=zh"

　　echo "ro.product.locale.region=CN"