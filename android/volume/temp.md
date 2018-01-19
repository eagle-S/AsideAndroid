## vold进程

1. 在init.rc中配置启动
2. vold起到一个承上启下的作用，帮助kernel和framework间通信
3. vold中包含两个socket，一个和kernel通信，一个和frameworks中MountService通信



