+++
author = "fuhua"
title = "Android-语音唤醒调试log"
date = "2020-09-18 15:09:16"
description = "手机技术笔记"
tags = [
    "markdown",
    "text",
]
weight = 10
draft = true
+++


- [背景](#背景)  
- [正文](#正文)  
    - [Qcom-Speech语音唤醒方案](#1-助听器构造)  
	    - [1:硬件层面](#1)
        - [2:软件](#2)
        - [3:HAL层中的定制](#3)
        - [4:MMITest软件中添加HAC测试的思路](#4)
    - [MTK-Alexa](#2-使用方式)  
    - [HAC部分硬件原理图](#3-硬件原理图)  
    - [硬件调试需求](#4-硬件调试需求)  
    - [软件平台及框架](#5-软件平台及框架)  
        - [1:audio_device.xml中添加hac path](#1)
        - [2:alsa用户空间，tinyalsa中注册HAC控件](#2)
        - [3:HAL层中的定制](#3)
        - [4:MMITest软件中添加HAC测试的思路](#4)
- [结语](#结语)  


## 背景
>耳机部分也是在工作范围内，所以有必要整理，方便回顾和查找。  


## Qcom-Speech语音唤醒方案


### 硬件层面

### 软件log

从main-log看，第一句触发语音识别的log是:`sound_trigger_hw: callback_thread_loop:[2] Received SNDRV_LSM_EVENT_STATUS_V3 status=0`,收到了语音唤醒的call back，LSM是listen stream manager(LSM)

```
01-05 12:29:55.133  1029 18750 I sound_trigger_hw: callback_thread_loop:[2] Received SNDRV_LSM_EVENT_STATUS_V3 status=0
01-05 12:29:55.135  1029 18750 D sound_trigger_hw: callback_thread_loop: params status 2
01-05 12:29:55.136  1029 18750 D sound_trigger_hw: active_state_fn:[2] handle event id 5
01-05 12:29:55.136  1029 18750 V sound_trigger_hw: get_detected_client:[2] single client detection
01-05 12:29:55.137  1029 18750 D sound_trigger_hw: get_detected_client: detected c2
01-05 12:29:55.137  1029 18750 V sound_trigger_hw: active_state_fn:[c2] second stage enabled, list_empty 0,det_requested 0
01-05 12:29:55.138  1029 18750 D sound_trigger_hw: get_first_stage_detection_params:[2] 1st stage kw_start = 0ms, kw_end = 2000ms,is_generic_event 0
01-05 12:29:55.139  1029 18750 D sound_trigger_hw: session[2]: active_state_fn ---> buffering_state_fn
01-05 12:29:55.140  1029 18750 I sound_trigger_hw: callback_thread_loop:[2] Waiting for SNDRV_LSM_EVENT_STATUS_V3
01-05 12:29:55.151  1029 14531 V sound_trigger_hw:ss: buffer_thread_loop: done waiting on cond, exit_buffering = 0
01-05 12:29:55.151  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: done waiting on cond, exit_buffering = 0
01-05 12:29:55.151  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: Enter
01-05 12:29:55.151  1029 14532 V sound_trigger_hw:ss: start_user_verification: Enter
01-05 12:29:55.151  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: waiting on cond, unread_bytes = 7680
01-05 12:29:55.151  1029 14532 V sound_trigger_hw:ss: start_user_verification: Issuing capi_set_param for param 6, uv_conf_score 16.000000
01-05 12:29:55.152  1029 14532 D vop_capi: capi_v2_set_param: Enter: [6]
01-05 12:29:55.152  1029 14532 D vop_capi: capi_v2_set_param: Exit
01-05 12:29:55.152  1029 14532 V sound_trigger_hw:ss: start_user_verification: waiting on cond, unread_bytes = 7680
01-05 12:29:55.171  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: done waiting on cond
01-05 12:29:55.171  1029 18751 D sound_trigger_hw: buffer_thread_loop: FTRT data transfer: 120ms of data received in 10ms
01-05 12:29:55.171  1029 14532 V sound_trigger_hw:ss: start_user_verification: done waiting on cond
01-05 12:29:55.171  1029 18751 D sound_trigger_hw: buffer_thread_loop: First real time frame took 19ms
01-05 12:29:55.171  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: waiting on cond, unread_bytes = 15360
01-05 12:29:55.171  1029 14532 V sound_trigger_hw:ss: start_user_verification: waiting on cond, unread_bytes = 15360
01-05 12:29:55.190  1029 14532 V sound_trigger_hw:ss: start_user_verification: done waiting on cond
01-05 12:29:55.191  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: done waiting on cond
01-05 12:29:55.191  1029 14532 V sound_trigger_hw:ss: start_user_verification: waiting on cond, unread_bytes = 23040
01-05 12:29:55.191  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: waiting on cond, unread_bytes = 23040
01-05 12:29:55.211  1029 14532 V sound_trigger_hw:ss: start_user_verification: done waiting on cond
01-05 12:29:55.211  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: done waiting on cond
01-05 12:29:55.212  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: waiting on cond, unread_bytes = 30720
01-05 12:29:55.212  1029 14532 V sound_trigger_hw:ss: process_frame_user_verification: Issuing capi_process
01-05 12:29:55.213  1029 14532 D sound_trigger_hw:ss: process_frame_user_verification combined_user_score:0.
01-05 12:29:55.214  1029 14532 D sound_trigger_hw:ss: start_user_verification: Processed 480ms of data in 62ms
01-05 12:29:55.214  1029 14532 D sound_trigger_hw:ss: start_user_verification: Detection success, confidence level = 0
01-05 12:29:55.215  1029 14532 V sound_trigger_hw:ss: start_user_verification: Issuing capi_set_param for param 3
01-05 12:29:55.215  1029 14529 V sound_trigger_hw: aggregator_thread_loop: done waiting on cond
01-05 12:29:55.216  1029 14532 D vop_capi: capi_v2_set_param: Enter: [3]
01-05 12:29:55.218  1029 14532 D vop_capi: capi_v2_set_param: Exit
01-05 12:29:55.219  1029 14532 V sound_trigger_hw:ss: start_user_verification: Exiting
01-05 12:29:55.220  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: waiting on cond
01-05 12:29:55.220  1029 14529 V sound_trigger_hw: aggregator_thread_loop: There is a second stage session pending, continuing
01-05 12:29:55.221  1029 14529 V sound_trigger_hw: aggregator_thread_loop: waiting on cond
01-05 12:29:55.230  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: done waiting on cond
01-05 12:29:55.230  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: done waiting on cond, exit_buffering = 1
01-05 12:29:55.231  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: waiting on cond, unread_bytes = 38400
01-05 12:29:55.231  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: waiting on cond
01-05 12:29:55.251  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: done waiting on cond
01-05 12:29:55.251  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: done waiting on cond, exit_buffering = 1
01-05 12:29:55.251  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: waiting on cond
01-05 12:29:55.251  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: waiting on cond, unread_bytes = 46080
01-05 12:29:55.271  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: done waiting on cond
01-05 12:29:55.271  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: done waiting on cond, exit_buffering = 1
01-05 12:29:55.271  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: waiting on cond, unread_bytes = 53760
01-05 12:29:55.271  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: waiting on cond
01-05 12:29:55.291  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: done waiting on cond, exit_buffering = 1
01-05 12:29:55.291  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: done waiting on cond
01-05 12:29:55.291  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: waiting on cond
01-05 12:29:55.291  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: waiting on cond, unread_bytes = 61440
01-05 12:29:55.310  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: done waiting on cond, exit_buffering = 1
01-05 12:29:55.311  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: done waiting on cond
01-05 12:29:55.311  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: waiting on cond, unread_bytes = 69120
01-05 12:29:55.311  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: waiting on cond
01-05 12:29:55.331  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: done waiting on cond
01-05 12:29:55.331  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: done waiting on cond, exit_buffering = 1
01-05 12:29:55.332  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: waiting on cond, unread_bytes = 76800
01-05 12:29:55.332  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: waiting on cond
01-05 12:29:55.351  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: done waiting on cond
01-05 12:29:55.351  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: done waiting on cond, exit_buffering = 1
01-05 12:29:55.352  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: waiting on cond
01-05 12:29:55.352  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: waiting on cond, unread_bytes = 84480
01-05 12:29:55.370  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: done waiting on cond, exit_buffering = 1
01-05 12:29:55.371  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: waiting on cond
01-05 12:29:55.372  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: done waiting on cond
01-05 12:29:55.373  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: waiting on cond, unread_bytes = 92160
01-05 12:29:55.390  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: done waiting on cond
01-05 12:29:55.390  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: done waiting on cond, exit_buffering = 1
01-05 12:29:55.390  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: waiting on cond, unread_bytes = 99840
01-05 12:29:55.390  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: waiting on cond

01-05 12:29:55.435  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: done waiting on cond, exit_buffering = 1
01-05 12:29:55.436  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: done waiting on cond
01-05 12:29:55.436  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: waiting on cond
01-05 12:29:55.436  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: waiting on cond, unread_bytes = 115200
01-05 12:29:55.451  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: done waiting on cond
01-05 12:29:55.451  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: done waiting on cond, exit_buffering = 1
01-05 12:29:55.451  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: waiting on cond
01-05 12:29:55.451  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: waiting on cond, unread_bytes = 122880
01-05 12:29:55.471  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: done waiting on cond
01-05 12:29:55.471  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: done waiting on cond, exit_buffering = 1
01-05 12:29:55.471  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: waiting on cond
01-05 12:29:55.472  1029 14531 D sound_trigger_hw:ss: keyword_data_buffer index is 0
01-05 12:29:55.472  1029 14531 V sound_trigger_hw:ss: process_frame_keyword_detection: Issuing capi_process
01-05 12:29:55.473  1029 14531 D sound_trigger_hw:ss: start_keyword_detection: Processed 2000ms of data in 321ms
01-05 12:29:55.473  1029 14531 D sound_trigger_hw:ss: start_keyword_detection: Detection success, confidence level = 0
01-05 12:29:55.473  1029 14531 D sound_trigger_hw:ss: start_keyword_detection: 2nd stage kw_start = 0ms, kw_end = 0ms
01-05 12:29:55.473  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: Issuing capi_set_param for param 5
01-05 12:29:55.473  1029 14529 V sound_trigger_hw: aggregator_thread_loop: done waiting on cond
01-05 12:29:55.474  1029 14531 D pdk_capi_wrapper: capi_v2_set_param: Enter: [5]
01-05 12:29:55.474  1029 14531 D pdk_capi_wrapper: capi_v2_set_param: Exit
01-05 12:29:55.475  1029 14531 V sound_trigger_hw:ss: start_keyword_detection: Exiting
01-05 12:29:55.476  1029 14531 V sound_trigger_hw:ss: buffer_thread_loop: waiting on cond
01-05 12:29:55.476  1029 14529 V sound_trigger_hw: check_and_extract_det_conf_levels_payload:[2] not merged
01-05 12:29:55.477  1029 14529 V sound_trigger_hw: process_detection_event_keyphrase: [0] kw_id 0 level 0
01-05 12:29:55.478  1029 14529 V sound_trigger_hw: process_detection_event_keyphrase: [0] user_id 1 level 0
01-05 12:29:55.478  1029 14529 I sound_trigger_hw: process_detection_event_keyphrase:[c2]
01-05 12:29:55.478  1029 14529 V sound_trigger_hw: process_detection_event_keyphrase:[c2] status=0, type=0, model=2, capture_avaiable=1, num_phrases=1 id=0
01-05 12:29:55.478  1029 14529 D sound_trigger_hw: aggregator_thread_loop:[c2] Second stage detected successfully, calling client callback
01-05 12:29:55.479  1029 14529 D sound_trigger_hw: aggregator_thread_loop: Total sthal processing time: 342ms
01-05 12:29:55.490  1029 14531 V sound_trigger_hw:ss: buffer_thread_loop: done waiting on cond, exit_buffering = 1
01-05 12:29:55.490  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: done waiting on cond, exit_buffering = 1
01-05 12:29:55.490  1029 14531 V sound_trigger_hw:ss: buffer_thread_loop: waiting on cond
01-05 12:29:55.490  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: waiting on cond
```

从Qcom的设计方案来看，
首先codec中有MAD(Mic Activity Detection)一直在跑，收集PCM samp，判断声压是否达标，然后给到SPE（SVA Process Engine）/CPE（Codec Process Engine）判断是否和keyword detach或者判断是否和soundModel匹配

在Ultra-Low-Power-Mode就匹配就通知HLOS中的sound triger driver,并且把MAD又打开，持续监听输入的信息。
在Low-Power-Mode中就通知LSM,然后给到ST(Sound Trigger)
在No codec模式中，MAD跑在LPASS(Low-Power-Audio-sub-System)中，其他逻辑同上。


如果是barge-in场景，当有audio-playback的时候，voice detection processing 从WDSP迁移到ADSP中进行；有LSM通知HLOS, 然后通知到JAVA层，由listen的服务发送wake up的广播唤醒系统，比如OVM/Qcom文档中写的是com.qualcomm.listen.ListenSoundModel  class

低功耗唤醒方案存在的问题是1级唤醒的准确率很低，所以有了2级唤醒，一般2级唤醒里面有AEC，然后是三级唤醒，即声纹识别。

所以给到了ST，经过`sound_trigger_hw:ss: start_keyword_detection: Enter`以及`sound_trigger_hw: aggregator_thread_loop:[c2] Second stage detected successfully, calling client callback`可以看到是经过了第二阶段了的。

接下来就是第三阶段OVMS的工作了
```
OVMS-WakeupService: onRecognition:
01-05 12:29:55.492  5037  5037 I OVMS-WakeupService: getVprintType(): vprintType 2
01-05 12:29:55.493  5037  5037 D OVMS-ExtendedSmMgr: getSmNameByHandle: smHandle = 1 smName = default.udm
01-05 12:29:55.493  5037  5037 I OVMS-WakeupService: acquireCPULock()
01-05 12:29:55.496   738   738 I android.system.suspend@1.0-service: mSuspendCounter=1 first holding wakelock name=PowerManagerService.WakeLocks: Success
01-05 12:29:55.498  5037  5037 W ContextImpl: Calling a method in the system process without a qualified user: android.app.ContextImpl.sendBroadcast:1147 android.content.ContextWrapper.sendBroadcast:478 com.oppo.ovoicemanager.midware.qcom.utils.Utils.sendFirstStageWakeupIntent:106 com.oppo.ovoicemanager.midware.qcom.service.WakeupService.onRecognition:761 com.oppo.ovoicemanager.midware.qcom.session.legacy.LegacyWakeupSession.onRecognition:215
01-05 12:29:55.499   990  3104 V ActivityManager: Broadcast: Intent { act=com.oppo.ovoicemanagertest.TEST_O_VOICE_MANAGER_SERVICE_FIRST pkg=null } ordered=false userid=0 resultTo null
01-05 12:29:55.499   990  3104 V ActivityManager: broadcastIntentLocked callingPid: 5037 callingUid=1000
01-05 12:29:55.502  5037  5037 V OVMS-OVSStatisticsUtils: onCommon eventID = 20169025, eventMap = {time=1609820995501}
01-05 12:29:55.502  5037  5037 D OVMS-WakeupService: RecognitionEvent.data length:133648, VOICE_DATA_LENGTH:128000
01-05 12:29:55.505  5037  5037 V OVMS-Utils: writeBufferToFile: stream created bufferSize 127968 filePath:/sdcard/Documents/OVMS/OVMS_Aispeech/requestAudio/st-2021-01-05-12-29-55-first.pcm
01-05 12:29:55.508 11111 11159 I [0]DCS-StrategyManager: isNeedUpload: cache, type: 2006, appID: 20169, logTag: 2016901, eventID: 20169025
01-05 12:29:55.513  5037  5037 V OVMS-Utils: writeBufferToFile: write false samples
01-05 12:29:55.515  5037  5037 V OVMS-Utils: writeBufferToFile: complete
01-05 12:29:55.516  5037  5037 V OVMS-Utils: writeBufferToFile: stream close
01-05 12:29:55.516  5037  5037 D OVMS-WakeupService: WakeupWords is open.
01-05 12:29:55.517  5037  5037 D OVMS-WakeupService: processRecognitionEvent: event = KeyphraseRecognitionEvent [keyphraseExtras=[KeyphraseRecognitionExtra [id=0, recognitionModes=3, coarseConfidenceLevel=0, confidenceLevels=[ConfidenceLevel [userId=1, confidenceLevel=0]]]], status=0, soundModelHandle=1, captureAvailable=true, captureSession=177, captureDelayMs=0, capturePreambleMs=0, triggerInData=false, sampleRate=16000, encoding=2, channelMask=12, data=133648] smName = default.udm
01-05 12:29:55.517  5037  5037 D OVMS-ExtendedSmMgr: getSoundModel: smFullName = default.udm
01-05 12:29:55.517  5037  5037 D OVMS-WakeupService: processRecognitionEvent4ThirdParty
01-05 12:29:55.518  5037  5037 D OVMS-ExtendedSmMgr: getSoundModel: smFullName = default.udm
01-05 12:29:55.518  5037  5037 I OVMS-WakeupService: recognize start.
01-05 12:29:55.519  5037  5037 I OVMS-WakeupService: recognize end. PCM file size:127968
01-05 12:29:55.520  5037 11909 D FespxKernel: feed wakeup data size: 127968
01-05 12:29:55.521  5037 11947 V duilite : wakeup & rollback
01-05 12:29:55.524  5037 11909 D FespxKernel: fespx stop begin
01-05 12:29:55.524  5037 11948 D FespxKernel: beamforming_callback wakeup_type return : {"wakeup_type": 2}
01-05 12:29:55.590  1029 14531 V sound_trigger_hw:ss: buffer_thread_loop: done waiting on cond, exit_buffering = 1
01-05 12:29:55.590  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: done waiting on cond, exit_buffering = 1
01-05 12:29:55.590  1029 14532 V sound_trigger_hw:ss: buffer_thread_loop: waiting on cond
01-05 12:29:55.590  1029 14531 V sound_trigger_hw:ss: buffer_thread_loop: waiting on cond
01-05 12:29:55.663  5037 11943 D FespxKernel: wakeup_callback return : {"wakeupWord":"ni hao xiao bu","major":0,"status":1,"boundary":0,"confidence":0.187653,"frmInc":256, "wkpIndex":115}
01-05 12:29:55.664  5037 11943 D FespxKernel: real wakeup
01-05 12:29:55.664  5037 11943 D FespxKernel: first wkp cb end
01-05 12:29:55.664  5037 11900 D FespxProcessor: >>>>>>Event: MSG_RESULT
01-05 12:29:55.664  5037 11900 D FespxProcessor: [Current]:STATE_RUNNING
01-05 12:29:55.665  5037 11900 W FespxProcessor: upload disable or new wakeup happens within 500ms, ignore
01-05 12:29:55.665  5037  5037 D AILocalWkpAndVpEngine: wakeup ni hao xiao bu confidence0.187653
01-05 12:29:55.665  5037  5037 D OVMS-OVMS_AispeechSdkInstance: mAILocalWakeupAndVprintListener onResults: aiResult = {"status":1,"wakeupWord":"ni hao xiao bu","confidence":0.187653}
01-05 12:29:55.666  5037  5037 I OVMS-WakeupService: AILocalWakeupAndVprintListener.onResults() called.
01-05 12:29:55.666  5037  5037 I OVMS-WakeupService: procSecondRecognitionResult: aiResult JSON = {"status":1,"wakeupWord":"ni hao xiao bu","confidence":0.187653}
01-05 12:29:55.666  5037  5037 I OVMS-WakeupService: ORMS accelerateCPU()
01-05 12:29:55.667  5037  5037 I OVMS-WakeupService: procSecondRecognitionResult: aiResultJson not has 'result'.
01-05 12:29:55.791  5037  5037 D OVMS-OVMS_StartSpeechAssist: isKeyguardLocked started
01-05 12:29:55.791  5037  5037 D OVMS-OVMS_StartSpeechAssist: isKeyguardLocked finished
01-05 12:29:55.791  5037  5037 D OVMS-OVMS_StartSpeechAssist: getIntent started
01-05 12:29:55.793  5037  5037 D OVMS-OVMS_StartSpeechAssist: getIntent finished
01-05 12:29:55.793  5037  5037 D OVMS-OVMS_StartSpeechAssist: wakeUpSpeechAssist caller_package: com.oppo.ovoicemanager
01-05 12:29:55.793  5037  5037 I OVMS-OVMS_StartSpeechAssist: wakeupCostTime:  300
01-05 12:29:55.793  5037  5037 V OVMS-OVSStatisticsUtils: onCommon eventID = 20169036, eventMap = {wakeUpCostTime=300}
01-05 12:29:55.794  5037  5037 D OVMS-OVMS_StartSpeechAssist: start_speechassist:1609820995793
01-05 12:29:55.794  5037  5037 I OVMS-OVMS_StartSpeechAssist: startSpeechAssistActivity started, session_ID: 177
```
