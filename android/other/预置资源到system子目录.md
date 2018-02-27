# 预置资源到/system子目录

在mk文件中添加 PRODUCT_COPY_FILES 关键字拷贝，不需要单独去创建拷贝后的文件。

## 单独拷贝

PRODUCT_COPY_FILES += \
    frameworks/native/data/etc/android.hardware.wifi.xml:system/etc/permissions/android.hardware.wifi.xml \
    frameworks/native/data/etc/android.hardware.wifi.direct.xml:system/etc/permissions/android.hardware.wifi.direct.xml \
    frameworks/native/data/etc/android.hardware.bluetooth.xml:system/etc/permissions/android.hardware.bluetooth.xml \
    frameworks/native/data/etc/android.hardware.bluetooth_le.xml:system/etc/permissions/android.hardware.bluetooth_le.xml \

## 批量拷贝

PRODUCT_COPY_FILES += \
    $(call find-copy-subdir-files,*,device/samsung/common/hardware/camera/libaw360/aw360,system/etc/aw360)

PRODUCT_COPY_FILES += \
    $(call find-copy-subdir-files,*.bmp,device/samsung/common/hardware/camera/libaw360/aw360,system/etc/aw360)
