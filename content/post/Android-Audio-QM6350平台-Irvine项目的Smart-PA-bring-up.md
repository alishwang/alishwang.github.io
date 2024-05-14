+++
author = "fuhua"
title = "Android - Audio - QM6350平台 - Irvine项目的Smart PA bring-up"
date = "2020-09-18 15:09:16"
description = "手机技术笔记"
tags = [
    "markdown",
    "text",
]
weight = 10
+++


>好久没写blog了，一是因为事情确实多，二是自己安排有点不对，三是加班太多状态也不好，最主要的是这些都是借口，还不是因为懒。

## 背景
Irvine项目需要上SmartPA，毕竟是6系列了，不加个好点的音效肯定不符合中端机的定位。当然在国内的配置就是中低端了。

## 确认原理图以及硬件接口
画图的同事很随意，所以有时候还是要确认；
比如他写的 `I2C address:0x68(W) 0x69(R)`，还需要自己移位才算出来是0x34，一眼找不到0x34就很迷惑；当然也有可能是思路不一样。

简单画一下：
```
+-------------------------+
|  AW88194                |
|                         |
|  +-----------+          +         +----------------+
|  |   SDL1    | +--NFC_I2S_SCL>--> |  NFC_I2S_SCL   |
|  +-----------+          +    |    +----------------+
|                         |    |
|  +-----------+          |    |
|  |   SDL2    +---------------+
|  +-----------+          |
|                         |
|  +-----------+          |         +----------------+
|  |   SDA     +----NFC_I2S_SDA---> |  NFC_I2C_SDA   |
|  +-----------+          |         +----------------+
|                         |
|  +-----------+          |
|  |   AD1     +---------------+-----+
|  +-----------+          |    ^     |
|                         |    |     |
|  +-----------+          |    |     v
|  |   AD2     +---------------+    GND
|  +-----------+          |
|                         |
|  +-----------+          |         +---------------+
|  |   BCK     +---------------+--> |  AUDIO_PA_BCK |
|  +-----------+          |    |    +---------------+
|                         |    |
|  +-----------+          |    +-------------------------------------+TP5001
|  |   WCK1    +<-------------+----+
|  +-----------+          |   ^ |  +----------------+
|                         |   | |  ++ AUDIO_PA_LRCK |
|  +-----------+          |   | |   +---------------+
|  |   WCK2    +--------------+ |
|  +-----------+          |     +------------------------------------+TP5002
|                         |
|  +-----------+          |         +---------------+
|  |  DATAI    +<--------------+----+ AUDIO_PA_DATAI|
|  +-----------+          |    |    +---------------+
|                         |    +-------------------------------------+TP5003
|  +-----------+          |         +---------------+
|  |  DATAO    +---------------+--->+ AUDIO_PA_DATAO|
|  +-----------+          |    |    +---------------+
|                         |    +-------------------------------------+TP5004
|  +-----------+          |         +---------------+
|  |  RSTN     +<-------------------+  SPK_PA_RST1  |
|  +-----------+          |         +---------------+
|                         |
|  +-----------+          |         +---------------+
|  |  INTN     +---------------^--->+ SPK_PA_INT1   |
|  +-----------+          |    |    +---------------+
|                         |    |
+-------------------------+    |    +---------------+
                               +----+  VDD_VIO_1P8V |
                                    +---------------+


```
由于和NFC共用了I2C，所以硬件同事直接标注了NFC的
原理图理清楚了之后，就是拿到实际硬件将调试线接出来，I2S的4条线
再一个，调试之前要确认声卡已经bring up了，不然问题更多。
## dtsi设备树部分
[qualcomm/platform/vendor/qcom/proprietary/devicetree-4.19.git] / qcom / lito-irvine-audio-overlay.dtsi
我是单独加了这样一个文件，在dtsi的overlay的最后再添加所有audio相关的overlay，方便集中管理和调试；如果单纯porting smart PA，则需要添加如下comb
```
  96 &qupv3_se8_i2c {
  97         status = "ok";
  98         aw881xx_smartpa@34 {
  99                 compatible = "awinic,aw881xx_smartpa";
 100                 reg = <0x34>;
 101                 reset-gpio = <&tlmm 12 0>;
 102                 irq-gpio = <&tlmm 11 0>;
 103                 monitor-flag = <1>;
 104                 monitor-timer-val = <30000>;
 105                 status = "okay";
 106         };
 107 };
```

## kernel部分
#####1. 针对AW的驱动和业务，在对应的配置文件中添加声明和定义
[qualcomm/kernel/msm-4.19.git] / arch / arm64 / configs / vendor / Irvine_defconfig
[qualcomm/kernel/msm-4.19.git] / arch / arm64 / configs / vendor / Irvine-perf_defconfig

```
+#begin mod by fuhua for task_id: 9660534  on 2020-07-22
+#CONFIG_SND_SOC_TFA98XX=y
+#CONFIG_SND_SOC_TFA98XX_MMI_TEST=y
+#CONFIG_SND_SMARTPA_AW881XX=y
+CONFIG_SND_SMARTPA_AW881XX_IRVINE=y
+CONFIG_TCT_SND_WSA_CNT=y
```

