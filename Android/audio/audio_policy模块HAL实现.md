### HAL结构体定义

hardware/libhardware/include/hardware/audio_policy.h
```c
/**
 * Every hardware module must have a data structure named HAL_MODULE_INFO_SYM
 * and the fields of this data structure must begin with hw_module_t
 * followed by module specific information.
 */
typedef struct audio_policy_module {
    struct hw_module_t common;
} audio_policy_module_t;

struct audio_policy_device {
    struct hw_device_t common;

    int (*create_audio_policy)(const struct audio_policy_device *device,
                               struct audio_policy_service_ops *aps_ops,
                               void *service,
                               struct audio_policy **ap);

    int (*destroy_audio_policy)(const struct audio_policy_device *device,
                                struct audio_policy *ap);
};

/** convenience API for opening and closing a supported device */

static inline int audio_policy_dev_open(const hw_module_t* module,
                                    struct audio_policy_device** device)
{
    return module->methods->open(module, AUDIO_POLICY_INTERFACE,
                                 (hw_device_t**)device);
}

static inline int audio_policy_dev_close(struct audio_policy_device* device)
{
    return device->common.close(&device->common);
}
```
定义结构体audio_policy_module_t 包含 hw_module_t
定义结构体audio_policy_device 包含 hw_device_t
定义打开、关闭函数audio_policy_dev_open、audio_policy_dev_close

### HAL结构体初始化
hardware/libhardware/modules/audio/audio_policy.c
```c
struct default_ap_module {
    struct audio_policy_module module;
};

struct default_ap_device {
    struct audio_policy_device device;
};

struct default_audio_policy {
    struct audio_policy policy;

    struct audio_policy_service_ops *aps_ops;
    void *service;
};

static struct hw_module_methods_t default_ap_module_methods = {
    .open = default_ap_dev_open,
};

struct default_ap_module HAL_MODULE_INFO_SYM = {
    .module = {
        .common = {
            .tag            = HARDWARE_MODULE_TAG,
            .version_major  = 1,
            .version_minor  = 0,
            .id             = AUDIO_POLICY_HARDWARE_MODULE_ID,
            .name           = "Default audio policy HAL",
            .author         = "The Android Open Source Project",
            .methods        = &default_ap_module_methods,
        },
    },
};
```
初始化结构体default_ap_module HAL_MODULE_INFO_SYM，并将default_ap_module_methods地址赋值给methods


hardware/libhardware_legacy/audio/audio_policy_hal.cpp
```c
struct legacy_ap_module {
    struct audio_policy_module module;
};

struct legacy_ap_device {
    struct audio_policy_device device;
};

struct legacy_audio_policy {
    struct audio_policy policy;

    void *service;
    struct audio_policy_service_ops *aps_ops;
    AudioPolicyCompatClient *service_client;
    AudioPolicyInterface *apm;
};

static struct hw_module_methods_t legacy_ap_module_methods = {
        open: legacy_ap_dev_open
};

struct legacy_ap_module HAL_MODULE_INFO_SYM = {
    module: {
        common: {
            tag: HARDWARE_MODULE_TAG,
            version_major: 1,
            version_minor: 0,
            id: AUDIO_POLICY_HARDWARE_MODULE_ID,
            name: "LEGACY Audio Policy HAL",
            author: "The Android Open Source Project",
            methods: &legacy_ap_module_methods,
            dso : NULL,
            reserved : {0},
        },
    },
};
```

### 调用流程示例
frameworks/av/services/audioflinger/AudioPolicyService.cpp中构造函数AudioPolicyService()
 ```c
    const struct hw_module_t *module;
    rc = hw_get_module(AUDIO_POLICY_HARDWARE_MODULE_ID, &module);
    if (rc)
        return;

    rc = audio_policy_dev_open(module, &mpAudioPolicyDev);
    ALOGE_IF(rc, "couldn't open audio policy device (%s)", strerror(-rc));
    if (rc)
        return;

    rc = mpAudioPolicyDev->create_audio_policy(mpAudioPolicyDev, &aps_ops, this,
                                               &mpAudioPolicy);
```
1. 获取模块hw_module_t
2. 调用open函数初始化hw_device_t及其他接口
3. 调用create_audio_policy初始化audio_policy

总结：
1. 调用hardware.h中 hw_get_module或hw_get_module_by_class函数初始化hw_module_t
2. 调用hw_module_t中open函数初始化audio_policy_device结构体，其中包含初始化hw_device_t及其他接口