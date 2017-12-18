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

AudioPolicyService实现，如下：
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





### 创建legacy_audio_policy
通过调用上一步得到的legacy_ap_device结构体的函数指针创建legacy_audio_policy
```c
mpAudioPolicyDev->create_audio_policy(mpAudioPolicyDev, &aps_ops, this,
                                               &mpAudioPolicy);
```
以上传入的



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