[qualcomm/kernel/msm-4.19.git] / sound / soc / codecs / Kconfig
```
+# begin mod by fuhua.wang for task 9660534 on 2020.01.19
+config SND_SMARTPA_AW881XX_IRVINE
+    tristate "Soc Audio for awinic a881xx series"
+    depends on I2C
+    help
+        This option enables support for aw881xx series Smart PA
+#end   mod by fuhua.wang for task 9660534 on 2020.01.19
```

#####2. 单独创建编译目录，方便兼容和管理。
[qualcomm/kernel/msm-4.19.git] / sound / soc / codecs / Makefile

```
+#begin mod by fuhua.wang for task 9554561 on 2020.07.10
+obj-$(CONFIG_SND_SMARTPA_AW881XX_IRVINE)  += aw881xx_irvine/aw881xx.o aw881xx_irvine/aw881xx_monitor.o aw881xx_irvine/aw881xx_cali.o
+#end   mod by fuhua.wang for task 9554561 on 2020.07.10
```

#####3. 剩下的就是将驱动放到对应目录下，当然驱动可以自己写，只是没必要。
```
sound/soc/codecs/aw881xx_irvine/aw881xx.c	[new file with mode: 0644]	blob
sound/soc/codecs/aw881xx_irvine/aw881xx.h	[new file with mode: 0644]	blob
sound/soc/codecs/aw881xx_irvine/aw881xx_cali.c	[new file with mode: 0644]	blob
sound/soc/codecs/aw881xx_irvine/aw881xx_cali.h	[new file with mode: 0644]	blob
sound/soc/codecs/aw881xx_irvine/aw881xx_monitor.c	[new file with mode: 0644]	blob
sound/soc/codecs/aw881xx_irvine/aw881xx_monitor.h	[new file with mode: 0644]	blob
sound/soc/codecs/aw881xx_irvine/aw881xx_reg.h	[new file with mode: 0644]	blob
```


## openkernel部分
1. 在Kona.c里面增加对应的dai。
[qualcomm/platform/vendor/opensource/audio-kernel.git] / asoc / kona.c

```
@@ -6879,6 +6879,8 @@ static struct snd_soc_dai_link msm_common_be_dai_links[] = {
                .be_hw_params_fixup = msm_be_hw_params_fixup,
                .ignore_suspend = 1,
        },
+/* begin mod by fuhua.wang task_id: 9660534 2020-07-18*/
+#ifndef CONFIG_SND_SMARTPA_AW881XX_IRVINE
 #ifndef CONFIG_TCT_SND_WSA_CNT // MODIFIED by hongwei.tian, 2020-07-28,BUG-9672091
        {
                .name = LPASS_BE_WSA_CDC_DMA_TX_0_VI,
@@ -6895,6 +6897,8 @@ static struct snd_soc_dai_link msm_common_be_dai_links[] = {
                .ops = &msm_cdc_dma_be_ops,
        },
 #endif // MODIFIED by hongwei.tian, 2020-07-28,BUG-9672091
+#endif
+/* end  mod by fuhua.wang task_id: 9660534 2020-07-18*/
 };
 
 
@@ -7327,8 +7331,15 @@ static struct snd_soc_dai_link msm_mi2s_be_dai_links[] = {
                .stream_name = "Quinary MI2S Playback",
                .cpu_dai_name = "msm-dai-q6-mi2s.4",
                .platform_name = "msm-pcm-routing",
-               .codec_name = "msm-stub-codec.1",
-               .codec_dai_name = "msm-stub-rx",
+/* begin mod by fuhua.wang task_id: 9660534 2020-07-18*/
+#ifdef CONFIG_SND_SMARTPA_AW881XX_IRVINE
+               .codec_name = "aw881xx_smartpa",
+               .codec_dai_name = "aw881xx-aif",
+#else
+        .codec_name = "msm-stub-codec.1",
+        .codec_dai_name = "msm-stub-rx",
+#endif
+/* end  mod by fuhua.wang task_id: 9660534 2020-07-18*/
                .no_pcm = 1,
                .dpcm_playback = 1,
                .id = MSM_BACKEND_DAI_QUINARY_MI2S_RX,
```



## mixer_path部分
或者说用户空间部分，其实用tinymix能调试出声音，就说明驱动差不多了，但可能会有一些要查漏补缺的地方。
[qualcomm/platform/hardware/qcom/audio.git] / configs / lito / mixer_paths_irvine.xml
这个就不全贴代码了，因为太多了，只能针对spk简单贴点
```
 525     <path name="deep-buffer-playback speaker">
 526         <ctl name="QUIN_MI2S_RX Audio Mixer MultiMedia1" value="1" />
 527         <ctl name="devkit Profile" value="hq" />
 528         <ctl name="aw881xx_mode_switch" value="spk" />
 529     </path>
 658     <path name="low-latency-playback">
 659         <!-- mod by fuhua for task id : 9554561 on 2020-08-11 -->
 660         <!--<ctl name="WSA_CDC_DMA_RX_0 Audio Mixer MultiMedia5" value="1" />-->
 661         <ctl name="QUIN_MI2S_RX Audio Mixer MultiMedia5" value="1" />
 662         <!--<ctl name="aw881xx_mode_switch" value="spk" />-->
 663     </path>
 664     <path name="low-latency-playback speaker">
 665         <ctl name="QUIN_MI2S_RX Audio Mixer MultiMedia5" value="1" />
 666     </path>
 667     <path name="low-latency-playback voice-speaker">
 668         <ctl name="QUIN_MI2S_RX Audio Mixer MultiMedia5" value="1" />
 669     </path>
```

