播放声音可以用MediaPlayer和AudioTrack，两者都提供了java API供应用开发者使用。虽然都可以播放声音，但两者还是有很大的区别的。其中最大的区别是MediaPlayer可以播放多种格式的声音文件，例如MP3，AAC，WAV，OGG，MIDI等。MediaPlayer会在framework层创建对应的音频解码器。而AudioTrack只能播放已经解码的PCM流，如果是文件的话只支持wav格式的音频文件，因为wav格式的音频文件大部分都是PCM流。AudioTrack不创建解码器，所以只能播放不需要解码的wav文件。当然两者之间还是有紧密的联系，MediaPlayer在framework层还是会创建AudioTrack，把解码后的PCM数流传递给AudioTrack，AudioTrack再传递给AudioFlinger进行混音，然后才传递给硬件播放,所以是MediaPlayer包含了AudioTrack。

The AudioTrack class manages and plays a single audio resource for Java applications. It allows streaming of PCM audio buffers to the audio sink for playback. This is achieved by "pushing" the data to the AudioTrack object using one of the write(byte[], int, int), write(short[], int, int), and write(float[], int, int, int) methods.

An AudioTrack instance can operate under two modes: static or streaming.
In Streaming mode, the application writes a continuous stream of data to the AudioTrack, using one of the write() methods. These are blocking and return when the data has been transferred from the Java layer to the native layer and queued for playback. The streaming mode is most useful when playing blocks of audio data that for instance are:

too big to fit in memory because of the duration of the sound to play,
too big to fit in memory because of the characteristics of the audio data (high sampling rate, bits per sample ...)
received or generated while previously queued audio is playing.

The static mode should be chosen when dealing with short sounds that fit in memory and that need to be played with the smallest latency possible. The static mode will therefore be preferred for UI and game sounds that are played often, and with the smallest overhead possible.
Upon creation, an AudioTrack object initializes its associated audio buffer. The size of this buffer, specified during the construction, determines how long an AudioTrack can play before running out of data.
For an AudioTrack using the static mode, this size is the maximum size of the sound that can be played from it.
For the streaming mode, data will be written to the audio sink in chunks of sizes less than or equal to the total buffer size. AudioTrack is not final and thus permits subclasses, but such use is not recommended.

### 使用流程
```java
    int bufsize = AudioTrack.getMinBufferSize(8000,
        AudioFormat.CHANNEL_CONFIGURATION_MONO,
        AudioFormat.ENCODING_PCM_16BIT);

    AudioTrack audioTrack = new AudioTrack(AudioManager.STREAM_MUSIC, 8000,
            AudioFormat.CHANNEL_CONFIGURATION_MONO,
            AudioFormat.ENCODING_PCM_16BIT, bufsize,
            AudioTrack.MODE_STREAM);

    audioTrack.play();

    ......
    audioTrack.write(flow, 0, flow.length);
    ......

    trackplayer.stop();
    trackplayer.release();
```
### 数据加载模式

### 创建AudioTrack


http://wiki.jikexueyuan.com/project/deep-android-v1/audio.html

http://blog.csdn.net/yangwen123/article/details/39989751