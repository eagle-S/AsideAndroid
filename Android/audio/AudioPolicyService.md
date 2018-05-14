[TOC]

## 简介

AudioPolicyService处于本地媒体服务层，提供音频策略服务，比如什么时候打开音频接口设备、某种Stream类型的音频对应什么设备等等。

## AudioPolicyService启动

AudioPolicyService服务运行在mediaserver进程中，随着mediaserver进程启动而启动。

mediaserver启动

```c
// frameworks/av/media/mediaserver/main_mediaserver.cpp

int main(int argc, char** argv)
{
    ......

        sp<ProcessState> proc(ProcessState::self());
        sp<IServiceManager> sm = defaultServiceManager();
        ALOGI("ServiceManager: %p", sm.get());
        AudioFlinger::instantiate();
        MediaPlayerService::instantiate();
        CameraService::instantiate();
        AudioPolicyService::instantiate();
        registerExtensions();
        ProcessState::self()->startThreadPool();
        IPCThreadState::self()->joinThreadPool();

}
```

AudioPolicyService继承了模板类BinderService，该类用于注册native service。

```c
// frameworks/native/include/binder/BinderService.h

template<typename SERVICE>
class BinderService
{
public:
    static status_t publish(bool allowIsolated = false) {
        sp<IServiceManager> sm(defaultServiceManager());
        return sm->addService(
                String16(SERVICE::getServiceName()),
                new SERVICE(), allowIsolated);
    }

    static void instantiate() { publish(); }
}
```

instantiate函数的实现可以看出，向ServiceManager注册的服务，服务名根据getServiceName()获取，服务使用无参数构造函数进行初始化。

## AudioPolicyService初始化

AudioPolicyService初始化实现，如下：

```c
static const char *getServiceName() ANDROID_API { return "media.audio_policy"; }

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

从构造函数可以看出初始化包括：
1. 创建AudioCommandThread（ApmTone、ApmAudio、ApmOutput）
2. 通过hw_get_module获取legacy_ap_module
3. 通过audio_policy_dev_open加载legacy_ap_device
4. 通过create_audio_policy创建legacy_audio_policy
5. 读取audio_effects.conf

### 创建AudioCommandThread线程

### 获取legacy_ap_module

详见

### 加载legacy_ap_device

详见

### 创建legacy_audio_policy

通过调用上一步得到的legacy_ap_device结构体的函数指针创建legacy_audio_policy

```c
mpAudioPolicyDev->create_audio_policy(mpAudioPolicyDev, &aps_ops, this,
                                               &mpAudioPolicy);
```

以上传入的aps_ops

```c
// hardware/libhardware_legacy/audio/audio_policy_hal.cpp

static int create_legacy_ap(const struct audio_policy_device *device,
                            struct audio_policy_service_ops *aps_ops,
                            void *service,
                            struct audio_policy **ap)
{
    struct legacy_audio_policy *lap;
    int ret;

    if (!service || !aps_ops)
        return -EINVAL;

    lap = (struct legacy_audio_policy *)calloc(1, sizeof(*lap));
    if (!lap)
        return -ENOMEM;

    lap->policy.set_device_connection_state = ap_set_device_connection_state;
    lap->policy.get_device_connection_state = ap_get_device_connection_state;
    lap->policy.set_phone_state = ap_set_phone_state;
    lap->policy.set_ringer_mode = ap_set_ringer_mode;
    lap->policy.set_force_use = ap_set_force_use;
    lap->policy.get_force_use = ap_get_force_use;
    lap->policy.set_can_mute_enforced_audible =
        ap_set_can_mute_enforced_audible;
    lap->policy.init_check = ap_init_check;
    lap->policy.get_output = ap_get_output;
    lap->policy.start_output = ap_start_output;
    lap->policy.stop_output = ap_stop_output;
    lap->policy.release_output = ap_release_output;
    lap->policy.get_input = ap_get_input;
    lap->policy.start_input = ap_start_input;
    lap->policy.stop_input = ap_stop_input;
    lap->policy.release_input = ap_release_input;
    lap->policy.init_stream_volume = ap_init_stream_volume;
    lap->policy.set_stream_volume_index = ap_set_stream_volume_index;
    lap->policy.get_stream_volume_index = ap_get_stream_volume_index;
    lap->policy.set_stream_volume_index_for_device = ap_set_stream_volume_index_for_device;
    lap->policy.get_stream_volume_index_for_device = ap_get_stream_volume_index_for_device;
    lap->policy.get_strategy_for_stream = ap_get_strategy_for_stream;
    lap->policy.get_devices_for_stream = ap_get_devices_for_stream;
    lap->policy.get_output_for_effect = ap_get_output_for_effect;
    lap->policy.register_effect = ap_register_effect;
    lap->policy.unregister_effect = ap_unregister_effect;
    lap->policy.set_effect_enabled = ap_set_effect_enabled;
    lap->policy.is_stream_active = ap_is_stream_active;
    lap->policy.is_stream_active_remotely = ap_is_stream_active_remotely;
    lap->policy.is_source_active = ap_is_source_active;
    lap->policy.dump = ap_dump;
    lap->policy.is_offload_supported = ap_is_offload_supported;