## 固件部分
[qualcomm/platform/vendor/qcom/lito.git] / Irvine.mk
```
 PRODUCT_PROPERTY_OVERRIDES += ro.logd.auditd.dmesg=false
 
-PRODUCT_COPY_FILES += \
-    device/qcom/lito/Irvine/aw8695/aw8695_haptic.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw8695_haptic.bin \
-    device/qcom/lito/Irvine/aw8695/aw8695_osc_rtp_24K_5s.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw8695_osc_rtp_24K_5s.bin
+#begin mod by fuhua.wang for task 9660534 on 2020.07.10
+
+# PRODUCT_COPY_FILES += \
+#     device/qcom/lito/Irvine/aw8695/aw8695_haptic.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw8695_haptic.bin \
+#     device/qcom/lito/Irvine/aw8695/aw8695_osc_rtp_24K_5s.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw8695_osc_rtp_24K_5s.bin
+
+# PRODUCT_COPY_FILES += \
+#     device/qcom/lito/Irvine/aw881xx/aw881xx_pid_01_fm_cfg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_01_fm_cfg.bin \
+#     device/qcom/lito/Irvine/aw881xx/aw881xx_pid_01_fm_fw.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_01_fm_fw.bin \
+#     device/qcom/lito/Irvine/aw881xx/aw881xx_pid_01_rcv_cfg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_01_rcv_cfg.bin \
+#     device/qcom/lito/Irvine/aw881xx/aw881xx_pid_01_rcv_fw.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_01_rcv_fw.bin \
+#     device/qcom/lito/Irvine/aw881xx/aw881xx_pid_01_spk_cfg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_01_spk_cfg.bin \
+#     device/qcom/lito/Irvine/aw881xx/aw881xx_pid_01_spk_fw.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_01_spk_fw.bin \
+#     device/qcom/lito/Irvine/aw881xx/aw881xx_pid_01_voice_cfg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_01_voice_cfg.bin \
+#     device/qcom/lito/Irvine/aw881xx/aw881xx_pid_01_voice_fw.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_01_voice_fw.bin \
+#     device/qcom/lito/Irvine/aw881xx/aw881xx_pid_03_fm_cfg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_03_fm_cfg.bin \
+#     device/qcom/lito/Irvine/aw881xx/aw881xx_pid_03_fm_fw.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_03_fm_fw.bin \
+#     device/qcom/lito/Irvine/aw881xx/aw881xx_pid_03_rcv_cfg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_03_rcv_cfg.bin \
+#     device/qcom/lito/Irvine/aw881xx/aw881xx_pid_03_rcv_fw.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_03_rcv_fw.bin \
+#     device/qcom/lito/Irvine/aw881xx/aw881xx_pid_03_spk_cfg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_03_spk_cfg.bin \
+#     device/qcom/lito/Irvine/aw881xx/aw881xx_pid_03_spk_fw.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_03_spk_fw.bin \
+#     device/qcom/lito/Irvine/aw881xx/aw881xx_pid_03_voice_cfg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_03_voice_cfg.bin \
+#     device/qcom/lito/Irvine/aw881xx/aw881xx_pid_03_voice_fw.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_03_voice_fw.bin \
+#     device/qcom/lito/Irvine/aw881xx/aw881xx_pid_xx_fm_reg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_xx_fm_reg.bin \
+#     device/qcom/lito/Irvine/aw881xx/aw881xx_pid_xx_rcv_reg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_xx_rcv_reg.bin \
+#     device/qcom/lito/Irvine/aw881xx/aw881xx_pid_xx_spk_reg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_xx_spk_reg.bin \
+#     device/qcom/lito/Irvine/aw881xx/aw881xx_pid_xx_voice_reg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_xx_voice_reg.bin
+
+PRODUCT_COPY_FILES += device/qcom/lito/Irvine/aw_smartpa_firmware_irvine/aw881xx_pid_01_rcv_cfg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_01_rcv_cfg.bin
+PRODUCT_COPY_FILES += device/qcom/lito/Irvine/aw_smartpa_firmware_irvine/aw881xx_pid_01_rcv_fw.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_01_rcv_fw.bin
+PRODUCT_COPY_FILES += device/qcom/lito/Irvine/aw_smartpa_firmware_irvine/aw881xx_pid_01_spk_cfg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_01_spk_cfg.bin
+PRODUCT_COPY_FILES += device/qcom/lito/Irvine/aw_smartpa_firmware_irvine/aw881xx_pid_01_spk_fw.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_01_spk_fw.bin
+PRODUCT_COPY_FILES += device/qcom/lito/Irvine/aw_smartpa_firmware_irvine/aw881xx_pid_03_rcv_cfg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_03_rcv_cfg.bin
+PRODUCT_COPY_FILES += device/qcom/lito/Irvine/aw_smartpa_firmware_irvine/aw881xx_pid_03_rcv_fw.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_03_rcv_fw.bin
+PRODUCT_COPY_FILES += device/qcom/lito/Irvine/aw_smartpa_firmware_irvine/aw881xx_pid_03_spk_cfg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_03_spk_cfg.bin
+PRODUCT_COPY_FILES += device/qcom/lito/Irvine/aw_smartpa_firmware_irvine/aw881xx_pid_03_spk_fw.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_03_spk_fw.bin
+PRODUCT_COPY_FILES += device/qcom/lito/Irvine/aw_smartpa_firmware_irvine/aw881xx_pid_xx_rcv_reg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_xx_rcv_reg.bin
+PRODUCT_COPY_FILES += device/qcom/lito/Irvine/aw_smartpa_firmware_irvine/aw881xx_pid_xx_spk_reg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_xx_spk_reg.bin
+#end   mod by fuhua.wang for task 9660534 on 2020.07.10
 
-PRODUCT_COPY_FILES += \
-    device/qcom/lito/Irvine/aw881xx/aw881xx_pid_01_fm_cfg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_01_fm_cfg.bin \
-    device/qcom/lito/Irvine/aw881xx/aw881xx_pid_01_fm_fw.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_01_fm_fw.bin \
-    device/qcom/lito/Irvine/aw881xx/aw881xx_pid_01_rcv_cfg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_01_rcv_cfg.bin \
-    device/qcom/lito/Irvine/aw881xx/aw881xx_pid_01_rcv_fw.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_01_rcv_fw.bin \
-    device/qcom/lito/Irvine/aw881xx/aw881xx_pid_01_spk_cfg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_01_spk_cfg.bin \
-    device/qcom/lito/Irvine/aw881xx/aw881xx_pid_01_spk_fw.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_01_spk_fw.bin \
-    device/qcom/lito/Irvine/aw881xx/aw881xx_pid_01_voice_cfg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_01_voice_cfg.bin \
-    device/qcom/lito/Irvine/aw881xx/aw881xx_pid_01_voice_fw.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_01_voice_fw.bin \
-    device/qcom/lito/Irvine/aw881xx/aw881xx_pid_03_fm_cfg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_03_fm_cfg.bin \
-    device/qcom/lito/Irvine/aw881xx/aw881xx_pid_03_fm_fw.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_03_fm_fw.bin \
-    device/qcom/lito/Irvine/aw881xx/aw881xx_pid_03_rcv_cfg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_03_rcv_cfg.bin \
-    device/qcom/lito/Irvine/aw881xx/aw881xx_pid_03_rcv_fw.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_03_rcv_fw.bin \
-    device/qcom/lito/Irvine/aw881xx/aw881xx_pid_03_spk_cfg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_03_spk_cfg.bin \
-    device/qcom/lito/Irvine/aw881xx/aw881xx_pid_03_spk_fw.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_03_spk_fw.bin \
-    device/qcom/lito/Irvine/aw881xx/aw881xx_pid_03_voice_cfg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_03_voice_cfg.bin \
-    device/qcom/lito/Irvine/aw881xx/aw881xx_pid_03_voice_fw.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_03_voice_fw.bin \
-    device/qcom/lito/Irvine/aw881xx/aw881xx_pid_xx_fm_reg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_xx_fm_reg.bin \
-    device/qcom/lito/Irvine/aw881xx/aw881xx_pid_xx_rcv_reg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_xx_rcv_reg.bin \
-    device/qcom/lito/Irvine/aw881xx/aw881xx_pid_xx_spk_reg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_xx_spk_reg.bin \
-    device/qcom/lito/Irvine/aw881xx/aw881xx_pid_xx_voice_reg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_xx_voice_reg.bin
 
 ifeq ($(TARGET_BUILD_MMITEST),true)
 PRODUCT_PROPERTY_OVERRIDES += persist.radio.multisim.config=dsds
```
该用的先用，如无必要勿增实体

