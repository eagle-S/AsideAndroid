[TOC]

## 硬件抽象层 (HAL)
### HAL简介

HAL的全称是Hardware Abstraction Layer,即硬件抽象层。

HAL 定义一个标准接口以供硬件供应商实现，这可让 Android 忽略较低级别的驱动程序实现。借助 HAL可以顺利实现相关功能，而不会影响或更改更高级别的系统。HAL 实现会被封装成模块，并由 Android 系统适时地加载。

Android的HAL是为了保护一些硬件提供商的知识产权而提出的，是为了避开linux的GPL束缚。思路是把控制硬件的动作都放到了Android HAL中，而linux driver仅仅完成一些简单的数据交互作用，甚至把硬件寄存器空间直接映射到user space。而Android是基于Aparch的license，因此硬件厂商可以只提供二进制代码，所以说Android只是一个开放的平台，并不是一个开源的平台。也许也正是因为Android不遵从GPL，所以Greg Kroah-Hartman才在2.6.33内核将Andorid驱动从linux中删除。GPL和硬件厂商目前还是有着无法弥合的裂痕。Android想要把这个问题处理好也是不容易的。

总结下来，Android HAL存在的原因主要有：
1. 并不是所有的硬件设备都有标准的linux kernel的接口
2. KERNEL DRIVER涉及到GPL的版权。某些设备制造商并不原因公开硬件驱动，所以才去用HAL方式绕过GPL。
3. 针对某些硬件，Android有一些特殊的需求.

### HAL接口介绍

为了保证 HAL 具有可预测的结构，每个特定于硬件的 HAL 接口都要具有 hardware/libhardware/include/hardware/hardware.h 中定义的属性。这类接口可让 Android 系统以一致的方式加载 HAL 模块的正确版本。HAL 接口包含两个组件：模块和设备。

#### HAL模块
```c
// hardware/libhardware/include/hardware/hardware.h 
typedef struct hw_module_t {
    /** tag must be initialized to HARDWARE_MODULE_TAG */
    uint32_t tag;

    /**
     * The API version of the implemented module. The module owner is
     * responsible for updating the version when a module interface has
     * changed.
     *
     * The derived modules such as gralloc and audio own and manage this field.
     * The module user must interpret the version field to decide whether or
     * not to inter-operate with the supplied module implementation.
     * For example, SurfaceFlinger is responsible for making sure that
     * it knows how to manage different versions of the gralloc-module API,
     * and AudioFlinger must know how to do the same for audio-module API.
     *
     * The module API version should include a major and a minor component.
     * For example, version 1.0 could be represented as 0x0100. This format
     * implies that versions 0x0100-0x01ff are all API-compatible.
     *
     * In the future, libhardware will expose a hw_get_module_version()
     * (or equivalent) function that will take minimum/maximum supported
     * versions as arguments and would be able to reject modules with
     * versions outside of the supplied range.
     */
    uint16_t module_api_version;
#define version_major module_api_version
    /**
     * version_major/version_minor defines are supplied here for temporary
     * source code compatibility. They will be removed in the next version.
     * ALL clients must convert to the new version format.
     */

    /**
     * The API version of the HAL module interface. This is meant to
     * version the hw_module_t, hw_module_methods_t, and hw_device_t
     * structures and definitions.
     *
     * The HAL interface owns this field. Module users/implementations
     * must NOT rely on this value for version information.
     *
     * Presently, 0 is the only valid value.
     */
    uint16_t hal_api_version;
#define version_minor hal_api_version

    /** Identifier of module */
    const char *id;

    /** Name of this module */
    const char *name;

    /** Author/owner/implementor of the module */
    const char *author;

    /** Modules methods */
    struct hw_module_methods_t* methods;

    /** module's dso */
    void* dso;

    /** padding to 128 bytes, reserved for future use */
    uint32_t reserved[32-7];

} hw_module_t;
```
表示模块的结构体 hw_module_t，其中包含：
- tag 标签，值必须为HARDWARE_MODULE_TAG
- version_major 主版本号
- version_minor 次版本号
- id   模块id
- name  名称
- author  作者
- methods  结构体 hw_module_methods_t 的指针

hw_module_t 结构体还包含指向另一个结构体 hw_module_methods_t 的指针，这个结构体中包含一个指向相应模块的 open 函数的指针。此 open 函数用于与相关硬件建立通信。
```c
typedef struct hw_module_methods_t {
    /** Open a specific device */
    int (*open)(const struct hw_module_t* module, const char* id,
            struct hw_device_t** device);

} hw_module_methods_t;
```

