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

```c
/*
 * Copyright (C) 2008 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

#ifndef ANDROID_INCLUDE_HARDWARE_HARDWARE_H
#define ANDROID_INCLUDE_HARDWARE_HARDWARE_H

#include <stdint.h>
#include <sys/cdefs.h>

#include <cutils/native_handle.h>
#include <system/graphics.h>

__BEGIN_DECLS

/*
 * Value for the hw_module_t.tag field
 */

#define MAKE_TAG_CONSTANT(A,B,C,D) (((A) << 24) | ((B) << 16) | ((C) << 8) | (D))

#define HARDWARE_MODULE_TAG MAKE_TAG_CONSTANT('H', 'W', 'M', 'T')
#define HARDWARE_DEVICE_TAG MAKE_TAG_CONSTANT('H', 'W', 'D', 'T')

#define HARDWARE_MAKE_API_VERSION(maj,min) \
            ((((maj) & 0xff) << 8) | ((min) & 0xff))

#define HARDWARE_MAKE_API_VERSION_2(maj,min,hdr) \
            ((((maj) & 0xff) << 24) | (((min) & 0xff) << 16) | ((hdr) & 0xffff))
#define HARDWARE_API_VERSION_2_MAJ_MIN_MASK 0xffff0000
#define HARDWARE_API_VERSION_2_HEADER_MASK  0x0000ffff


/*
 * The current HAL API version.
 *
 * All module implementations must set the hw_module_t.hal_api_version field
 * to this value when declaring the module with HAL_MODULE_INFO_SYM.
 *
 * Note that previous implementations have always set this field to 0.
 * Therefore, libhardware HAL API will always consider versions 0.0 and 1.0
 * to be 100% binary compatible.
 *
 */
#define HARDWARE_HAL_API_VERSION HARDWARE_MAKE_API_VERSION(1, 0)

/*
 * Helper macros for module implementors.
 *
 * The derived modules should provide convenience macros for supported
 * versions so that implementations can explicitly specify module/device
 * versions at definition time.
 *
 * Use this macro to set the hw_module_t.module_api_version field.
 */
#define HARDWARE_MODULE_API_VERSION(maj,min) HARDWARE_MAKE_API_VERSION(maj,min)
#define HARDWARE_MODULE_API_VERSION_2(maj,min,hdr) HARDWARE_MAKE_API_VERSION_2(maj,min,hdr)

/*
 * Use this macro to set the hw_device_t.version field
 */
#define HARDWARE_DEVICE_API_VERSION(maj,min) HARDWARE_MAKE_API_VERSION(maj,min)
#define HARDWARE_DEVICE_API_VERSION_2(maj,min,hdr) HARDWARE_MAKE_API_VERSION_2(maj,min,hdr)

struct hw_module_t;
struct hw_module_methods_t;
struct hw_device_t;

/**
 * Every hardware module must have a data structure named HAL_MODULE_INFO_SYM
 * and the fields of this data structure must begin with hw_module_t
 * followed by module specific information.
 */
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

typedef struct hw_module_methods_t {
    /** Open a specific device */
    int (*open)(const struct hw_module_t* module, const char* id,
            struct hw_device_t** device);

} hw_module_methods_t;

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

/**
 * Name of the hal_module_info
 */
#define HAL_MODULE_INFO_SYM         HMI

/**
 * Name of the hal_module_info as a string
 */
#define HAL_MODULE_INFO_SYM_AS_STR  "HMI"

/**
 * Get the module info associated with a module by id.
 *
 * @return: 0 == success, <0 == error and *module == NULL
 */
int hw_get_module(const char *id, const struct hw_module_t **module);

/**
 * Get the module info associated with a module instance by class 'class_id'
 * and instance 'inst'.
 *
 * Some modules types necessitate multiple instances. For example audio supports
 * multiple concurrent interfaces and thus 'audio' is the module class
 * and 'primary' or 'a2dp' are module interfaces. This implies that the files
 * providing these modules would be named audio.primary.<variant>.so and
 * audio.a2dp.<variant>.so
 *
 * @return: 0 == success, <0 == error and *module == NULL
 */
int hw_get_module_by_class(const char *class_id, const char *inst,
                           const struct hw_module_t **module);

__END_DECLS

#endif  /* ANDROID_INCLUDE_HARDWARE_HARDWARE_H */
```

#### HAL模块
表示模块的结构体 hw_module_t，其中包含模块的版本、名称和作者等元数据。Android 会根据这些元数据来找到并正确加载 HAL 模块。