然后就将bin文件拷贝到当前目录下：
```
Irvine/aw_smartpa_firmware_irvine/aw881xx_pid_01_rcv_cfg.bin	[new file with mode: 0644]	blob
Irvine/aw_smartpa_firmware_irvine/aw881xx_pid_01_rcv_fw.bin	[new file with mode: 0644]	blob
Irvine/aw_smartpa_firmware_irvine/aw881xx_pid_01_spk_cfg.bin	[new file with mode: 0644]	blob
Irvine/aw_smartpa_firmware_irvine/aw881xx_pid_01_spk_fw.bin	[new file with mode: 0644]	blob
Irvine/aw_smartpa_firmware_irvine/aw881xx_pid_03_rcv_cfg.bin	[new file with mode: 0644]	blob
Irvine/aw_smartpa_firmware_irvine/aw881xx_pid_03_rcv_fw.bin	[new file with mode: 0644]	blob
Irvine/aw_smartpa_firmware_irvine/aw881xx_pid_03_spk_cfg.bin	[new file with mode: 0644]	blob
Irvine/aw_smartpa_firmware_irvine/aw881xx_pid_03_spk_fw.bin	[new file with mode: 0644]	blob
Irvine/aw_smartpa_firmware_irvine/aw881xx_pid_xx_rcv_reg.bin	[new file with mode: 0644]	blob
Irvine/aw_smartpa_firmware_irvine/aw881xx_pid_xx_spk_reg.bin	[new file with mode: 0644]	blob
Irvine/aw_smartpa_firmware_irvine/sktPreset.sat	[new file with mode: 0644]	blob
```
## 算法部分
这部分加在adsp里面，
[qualcomm/SPF2020/amss_6350_spf1.0.git] / ADSP.VT.5.6 / adsp_proc / hap / oem / build / bitra / custom_libs_cfg.json

