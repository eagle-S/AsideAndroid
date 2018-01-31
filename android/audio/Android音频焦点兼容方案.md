# Android音频焦点兼容方案

[TOC]

> 此文仅针对Android4.4实现

## Android原生音频焦点实现

Android使用栈来管理音频焦点，请求焦点会涉及到入栈操作，释放焦点涉及到出栈操作。

### 相关知识点

主要涉及到两个函数：
1. 请求焦点 requestAudioFocus(OnAudioFocusChangeListener l, int streamType, int durationHint)
2. 释放焦点 abandonAudioFocus(OnAudioFocusChangeListener l)

参数简介：

l  音频焦点改变监听，用于监听音频焦点状态变化，也用于标识请求id，同一个进程中只要OnAudioFocusChangeListener是不同对象，其生成的id就不同

``` java
    private String getIdForAudioFocusListener(OnAudioFocusChangeListener l) {
        if (l == null) {
            return new String(this.toString());
        } else {
            return new String(this.toString() + l.toString());
        }
    }
```

streamType 请求的音频类型，暂未看到有什么作用

durationHint  请求音频焦点持续时间，有四种：

AUDIOFOCUS_GAIN  长时间获取焦点
AUDIOFOCUS_GAIN_TRANSIENT  临时获取焦点，在短时间内会释放
AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK 临时获取焦点，之前获取焦点的应用继续压低音量播放
AUDIOFOCUS_GAIN_TRANSIENT_EXCLUSIVE 临时获取焦点，此时系统在此状态下不会播放像通知铃声之类的易混乱的声音

### requestAudioFocus函数分析

```java
    protected int requestAudioFocus(int mainStreamType, int focusChangeHint, IBinder cb,
            IAudioFocusDispatcher fd, String clientId, String callingPackageName) {
        Log.i(TAG, " AudioFocus  requestAudioFocus() from " + clientId);
        // we need a valid binder callback for clients
        if (!cb.pingBinder()) {
            Log.e(TAG, " AudioFocus DOA client for requestAudioFocus(), aborting.");
            return AudioManager.AUDIOFOCUS_REQUEST_FAILED;
        }

        if (mAppOps.noteOp(AppOpsManager.OP_TAKE_AUDIO_FOCUS, Binder.getCallingUid(),
                callingPackageName) != AppOpsManager.MODE_ALLOWED) {
            return AudioManager.AUDIOFOCUS_REQUEST_FAILED;
        }

        synchronized(mAudioFocusLock) {
            if (!canReassignAudioFocus()) {
                return AudioManager.AUDIOFOCUS_REQUEST_FAILED;
            }

            // handle the potential premature death of the new holder of the focus
            // (premature death == death before abandoning focus)
            // Register for client death notification
            AudioFocusDeathHandler afdh = new AudioFocusDeathHandler(cb);
            try {
                cb.linkToDeath(afdh, 0);
            } catch (RemoteException e) {
                // client has already died!
                Log.w(TAG, "AudioFocus  requestAudioFocus() could not link to "+cb+" binder death");
                return AudioManager.AUDIOFOCUS_REQUEST_FAILED;
            }

            if (!mFocusStack.empty() && mFocusStack.peek().hasSameClient(clientId)) {
                // if focus is already owned by this client and the reason for acquiring the focus
                // hasn't changed, don't do anything
                if (mFocusStack.peek().getGainRequest() == focusChangeHint) {
                    // unlink death handler so it can be gc'ed.
                    // linkToDeath() creates a JNI global reference preventing collection.
                    cb.unlinkToDeath(afdh, 0);
                    return AudioManager.AUDIOFOCUS_REQUEST_GRANTED;
                }
                // the reason for the audio focus request has changed: remove the current top of
                // stack and respond as if we had a new focus owner
                FocusRequester fr = mFocusStack.pop();
                fr.release();
            }

            // focus requester might already be somewhere below in the stack, remove it
            removeFocusStackEntry(clientId, false /* signal */);

            // propagate the focus change through the stack
            if (!mFocusStack.empty()) {
                propagateFocusLossFromGain_syncAf(focusChangeHint);
            }

            // push focus requester at the top of the audio focus stack
            mFocusStack.push(new FocusRequester(mainStreamType, focusChangeHint, fd, cb,
                    clientId, afdh, callingPackageName, Binder.getCallingUid()));

            // there's a new top of the stack, let the remote control know
            synchronized(mRCStack) {
                checkUpdateRemoteControlDisplay_syncAfRcs(RC_INFO_ALL);
            }
        }//synchronized(mAudioFocusLock)

        return AudioManager.AUDIOFOCUS_REQUEST_GRANTED;
    }
```

从代码中可以看出音频焦点请求会封装成FocusRequester对象保存在mFocusStack栈中。

