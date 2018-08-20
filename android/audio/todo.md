Android 4.4 mediaserver 启动

/system/core/rootdir/init.rc:service media /system/bin/mediaserver

main类：
Z:\android\frameworks\av\media\mediaserver\main_mediaserver.cpp

main中启动服务：
        AudioFlinger::instantiate();
        MediaPlayerService::instantiate();
        CameraService::instantiate();
        AudioPolicyService::instantiate();


Android7.1 CameraService

Z:\android\frameworks\av\camera\cameraserver
    cameraserver.rc
    main_cameraserver.cpp