```

ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_audio/build/fvsam_audio.scons	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_audio/fvsam_audio_wrp/build/avs_libs/qdsp6/bitra.adsp.prod/fvsam_audio_wrp.lib	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_audio/fvsam_audio_wrp/build/avs_libs/qdsp6/bitra.adsp.prod/src/fvsam_audio.o	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_audio/fvsam_audio_wrp/build/fvsam_audio_wrp.scons	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_audio/fvsam_audio_wrp/inc/fvsam_audio.h	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_audio/fvsam_audio_wrp/src/fvsam_audio.cpp	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_voice/build/fvsam_voice.scons	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_voice/fvsam_lib/build/avs_libs/qdsp6/6150.adsp.prod/fvsam_lib.lib	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_voice/fvsam_lib/build/avs_libs/qdsp6/7150.adsp.prod/fvsam_lib.lib	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_voice/fvsam_lib/build/avs_libs/qdsp6/855.adsp.prod/fvsam_lib.lib	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_voice/fvsam_lib/build/avs_libs/qdsp6/bitra.adsp.prod/fvsam_lib.lib	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_voice/fvsam_lib/build/avs_libs/qdsp6/nicobar.adsp.prod/fvsam_lib.lib	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_voice/fvsam_lib/build/avs_libs/qdsp6/saipan.adsp.prod/fvsam_lib.lib	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_voice/fvsam_lib/build/fvsam_lib.scons	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_voice/fvsam_lib/inc/fvsam_comm.h	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_voice/fvsam_lib/inc/fvsam_sapi.h	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_voice/fvsam_voice_wrp/build/avs_libs/qdsp6/bitra.adsp.prod/fvsam_voice_wrp.lib	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_voice/fvsam_voice_wrp/build/avs_libs/qdsp6/bitra.adsp.prod/src/fvsam_voice_imc.o	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_voice/fvsam_voice_wrp/build/avs_libs/qdsp6/bitra.adsp.prod/src/fvsam_voice_rx.o	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_voice/fvsam_voice_wrp/build/avs_libs/qdsp6/bitra.adsp.prod/src/fvsam_voice_tx.o	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_voice/fvsam_voice_wrp/build/fvsam_voice_wrp.scons	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_voice/fvsam_voice_wrp/inc/fvsam_voice_rx.h	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_voice/fvsam_voice_wrp/inc/fvsam_voice_tx.h	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_voice/fvsam_voice_wrp/src/fvsam_voice_imc.cpp	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_voice/fvsam_voice_wrp/src/fvsam_voice_imc.h	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_voice/fvsam_voice_wrp/src/fvsam_voice_rx.cpp	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/audio/fvsam_voice/fvsam_voice_wrp/src/fvsam_voice_tx.cpp	[new file with mode: 0644]	blob
ADSP.VT.5.6/adsp_proc/hap/oem/build/bitra/custom_libs_cfg.json	[changed mode: 0755->0644]	diff | blob | history
```

可能要改一下xml
[qualcomm/platform/hardware/qcom/audio.git] / configs / lito / audio_platform_info_irvine.xml