1. 首先判断栈顶FocusRequester和请求的clientId是不是同一个clientId和状态
2. 然后移除栈中相同clientId的FocusRequester
3. 通知栈中剩余FocusRequester状态改变
4. 将最新音频焦点请求入栈

第2步移除栈中相同clientId的FocusRequester

```java
    private void removeFocusStackEntry(String clientToRemove, boolean signal) {
        // is the current top of the focus stack abandoning focus? (because of request, not death)
        if (!mFocusStack.empty() && mFocusStack.peek().hasSameClient(clientToRemove))
        {
            //Log.i(TAG, "   removeFocusStackEntry() removing top of stack");
            FocusRequester fr = mFocusStack.pop();
            fr.release();
            if (signal) {
                // notify the new top of the stack it gained focus
                notifyTopOfAudioFocusStack();
                // there's a new top of the stack, let the remote control know
                synchronized(mRCStack) {
                    checkUpdateRemoteControlDisplay_syncAfRcs(RC_INFO_ALL);
                }
            }
        } else {
            // focus is abandoned by a client that's not at the top of the stack,
            // no need to update focus.
            // (using an iterator on the stack so we can safely remove an entry after having
            //  evaluated it, traversal order doesn't matter here)
            Iterator<FocusRequester> stackIterator = mFocusStack.iterator();
            while(stackIterator.hasNext()) {
                FocusRequester fr = (FocusRequester)stackIterator.next();
                if(fr.hasSameClient(clientToRemove)) {
                    Log.i(TAG, "AudioFocus  removeFocusStackEntry(): removing entry for "
                            + clientToRemove);
                    stackIterator.remove();
                    fr.release();
                }
            }
        }
    }

    private void notifyTopOfAudioFocusStack() {
        // notify the top of the stack it gained focus
        if (!mFocusStack.empty()) {
            if (canReassignAudioFocus()) {
                mFocusStack.peek().handleFocusGain(AudioManager.AUDIOFOCUS_GAIN);
            }
        }
    }
```

这里有两个参数：
clientToRemove 表示将要被移除的clientId

signal 表示移除栈顶数据后要不要通知下一个数据获取焦点。在请求焦点时signal传入false，表示请求的clientId和栈顶相同时，移除栈顶数据不需要通知下一个数据，但是主动释放焦点时signal传入true，表示栈顶的自己释放焦点了，下一个数据将获取焦点。

移除逻辑分两种情况：
1. 栈顶FocusRequester和将要移除的是同一个client
2. 栈顶FocusRequester和将要移除的是不同的client

第3步是通知栈中剩余数据焦点状态改变

```java

    private void propagateFocusLossFromGain_syncAf(int focusGain) {
        // going through the audio focus stack to signal new focus, traversing order doesn't
        // matter as all entries respond to the same external focus gain
        Iterator<FocusRequester> stackIterator = mFocusStack.iterator();
        while(stackIterator.hasNext()) {
            stackIterator.next().handleExternalFocusGain(focusGain);
        }
    }
```

```java
    void handleExternalFocusGain(int focusGain) {
        int focusLoss = focusLossForGainRequest(focusGain);
        handleFocusLoss(focusLoss);
    }

    void handleFocusGain(int focusGain) {
        try {
            if (mFocusDispatcher != null) {
                if (DEBUG) {
                    Log.v(TAG, "dispatching " + focusChangeToString(focusGain) + " to "
                        + mClientId);
                }
                mFocusDispatcher.dispatchAudioFocusChange(focusGain, mClientId);
            }
            mFocusLossReceived = AudioManager.AUDIOFOCUS_NONE;
        } catch (android.os.RemoteException e) {
            Log.e(TAG, "Failure to signal gain of audio focus due to: ", e);
        }
    }
```

```java
    private int focusLossForGainRequest(int gainRequest) {
        switch(gainRequest) {
            case AudioManager.AUDIOFOCUS_GAIN:
                switch(mFocusLossReceived) {
                    case AudioManager.AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK:
                    case AudioManager.AUDIOFOCUS_LOSS_TRANSIENT:
                    case AudioManager.AUDIOFOCUS_LOSS:
                    case AudioManager.AUDIOFOCUS_NONE:
                        return AudioManager.AUDIOFOCUS_LOSS;
                }
            case AudioManager.AUDIOFOCUS_GAIN_TRANSIENT_EXCLUSIVE:
            case AudioManager.AUDIOFOCUS_GAIN_TRANSIENT:
                switch(mFocusLossReceived) {
                    case AudioManager.AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK:
                    case AudioManager.AUDIOFOCUS_LOSS_TRANSIENT:
                    case AudioManager.AUDIOFOCUS_NONE:
                        return AudioManager.AUDIOFOCUS_LOSS_TRANSIENT;
                    case AudioManager.AUDIOFOCUS_LOSS:
                        return AudioManager.AUDIOFOCUS_LOSS;
                }
            case AudioManager.AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK:
                switch(mFocusLossReceived) {
                    case AudioManager.AUDIOFOCUS_NONE:
                    case AudioManager.AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK:
                        return AudioManager.AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK;
                    case AudioManager.AUDIOFOCUS_LOSS_TRANSIENT:
                        return AudioManager.AUDIOFOCUS_LOSS_TRANSIENT;
                    case AudioManager.AUDIOFOCUS_LOSS:
                        return AudioManager.AUDIOFOCUS_LOSS;
                }
            default:
                Log.e(TAG, "focusLossForGainRequest() for invalid focus request "+ gainRequest);
                        return AudioManager.AUDIOFOCUS_NONE;
        }
    }
```