    lap->service = service;
    lap->aps_ops = aps_ops;
    lap->service_client =
        new AudioPolicyCompatClient(aps_ops, service);
    if (!lap->service_client) {
        ret = -ENOMEM;
        goto err_new_compat_client;
    }

    lap->apm = createAudioPolicyManager(lap->service_client);
    if (!lap->apm) {
        ret = -ENOMEM;
        goto err_create_apm;
    }

    *ap = &lap->policy;
    return 0;

err_create_apm:
    delete lap->service_client;
err_new_compat_client:
    free(lap);
    *ap = NULL;
    return ret;
}

```

创建legacy_audio_policy对象给函数指针及各成员赋值。这里赋值的同时会初始化AudioPolicyCompatClient及AudioPolicyManager

#### 创建AudioPolicyCompatClient

```c
// hardware/libhardware_legacy/audio/AudioPolicyCompatClient.h
    AudioPolicyCompatClient(struct audio_policy_service_ops *serviceOps,
                            void *service) :
            mServiceOps(serviceOps) , mService(service) {}
```

查看hardware/libhardware_legacy/audio/AudioPolicyCompatClient.cpp的实现可以发现，AudioPolicyCompatClient是对audio_policy_service_ops的封装类，对外提供audio_policy_service_ops数据结构中定义的接口。
而初始化传入的audio_policy_service_ops在AudioPolicyService.cpp中初始化

```c
    struct audio_policy_service_ops aps_ops = {
        open_output           : aps_open_output,
        open_duplicate_output : aps_open_dup_output,
        close_output          : aps_close_output,
        suspend_output        : aps_suspend_output,
        restore_output        : aps_restore_output,
        open_input            : aps_open_input,
        close_input           : aps_close_input,
        set_stream_volume     : aps_set_stream_volume,
        set_stream_output     : aps_set_stream_output,
        set_parameters        : aps_set_parameters,
        get_parameters        : aps_get_parameters,
        start_tone            : aps_start_tone,
        stop_tone             : aps_stop_tone,
        set_voice_volume      : aps_set_voice_volume,
        move_effects          : aps_move_effects,
        load_hw_module        : aps_load_hw_module,
        open_output_on_module : aps_open_output_on_module,
        open_input_on_module  : aps_open_input_on_module,
    };
```

#### 创建AudioPolicyManager

```c
// hardware/libhardware_legacy/audio/AudioPolicyManagerDefault.h
class AudioPolicyManagerDefault: public AudioPolicyManagerBase
{

public:
                AudioPolicyManagerDefault(AudioPolicyClientInterface *clientInterface)
                : AudioPolicyManagerBase(clientInterface) {}

        virtual ~AudioPolicyManagerDefault() {}

};
};

// hardware/libhardware_legacy/audio/AudioPolicyManagerDefault.cpp
extern "C" AudioPolicyInterface* createAudioPolicyManager(AudioPolicyClientInterface *clientInterface)
{
    return new AudioPolicyManagerDefault(clientInterface);
}
```

AudioPolicyManagerDefault继承于AudioPolicyManagerBase，看下AudioPolicyManagerBase初始化

```c
AudioPolicyManagerBase::AudioPolicyManagerBase(AudioPolicyClientInterface *clientInterface)
    :
#ifdef AUDIO_POLICY_TEST
    Thread(false),