```
@@ -46,12 +46,29 @@
         <device name="SND_DEVICE_IN_USB_HEADSET_HEX_MIC_AEC" acdb_id="162"/>
         <device name="SND_DEVICE_IN_UNPROCESSED_USB_HEADSET_HEX_MIC" acdb_id="162"/>
         <device name="SND_DEVICE_IN_VOCE_RECOG_USB_HEADSET_HEX_MIC" acdb_id="162"/>
-        <!-- mod by fuhua for task id : 9824953 on 2020-08-28
+        <!-- mod by fuhua for task id : 9824953 on 2020-08-28        -->
+        <!--
         <device name="SND_DEVICE_IN_HANDSET_MIC_AEC_NS" acdb_id="4"/>
         <device name="SND_DEVICE_IN_SPEAKER_MIC_AEC_NS" acdb_id="11"/>
         <device name="SND_DEVICE_IN_HEADSET_MIC_FLUENCE" acdb_id="8"/>
         <device name="SND_DEVICE_IN_HANDSET_MIC" acdb_id="118"/>
         -->
+        <device name="SND_DEVICE_IN_VOICE_SPEAKER_MIC" acdb_id="43"/>
+        <device name="SND_DEVICE_IN_VOICE_SPEAKER_MIC_SB" acdb_id="43"/>
+        <device name="SND_DEVICE_IN_VOICE_SPEAKER_MIC_NN" acdb_id="43"/>
+        <device name="SND_DEVICE_IN_HANDSET_MIC" acdb_id="41"/>
+        <device name="SND_DEVICE_IN_HANDSET_MIC_SB" acdb_id="41"/>
+        <device name="SND_DEVICE_IN_HANDSET_MIC_NN" acdb_id="41"/>
+        <device name="SND_DEVICE_IN_HANDSET_MIC_AEC" acdb_id="41"/>
+        <device name="SND_DEVICE_IN_HANDSET_MIC_AEC_SB" acdb_id="41"/>
+        <device name="SND_DEVICE_IN_HANDSET_MIC_AEC_NN" acdb_id="41"/>
+        <device name="SND_DEVICE_IN_HANDSET_MIC_NS" acdb_id="41"/>
+        <device name="SND_DEVICE_IN_HANDSET_MIC_NS_SB" acdb_id="41"/>
+        <device name="SND_DEVICE_IN_HANDSET_MIC_NS_NN" acdb_id="41"/>
+        <device name="SND_DEVICE_IN_HANDSET_MIC_AEC_NS" acdb_id="41"/>
+        <device name="SND_DEVICE_IN_HANDSET_MIC_AEC_NS_SB" acdb_id="41"/>
+        <device name="SND_DEVICE_IN_HANDSET_MIC_AEC_NS_NN" acdb_id="41"/>
+
     </acdb_ids>
 
```


添加还是很方便的，验证的话需要两部分走，1是greo一下Nonhlos是否有编译进去算法，比如`grep -rns awinic_sp`

然后就是在播放音乐等场景下抓QXDM log，转换成txt，搜索关键子看看有没有咯。


## 校准
aw会提供校准执行程序，需要我们自己集成，主要注意三点，1.路径，2.权限，3.I2C地址
#####1.改一下MMITest
[qualcomm/vendor/tct-source/mmitest.git] / MMITest / src / com / android / mmi / SmartPATest.java
```
@@ -142,8 +142,9 @@ public class SmartPATest extends TestBase {
                     // TODO Auto-generated catch block
                     e.printStackTrace();
                 }
-
-                String commond = "aw881xx_cali start_cali aw881xx_smartpa 0x00 0x43";
+//Begin modify by lanying.he for XR9824828 on 2020-08-28
+                String commond = "aw881xx_cali start_cali aw881xx_smartpa 0x03 0x34";
+//End modify by lanying.he for XR9824828 on 2020-08-28
                 String result;
                 result = runShellCmd(commond);
```
[qualcomm/vendor/tct-source/mmitest.git] / MMITest / assets / limit_irvine.xml
```
 <?xml version="1.0" encoding="utf-8"?>
 <configuration>
      <config
-     rdc_min = "6.8"
-     rdc_max = "9.2"
-     f0_min = "740"
-     f0_max = "920"
+     rdc_min = "6.4"
+     rdc_max = "8.6"
+     f0_min = "765"
+     f0_max = "1035"
      otp_infinity_64m_min = "310"
      otp_infinity_64m_max = "610"
```

#####2. 将校准程序拷贝到手机
[qualcomm/device/qcom/qssi.git] / qssi.mk
```
 PRODUCT_PACKAGES += libjni_cali_verify \
     libjni_macro
-
+#Begin modify by lanying.he for XR9824828 on 2020-08-28
 PRODUCT_COPY_FILES += \
-    device/qcom/lito/seattle/aw881xx/aw881xx_cali:system/bin/aw881xx_cali
+    device/qcom/lito/Irvine/aw881xx/aw881xx_cali:system/bin/aw881xx_cali
+#End modify by lanying.he for XR9824828 on 2020-08-28
 # Task: 8749286, add oempersist mount point
```