实际使用中每个特定硬件的 HAL 通常都会使用附加信息为该特定硬件扩展通用的 hw_module_t 结构体。但必须保证hw_module_t处于此结体的第一个位置，即处于处于结构体的首地址的位置，在相机 HAL 中，camera_module_t 结构体会包含一个 hw_module_t 结构体以及其他特定于相机的函数指针：
```c
typedef struct camera_module {
    hw_module_t common;
    int (*get_number_of_cameras)(void);
    int (*get_camera_info)(int camera_id, struct camera_info *info);
} camera_module_t;
```

实现 HAL 并创建模块结构体时，必须将其命名为 HAL_MODULE_INFO_SYM， HAL_MODULE_INFO_SYM在hardware/libhardware/include/hardware/hardware.h中定义，此为寻找hw_module_t模块首地址标志。
```c
/**
 * Name of the hal_module_info
 */
#define HAL_MODULE_INFO_SYM         HMI

/**
 * Name of the hal_module_info as a string
 */
#define HAL_MODULE_INFO_SYM_AS_STR  "HMI"
```

HAL模块定义举例：
```c
struct audio_module HAL_MODULE_INFO_SYM = {
    .common = {
        .tag = HARDWARE_MODULE_TAG,
        .module_api_version = AUDIO_MODULE_API_VERSION_0_1,
        .hal_api_version = HARDWARE_HAL_API_VERSION,
        .id = AUDIO_HARDWARE_MODULE_ID,
        .name = "NVIDIA Tegra Audio HAL",
        .author = "The Android Open Source Project",
        .methods = &hal_module_methods,
    },
};
```

#### HAL设备

设备是产品硬件的抽象表示。例如，一个音频模块可能包含主音频设备、USB 音频设备或蓝牙 A2DP 音频设备。

设备由 hw_device_t 结构体表示。
```c
// hardware/libhardware/include/hardware/hardware.h 

/**
 * Every device data structure must begin with hw_device_t
 * followed by module specific public methods and attributes.
 */
typedef struct hw_device_t {
    /** tag must be initialized to HARDWARE_DEVICE_TAG */
    uint32_t tag;

    /**
     * Version of the module-specific device API. This value is used by
     * the derived-module user to manage different device implementations.
     *
     * The module user is responsible for checking the module_api_version
     * and device version fields to ensure that the user is capable of
     * communicating with the specific module implementation.
     *
     * One module can support multiple devices with different versions. This
     * can be useful when a device interface changes in an incompatible way
     * but it is still necessary to support older implementations at the same
     * time. One such example is the Camera 2.0 API.
     *
     * This field is interpreted by the module user and is ignored by the
     * HAL interface itself.
     */
    uint32_t version;

    /** reference to the module this device belongs to */
    struct hw_module_t* module;

    /** padding reserved for future use */
    uint32_t reserved[12];

    /** Close this device */
    int (*close)(struct hw_device_t* device);

} hw_device_t;
```

与模块类似，实际使用中每类设备都定义了一个通用 hw_device_t 的详细版本，其中包含指向硬件特定功能的函数指针。例如，audio_hw_device_t 结构体类型会包含指向音频设备操作的函数指针：

```c
struct audio_hw_device {
    struct hw_device_t common;

    /**
     * used by audio flinger to enumerate what devices are supported by
     * each audio_hw_device implementation.
     *
     * Return value is a bitmask of 1 or more values of audio_devices_t
     */
    uint32_t (*get_supported_devices)(const struct audio_hw_device *dev);
  ...
};
typedef struct audio_hw_device audio_hw_device_t;
```

### HAL加载

HAL加载具体实现在hardware/libhardware/hardware.c文件中

#### 加载HAL模块，获取hw_module_t

