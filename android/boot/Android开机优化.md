# Android启动优化

[TOC]

- Android启动流程
- 启动时间测量
- 优化实施

## Android启动流程

- Bootloader
- Kernel
- Init
   - init.rc
- Zygote
   - createVm
   - preload
     - preloadClasses
     - preloadResources
     - preloadOpenGL
     - preloadSharedLibraries
- SystemServer
   - PackageManagerService
   - ActivityManagerService
- Launcher
- bootanimation

### Bootloader启动

### Kernel启动

### Init启动

### Zygote启动

### SystemServer

SystemServer启动后，会加载各种系统服务，PackageManagerService， ActivityManagerService

### Launcher启动


### bootanimation流程

## 启动时间测量

### bootchart

### log

## 优化实施

## 总结