### abandonAudioFocus函数分析

```java
    protected int abandonAudioFocus(IAudioFocusDispatcher fl, String clientId) {
        Log.i(TAG, " AudioFocus  abandonAudioFocus() from " + clientId);
        try {
            // this will take care of notifying the new focus owner if needed
            synchronized(mAudioFocusLock) {
                removeFocusStackEntry(clientId, true /*signal*/);
            }
        } catch (java.util.ConcurrentModificationException cme) {
            // Catching this exception here is temporary. It is here just to prevent
            // a crash seen when the "Silent" notification is played. This is believed to be fixed
            // but this try catch block is left just to be safe.
            Log.e(TAG, "FATAL EXCEPTION AudioFocus  abandonAudioFocus() caused " + cme);
            cme.printStackTrace();
        }

        return AudioManager.AUDIOFOCUS_REQUEST_GRANTED;
    }
```

此处调用removeFocusStackEntry，传入的signal参数为true，表示在如果栈顶数据和释放焦点的clientId相同，则通知栈中下一个数据将获取焦点

## 兼容方案

### 车机音频通用设计方案概述

车辆中喇叭播放的声音均为当前播放源和当前优先级最高的音频事件的结合播放声音，具体结合方式根据不同的音频事件和需求会有不同，一般是混音，静音或者前后左右分工播放。

关于播放源和音频事件的描述如下。
播放源：同一时间有且仅有一个播放源播放，不能被退出，只能被另一播放源替换。之后统一称为“源”（source/moden）
音频事件：同一时间可以有多个事件同时发生，彼此存在优先级关系，需要有事件的切入及切出两个消息响应对应动作。

### 需求

源切换或事件切入时被切源处理汇总

#### 源

- 音乐：切源统一暂停
- 视频：切源统一暂停
- 蓝牙音乐：切源统一暂停
- 收音：切源时不需要处理，通道会切换
- 第三方音频应用：切源时统一暂停
- TBOX（预留）

#### 事件

- 蓝牙电话：优先级最高，四个喇叭全播放蓝牙电话，播放源按切源操作执行
- 倒车：整车静音，播放源按切源操作执行
- 导航播报：前左前右喇叭按固定比例和播放源混音播出，后两喇叭播放当前源声
- 语音交互：整车出语音交互声音，播放源按切源操作执行

### 分析与实现

#### 分析

以上需求看出主要功能有两个：
1. 控制音频播放与暂停
2. 控制音频播放时通道切换

需求一由源之间协调来控制，需求二需发MCU指令给MCU来实现切换

音频播放与暂停场景：
场景一：源与源之间音频暂停播放
可以使用Android原生音频焦点实现。
源自身播放时请求焦点成功后开始播放，其他源收到焦点丢失时暂停处理。针对收音机收到焦点丢失不做任何处理即可。
当再次获得焦点时继续播放

场景二：源与事件之间音频暂停播放
可以使用Android原生音频焦点实现。
当事件发生时，系统会请求一个特殊的音频焦点，源收到焦点丢失时暂停处理。此时特殊音频焦点会处于音频栈顶部，此时源请求焦点将失败。由于导航播报不暂停源，所以导航播报时不会请求特殊的音频焦点，源可以随意请求焦点

音频通道切换场景：
在请求焦点的同时切换通道，如果第三方音频发声需要特殊通道，在第三方请求焦点时帮其切换通道。或在第三方提供的播放、暂停接口中切换通道

#### 应用调用

使用Android原生口：requestAudioFocus、abandonAudioFocus， 在requestAudioFocus成功后处理切源
（替换掉源进、源出、事件进、事件出处理）

注意：抢音频焦点和切源同时进行

针对事件：蓝牙电话、倒车在sdk内部实现请求焦点和切通道操作，导航播报、语音交互仍调用sdk切通道接口，其他逻辑由sdk和frameworks内部实现

蓝牙电话、倒车、语音交互请求特殊的焦点及切通道
导航播报不抢焦点，不处理焦点丢失情况（屏蔽导航应用抢焦点操作）

## 其他

### dumpsys audio