#endif //AUDIO_POLICY_TEST
    mPrimaryOutput((audio_io_handle_t)0),
    mAvailableOutputDevices(AUDIO_DEVICE_NONE),
    mPhoneState(AudioSystem::MODE_NORMAL),
    mLimitRingtoneVolume(false), mLastVoiceVolume(-1.0f),mLastFMVolume(-1.0f),
    mTotalEffectsCpuLoad(0), mTotalEffectsMemory(0),
    mA2dpSuspended(false), mHasA2dp(false), mHasUsb(false), mHasRemoteSubmix(false),
    mSpeakerDrcEnabled(false)
{
    mpClientInterface = clientInterface;

    for (int i = 0; i < AudioSystem::NUM_FORCE_USE; i++) {
        mForceUse[i] = AudioSystem::FORCE_NONE;
    }

    mA2dpDeviceAddress = String8("");
    mScoDeviceAddress = String8("");
    mUsbCardAndDevice = String8("");

    if (loadAudioPolicyConfig(AUDIO_POLICY_VENDOR_CONFIG_FILE) != NO_ERROR) {
        if (loadAudioPolicyConfig(AUDIO_POLICY_CONFIG_FILE) != NO_ERROR) {
            ALOGE("could not load audio policy configuration file, setting defaults");
            defaultAudioPolicyConfig();
        }
    }
    // must be done after reading the policy
    initializeVolumeCurves();

    // open all output streams needed to access attached devices
    for (size_t i = 0; i < mHwModules.size(); i++) {
        mHwModules[i]->mHandle = mpClientInterface->loadHwModule(mHwModules[i]->mName);
        if (mHwModules[i]->mHandle == 0) {
            ALOGW("could not open HW module %s", mHwModules[i]->mName);
            continue;
        }
        // open all output streams needed to access attached devices
        // except for direct output streams that are only opened when they are actually
        // required by an app.
        for (size_t j = 0; j < mHwModules[i]->mOutputProfiles.size(); j++)
        {
            const IOProfile *outProfile = mHwModules[i]->mOutputProfiles[j];

            if ((outProfile->mSupportedDevices & mAttachedOutputDevices) &&
                    ((outProfile->mFlags & AUDIO_OUTPUT_FLAG_DIRECT) == 0)) {
                AudioOutputDescriptor *outputDesc = new AudioOutputDescriptor(outProfile);
            if(!(strcmp(mHwModules[i]->mName, AUDIO_HARDWARE_MODULE_ID_EXTERNAL))
                 && (outProfile->mSupportedDevices & AUDIO_DEVICE_OUT_EXTERNAL))
             {
                 ALOGV("external devices supported, force open!");
                 outputDesc->mDevice = AUDIO_DEVICE_OUT_EXTERNAL;
             }
            else
             {
                outputDesc->mDevice = (audio_devices_t)(mDefaultOutputDevice &
                                                            outProfile->mSupportedDevices);
            }

                audio_io_handle_t output = mpClientInterface->openOutput(
                                                outProfile->mModule->mHandle,
                                                &outputDesc->mDevice,
                                                &outputDesc->mSamplingRate,
                                                &outputDesc->mFormat,
                                                &outputDesc->mChannelMask,
                                                &outputDesc->mLatency,
                                                outputDesc->mFlags);
                if (output == 0) {
                    delete outputDesc;
                } else {
                    mAvailableOutputDevices = (audio_devices_t)(mAvailableOutputDevices |
                                            (outProfile->mSupportedDevices & mAttachedOutputDevices));
                    if (mPrimaryOutput == 0 &&
                            outProfile->mFlags & AUDIO_OUTPUT_FLAG_PRIMARY) {
                        mPrimaryOutput = output;
                    }
                    addOutput(output, outputDesc);
                    setOutputDevice(output,
                                    (audio_devices_t)(mDefaultOutputDevice &
                                                        outProfile->mSupportedDevices),
                                    true);
                }
            }
        }
    }

    ALOGE_IF((mAttachedOutputDevices & ~mAvailableOutputDevices),
             "Not output found for attached devices %08x",
             (mAttachedOutputDevices & ~mAvailableOutputDevices));

    ALOGE_IF((mPrimaryOutput == 0), "Failed to open primary output");

    updateDevicesAndOutputs();

#ifdef AUDIO_POLICY_TEST
    if (mPrimaryOutput != 0) {
        AudioParameter outputCmd = AudioParameter();
        outputCmd.addInt(String8("set_id"), 0);
        mpClientInterface->setParameters(mPrimaryOutput, outputCmd.toString());

        mTestDevice = AUDIO_DEVICE_OUT_SPEAKER;
        mTestSamplingRate = 44100;
        mTestFormat = AudioSystem::PCM_16_BIT;
        mTestChannels =  AudioSystem::CHANNEL_OUT_STEREO;
        mTestLatencyMs = 0;
        mCurOutput = 0;
        mDirectOutput = false;
        for (int i = 0; i < NUM_TEST_OUTPUTS; i++) {
            mTestOutputs[i] = 0;
        }

        const size_t SIZE = 256;
        char buffer[SIZE];
        snprintf(buffer, SIZE, "AudioPolicyManagerTest");
        run(buffer, ANDROID_PRIORITY_AUDIO);
    }
#endif //AUDIO_POLICY_TEST
}