####3. 给节点添加权限
[qualcomm/platform/vendor/qcom/lito.git] / init.target.rc
```
     chown system system /sys/bus/i2c/devices/1-0038/fts_glove_mode
     chmod 0666 /sys/bus/i2c/devices/1-0038/fts_glove_mode
     #[TCT-ROM][Glove Mode]End modified by xiang.zhang for task 8956524 on 2020/3/4
-    chown system system /sys/devices/platform/soc/880000.i2c/i2c-0/0-0043/cali
-    chown system system /sys/devices/platform/soc/880000.i2c/i2c-0/0-0043/f0
-    chown system system /sys/devices/platform/soc/880000.i2c/i2c-0/0-0043/re
-    chown system system /sys/devices/platform/soc/880000.i2c/i2c-0/0-0043/rw
-    chown system system /sys/devices/platform/soc/880000.i2c/i2c-0/0-0043/monitor
+    #Begin modify by lanying.he for XR9824828 on 2020-08-28
+    chown system system /sys/devices/platform/soc/988000.i2c/i2c-3/3-0034/cali
+    chown system system /sys/devices/platform/soc/988000.i2c/i2c-3/3-0034/f0
+    chown system system /sys/devices/platform/soc/988000.i2c/i2c-3/3-0034/re
+    chown system system /sys/devices/platform/soc/988000.i2c/i2c-3/3-0034/rw
+    chown system system /sys/devices/platform/soc/988000.i2c/i2c-3/3-0034/monitor
+    #End modify by lanying.he for XR9824828 on 2020-08-28
 
     chown system system /sys/bus/i2c/devices/1-0038/panel_vendor
```
[qualcomm/device/qcom/sepolicy.git] / generic / vendor / common / file_contexts
```
 /sys/devices/platform/soc/984000.i2c/i2c-1/1-0038/panel_vendor                    u:object_r:tct_mmitest_sysfs:s0
 /sys/devices/platform/soc/880000.spi/spi_master/spi0/spi0.0/fts_test              u:object_r:tct_mmitest_sysfs:s0
 # smartpa
-/sys/devices/platform/soc/880000.i2c/i2c-0/0-0043/cali                  u:object_r:tct_mmitest_sysfs:s0
-/sys/devices/platform/soc/880000.i2c/i2c-0/0-0043/f0                    u:object_r:tct_mmitest_sysfs:s0
-/sys/devices/platform/soc/880000.i2c/i2c-0/0-0043/re                    u:object_r:tct_mmitest_sysfs:s0
-/sys/devices/platform/soc/880000.i2c/i2c-0/0-0043/rw                    u:object_r:tct_mmitest_sysfs:s0
-/sys/devices/platform/soc/880000.i2c/i2c-0/0-0043/monitor               u:object_r:tct_mmitest_sysfs:s0
-
+#Begin modify by lanying.he for XR9824828 on 2020-08-28
+/sys/devices/platform/soc/988000.i2c/i2c-3/3-0034/cali                  u:object_r:tct_mmitest_sysfs:s0
+/sys/devices/platform/soc/988000.i2c/i2c-3/3-0034/f0                    u:object_r:tct_mmitest_sysfs:s0
+/sys/devices/platform/soc/988000.i2c/i2c-3/3-0034/re                    u:object_r:tct_mmitest_sysfs:s0
+/sys/devices/platform/soc/988000.i2c/i2c-3/3-0034/rw                    u:object_r:tct_mmitest_sysfs:s0
+/sys/devices/platform/soc/988000.i2c/i2c-3/3-0034/monitor               u:object_r:tct_mmitest_sysfs:s0
+#End modify by lanying.he for XR9824828 on 2020-08-28
 #npi down status
 /sys/class/npi_down_status/status u:object_r:tct_mmitest_sysfs:s0
```
##### 4. 更改校准驱动中的文件权限
[qualcomm/kernel/msm-4.19.git] / sound / soc / codecs / aw881xx_irvine / aw881xx_cali.c
```
 #ifdef AW_CALI_STORE_EXAMPLE\r
 \r
-#define AWINIC_CALI_FILE  "/mnt/vendor/persist/factory/audio/aw_cali.bin"\r
+#define AWINIC_CALI_FILE  "/oempersist/audio/aw_cali.bin" //modify by lanying.he for XR9824828 on 2020-08-31\r
 #define AW_INT_DEC_DIGIT 10\r
 static int aw881xx_write_cali_re_to_file(uint32_t cali_re, char *name_suffix)\r
 {\r
@@ -55,7 +55,7 @@ static int aw881xx_write_cali_re_to_file(uint32_t cali_re, char *name_suffix)
                return -EINVAL;\r
        }\r
 \r
-       fp = filp_open(AWINIC_CALI_FILE, O_RDWR | O_CREAT, 0644);\r
+       fp = filp_open(AWINIC_CALI_FILE, O_RDWR | O_CREAT, 0664); //modify by lanying.he for XR9824828 on 2020-08-31\r
        if (IS_ERR(fp)) {\r
                pr_err("%s:channel:%s open %s failed!\n",\r
                        __func__, channel, AWINIC_CALI_FILE);\r
@@ -221,6 +221,7 @@ static ssize_t aw881xx_cali_store(struct device *dev,
                return ret;\r
 \r
        cali_attr->cali_re = databuf[0] * (1 << AW881XX_DSP_RE_SHIFT) / 1000;\r
+       pr_info("%s : cali_re = %d \n ",__func__,cali_attr->cali_re);   //added by lanying.he for XR9824828 on 2020-08-31\r
        aw881xx_set_cali_re(cali_attr);\r
 \r
        return count;\r
@@ -249,13 +250,20 @@ static ssize_t aw881xx_re_store(struct device *dev,
        struct aw881xx_cali_attr *cali_attr = &aw881xx->cali_attr;\r
        int ret = -1;\r
        unsigned int databuf[2] = { 0 };\r
-\r
+    //Begin modify by lanying.he for XR9824828 on 2020-08-31\r
+    pr_info("%s : \n ",__func__);\r
        ret = kstrtouint(buf, 0, &databuf[0]);\r
        if (ret < 0)\r
                return ret;\r
 \r
        cali_attr->re = databuf[0];\r
+    pr_info("%s : re = %d \n ",__func__,cali_attr->re);\r
+    cali_attr->cali_re = cali_attr->re * (1 << AW881XX_DSP_RE_SHIFT) / 1000;\r
+\r
+    pr_info("%s : cali_re = %d \n ",__func__,cali_attr->cali_re);\r
 \r
+    aw881xx_set_cali_re(cali_attr);\r
+    //Begin modify by lanying.he for XR9824828 on 2020-08-31\r
        return count;\r
 }\r
 \r
```
由于确实忙不过来了，这部分就交给了同事lanying做的，所以注释也是她的。^_^