hw_module_t 结构体还包含指向另一个结构体 hw_module_methods_t 的指针，这个结构体中包含一个指向相应模块的 open 函数的指针。此 open 函数用于与相关硬件建立通信。
每个特定硬件的 HAL 通常都会使用附加信息为该特定硬件扩展通用的 hw_module_t 结构体。但必须保证hw_module_t处于此结体的第一个位置，即处于处于结构体的首地址的位置，在相机 HAL 中，camera_module_t 结构体会包含一个 hw_module_t 结构体以及其他特定于相机的函数指针：
```c
typedef struct camera_module {
    hw_module_t common;
    int (*get_number_of_cameras)(void);
    int (*get_camera_info)(int camera_id, struct camera_info *info);
} camera_module_t;
```

实现 HAL 并创建模块结构体时，您必须将其命名为 HAL_MODULE_INFO_SYM。
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

设备由 hw_device_t 结构体表示。与模块类似，每类设备都定义了一个通用 hw_device_t 的详细版本，其中包含指向硬件特定功能的函数指针。例如，audio_hw_device_t 结构体类型会包含指向音频设备操作的函数指针：

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

hardware/libhardware/hardware.c


## 寻找so库对应的模块


Y:\kitkat_T3\android\frameworks\av\services\audioflinger\AudioPolicyService.cpp

 在AudioPolicyService初始化时

``` c
 AudioPolicyService::AudioPolicyService()
    : BnAudioPolicyService() , mpAudioPolicyDev(NULL) , mpAudioPolicy(NULL)
{
    char value[PROPERTY_VALUE_MAX];
    const struct hw_module_t *module;
    int forced_val;
    int rc;

    Mutex::Autolock _l(mLock);

    // start tone playback thread
    mTonePlaybackThread = new AudioCommandThread(String8("ApmTone"), this);
    // start audio commands thread
    mAudioCommandThread = new AudioCommandThread(String8("ApmAudio"), this);
    // start output activity command thread
    mOutputCommandThread = new AudioCommandThread(String8("ApmOutput"), this);
    /* instantiate the audio policy manager */
    rc = hw_get_module(AUDIO_POLICY_HARDWARE_MODULE_ID, &module);
    if (rc)
        return;

    rc = audio_policy_dev_open(module, &mpAudioPolicyDev);
    ALOGE_IF(rc, "couldn't open audio policy device (%s)", strerror(-rc));
    if (rc)
        return;

    rc = mpAudioPolicyDev->create_audio_policy(mpAudioPolicyDev, &aps_ops, this,
                                               &mpAudioPolicy);
    ALOGE_IF(rc, "couldn't create audio policy (%s)", strerror(-rc));
    if (rc)
        return;

    rc = mpAudioPolicy->init_check(mpAudioPolicy);
    ALOGE_IF(rc, "couldn't init_check the audio policy (%s)", strerror(-rc));
    if (rc)
        return;

    ALOGI("Loaded audio policy from %s (%s)", module->name, module->id);

    // load audio pre processing modules
    if (access(AUDIO_EFFECT_VENDOR_CONFIG_FILE, R_OK) == 0) {
        loadPreProcessorConfig(AUDIO_EFFECT_VENDOR_CONFIG_FILE);
    } else if (access(AUDIO_EFFECT_DEFAULT_CONFIG_FILE, R_OK) == 0) {
        loadPreProcessorConfig(AUDIO_EFFECT_DEFAULT_CONFIG_FILE);
    }
}
```

其中hw_get_module
``` c
Y:\kitkat_T3\android\hardware\libhardware\hardware.c

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

/** Base path of the hal modules */
#define HAL_LIBRARY_PATH1 "/system/lib/hw"
#define HAL_LIBRARY_PATH2 "/vendor/lib/hw"


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

会先后从/vendor/lib/hw、/system/lib/hw查找命名格式为<MODULE_ID>.variant.so库，
其中MODULE_ID为硬件模块id，
variant为从属性"ro.product.board", "ro.board.platform","ro.arch" 中获取，如果均获取失败则获取<MODULE_ID>.default.so

获取到so后会，在加载so时查找“HMI”这个导出符号，并获取其地址。
``` c
    const char *sym = HAL_MODULE_INFO_SYM_AS_STR;
    hmi = (struct hw_module_t *)dlsym(handle, sym);
```
/**
 * Every hardware module must have a data structure named HAL_MODULE_INFO_SYM
 * and the fields of this data structure must begin with hw_module_t
 * followed by module specific information.
 */

 /**
 * Name of the hal_module_info
 */
#define HAL_MODULE_INFO_SYM         HMI

/**
 * Name of the hal_module_info as a string
 */
#define HAL_MODULE_INFO_SYM_AS_STR  "HMI"


 android硬件抽象层规定，每个硬件模块必须有一个名为HAL_MODULE_INFO_SYM的结构体，HAL_MODULE_INFO_SYM其实是个宏在hardware.h 中定义#define HAL_MODULE_INFO_SYM   HMI，并且结构体第一个成员地址必须是hw_module_t的结构体


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