```

AudioPolicyManagerBase对象构造过程中主要完成以下几个步骤：
1. 加载audio_policy.conf配置文件:loadAudioPolicyConfig(AUDIO_POLICY_CONFIG_FILE)
2. 初始化各种音频流对应的音量调节点:initializeVolumeCurves()
3. 加载各audio硬件模块：mpClientInterface->loadHwModule(mHwModules[i]->mName)
4. 打开attached_output_devices输出：mpClientInterface->openOutput()；
5. 保存输出设备描述符对象：addOutput(output, outputDesc);

##### 加载audio_policy.conf配置文件

```c
#define AUDIO_POLICY_CONFIG_FILE "/system/etc/audio_policy.conf"
#define AUDIO_POLICY_VENDOR_CONFIG_FILE "/vendor/etc/audio_policy.conf"

if (loadAudioPolicyConfig(AUDIO_POLICY_VENDOR_CONFIG_FILE) != NO_ERROR) {
        if (loadAudioPolicyConfig(AUDIO_POLICY_CONFIG_FILE) != NO_ERROR) {
            ALOGE("could not load audio policy configuration file, setting defaults");
            defaultAudioPolicyConfig();
        }
    }
```

audio_policy.conf优先加载/vendor/etc/audio_policy.conf，如果加载失败再加载/system/etc/audio_policy.conf，如果均加载失败则会加载一个默认的HwModule，并命名为primary module，从这可以看出，音频系统中一定必须存在的module就是primary了。

##### audio_policy.conf介绍

/system/etc/audio_policy.conf文件开头有对此文件介绍

```
# Global configuration section: lists input and output devices always present on the device
# as well as the output device selected by default.
# Devices are designated by a string that corresponds to the enum in audio.h

全局配置部分：列出了设备上总是存在的输入输出设备，同时指定了系统默认选择的输出设备。这里的设备必须是audio.h中列出的枚举类型

global_configuration {
  attached_output_devices AUDIO_DEVICE_OUT_EARPIECE|AUDIO_DEVICE_OUT_SPEAKER
  default_output_device AUDIO_DEVICE_OUT_SPEAKER
  attached_input_devices AUDIO_DEVICE_IN_BUILTIN_MIC|AUDIO_DEVICE_IN_BACK_MIC|AUDIO_DEVICE_IN_REMOTE_SUBMIX|AUDIO_DEVICE_IN_FM_RX|AUDIO_DEVICE_IN_FM_RX_A2DP|AUDIO_DEVICE_IN_VOICE_CALL
}

# audio hardware module section: contains descriptors for all audio hw modules present on the
# device. Each hw module node is named after the corresponding hw module library base name.
# For instance, "primary" corresponds to audio.primary.<device>.so.
# The "primary" module is mandatory and must include at least one output with
# AUDIO_OUTPUT_FLAG_PRIMARY flag.
# Each module descriptor contains one or more output profile descriptors and zero or more
# input profile descriptors. Each profile lists all the parameters supported by a given output
# or input stream category.
# The "channel_masks", "formats", "devices" and "flags" are specified using strings corresponding
# to enums in audio.h and audio_policy.h. They are concatenated by use of "|" without space or "\n".

音频硬件模块部分：包含了机器中所有存在的音频硬件模块的描述，每一个硬件模块节点的名称必须与音频模块共享库so的名称相对应，例如："primary"与audio.primary.<device>.so库相对应。其中"primary"模块是强制需要的，并且它必须包含一个以AUDIO_OUTPUT_FLAG_PRIMARY作为flag的output。每个模块描述中包含一个或多个output输入配置描述，零个或多个input输入配置描述。每一个配置列举了输入输出流类型支持的参数，其中的 "channel_masks"、"formats"、"devices"和"flags"必须为audio.h、audio_policy.h中定义的枚举值，它们可以通过|连接，但不能包含空格和"\n"。
```

在代码中每种音频硬件模块被抽象成HwModule

```c
// hardware/libhardware_legacy/include/hardware_legacy/AudioPolicyManagerBase.h
class HwModule {
        public:
                    HwModule(const char *name);
                    ~HwModule();

            void dump(int fd);

