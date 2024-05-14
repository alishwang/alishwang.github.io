
+++
author = "fuhua"
title = "Android - Audio - audioTrack如何调用audioFlinger播放声音的（基于QM439分析）"
date = "2020-01-12 16:56:23"
description = "手机技术笔记"
tags = [
    "markdown",
    "text",
]
weight = 10
+++


## __背景__
>如今android越来越庞大，遇到的问题也越来越多的客制化，按照本人的理解，音频的hal层以下只是向上提供接口，在硬件条件满足的情况下完成声音输出的各种需求。  
本文将从电话挂断的那段生成的声音（我们的项目挂断的时候是“嘟”的一声）进行分析。达到分析AT和AF的目的。  

## 从ToneGenerator开始  
我们首先从 `frameworks\av\media\libaudioclient\ToneGenerator.cpp`分析。  
找到AudioTrack->set函数  
```
 status_t status = mpAudioTrack->set(
            AUDIO_STREAM_DEFAULT,
            0,    // sampleRate
            AUDIO_FORMAT_PCM_16_BIT,
            AUDIO_CHANNEL_OUT_MONO,
            frameCount,
            AUDIO_OUTPUT_FLAG_FAST,
            audioCallback,
            this, // user
            0,    // notificationFrames
            0,    // sharedBuffer
            mThreadCanCallJava,
            AUDIO_SESSION_ALLOCATE,
            AudioTrack::TRANSFER_CALLBACK,
            nullptr,
            AUDIO_UID_INVALID,
            -1,
            &attr);
```
我们可以看到有一个audioCallback函数，在其中加上log，顺着代码再往下阅读，有`ToneGenerator::WaveGenerator::WaveGenerato(uint32_t samplingRate,unsigned short frequency, float volume)`这样一个函数，也在其中添加上log，最后拨打电话，挂断，“嘟”的那一声打印出来的log如下。  
```
01-01 08:11:23.859  1356  9272 E ToneGenerator:  fuhua --- WaveGenerator init, mA1_Q14: 32723, mS2_0: -1674, mAmplitude_Q15: 10065
01-01 08:11:23.859  1356  9272 E ToneGenerator:  fuhua --- WaveGenerator init, mA1_Q14: 32364, mS2_0: -5005, mAmplitude_Q15: 10065
01-01 08:11:23.864  1356  9273 E ToneGenerator: fuhua --- audioCallback --- begin
01-01 08:11:23.864  1356  9273 E ToneGenerator: fuhua --- audioCallback --- while(lNumSmp)
01-01 08:11:23.865  1356  9273 E ToneGenerator: fuhua --- audioCallback --- end

(重复出现多次)
01-01 08:11:24.053  1356  9273 E ToneGenerator: fuhua --- audioCallback --- begin
01-01 08:11:24.053  1356  9273 E ToneGenerator: fuhua --- audioCallback --- while(lNumSmp)
01-01 08:11:24.053  1356  9273 E ToneGenerator: fuhua --- audioCallback --- end
01-01 08:11:24.057  1356  9273 E ToneGenerator: fuhua --- audioCallback --- begin
01-01 08:11:24.057  1356  9273 E ToneGenerator: fuhua --- audioCallback --- while(lNumSmp)
01-01 08:11:24.057  1356  9273 E ToneGenerator: fuhua  Cbk Stopped track
01-01 08:11:24.058  1356  9273 E ToneGenerator: fuhua --- audioCallback --- end
01-01 08:11:24.086  1356  9272 E ToneGenerator: fuhua  Delete Track: 0x7815a94000
01-01 08:11:24.093   617  7252 E DeviceHalHidl: fuhua-check-devicehalHidl-4.0-hidl

01-01 08:11:24.950  8967  8967 E ToneGenerator: fuhua- in initAudioTrack() --AudioTrack(0x78172e1800) created
01-01 08:12:24.172  8967  8967 E ToneGenerator: fuhua  Delete Track: 0x78172e18
```

从这一段我们可以看出在循环调用audioCallback函数  
是为什么调用audioCallback函数呢？通过查找，能找到audioCallback给了AudioTrack里面有一个mCbf的函数，由该函数进行处理。  

该声音的流程是如何呢？__如何生成，何时开始，何时结束？__  
我们首先聚焦时间，毕竟该问题的起因是在处理一个杂音问题的时候发现的，因为“嘟”相对过长导致hal层内处理voice-path，还未停止，low-latency的“嘟”出现了，当然在这期间能够完成播放也就罢了，但是voice-path有可能提前就完成了stop，那么寄存器的值相对就有变化，进而导致了pop音，我这边解决方案其实就是在关闭voice-path的时候延时500ms，因为“嘟”很短，当然还有一个办法就是缩短“嘟”声，但是这样改一是可能改了声音设计，让声音出来时自己延时一下（虽然我司定制成分极少），一个根本原因就是当时找不到在哪里改23333，但是研究完该流程，也应该知道在哪里改了。  

