# Android音频焦点兼容方案

> 此文仅针对Android4.4实现

## Android原生音频焦点实现

Android使用栈来管理音频焦点，请求焦点会涉及到入栈操作，释放焦点涉及到出栈操作。


### 相关知识点

主要涉及到两个函数：
1. 请求焦点 requestAudioFocus(OnAudioFocusChangeListener l, int streamType, int durationHint)
2. 释放焦点 abandonAudioFocus(OnAudioFocusChangeListener l)

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

### requestAudioFocus

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

第二步移除栈中相同clientId的FocusRequester

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
```

这里有两个参数：
clientToRemove 表示将要被移除的clientId

signal 表示移除栈顶数据后要不要通知下一个数据获取焦点。在请求焦点时signal传入false，表示请求的clientId和栈顶相同时，移除栈顶数据不需要通知下一个数据，但是主动释放焦点时signal传入true，表示栈顶的自己释放焦点了，下面的数据将获取焦点。

移除逻辑分两种情况：
1. 栈顶FocusRequester和将要移除的是同一个client
2. 栈顶FocusRequester和将要移除的是不同的client

### abandonAudioFocus

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


## 兼容方案



## 其他

### dumpsys audio