            const char *const mName; // base name of the audio HW module (primary, a2dp ...)
            audio_module_handle_t mHandle;
            Vector <IOProfile *> mOutputProfiles; // output profiles exposed by this module
            Vector <IOProfile *> mInputProfiles;  // input profiles exposed by this module
        };

        // the IOProfile class describes the capabilities of an output or input stream.
        // It is currently assumed that all combination of listed parameters are supported.
        // It is used by the policy manager to determine if an output or input is suitable for
        // a given use case,  open/close it accordingly and connect/disconnect audio tracks
        // to/from it.
        class IOProfile
        {
        public:
            IOProfile(HwModule *module);
            ~IOProfile();

            bool isCompatibleProfile(audio_devices_t device,
                                     uint32_t samplingRate,
                                     uint32_t format,
                                     uint32_t channelMask,
                                     audio_output_flags_t flags) const;

            void dump(int fd);

            // by convention, "0' in the first entry in mSamplingRates, mChannelMasks or mFormats
            // indicates the supported parameters should be read from the output stream
            // after it is opened for the first time
            Vector <uint32_t> mSamplingRates; // supported sampling rates
            Vector <audio_channel_mask_t> mChannelMasks; // supported channel masks
            Vector <audio_format_t> mFormats; // supported audio formats
            audio_devices_t mSupportedDevices; // supported devices (devices this output can be
                                               // routed to)
            audio_output_flags_t mFlags; // attribute flags (e.g primary output,
                                                // direct output...). For outputs only.
            HwModule *mModule;                     // audio HW module exposing this I/O stream
        };
```

HwModule中包含了模块名称，输入、输入描述及int类型的mHandle

audio.h中定义的音频模块名称

```c
// hardware/libhardware/include/hardware/audio.h

#define AUDIO_HARDWARE_MODULE_ID_PRIMARY "primary"
#define AUDIO_HARDWARE_MODULE_ID_A2DP "a2dp"
#define AUDIO_HARDWARE_MODULE_ID_USB "usb"
#define AUDIO_HARDWARE_MODULE_ID_REMOTE_SUBMIX "r_submix"
#define AUDIO_HARDWARE_MODULE_ID_CODEC_OFFLOAD "codec_offload"
```

加载流程如下:

```c
status_t AudioPolicyManagerBase::loadAudioPolicyConfig(const char *path)
{
    cnode *root;
    char *data;

    data = (char *)load_file(path, NULL);
    if (data == NULL) {
        return -ENODEV;
    }
    root = config_node("", "");
    config_load(root, data);

    loadGlobalConfig(root);
    loadHwModules(root);

    config_free(root);
    free(root);
    free(data);

    ALOGI("loadAudioPolicyConfig() loaded %s\n", path);

    return NO_ERROR;
}

```

##### 初始化各种音频流对应的音量调节点

待续

##### 加载各audio硬件模块

```c
mHwModules[i]->mHandle = mpClientInterface->loadHwModule(mHwModules[i]->mName);
```

由之前对AudioPolicyCompatClient分析可知，此处loadHwModule实际是调用了AudioFlinger的loadHwModule

```c
audio_module_handle_t AudioFlinger::loadHwModule(const char *name)
{
    if (!settingsAllowed()) {
        return 0;
    }
    Mutex::Autolock _l(mLock);
    return loadHwModule_l(name);
}