通过阅读代码并结合log，我们可以把ToneGenerator.cpp分为三个部分，顶部设置了各种声音，比如按键音TONE_DTMF,call waiting音TONE_SUP_CALL_WAITING,我们这个问题，实际上是TONE_PROP_NACK,这个声音。  
中部就是初始化函数以及和AudioTrack有关的函数，比如构造函数，析构函数，startTone,stopTone，initAudioTrack,audioCallback。  
底部就是生成波形的函数。这些函数主要处理的是顶部那些设置，生成对应的频率，振幅，一帧一帧的给audioCallback进行处理，我们暂且对生成的何种数据不做关心，总归要给到AudioTrack的，我将在下文对AudioTrack如何处理的进行深入的探求。当然底部还有一个就是WaveGenerator成员，顾名思义，在此不探究。  


那根据查证，发现“嘟”的一声完后会走到`audioCallback`函数中的如下`if (lpToneGen->mTotalSmp > lpToneGen->mNextSegSmp)`判断，由此找到`mNextSegSmp`是关键的判断量，那么继续查找并打印log，发现在`prepareWave`函数中初始化了该变量  
```
    // Initialize tone sequencer
    mTotalSmp = 0;
    mCurSegment = 0;
    mCurCount = 0;
    mLoopCounter = 0;
    if (mpToneDesc->segments[0].duration == TONEGEN_INF) {
        mNextSegSmp = TONEGEN_INF;
		ALOGE("fuhua----prepareWave,if 1 mNextSegSmp = %d,duration = %d",mNextSegSmp,mpToneDesc->segments[0].duration);
    } else{
        mNextSegSmp = (mpToneDesc->segments[0].duration * mSamplingRate) / 1000;
		ALOGE("fuhua----prepareWave,if 2 mNextSegSmp = %d,duration = %d",mNextSegSmp,mpToneDesc->segments[0].duration);
    }
```
对应确实有如下log：  

`ToneGenerator: fuhua----prepareWave,if 2 mNextSegSmp = 480,duration = 10`  


之前提出的三个疑问，首先可以回答 __如何生成__ 了，通过preparewave函数生成的。时间参数、频率参数等在ToneGenerator.cpp顶部的配置。  

那么 __何时开始__ 呢？自然是在构造ToneGenerator之后，调用startTone的时候咯。  
由于本人也没有做app，所以怀疑如下可能相关的调用代码：  
```
packages/services/Telecomm/src/com/android/server/telecom/	InCallTonePlayer.java
255 toneGenerator.startTone(toneType); in run() 


vendor/tct/apps/TctTelecomm/src/com/android/server/telecom/DtmfLocalTonePlayer.java
69 mToneGenerator.startTone(toneType, durationMs); in startTone()
108 mToneGeneratorProxy.startTone(tone, -1 /* toneDuration */); in handleMessage() 

vendor/tct/apps/TctTelecomm/src/com/android/server/telecom/InCallTonePlayer.java
toneGenerator.startTone(toneType); in run() 

vendor/tct/apps/TctTelephony/src/com/android/phone/CallNotifier.java
toneGenerator.startTone(toneType); in run() 

packages/apps/Dialer/java/com/android/incallui/ringtone/InCallTonePlayer.java
110 toneGenerator.startTone(info.tone); in playOnBackgroundThread() 

packages/apps/Dialer/java/com/android/dialer/dialpadview/DialpadFragment.java	
1182 toneGenerator.startTone(tone, durationMs); in playTone() 

```

那么 __何时结束__ 呢？我们在析构函数中能找到stopTone的调用，但是这里面就只有超时的时候调用AudioTrack->stop，估计不在这里面，而`audioCallback`中的totalsmp > nextsgesmp 的时候会将mstate置为`lpToneGen->mState = TONE_STOPPING`  既然如此就会去到`audioCallback_EndLoop:`，然后就会调用`lpToneGen->mpAudioTrack->stop();`。
```
audioCallback_EndLoop:

        switch (lpToneGen->mState) {
        case TONE_RESTARTING:

        .....

        case TONE_STOPPED:
            lpToneGen->mState = TONE_INIT;
            ALOGE("fuhua  Cbk Stopped track");
            lpToneGen->mpAudioTrack->stop();
            // Force loop exit
            lNumSmp = 0;
            buffer->size = 0;
            lSignal = true;
            break;
```
AudioTrack倒是stop掉了但是还是没有找到触发ToneGenerator析构的逻辑。  
从AudioTrack的stop和其中的`mProxy->stop();`来看，应该是audiotrack的生命周期到了之后ToneGenerator才会析构，因为上层的代码来看也没有看到多久去析构，声明周期多久结束，而是native层自己控制的？？？也还有种可能，就是audiotrack stop或者release的时候通过某种机制传递给了JNI或者Framework层，比如`mProxy`，然后再通过上层的生命周期结束掉ToneGenerator，因为ToneGenerator->stop里调用audiotrack->stop的地方是超时的时候才去做的。还有一种可能就是调用startTone的地方自己析构了，这样也能层层析构下来。    