## 其余Audio部分
speaker有声音了只是部分，但是mic和receiver以及耳机用的codec都还是wcd937x，所以需要配置驱动，一般来讲，将数字cdc和dai的控件注册上之后应该问题都不大，这个调试需要对着log调，1是看comb是否有注册上，2是是不是wsa的干扰等因素导致ret直接error了。要在kona.c里面添加log。


还有声卡加载定制部分，有如下文件需要改动：
projects / qualcomm / platform / hardware / qcom / audio.git / commit
```
configs/lito/audio_platform_info_irvine.xml	
configs/lito/lito.mk
configs/lito/mixer_paths_irvine.xml
hal/Android.mk
hal/msm8974/platform.c
```
[qualcomm/platform/hardware/qcom/audio.git] / configs / lito / lito.mk
```
+#begin mod by fuhua for task_id: 9660534  on 2020-07-22
+ifeq ($(CUSTOMIZE_AUDIO_FEATURE_SUPPORT),IRVINE)
+PRODUCT_COPY_FILES += \
+    vendor/qcom/opensource/audio-hal/primary-hal/configs/lito/mixer_paths_irvine.xml:$(TARGET_COPY_OUT_VENDOR)/etc/mixer_paths_irvine.xml \
+    vendor/qcom/opensource/audio-hal/primary-hal/configs/lito/audio_platform_info_irvine.xml:$(TARGET_COPY_OUT_VENDOR)/etc/audio_platform_info_irvine.xml
+endif
+#end   mod by fuhua for task_id: 9660534  on 2020-07-22
```
[qualcomm/platform/hardware/qcom/audio.git] / hal / Android.mk
```
+#begin mod by fuhua for task_id: 9660534  on 2020-07-22
+ifeq ($(CUSTOMIZE_AUDIO_FEATURE_SUPPORT),IRVINE)
+    LOCAL_CFLAGS += -DCUSTOMIZE_AUDIO_FEATURE_IRVINE
+    #LOCAL_C_INCLUDES += vendor/qcom/opensource/audio-hal/primary-hal/climax/app/exTfa98xx/inc
+    #LOCAL_SHARED_LIBRARIES += libtfatest
+endif
+#end   mod by fuhua for task_id: 9660534  on 2020-07-22
```
[qualcomm/platform/hardware/qcom/audio.git] / hal / msm8974 / platform.c
```
 #endif
 #define PLATFORM_INFO_XML_PATH_SEATTLE  "/vendor/etc/audio_platform_info_seattle.xml"
 
+#ifdef CUSTOMIZE_AUDIO_FEATURE_IRVINE
+#define PLATFORM_INFO_XML_PATH_IRVINE  "/vendor/etc/audio_platform_info_irvine.xml" // begin add by fuhua for task_id: 9660534  on 2020-07-22
+#endif
+
 #include <linux/msm_audio.h>
 #if defined (PLATFORM_MSM8998) || (PLATFORM_SDM845) || (PLATFORM_SDM710) || \
     defined (PLATFORM_QCS605) || defined (PLATFORM_MSMNILE) || \
	 
......

@@ -1882,6 +1886,14 @@ static void update_codec_type_and_interface(struct platform_data * my_data,
 #endif
 /* MODIFIED-END by bo.yu,BUG-8676815*/
                    /* MODIFIED-END by hongwei.tian,BUG-8580187*/
+
+/* begin add by fuhua for task_id: 9660534  on 2020-07-22 */
+#ifdef CUSTOMIZE_AUDIO_FEATURE_IRVINE
+         !strncmp(snd_card_name, "lito-irvine-snd-card",
+                   sizeof("lito-irvine-snd-card")) ||
+#endif
+/* end   add by fuhua for task_id: 9660534  on 2020-07-22 */
+
          !strncmp(snd_card_name, "atoll-qrd-snd-card",
                    sizeof("atoll-qrd-snd-card")) ||
          !strncmp(snd_card_name, "bengal-idp-snd-card",
@@ -3459,6 +3471,15 @@ void *platform_init(struct audio_device *adev)
         platform_info_init(PLATFORM_INFO_XML_PATH_SEATTLE, my_data, PLATFORM);
		 
......
@@ -3459,6 +3471,15 @@ void *platform_init(struct audio_device *adev)
+/* begin add by fuhua for task_id: 9660534  on 2020-07-22 */
+#ifdef CUSTOMIZE_AUDIO_FEATURE_IRVINE
+    else if (!strncmp(snd_card_name, "lito-irvine-snd-card",
+               sizeof("lito-irvine-snd-card")))
+        platform_info_init(PLATFORM_INFO_XML_PATH_IRVINE, my_data, PLATFORM);
+#endif
+/* end   add by fuhua for task_id: 9660534  on 2020-07-22 */
```