// loadHwModule_l() must be called with AudioFlinger::mLock held
audio_module_handle_t AudioFlinger::loadHwModule_l(const char *name)
{
    for (size_t i = 0; i < mAudioHwDevs.size(); i++) {
        if (strncmp(mAudioHwDevs.valueAt(i)->moduleName(), name, strlen(name)) == 0) {
            ALOGW("loadHwModule() module %s already loaded", name);
            return mAudioHwDevs.keyAt(i);
        }
    }

    audio_hw_device_t *dev;

    int rc = load_audio_interface(name, &dev);
    if (rc) {
        ALOGI("loadHwModule() error %d loading module %s ", rc, name);
        return 0;
    }

    mHardwareStatus = AUDIO_HW_INIT;
    rc = dev->init_check(dev);
    mHardwareStatus = AUDIO_HW_IDLE;
    if (rc) {
        ALOGI("loadHwModule() init check error %d for module %s ", rc, name);
        return 0;
    }

    // Check and cache this HAL's level of support for master mute and master
    // volume.  If this is the first HAL opened, and it supports the get
    // methods, use the initial values provided by the HAL as the current
    // master mute and volume settings.

    AudioHwDevice::Flags flags = static_cast<AudioHwDevice::Flags>(0);
    {  // scope for auto-lock pattern
        AutoMutex lock(mHardwareLock);

        if (0 == mAudioHwDevs.size()) {
            mHardwareStatus = AUDIO_HW_GET_MASTER_VOLUME;
            if (NULL != dev->get_master_volume) {
                float mv;
                if (OK == dev->get_master_volume(dev, &mv)) {
                    mMasterVolume = mv;
                }
            }

            mHardwareStatus = AUDIO_HW_GET_MASTER_MUTE;
            if (NULL != dev->get_master_mute) {
                bool mm;
                if (OK == dev->get_master_mute(dev, &mm)) {
                    mMasterMute = mm;
                }
            }
        }

        mHardwareStatus = AUDIO_HW_SET_MASTER_VOLUME;
        if ((NULL != dev->set_master_volume) &&
            (OK == dev->set_master_volume(dev, mMasterVolume))) {
            flags = static_cast<AudioHwDevice::Flags>(flags |
                    AudioHwDevice::AHWD_CAN_SET_MASTER_VOLUME);
        }

        mHardwareStatus = AUDIO_HW_SET_MASTER_MUTE;
        if ((NULL != dev->set_master_mute) &&
            (OK == dev->set_master_mute(dev, mMasterMute))) {
            flags = static_cast<AudioHwDevice::Flags>(flags |
                    AudioHwDevice::AHWD_CAN_SET_MASTER_MUTE);
        }

        mHardwareStatus = AUDIO_HW_IDLE;
    }

    audio_module_handle_t handle = nextUniqueId();
    mAudioHwDevs.add(handle, new AudioHwDevice(name, dev, flags));

    ALOGI("loadHwModule() Loaded %s audio interface from %s (%s) handle %d",
          name, dev->common.module->name, dev->common.module->id, handle);

    return handle;

}
```

此函数主要做了一下事情：
1. 函数首先加载系统定义的音频接口对应的so库，并打开该音频接口的抽象硬件设备audio_hw_device_t
2. 设置音频模块主音量
3. 为每个音频接口设备生成独一无二的ID号，同时将打开的音频接口设备封装为AudioHwDevice对象，将系统中所有的音频接口设备保存到AudioFlinger的成员变量mAudioHwDevs中。

此函数返回了音频设备唯一id，赋值给AudioPolicyManagerBase中的mHwModules，这样使得配置项与每个设备接口真正的对应了起来
mHwModules[i]->mHandle = mpClientInterface->loadHwModule(mHwModules[i]->mName);

load_audio_interface加载各音频模块，初始化audio_hw_device_t

```c
static int load_audio_interface(const char *if_name, audio_hw_device_t **dev)
{
    const hw_module_t *mod;
    int rc;

    rc = hw_get_module_by_class(AUDIO_HARDWARE_MODULE_ID, if_name, &mod);
    ALOGE_IF(rc, "%s couldn't load audio hw module %s.%s (%s)", __func__,
                 AUDIO_HARDWARE_MODULE_ID, if_name, strerror(-rc));
    if (rc) {
        goto out;
    }
    rc = audio_hw_device_open(mod, dev);
    ALOGE_IF(rc, "%s couldn't open audio hw device in %s.%s (%s)", __func__,
                 AUDIO_HARDWARE_MODULE_ID, if_name, strerror(-rc));
    if (rc) {
        goto out;
    }
    if ((*dev)->common.version != AUDIO_DEVICE_API_VERSION_CURRENT) {
        ALOGE("%s wrong audio hw device version %04x", __func__, (*dev)->common.version);
        rc = BAD_VALUE;
        goto out;
    }
    return 0;

out:
    *dev = NULL;
    return rc;
}
```

此处加载各种音频设备与之前的一致。

##### 打开attached_output_devices输出

获取全局配置中attached_output_devices 配置的devices，与mHwModules中mOutputProfiles匹配，如果匹配到且outProfile->mFlags不为AUDIO_OUTPUT_FLAG_DIRECT时，将会打开输出

打开输出：

```c
// hardware/libhardware_legacy/audio/AudioPolicyManagerBase.cpp
    audio_io_handle_t output = mpClientInterface->openOutput(
                                    outProfile->mModule->mHandle,
                                    &outputDesc->mDevice,
                                    &outputDesc->mSamplingRate,
                                    &outputDesc->mFormat,
                                    &outputDesc->mChannelMask,
                                    &outputDesc->mLatency,
                                    outputDesc->mFlags);