加载HAL模块使用hardware中hw_get_module或hw_get_module_by_class函数：
``` c
// hardware/libhardware/hardware.c

int hw_get_module(const char *id, const struct hw_module_t **module)
{
    return hw_get_module_by_class(id, NULL, module);
}

int hw_get_module_by_class(const char *class_id, const char *inst,
                           const struct hw_module_t **module)
{
    int status;
    int i;
    const struct hw_module_t *hmi = NULL;
    char prop[PATH_MAX];
    char path[PATH_MAX];
    char name[PATH_MAX];

    if (inst)
        snprintf(name, PATH_MAX, "%s.%s", class_id, inst);
    else
        strlcpy(name, class_id, PATH_MAX);

    /*
     * Here we rely on the fact that calling dlopen multiple times on
     * the same .so will simply increment a refcount (and not load
     * a new copy of the library).
     * We also assume that dlopen() is thread-safe.
     */

    /* Loop through the configuration variants looking for a module */
    for (i=0 ; i<HAL_VARIANT_KEYS_COUNT+1 ; i++) {
        if (i < HAL_VARIANT_KEYS_COUNT) {
            if (property_get(variant_keys[i], prop, NULL) == 0) {
                continue;
            }
            snprintf(path, sizeof(path), "%s/%s.%s.so",
                     HAL_LIBRARY_PATH2, name, prop);
            if (access(path, R_OK) == 0) break;

            snprintf(path, sizeof(path), "%s/%s.%s.so",
                     HAL_LIBRARY_PATH1, name, prop);
            if (access(path, R_OK) == 0) break;
        } else {
            snprintf(path, sizeof(path), "%s/%s.default.so",
                     HAL_LIBRARY_PATH2, name);
            if (access(path, R_OK) == 0) break;

            snprintf(path, sizeof(path), "%s/%s.default.so",
                     HAL_LIBRARY_PATH1, name);
            if (access(path, R_OK) == 0) break;
        }
    }

    status = -ENOENT;
    if (i < HAL_VARIANT_KEYS_COUNT+1) {
        /* load the module, if this fails, we're doomed, and we should not try
         * to load a different variant. */
        status = load(class_id, path, module);
    }

    return status;
}
```
1. 获取模块名称name，name由传入的参数char *class_id, char *inst组成，如果inst为null则name=class_id，否则name=class_id.inst
2. 在特定目录下，查找变体so共享库，获取其路径path

```c
/** Base path of the hal modules */
#define HAL_LIBRARY_PATH1 "/system/lib/hw"
#define HAL_LIBRARY_PATH2 "/vendor/lib/hw"
```
so库查找路径有两个，且/vendor/lib/hw 优先级大于/system/lib/hw
```c
/**
 * There are a set of variant filename for modules. The form of the filename
 * is "<MODULE_ID>.variant.so" so for the led module the Dream variants 
 * of base "ro.product.board", "ro.board.platform" and "ro.arch" would be:
 *
 * led.trout.so
 * led.msm7k.so
 * led.ARMV6.so
 * led.default.so
 */

static const char *variant_keys[] = {
    "ro.hardware",  /* This goes first so that it can pick up a different
                       file on the emulator. */
    "ro.product.board",
    "ro.board.platform",
    "ro.arch"
};
```
每个模块可能有一系列变体名，变体名格式为<MODULE_ID>.variant.so，例如MODULE_ID=led时，变体名可能是led.trout.so、led.msm7k.so。变体值根据系统属性"ro.product.board", "ro.board.platform" and "ro.arch" 依次获取。如果这些属性获取不到值，则查找<MODULE_ID>.default.so

查找路径及变体so库名称最终组装成path，传入到load函数中
```c
/**
 * Load the file defined by the variant and if successful
 * return the dlopen handle and the hmi.
 * @return 0 = success, !0 = failure.
 */
static int load(const char *id,
        const char *path,
        const struct hw_module_t **pHmi)
{
    int status;
    void *handle;
    struct hw_module_t *hmi;

    /*
     * load the symbols resolving undefined symbols before
     * dlopen returns. Since RTLD_GLOBAL is not or'd in with
     * RTLD_NOW the external symbols will not be global
     */
    handle = dlopen(path, RTLD_NOW);
    if (handle == NULL) {
        char const *err_str = dlerror();
        ALOGE("load: module=%s\n%s", path, err_str?err_str:"unknown");
        status = -EINVAL;
        goto done;
    }

    /* Get the address of the struct hal_module_info. */
    const char *sym = HAL_MODULE_INFO_SYM_AS_STR;
    hmi = (struct hw_module_t *)dlsym(handle, sym);
    if (hmi == NULL) {
        ALOGE("load: couldn't find symbol %s", sym);
        status = -EINVAL;
        goto done;
    }

    /* Check that the id matches */
    if (strcmp(id, hmi->id) != 0) {
        ALOGE("load: id=%s != hmi->id=%s", id, hmi->id);
        status = -EINVAL;
        goto done;
    }

    hmi->dso = handle;

    /* success */
    status = 0;

    done:
    if (status != 0) {
        hmi = NULL;
        if (handle != NULL) {
            dlclose(handle);
            handle = NULL;
        }
    } else {
        ALOGV("loaded HAL id=%s path=%s hmi=%p handle=%p",
                id, path, *pHmi, handle);
    }

    *pHmi = hmi;

    return status;
}
```
1. 调用handle = dlopen(path, RTLD_NOW)根据path路径打开so文件
2. 调用hmi = (struct hw_module_t *)dlsym(handle, sym)，根据HAL_MODULE_INFO_SYM_AS_STR字符串"HMI"获取hw_module_t地址


