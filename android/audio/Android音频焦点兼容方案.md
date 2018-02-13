# Android音频焦点兼容方案

[TOC]

> 此文仅针对Android4.4实现

## Android原生音频焦点实现

Android机器上会装有各种五法八门的应用，并且可能会有多种应用程序会播放音频，比如播放音乐、视频，或收音机、闹铃、电话铃声等。当音频之间交互时应当有统一的交互规则，否则机器中各种声音混杂在一起不受控制，是让人极其厌恶的体验。为此Android系统提供了统一交互方式，使用音频焦点来进行统一管理。

正常的音频间交互场景：
收音机开始播放音乐暂停，收音机退出后音乐开始继续播放
导航播报时后台音乐暂停或降低音量，播报完毕后后台音乐继续播放或升高音量播放

音频焦点的作用即是协助各应用间音频状态的交互。

1. 请求音频焦点，其他应用失去焦点
2. 请求音频焦点，其他应用失去焦点，可以继续播放，但需要减小音量
3. 请求音频焦点，其他应用无法获取焦点

### 接口介绍

> 相关代码：
> frameworks/base/media/java/android/media/AudioManager.java
> frameworks/base/media/java/android/media/AudioService.java
> frameworks/base/media/java/android/media/MediaFocusControl.java

AudioManager为客户端接口
AudioService为服务端
MediaFocusControl音频焦点管理类，使用栈来管理音频焦点，在请求和释放焦点焦点时会响应进行入栈和出栈操作

AudioManager中接口函数：
1. 请求焦点 requestAudioFocus(OnAudioFocusChangeListener l, int streamType, int durationHint)
2. 释放焦点 abandonAudioFocus(OnAudioFocusChangeListener l)

参数简介：
OnAudioFocusChangeListener l  音频焦点改变监听，用于监听音频焦点状态变化。

```java
    public interface OnAudioFocusChangeListener {
        public void onAudioFocusChange(int focusChange);
    }
```

l也用于标识请求id，同一个进程中只要是不同的OnAudioFocusChangeListener对象，其生成的id就不同，即同一个应用内也可以相互之间抢焦点

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

1. 判断栈顶FocusRequester和请求的clientId是不是同一个clientId和状态，完全相同则表示获取成功，否则移除栈顶数据
2. 移除栈中相同clientId的FocusRequester
3. 通知栈中剩余FocusRequester状态改变
4. 将最新音频焦点请求对象入栈

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

#### SDK目前实现

SDK将能发声的应用定义为不同的音源，车机上的一些需要静音的特殊操作（例如倒车）定义为事件。 源分两种动作：源进、源出。事件也分两种：事件进、事件出。应用需注册监听源进、源出、事件进、事件出等状态。

sdkservice会解析MCU发送sdk的数据，也会解析应用调用sdk发送给MCU的数据，判断是源相关的操作，会回调给各应用的sdk接口，各应用sdk会根据当前源和上次的源来判断是否回调给各应用处理。各应用在回调中处理音频的播放和暂停状态

优点：逻辑比较独立，方便移植；接口中传递源和事件的种类，应用可以做更细致的逻辑处理
缺点：与android音频焦点切换不兼容，不能与第三方音频焦点进行交互；切换音频通道与音频焦点两种逻辑混在了一起

#### 新方案实现

使用Android原生音频焦点逻辑，将原车事件转换为抢焦点方式与应用进行交互，将切源操作仅用于切换音频通道.

应用实现：使用Android音频焦点接口：requestAudioFocus、abandonAudioFocus， 在requestAudioFocus成功后调用切源（用于切换通道）
系统实现：将车机特殊事件转换为音频焦点处理，帮助第三方应用切换音频通道，实现音频交互的特殊需求（有的导航播报时会请求焦点，并在焦点丢失后中断播报，但需求不希望被中断）

针对事件：蓝牙电话、倒车在sdk内部实现请求焦点和切通道操作，导航播报、语音交互仍调用sdk切通道接口，其他逻辑由sdk和frameworks内部实现

蓝牙电话、倒车、语音交互请求特殊的焦点及切通道
导航播报不抢焦点，不处理焦点丢失情况（屏蔽导航应用抢焦点操作）

优点：与android原生音频焦点切换兼容，可以与第三方应用共享音频焦点逻辑；切换音频通道与音频焦点两种逻辑分开处理
缺点：需要将事件转换为android音频焦点处理，增加逻辑复杂度；无法判断当前源或事件。

## 其他

### 打印信息

dumpsys audio命令能打印音频焦点栈中各个客户端的焦点状态