```

最终调用到AudioFlinger.openOutput

```c
audio_io_handle_t AudioFlinger::openOutput(audio_module_handle_t module,
                                           audio_devices_t *pDevices,
                                           uint32_t *pSamplingRate,
                                           audio_format_t *pFormat,
                                           audio_channel_mask_t *pChannelMask,
                                           uint32_t *pLatencyMs,
                                           audio_output_flags_t flags,
                                           const audio_offload_info_t *offloadInfo)
{
    PlaybackThread *thread = NULL;
    struct audio_config config;
    config.sample_rate = (pSamplingRate != NULL) ? *pSamplingRate : 0;
    config.channel_mask = (pChannelMask != NULL) ? *pChannelMask : 0;
    config.format = (pFormat != NULL) ? *pFormat : AUDIO_FORMAT_DEFAULT;
    if (offloadInfo) {
        config.offload_info = *offloadInfo;
    }

    audio_stream_out_t *outStream = NULL;
    AudioHwDevice *outHwDev;

    ALOGV("openOutput(), module %d Device %x, SamplingRate %d, Format %#08x, Channels %x, flags %x",
              module,
              (pDevices != NULL) ? *pDevices : 0,
              config.sample_rate,
              config.format,
              config.channel_mask,
              flags);
    ALOGV("openOutput(), offloadInfo %p version 0x%04x",
          offloadInfo, offloadInfo == NULL ? -1 : offloadInfo->version );

    if (pDevices == NULL || *pDevices == 0) {
        return 0;
    }

    Mutex::Autolock _l(mLock);

    outHwDev = findSuitableHwDev_l(module, *pDevices);
    if (outHwDev == NULL)
        return 0;

    audio_hw_device_t *hwDevHal = outHwDev->hwDevice();
    audio_io_handle_t id = nextUniqueId();

    mHardwareStatus = AUDIO_HW_OUTPUT_OPEN;

    status_t status = hwDevHal->open_output_stream(hwDevHal,
                                          id,
                                          *pDevices,
                                          (audio_output_flags_t)flags,
                                          &config,
                                          &outStream);

    mHardwareStatus = AUDIO_HW_IDLE;
    ALOGV("openOutput() openOutputStream returned output %p, SamplingRate %d, Format %#08x, "
            "Channels %x, status %d",
            outStream,
            config.sample_rate,
            config.format,
            config.channel_mask,
            status);

    if (status == NO_ERROR && outStream != NULL) {
        AudioStreamOut *output = new AudioStreamOut(outHwDev, outStream, flags);

        if (flags & AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD) {
            thread = new OffloadThread(this, output, id, *pDevices);
            ALOGV("openOutput() created offload output: ID %d thread %p", id, thread);
        } else if ((flags & AUDIO_OUTPUT_FLAG_DIRECT) ||
            (config.format != AUDIO_FORMAT_PCM_16_BIT) ||
            (config.channel_mask != AUDIO_CHANNEL_OUT_STEREO)) {
            thread = new DirectOutputThread(this, output, id, *pDevices);
            ALOGV("openOutput() created direct output: ID %d thread %p", id, thread);
        } else {
            thread = new MixerThread(this, output, id, *pDevices);
            ALOGV("openOutput() created mixer output: ID %d thread %p", id, thread);
        }
        mPlaybackThreads.add(id, thread);

        if (pSamplingRate != NULL) {
            *pSamplingRate = config.sample_rate;
        }
        if (pFormat != NULL) {
            *pFormat = config.format;
        }
        if (pChannelMask != NULL) {
            *pChannelMask = config.channel_mask;
        }
        if (pLatencyMs != NULL) {
            *pLatencyMs = thread->latency();
        }

        // notify client processes of the new output creation
        thread->audioConfigChanged_l(AudioSystem::OUTPUT_OPENED);

        // the first primary output opened designates the primary hw device
        if ((mPrimaryHardwareDev == NULL) && (flags & AUDIO_OUTPUT_FLAG_PRIMARY)) {
            ALOGI("Using module %d has the primary audio interface", module);
            mPrimaryHardwareDev = outHwDev;

            AutoMutex lock(mHardwareLock);
            mHardwareStatus = AUDIO_HW_SET_MODE;
            hwDevHal->set_mode(hwDevHal, mMode);
            mHardwareStatus = AUDIO_HW_IDLE;
        }
        return id;
    }

    return 0;
}
```

1. 查找对应的音频接口设备audio_hw_device_t
2. 初始化音频输出流对象audio_stream_out_t
3. 将AudioHwDevice及audio_stream_out_t封装成AudioStreamOut
4. 根据不同类型flags创建PlaybackThread
5. 给每个thread对应一个唯一id缓存下来mPlaybackThreads.add(id, thread);

##### 保存输出设备描述符对象

### 其他

#### 查找对应的音频输出设备

[hardware/libhardware_legacy/audio/AudioPolicyManagerBase.cpp]

```java
audio_io_handle_t AudioPolicyManagerBase::getOutput(AudioSystem::stream_type stream,
                                    uint32_t samplingRate,
                                    uint32_t format,
                                    uint32_t channelMask,
                                    AudioSystem::output_flags flags,
                                    const audio_offload_info_t *offloadInfo)
{
    audio_io_handle_t output = 0;
    uint32_t latency = 0;
    routing_strategy strategy = getStrategy((AudioSystem::stream_type)stream);
    audio_devices_t device = getDeviceForStrategy(strategy, false /*fromCache*/);

    .........

    IOProfile *profile = NULL;
    if (((flags & AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD) == 0) ||
            !isNonOffloadableEffectEnabled()) {
        profile = getProfileForDirectOutput(device,
                                           samplingRate,
                                           format,
                                           channelMask,
                                           (audio_output_flags_t)flags);
    }

    ..........

        outputDesc = new AudioOutputDescriptor(profile);
        outputDesc->mDevice = device;
        outputDesc->mSamplingRate = samplingRate;
        outputDesc->mFormat = (audio_format_t)format;
        outputDesc->mChannelMask = (audio_channel_mask_t)channelMask;
        outputDesc->mLatency = 0;
        outputDesc->mFlags =(audio_output_flags_t) (outputDesc->mFlags | flags);
        outputDesc->mRefCount[stream] = 0;
        outputDesc->mStopTime[stream] = 0;
        outputDesc->mDirectOpenCount = 1;
        output = mpClientInterface->openOutput(profile->mModule->mHandle,
                                        &outputDesc->mDevice,
                                        &outputDesc->mSamplingRate,
                                        &outputDesc->mFormat,
                                        &outputDesc->mChannelMask,
                                        &outputDesc->mLatency,
                                        outputDesc->mFlags,
                                        offloadInfo);

         .................
        return output;
    }
