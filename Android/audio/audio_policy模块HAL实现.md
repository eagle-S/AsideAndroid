# audio_policy HAL

[TOC]

## 相关代码

hardware/libhardware/include/hardware/audio_policy.h
hardware/libhardware_legacy/audio/audio_policy_hal.cpp
frameworks/av/services/audioflinger/AudioPolicyService.cpp

## HAL结构体定义

```c
// hardware/libhardware_legacy/audio/audio_policy_hal.cpp
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
```

- legacy_ap_module包含audio_policy_module
- legacy_ap_device包含audio_policy_device

audio_policy_module与audio_policy_device均在audio_policy.h中定义

```c
// hardware/libhardware/include/hardware/audio_policy.h
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

## 初始化

```c
// hardware/libhardware_legacy/audio/audio_policy_hal.cpp
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

## 调用流程示例

frameworks/av/services/audioflinger/AudioPolicyService.cpp中构造函数AudioPolicyService()

 ```c
 // frameworks/av/services/audioflinger/AudioPolicyService.cpp
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

1. 通过hw_get_module函数加载so库，获取模块hw_module_t
2. 调用模块hw_module_t中hw_module_methods_t的open函数即module->methods->open，初始化hw_device_t及其他接口
3. 调用设备hw_device_t中create_audio_policy初始化audio_policy