#### 获取hw_device_t
在获取hw_module_t地址后，hw_module_t中函数指针地址也被初始化。在调用open函数时对hw_device_t进行赋值

### 寻找hw_module_t首地址
```
linux@ubuntu:~/eclair_2.1_farsight/out/target/product/fs100/system/lib/hw$ file led.default.so   
led.default.so: ELF 32-bit LSB shared object, ARM, version 1 (SYSV), dynamically linked, stripped 
```
使用file命令查看so文件，会发现so是以ELF头开始的文件。

ELF = Executable and Linkable Format，可执行连接格式，是UNIX系统实验室（USL）作为应用程序二进制接口（Application Binary Interface，ABI）而开发和发布的，扩展名为elf。它保存了路线图(road map)，描述了该文件的组织情况。sections保存着object 文件的信息，从连接角度看：包括指令，数据,符号表，重定位信息等等。

可以使用readelf命令进一步查看相应的符号信息
```
linux@ubuntu:~/eclair_2.1_farsight/out/target/product/fs100/system/lib/hw$ readelf -s led.default.so 

Symbol table '.dynsym' contains 25 entries:
   Num:    Value  Size Type    Bind   Vis      Ndx Name
     0: 00000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 000004c8     0 SECTION LOCAL  DEFAULT    7 
     2: 00001000     0 SECTION LOCAL  DEFAULT   11 
     3: 00000000     0 FUNC    GLOBAL DEFAULT  UND ioctl
     4: 000006d4     0 NOTYPE  GLOBAL DEFAULT  ABS __exidx_end
     5: 00000000     0 FUNC    GLOBAL DEFAULT  UND __aeabi_unwind_cpp_pr0
     6: 00001178     0 NOTYPE  GLOBAL DEFAULT  ABS _bss_end__
     7: 00000000     0 FUNC    GLOBAL DEFAULT  UND malloc
     8: 00001174     0 NOTYPE  GLOBAL DEFAULT  ABS __bss_start__
     9: 00000000     0 FUNC    GLOBAL DEFAULT  UND __android_log_print
    10: 000006ab     0 NOTYPE  GLOBAL DEFAULT  ABS __exidx_start
    11: 00001174     4 OBJECT  GLOBAL DEFAULT   15 fd
    12: 000005d5    60 FUNC    GLOBAL DEFAULT    7 led_set_off
    13: 00001178     0 NOTYPE  GLOBAL DEFAULT  ABS __bss_end__
    14: 00001174     0 NOTYPE  GLOBAL DEFAULT  ABS __bss_start
    15: 00000000     0 FUNC    GLOBAL DEFAULT  UND memset
    16: 00001178     0 NOTYPE  GLOBAL DEFAULT  ABS __end__
    17: 00001174     0 NOTYPE  GLOBAL DEFAULT  ABS _edata
    18: 00001178     0 NOTYPE  GLOBAL DEFAULT  ABS _end
    19: 00000000     0 FUNC    GLOBAL DEFAULT  UND open
    20: 00080000     0 NOTYPE  GLOBAL DEFAULT  ABS _stack
    21: 00001000   128 OBJECT  GLOBAL DEFAULT   11 HMI
    22: 00001170     0 NOTYPE  GLOBAL DEFAULT   14 __data_start
    23: 00000000     0 FUNC    GLOBAL DEFAULT  UND close
    24: 00000000     0 FUNC    GLOBAL DEFAULT  UND free
```
在21行我们发现，名字就是“HMI”，对应于hw_module_t结构体。

所以在定义硬件模块时必须使用HAL_MODULE_INFO_SYM为变量名，这样编译器才会将这个结构体的导出符号变为“HMI”，这样这个结构体才能被dlsym函数找到！



参考：

android官方介绍
https://source.android.com/devices/architecture/hal?hl=zh-cn

hw_get_module解析
http://blog.csdn.net/mdx20072419/article/details/10354651

如何实现HAL接口
http://blog.csdn.net/flydream0/article/details/7086273

HAL如何向上层提供接口总结-hw_device_t
http://blog.csdn.net/myarrow/article/details/7175204

Android HAL的被调用流程
http://blog.csdn.net/myarrow/article/details/7175714