```

1. 根据音频类型得到播放策略Strategy

    enum routing_strategy {
            STRATEGY_MEDIA,
            STRATEGY_PHONE,
            STRATEGY_SONIFICATION,
            STRATEGY_SONIFICATION_RESPECTFUL,
            STRATEGY_DTMF,
            STRATEGY_ENFORCED_AUDIBLE,
            STRATEGY_EXTERNAL,
            NUM_STRATEGIES
        };

2. 根据播放策略Strategy找到音频的设备id

    AUDIO_DEVICE_OUT_EARPIECE                  = 0x1,
    AUDIO_DEVICE_OUT_SPEAKER                   = 0x2,
    AUDIO_DEVICE_OUT_WIRED_HEADSET             = 0x4,
    AUDIO_DEVICE_OUT_WIRED_HEADPHONE           = 0x8,
    AUDIO_DEVICE_OUT_BLUETOOTH_SCO             = 0x10,
    AUDIO_DEVICE_OUT_BLUETOOTH_SCO_HEADSET     = 0x20,
    AUDIO_DEVICE_OUT_BLUETOOTH_SCO_CARKIT      = 0x40,
    AUDIO_DEVICE_OUT_BLUETOOTH_A2DP            = 0x80,
    AUDIO_DEVICE_OUT_BLUETOOTH_A2DP_HEADPHONES = 0x100,
    AUDIO_DEVICE_OUT_BLUETOOTH_A2DP_SPEAKER    = 0x200,
    AUDIO_DEVICE_OUT_AUX_DIGITAL               = 0x400,
    AUDIO_DEVICE_OUT_ANLG_DOCK_HEADSET         = 0x800,
    AUDIO_DEVICE_OUT_DGTL_DOCK_HEADSET         = 0x1000,
    AUDIO_DEVICE_OUT_USB_ACCESSORY             = 0x2000,
    AUDIO_DEVICE_OUT_USB_DEVICE                = 0x4000,
    AUDIO_DEVICE_OUT_REMOTE_SUBMIX             = 0x8000,
    AUDIO_DEVICE_OUT_EXTERNAL                  = 0x10000,

3. 再根据device, samplingRate,format, channelMask,flags 从audio_policy.conf中匹配，找到合适的设备