那么至此算是理到了Audiotrack的开始和结束，而AudioTrack和AudioFlinger的关系才刚刚开始，也算是为了搞清楚 __何时结束__ 吧。  

先附上分析之后的流程草图，有利于理清关系和记忆：  
![流程图](http://img.alish.wang/d0/68a048bcb37848e67d06ed6c1e2240f643128b.jpg)


## AudioTrack和ToneGenerator的联系  
```
void ToneGenerator::audioCallback(int event, void* user, void *info) {
    if (event != AudioTrack::EVENT_MORE_DATA) return;

    AudioTrack::Buffer *buffer = static_cast<AudioTrack::Buffer *>(info);//从AudioTrack那里拿到了buffer的大小。
    ToneGenerator *lpToneGen = static_cast<ToneGenerator *>(user);//从callback函数中拿到了自己。
```
lpToneGen实际上是拿到了之前传给AudioTrack的自己，因为`lpToneGen->mNextSegSmp`在初始化之后再从callback拿回来都是没变的。  

我们进入AudioTrack，加上log发现AudioTrack中打印mcbf，一直都是走的`mCbf(EVENT_MORE_DATA, mUserData, &audioBuffer);`整个文件唯独一处，在`nsecs_t AudioTrack::processAudioBuffer()`函数当中 ( 返回值是上一次运行结束的时间？这仔细查看一下该函数。),也没有走其他的EVENT命令，所以再次确认“嘟”的停止大概率不在AudioTrack控制。  

那么AudioTrack又是多久开始处理数据？数据如何传输给AudioTrack，AudioTrack如何处理数据，多久结束呢？  

从ToneGenerator来看，开始和结束两个问题都比较好回答，我们都能找到` mpAudioTrack->start()`(StartTone函数里面)和` mpAudioTrack->stop()`函数，而且上文已经找到了对应的位置。那么剩下两个问题，一个是数据如何给到AudioTrack的，一个是AudioTrack如何处理数据的。

### 数据如何给到AudioTrack  

我们可以看到在ToneGenerator里面有`bool ToneGenerator::prepareWave()`new了一个WaveGenerator，然后添加到了`mWaveGens.add(frequency, lpWaveGen);`，而`mWaveGens`是一个容器，然后在audiocallback中给了`WaveGenerator *lpWaveGen = lpToneGen->mWaveGens.valueFor(lFrequency);`，然而这个lpWaveGen没怎么用到？？？我们只看到了它调用`getSamples`函数，对该函数进行解读，发现它实际上是在不断的计算之后给到`short *outBuffer`的，所以数据是如何给的，其实已经了然了。所有的计算，控制逻辑其实都能在ToneGenerator.cpp文件中体现。那么接下来就是看outBuffer怎么处理这个数据的了



### AudioTrack如何处理数据  

猜想是避免不了AudioTrack的，AudioTrack.cpp位置在`frameworks\av\media\libaudioclient\AudioTrack.cpp`，应该是属于native层的（只是说一下，23333）。  
要想知道如何处理数据，那么就要知道数据在存哪里，怎么申请的(防止内存泄漏)，谁去用了它，怎么用的它。所以接着outbuffer来看，我们看到audiocallback中调用`lpWaveGen->getSamples(lpOut, lGenSmp, lWaveCmd);`地方只有两处，一处是上文分析的停止的时候，一处是一直在调用audiocallback的时候，所以基本能确定就是lpOut了，而它来自于`AudioTrack::Buffer *buffer = static_cast<AudioTrack::Buffer *>(info);`，所以得去到AudioTrack分析。  

#### 怎么申请  

我们已经知道在set的时候就已经把audiocallback传给了cbf函数，然后audiotrack的set函数又将cbf给了`mCbf`，也可以知道了audioTrack应该是构造之后常驻内存的了。通过上文的分析，AudioTrack->processAudioBuffer()走的`mCbf(EVENT_MORE_DATA, mUserData, &audioBuffer);`这个函数，那么`&audioBuffer`就至关重要了。然而我们发现了AudioTrack的代理.....这就增加了阅读难度了  
```
    Proxy::Buffer buffer;
    ......

    sp<AudioTrackClientProxy> proxy;

    ......

    buffer.mFrameCount = audioBuffer->frameCount;
    // FIXME starts the requested timeout and elapsed over from scratch
    status = proxy->obtainBuffer(&buffer, requested, elapsed);
```

发现`AudioTrackClientProxy`是在`AudioTrackShared.cpp`文件当中，也仅在这个文件当中，那么从.h文件中我们能看到`class AudioTrackClientProxy : public ClientProxy `继承了ClientProxy，所以我们直接找ClientProxy的obtainBuffer,暂时不管ServerProxy的，可以从下面的代码看到buffer的操作全是用的front和rear，这样一个线索，  

```
status_t ClientProxy::obtainBuffer(Buffer* buffer, const struct timespec *requested,
        struct timespec *elapsed)

        ......
    for (;;) {
        ......

        int32_t front;
        int32_t rear;
        if (mIsOut) {

            front = android_atomic_acquire_load(&cblk->u.mStreaming.mFront);
            rear = cblk->u.mStreaming.mRear;


        ......
            if (part1 > buffer->mFrameCount) {
                part1 = buffer->mFrameCount;
            }
            buffer->mFrameCount = part1;
            buffer->mRaw = part1 > 0 ?
                    &((char *) mBuffers)[(mIsOut ? rear : front) * mFrameSize] : NULL;
            buffer->mNonContig = avail - part1;

```
而front和rear都来自于cblk，也就是`audio_track_cblk_t`这个东西(cblk就是control block的缩写),该函数也有定义`audio_track_cblk_t* cblk = mCblk;`，这个东西在初始化列表的时候似乎已经传进来了，也就是说要去找到构造函数，那么通过搜索Proxy 等关键字，很快我们能找到只有一个地方做了这样的操作，如下：
```
status_t AudioTrack::createTrack_l()
{

    sp<IAudioTrack> track = audioFlinger->createTrack(input,
                                                    output,
                                                    &status);
    ......

    sp<IMemory> iMem = track->getCblk();

    ......


    void *iMemPointer = iMem->pointer();

    .....

    audio_track_cblk_t* cblk = static_cast<audio_track_cblk_t*>(iMemPointer);

    ......

    // update proxy
    if (mSharedBuffer == 0) {
        mStaticProxy.clear();
        mProxy = new AudioTrackClientProxy(cblk, buffers, mFrameCount, mFrameSize);
    } else {
        mStaticProxy = new StaticAudioTrackClientProxy(cblk, buffers, mFrameCount, mFrameSize);
        mProxy = mStaticProxy;
    }

```

我们暂且不管AudioTrackClientProxy是否Static，只关注cblk的来历，以此找到申请buffer的具体函数(当然整个过程我们还是追到了createTrack_l，其实看这个应该还是先看构造函数和create函数的)。函数从下往上追，就追到了audioFlinger的createTrack函数了。也就是说audioTrack的内存控制的逻辑部分是是掌握在自己手中的，而实际的内存，即共享内存是在AudioFlinger中申请的？  

我们执果索因，返回给AudioTrack的是trackhandl，那么就往上追溯，代码如下：  

```
sp<IAudioTrack> AudioFlinger::createTrack(const CreateTrackInput& input,
                                          CreateTrackOutput& output,
                                          status_t *status)
{
    sp<PlaybackThread::Track> track;
    sp<TrackHandle> trackHandle;

    ......

    PlaybackThread *thread = checkPlaybackThread_l(output.outputId);

    .....

    track = thread->createTrack_l(client, streamType, input.attr, &output.sampleRate,
                                    input.config.format, input.config.channel_mask,
                                    &output.frameCount, &output.notificationFrameCount,
                                    input.notificationsPerBuffer, input.speed,
                                    input.sharedBuffer, sessionId, &output.flags,
                                    input.clientInfo.clientTid, clientUid, &lStatus, portId);

    ......

    // return handle to client
    trackHandle = new TrackHandle(track);

Exit:
    if (lStatus != NO_ERROR && output.outputId != AUDIO_IO_HANDLE_NONE) {
        AudioSystem::releaseOutput(output.outputId, streamType, sessionId);
    }
    *status = lStatus;
    return trackHandle;
}
```


首先就分析`checkPlaybackThread_l`这个函数。  

```
// checkPlaybackThread_l() must be called with AudioFlinger::mLock held
AudioFlinger::PlaybackThread *AudioFlinger::checkPlaybackThread_l(audio_io_handle_t output) const
{
    return mPlaybackThreads.valueFor(output).get();
}
```

这个函数就是从一个容器里面拿outputid的操作，应该是找到对应的线程（实际上分析这个流程有几个个初衷，其中一个就是想看怎么找到对应的实际线程）。应该找到了就是一个id，后面debug的时候可以打印一下看看，好像也没有其他的东西了，然后就是很大一坨的`track = thread->createTrack_l(...)`函数，这个函数的实现跑到了`frameworks\av\services\audioflinger\Threads.cpp`虽然是和`AudioFlinger.cpp`是一个路径，  

```
// PlaybackThread::createTrack_l() must be called with AudioFlinger::mLock held
sp<AudioFlinger::PlaybackThread::Track> AudioFlinger::PlaybackThread::createTrack_l(

```
而sp<AudioFlinger::PlaybackThread::Track> 这个模版函数中的Track又有一个初始化列表，PlaybackThread也有长得像的，但是内容和buffer基本没有关系，
```
// Track constructor must be called with AudioFlinger::mLock and ThreadBase::mLock held
AudioFlinger::PlaybackThread::Track::Track(
            PlaybackThread *thread,
            const sp<Client>& client,
            audio_stream_type_t streamType,
            const audio_attributes_t& attr,
            uint32_t sampleRate,
            audio_format_t format,
            audio_channel_mask_t channelMask,
            size_t frameCount,
            void *buffer,
            size_t bufferSize,
            const sp<IMemory>& sharedBuffer,
            audio_session_t sessionId,
            uid_t uid,
            audio_output_flags_t flags,
            track_type type,
            audio_port_handle_t portId)
    :   TrackBase(thread, client, attr, sampleRate, format, channelMask, frameCount,
                  (sharedBuffer != 0) ? sharedBuffer->pointer() : buffer,
                  (sharedBuffer != 0) ? sharedBuffer->size() : bufferSize,
                  sessionId, uid, true /*isOut*/,
                  (type == TYPE_PATCH) ? ( buffer == NULL ? ALLOC_LOCAL : ALLOC_NONE) : ALLOC_CBLK,
                  type, portId),
    mFillingUpStatus(FS_INVALID),
    // mRetryCount initialized later when needed
    mSharedBuffer(sharedBuffer),
    mStreamType(streamType),
    mName(TRACK_NAME_FAILURE),  // set to TRACK_NAME_PENDING on constructor success.
    mMainBuffer(thread->sinkBuffer()),
    mAuxBuffer(NULL),
    mAuxEffectId(0), mHasVolumeController(false),
    mPresentationCompleteFrames(0),
    mFrameMap(16 /* sink-frame-to-track-frame map memory */),
    mVolumeHandler(new media::VolumeHandler(sampleRate)),
    // mSinkTimestamp
    mFastIndex(-1),
    mCachedVolume(1.0),
    /* The track might not play immediately after being active, similarly as if its volume was 0.
     * When the track starts playing, its volume will be computed. */
    mFinalVolume(0.f),
    mResumeToStopping(false),
    mFlushHwPending(false),
    mFlags(flags)

```

这其中的`TrackBase`似乎有点线索，可以看到这个文件也改变了，`frameworks\av\services\audioflinger\Tracks.cpp`，截取其中部分代码：  

```
    size_t size = sizeof(audio_track_cblk_t);
    if (buffer == NULL && alloc == ALLOC_CBLK) {
        // check overflow when computing allocation size for streaming tracks.
        if (size > SIZE_MAX - bufferSize) {
            android_errorWriteLog(0x534e4554, "34749571");
            return;
        }
        size += bufferSize;
    }

    if (client != 0) {
        mCblkMemory = client->heap()->allocate(size);
        if (mCblkMemory == 0 ||
                (mCblk = static_cast<audio_track_cblk_t *>(mCblkMemory->pointer())) == NULL) {
            ALOGE("not enough memory for AudioTrack size=%zu", size);
            client->heap()->dump("AudioTrack");
            mCblkMemory.clear();
            return;
        }
    } else {
        mCblk = (audio_track_cblk_t *) malloc(size);
        if (mCblk == NULL) {
            ALOGE("not enough memory for AudioTrack size=%zu", size);
            return;
        }
    }
```
只要不出幺蛾子其中的
```
size_t size = sizeof(audio_track_cblk_t);
size += bufferSize;
mCblkMemory = client->heap()->allocate(size);
```
已然是分配好空间了。继续看是如下代码
```
sp<MemoryDealer> AudioFlinger::Client::heap() const
{
    return mMemoryDealer;
}
```

那这个mMemoryDealer又是如何分配的呢？继续搜索好像是在`frameworks/native/libs/binder/MemoryDealer.cpp`这个路径下的文件了，随便找了一下，这样的代码应该还是很好解释了

```
241  sp<IMemory> MemoryDealer::allocate(size_t size)
242  {
243      sp<IMemory> memory;
244      const ssize_t offset = allocator()->allocate(size);
245      if (offset >= 0) {
246          memory = new Allocation(this, heap(), offset, size);
247      }
248      return memory;
249  }
```

那么怎么申请的这个问题就算是回答上了。总的来说就是AudioTrack初始化的时候或者说create的时候就会找AudioFlinger，然后给一个id号，然后通过thread确认之后通过buffersize大小给分配内存，所以说“嘟”的那个铃声大小buffer，是通过如下代码决定的，之前看得都是cblk有关的东西。

```
    input.speed = 1.0;
    if (audio_has_proportional_frames(mFormat) && mSharedBuffer == 0 &&
            (mFlags & AUDIO_OUTPUT_FLAG_FAST) == 0) {
        input.speed  = !isPurePcmData_l() || isOffloadedOrDirect_l() ? 1.0f :
                        max(mMaxRequiredSpeed, mPlaybackRate.mSpeed);
    }
```



##### 谁去用了它，怎么用的它（数据）
思路自然是找start到stop之间调用的函数，然后查看这些函数的流程，既然都说AudioFlinger是audio的实现，那么有没有会用它来联系hal层提供的接口呢？  

线索1：可以看出mOutput实际上就是 audio_io_handle_t 类型，结合上文，估计这个就是拿到工作线程，具体是哪个，还需要打印log  

```
    //mOutput != output includes the case where mOutput == AUDIO_IO_HANDLE_NONE for first creation
    if (mDeviceCallback != 0 && mOutput != output.outputId) {
        if (mOutput != AUDIO_IO_HANDLE_NONE) {
            AudioSystem::removeAudioDeviceCallback(this, mOutput);
        }
        AudioSystem::addAudioDeviceCallback(this, output.outputId);
        callbackAdded = true;
    }
```

log如下：
`fuhua---check mNotificationFramesAct = 192，，input.speed = 1.000000,output.outputId = 13`

思路1：直接查看audioCallback的调用函数，看其中有咩有什么线索。  
我们基本上能知道`mCbf(EVENT_MORE_DATA, mUserData, &audioBuffer)`的逻辑了，第一个参数cmd是执行`ToneGenerator.audioCallback`的依据，而mUserData主要是`ToneGenerator`本身，`&audioBuffer`则只是传递了buffer的特征，整个控制逻辑都是buffer的控制，和谁去用了，怎么用的毫无关联，也和HAL层联系不上，所以可能还需要结合create这样的正常生命周期去看看。  

结合之前看到的PlaybackThread来看，构造函数有传一个audiostreamout的对象  

```
    PlaybackThread(const sp<AudioFlinger>& audioFlinger, AudioStreamOut* output,
                   audio_io_handle_t id, audio_devices_t device, type_t type, bool systemReady);
```

而AudioStreamOut从定义来看是管理HAL层的接口。

```
32  /**
33   * Managed access to a HAL output stream.
34   */
35  class AudioStreamOut {
36  public:
37  // AudioStreamOut is immutable, so its fields are const.
38  // For emphasis, we could also make all pointers to them be "const *",
39  // but that would clutter the code unnecessarily.
40      AudioHwDevice * const audioHwDev;
41      sp<StreamOutHalInterface> stream;
42      const audio_output_flags_t flags;
43  
44      sp<DeviceHalInterface> hwDev() const;
45  
46      AudioStreamOut(AudioHwDevice *dev, audio_output_flags_t flags);
```

其中的AudioHwDevice则是很早之前看过的和HAL层的连接接口。这个直接导致了第二条思路。  

思路2：从hal层入手，找一下是如何给地址或者数据到hal层接口的，毕竟Android只做native及以上的。  
纵览接口，发现只有如下函数可能是关联的，其他的都和control flow关系大。  

```
status_t DeviceHalLocal::openOutputStream(
        audio_io_handle_t handle,
        audio_devices_t devices,
        audio_output_flags_t flags,
        struct audio_config *config,
        const char *address,
        sp<StreamOutHalInterface> *outStream) {
    audio_stream_out_t *halStream;
	ALOGE("fuhua-check-DeviceHalLocal-4.0-hidl---openOutputStream-start");
    ALOGV("open_output_stream handle: %d devices: %x flags: %#x"
            "srate: %d format %#x channels %x address %s",
            handle, devices, flags,
            config->sample_rate, config->format, config->channel_mask,
            address);
    int openResut = mDev->open_output_stream(
            mDev, handle, devices, flags, config, &halStream, address);
    if (openResut == OK) {
        *outStream = new StreamOutHalLocal(halStream, this);
    }
    ALOGV("open_output_stream status %d stream %p", openResut, halStream);
	ALOGE("fuhua-check-DeviceHalLocal-4.0-hidl---openOutputStream-end");
    return openResut;
}

```

其中只有address和outStream可能会藏有data flow的操作。我们找到audio_hw.c去确认这个接口到底用的哪一个或者两个都和数据有关系，发现如下代码  

```
int adev_open_output_stream(struct audio_hw_device *dev,
                            audio_io_handle_t handle,
                            audio_devices_t devices,
                            audio_output_flags_t flags,
                            struct audio_config *config,
                            struct audio_stream_out **stream_out,
                            const char *address __unused)
{
    struct stream_out *out;
    ......
    *stream_out = NULL;
    out = (struct stream_out *)calloc(1, sizeof(struct stream_out));

    ......

    out->stream.set_volume = out_set_volume;
    out->stream.write = out_write;

    ......
```

address 居然被设置成了没用？纵观整个文件，确实没有用到这个变量（其实还不少），那么就只有`*outStream`这一个线索了；在这个函数当中，我们可以看到  

```

static ssize_t out_write(struct audio_stream_out *stream, const void *buffer,
                         size_t bytes)
{
    struct stream_out *out = (struct stream_out *)stream;
    struct audio_device *adev = out->dev;
    ssize_t ret = 0;
    int channels = 0;
    const size_t frame_size = audio_stream_out_frame_size(stream);
    const size_t frames = (frame_size != 0) ? bytes / frame_size : bytes;

    ......

            ALOGVV("%s: writing buffer (%zu bytes) to pcm device", __func__, bytes);

            if (use_mmap)
                ret = pcm_mmap_write(out->pcm, (void *)buffer, bytes_to_write);
            else if (out->hal_op_format != out->hal_ip_format &&
                       out->convert_buffer != NULL) {

                memcpy_by_audio_format(out->convert_buffer,
                                       out->hal_op_format,
                                       buffer,
                                       out->hal_ip_format,
                                       out->config.period_size * out->config.channels);

                ret = pcm_write(out->pcm, out->convert_buffer,
                                 (out->config.period_size *
                                 out->config.channels *
                                 format_to_bitwidth_table[out->hal_op_format]));
            } else {


```

其中就有写pcm数据的函数，而这个`out`就是关键，通过上文，我们可以看到`out`其实就是`stream_out`，也就是接口中的`sp<StreamOutHalInterface> *outStream`。那么native在哪个地方用到了这个接口呢？只有一层一层往上找：
基于之前对audio初始化流程的了解和本来的基础，首先想到的就是`AudioStreamOut.cpp`，果不其然，就在这里建立连接的。

```
status_t AudioStreamOut::open(
        audio_io_handle_t handle,
        audio_devices_t devices,
        struct audio_config *config,
        const char *address)
{
    sp<StreamOutHalInterface> outStream;

    audio_output_flags_t customFlags = (config->format == AUDIO_FORMAT_IEC61937)
                ? (audio_output_flags_t)(flags | AUDIO_OUTPUT_FLAG_IEC958_NONAUDIO)
                : flags;

    int status = hwDev()->openOutputStream(
            handle,
            devices,
            customFlags,
            config,
            address,
            &outStream);
    ALOGV("AudioStreamOut::open(), HAL returned "
            " stream %p, sampleRate %d, Format %#x, "
            "channelMask %#x, status %d",
            outStream.get(),
            config->sample_rate,
            config->format,
            config->channel_mask,
            status);

    ......

}

ssize_t AudioStreamOut::write(const void *buffer, size_t numBytes)
{
    size_t bytesWritten;
    status_t result = stream->write(buffer, numBytes, &bytesWritten);
    return result == OK ? bytesWritten : result;
}


```

那么这个AudioStreamOut是在哪里调用的呢？顺着之前看“嘟”的思路，看AudioFlinger里有一个`openOutput_l`的函数，其中就有一段  

```
    AudioStreamOut *outputStream = NULL;
    status_t status = outHwDev->openOutputStream(
            &outputStream,
            *output,
            devices,
            flags,
            config,
            address.string());

    ......
     if (status == NO_ERROR) {
        if (flags & AUDIO_OUTPUT_FLAG_MMAP_NOIRQ) {
            sp<MmapPlaybackThread> thread =
                    new MmapPlaybackThread(this, *output, outHwDev, outputStream,
                                          devices, AUDIO_DEVICE_NONE, mSystemReady);
            mMmapThreads.add(*output, thread);
            ALOGV("openOutput_l() created mmap playback thread: ID %d thread %p",
                  *output, thread.get());
            return thread;
        } else {
            sp<PlaybackThread> thread;
            if (flags & AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD) {
                thread = new OffloadThread(this, outputStream, *output, devices, mSystemReady);
                ALOGV("openOutput_l() created offload output: ID %d thread %p",
                      *output, thread.get());
            } else if ((flags & AUDIO_OUTPUT_FLAG_DIRECT)
                    || !isValidPcmSinkFormat(config->format)
                    || !isValidPcmSinkChannelMask(config->channel_mask)) {
                thread = new DirectOutputThread(this, outputStream, *output, devices, mSystemReady);
                ALOGV("openOutput_l() created direct output: ID %d thread %p",
                      *output, thread.get());
            } else {
                thread = new MixerThread(this, outputStream, *output, devices, mSystemReady);
                ALOGV("openOutput_l() created mixer output: ID %d thread %p",
                      *output, thread.get());
            }
            mPlaybackThreads.add(*output, thread);
            return thread;
        }
    }

```

而这个outputStream频繁的被这么多的thread调用，很有可能thread.cpp里面有相关内容，而且PlaybackThread也是我们重点关注的对象，  

```
ssize_t AudioFlinger::PlaybackThread::threadLoop_write()
{
        bytesWritten = mOutput->write((char *)mSinkBuffer + offset, mBytesRemaining);
```

果不其然，就是在这里写的，为了验证思路，还是加log比较稳妥，但是从log看，铃声的write不再这threadLoop_write()当中，然后用了笨办法，直接在目录下面搜索,筛选出可能的地方

```
user@user-HP-ProDesk-600-G1-TWR:/local/code/Qcom-code/Gauss/Gauss-P-3H26-2-EU/frameworks/av/services/audioflinger$ grep -rns "\->write"
AudioStreamOut.cpp:198:    status_t result = stream->write(buffer, numBytes, &bytesWritten);
FastCapture.cpp:195:            ssize_t framesWritten = mPipeSink->write(mReadBuffer, mReadBufferState);
BufLog.cpp:88:    return pBLStream->write(buf, size);
FastMixer.cpp:451:            (void) teeSink->write(buffer, frameCount);
FastMixer.cpp:457:        ssize_t framesWritten = mOutputSink->write(buffer, frameCount);
Tracks.cpp:278:        (void) mTeeSink->write(buffer->raw, buffer->frameCount);
Threads.cpp:2860:        ssize_t framesWritten = mNormalSink->write((char *)mSinkBuffer + offset, count);
Threads.cpp:2880:        bytesWritten = mOutput->write((char *)mSinkBuffer + offset, mBytesRemaining);
Threads.cpp:4060:                mOutput->write((char *)mSinkBuffer, 0);
Threads.cpp:6208:        outputTracks[i]->write(mSinkBuffer, writeFrames);
Threads.cpp:6851:            (void) mTeeSink->write((uint8_t*)mRsmpInBuffer + rear * mFrameSize, framesRead);
```

经过log，可以看到有走了如下函数：  

```
01-01 08:29:35.611   621  1125 E AudioFlinger: fuhua---check AudioStreamOut::write result = 0
01-01 08:29:35.611   621  1125 E AudioFlinger: fuhua---check AudioFlinger::MixerThread::threadLoop_write
01-01 08:29:35.614   621  1125 E AudioFlinger: fuhua --- check PlaybackThread::threadLoop_write()---begin
01-01 08:29:35.615   621  1124 E FastMixer: fuhua --- check FastMixer::onWork()
01-01 08:29:35.616   621  1124 E FastMixer: fuhua --- check FastMixer::onWork()
01-01 08:29:35.621   621  1124 E FastMixer: fuhua --- check FastMixer::onWork()
01-01 08:29:35.624   621  1124 E FastMixer: fuhua --- check FastMixer::onWork()
01-01 08:29:35.629   621  1124 E FastMixer: fuhua --- check FastMixer::onWork()
01-01 08:29:35.632   621  1124 E FastMixer: fuhua --- check FastMixer::onWork()
01-01 08:29:35.636   621  1124 E FastMixer: fuhua --- check FastMixer::onWork()
01-01 08:29:35.638   621  1125 E AudioFlinger: fuhua --- check PlaybackThread::threadLoop_write()---mOutput->write in if
01-01 08:29:35.638   621  1125 E AudioFlinger: fuhua --- check PlaybackThread::threadLoop_write()---end

```

大概是在生成波形之后50ms开始走AudioFlinger的吧。到这里基本上理清了数据的调用流程，open的时候建立与hal层的连接，然后通过AudioTrack调用AudioFlinger，AudioFlinger通过Mixer线程，Playback线程，FastMixer线程调用AudioOutputStream接口，然后就是write函数了，这个write函数到后面就是HAL层的write_out函数的实现。所以想起之前有个发问，问的openoutputstream的概念，或许是想问我对这个函数的理解吧，这个是连接native和hal层的audio设备的接口函数，通过此函数将各种操作hal层的接口给连接起来；满足Android对audio的功能需求。   

接下来就是详细分析一下AudioTrack和AudioFlinger还有Mixer的逻辑？也就是怎么用的这段数据？这个放到后面再看，工作上生活上都还有好多事情要做，暂时没这个时间。  

### 如何debug native & hal 层的pcm数据  

其实了解完了这些最重要的就是要提升解决问题的能力和搭建框架的能力，不然只是了解框架，疯狂输出的文章全是科普，一点没有含金量，个人是感觉到了一定程度一点意义都没有。  
那么我们顺手就能在native 层和 hal层加抓取pcm数据的地方，结合QXDM工具，其实就能判断出到底是audio哪一层出现了问题，然后再针对解决。  













