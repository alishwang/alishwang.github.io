+++
author = "fuhua"
title = "Android - Audio - MT67平台 - Milan项目的Smart PA bring-up"
date = "2020-01-16 22:33:11"
description = "手机技术笔记"
tags = [
    "markdown",
    "text",
]
weight = 10
+++



- [背景](#背景)  
- [Smart-PA是什么](#Smart-PA是什么)  
- [为什么要用Smart PA](#为什么要用Smart-PA)  
- [怎样用它](#怎么样用到项目当中。)  
    - [结构和硬件](#结构怎么做，用到了哪些引脚。)  
	- [软件部分](#软件当中要做哪些事情。)  
		- [kernel](#kernel)  
		    - [I2C](#I2C)  
			- [I2S](#I2S)  
			- [SPI](#SPI)  
- [具体修改内容](#修改目录以及详细修改内容)  
    - [kernel](#kernel部分)  
	- [hal层](#hal部分)
	- [其他目录](#其他patch目录)  
- [调试](#调试)  
- [结语](#总结)  
- [额外的补充内容](#PS)  




## 背景
>公司的项目（ MTK平台 ）要用AW的Smart PA，又有文档输出的要求，所以准备顺便写一下；作为个人知识积累的一部分。  
本文暂时主要聚焦软件的bring up，其余内容可能不够深入。


## Smart PA是什么  
一种有反馈参数的放大器，主要功能都是保护喇叭，比如断路保护，过温保护，欠压保护，超压保护；有对应的算法和动态调节机制(升高或者降低电压)；如公司用的AW88194A具有高射频抑制和消除TDD 噪音。


## 为什么要用Smart PA  
__与传统PA不同的地方是哪里？市面上那么多厂商都用了Smart PA，到底效果如何？__  
因为能保护器件，不同的就是那些动态调节和算法，肯定要优异一些，去除噪音，抑制高频信号等等。
市面上用了Smart PA 的厂商实际上audio其他部分也不会差，比如音腔的设计，喇叭的选型，位置摆放，算法调试等等，从软件到硬件都会有层级上的区别。效果自然要好很多。
如果单纯比较的话还是可以通过Golden sample来对比一下，可以使用milan项目和Gauss来对比，把喇叭焊出来就知道了。  


## 怎么样用到项目当中。  
单纯从smart pa这样的一个芯片来看，直接替换掉原有的PA就ok了，但是从audio整体来看，这个东西是软硬件结合的，有结构和硬件，也有软件的部分。
按照个人理解，硬件和结构的目的是在合适的位置安放喇叭，结合项目保证音质最佳，然后会考虑各种消噪和保护。

### 结构怎么做，用到了哪些引脚。  
单纯从milan项目和smart PA芯片来讲，结构并没有做任何改变，因为这仅仅是一个芯片，想要提高音质，需要良好的音腔。  

引脚倒是用到了不少,关键引脚如下（AW88194的INIT）
```
                        VIO18_PMU

                         |
                         | AD2
                         v
                 AD1   +---------+  SCL5              +--------+
      GND       -----> |         | <----------------- |        |
                       |         |                    |        |
                       |         |  SDA5              |        |
                       |         | <----------------- |        |
                       |         |                    |        |
+-------------+  VON   |         |  I2S_BCK           |        |
| AU_SPKN_SUB | <----- |         | <----------------- |        |
+-------------+        |         |                    |        |
                       |         |  I2S_LRCK          |        |
                       |         | <----------------- |        |
                       | AW88194 |                    | MT6762 |
+-------------+  VOP   |         |  I2S_DO            |        |
| AU_SPKP_SUB | <----- |         | <----------------- |        |
+-------------+        |         |                    |        |
                       |         |  I2S_DI            |        |
                       |         | <----------------- |        |
                       |         |                    |        |
                       |         |  RSTN(GPIO_EXT8)   |        |
                       |         | <----------------- |        |
                       |         |                    |        |
                       |         |  INTN(GPIO_AC24)   |        |
                       |         | <----------------- |        |
                       +---------+                    +--------+


```

### 软件当中要做哪些事情。  

#### kernel  
(1)Asoc架构，machine，platform，codec，dai，dai_link分别是什么，互相之间什么关系。  

(2)注册设备，这款AW的smart PA，就需要用I2C通信，通过I2C设备注册到总线上，然后在注册成codec设备，在dai link的时候就能连接了。  

##### I2C  
1. 是什么技术：
I2C(Inter-Integrated Circuit)字面上理解就是内部集成电路，I2C Bus简称，是一种串行通信总线，由飞利浦在1980年为了让主板、嵌入式系统、手机用来连接低速外设发展，当时主要用途在于控制飞利浦所生产的芯片，2006年，使用i2c协议不需要支付专利费用了，但据说制造商仍需要付费来获取I2C从属设备地址？  

2. 为什么有这项技术： 
像I2C这样的总线之所以那么火是因为电脑工程师发现对于集成电路设计而言，许多的制造成本源自于封装尺寸及接脚数量。更小的包装通常能够减少重量及电源的消耗，这对于移动电话以及手持式电脑来说格外重要。  

3. 这项技术能干什么：  
当然是连接设备，和其余设备通信，组成网络撒。  

4. 使用规则及协议：
两条线，SDA-串行数据，SCL-串行时钟，典型电压标准为+3.3V或者+5V，7比特长度的地址空间但保留了16个地址。常见速率：标准模式100kbit/s，低速模式10kbit/s；新一代I2C总线支持10比特长度的地址空间。

协议开销包括一个字节从地址以及每个字节的ACK/NCK比特


```
        ————          —————————————                    ————————————————                        ————————————————————
SDA:        |        |             |                  |                |                      |
             ————————               ——————————————————                  ——————————————————————

        ————————       —————————          ——————————        ————————          —————————————————————————
SCL:            |     |         |        |          |      |        |        |
                 —————           ————————            ——————          ————————
```




##### I2S  
1. 是什么样的技术：  
I2S(Inter-IC Sound)是IC间传输数字音频数据的一种接口标准，采用序列的方式传输2组（左右声道）数据。I2S常被使用在发送PCM音频数据到DAC中。  

2. 为什么是它呢？  
cpu->audio codec的内部传输，I2S是最方便省力的。  

3. 这项技术能干什么：
传输左右声道的数据。因为只需要双声道，所以i2s最省，如果有的系统需要更高的要求，或许cpu和codec之间就得用HD-Audio这样的接口了。    

4. 使用规则及协议：  
如上图，BCK和LRCK作为位时钟和帧时钟，DO和DI作为DATA数据线。  
每发送1位数字音频数据，SCK上都有1个脉冲。  
LRCK作为声道选择，用于切换左右声道数据，WS为1表示传输的是左声道数据，0表示右声道。  


##### SPI  
1. 和I2C比有什么优势么：  
结合现在的标准速度快很多，而且清晰明了，所以有存在的意义。  
补充一个知识点：

>单工、半双工、全双工  
单工数据传输只支持数据在一个方向上传输；  
半双工数据传输允许数据在两个方向上传输，但是，在某一时刻，只允许数据在一个方向上传输，它实际上是一种切换方向的单工通信；  
全双工数据通信允许数据同时在两个方向上传输，因此，全双工通信是两个单工通信方式的结合，它要求发送设备和接收设备都有独立的接收和发送能力。  
I2C是半双工，SPI的全双工，uart是全双工。  
[这篇文章讲得蛮清楚的样子](https://blog.csdn.net/chenpuo/article/details/81023882)



2. 为什么有这项技术：  
追述历史，好像是摩托罗拉在1979年搞的总线协议？反正和飞利浦不一样，可能也是为了规避技术专利吧。  

3. 这项技术能干什么：  
和I2C差不多，都是系统内部的小协议，组内网呗。  

4. 使用规则及协议：  
网上资料多，用到的时候再总结。   


## 修改目录以及详细修改内容  
该部分是记录详细修改的代码，也方便回顾
### kernel部分
```
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

    modified:   drivers/misc/mediatek/dws/mt6765/k62v1_64_bsp.dws
	modified:   arch/arm64/boot/dts/mediatek/k62v1_64_bsp.dts
	modified:   arch/arm64/configs/k62v1_64_bsp_debug_defconfig
	modified:   arch/arm64/configs/k62v1_64_bsp_defconfig
	modified:   sound/soc/codecs/Makefile
    modified:   sound/soc/codecs/Kconfig
	modified:   drivers/base/firmware_class.c
	modified:   sound/soc/mediatek/common_int/mtk-soc-machine.c
	modified:   sound/soc/mediatek/common_int/mtk-soc-pcm-dl1-i2s0.c
	modified:   sound/soc/soc-core.c

Untracked files:
  (use "git add <file>..." to include in what will be committed)

	sound/soc/codecs/aw88194.c
	sound/soc/codecs/aw88194.h
	sound/soc/codecs/aw88194_reg.h
	sound/soc/codecs/aw88195.c
	sound/soc/codecs/aw88195.h
	sound/soc/codecs/aw88195_reg.h
	sound/soc/codecs/aw881xx.c
	sound/soc/codecs/aw881xx.h
```
如上所述，首先是dts文件，mtk的项目同事建议最好在dws文件里面改，方便维护，但是原生的都是在dts里面直接加的，而且维护两个地方实际上很麻烦。  
由于smartPA的GPIO口已经配好了的，而且MTK默认有一套SmartPA的方案，I2S3的GPIO可以直接用上去，所以dts里面只需要关注器件的I2C通讯就行(这里要注意I2C速率，器件是否上电，地址)。    

modified:   drivers/misc/mediatek/dws/mt6765/k62v1_64_bsp.dws

```
    <i2c_bus5>
        <speed_kbps>400</speed_kbps>
        <pullPushEn>true</pullPushEn>
    </i2c_bus5>
```

modified:   arch/arm64/boot/dts/mediatek/k62v1_64_bsp.dts
```
/*begin add by fuhu AWINIC AW881XX Smart PA for task 8740652*/
&i2c5 {
	aw881xx_smartpa@36 {
		compatible = "awinic,aw881xx_smartpa";
		reg = <0x36>;
		reset-gpio = <&pio 171 0>;
		irq-gpio = <&pio 153 0>;
		monitor-flag = <1>;
		monitor-timer-val = <30000>;
		status = "okay";
	};
};
/*end  add by fuhua AWINIC AW881XX Smart PA for task 8740652*/

```
（I2S的线已经默认配好了的，如果没有配置，则需要另外配置，可以检查编译出来的dtsi有没有将I2S给配置上去）

modified:   arch/arm64/configs/k62v1_64_bsp_debug_defconfig
modified:   arch/arm64/configs/k62v1_64_bsp_defconfig  
这两个文件添加CONFIG并且注释掉默认的smartPA方案。  
```
#begin add by fuhua.wang for task 8740652 on 2020.01.19
#CONFIG_SND_SOC_RT5509=y
CONFIG_SND_SMARTPA_AW881XX
#end   add by fuhua.wang for task 8740652 on 2020.01.19
```

modified:   sound/soc/codecs/Makefile
modified:   sound/soc/codecs/Kconfig
这个目录顺便就把`Untracked files:`一并加上去，这些都是供应商提供的驱动代码， 里面或许有bug，粗略的检查问题不大。  
```
#begin mod by fuhua.wang for task 8740652 on 2020.01.19
#snd-soc-rt5509-objs := rt5509.o rt5509-regmap.o rt5509-calib.o
#end   mod by fuhua.wang for task 8740652 on 2020.01.19

......

#begin mod by fuhua.wang for task 8740652 on 2020.01.19
#obj-$(CONFIG_SND_SOC_RT5509)	+= snd-soc-rt5509.o
obj-$(CONFIG_SND_SOC_MT6660)	+= snd-soc-mt6660.o
obj-$(CONFIG_SND_SMARTPA_AW881XX)	+= aw881xx.o aw88194.o aw88195.o
#end   mod by fuhua.wang for task 8740652 on 2020.01.19

-----

config SND_SMARTPA_AW881XX
	tristate "Soc Audio for awinic a881xx series"
	depends on I2C
	help
		This option enables support for aw881xx series Smart PA
```

modified:   drivers/base/firmware_class.c  
添加固件的位置  

```
static const char * const fw_path[] = {
	fw_path_para,
	"/system/vendor/firmware",//add by fuhua.wang for task 8740652 on 2020.01.19
	"/lib/firmware/updates/" UTS_RELEASE,
	"/lib/firmware/updates",

```


modified:   sound/soc/mediatek/common_int/mtk-soc-machine.c
这里主要是添加dai接口，以及屏蔽掉mtk默认的冲掉dai配置的地方(供应商有这么说，实际验证确实如此，还需要细致查看代码才能了解清楚。)  

```
		.ignore_suspend = 1,
		.ignore_pmdown_time = true,
		.dai_fmt = SND_SOC_DAIFMT_I2S | SND_SOC_DAIFMT_CBS_CFS |
			   SND_SOC_DAIFMT_NB_NF,
		.ops = &cs35l35_ops,
#else
//begin mod by fuhua.wang for task 8740652 on 2020.01.19
//		.codec_dai_name = "snd-soc-dummy-dai",
//		.codec_name = "snd-soc-dummy",
		.ignore_suspend = 1,
		.ignore_pmdown_time = true,
		.codec_dai_name = "aw881xx-aif",
		.codec_name = "aw881xx_smartpa",
		.ops = &mt_machine_audio_ops,
//end   mod by fuhua.wang for task 8740652 on 2020.01.19
#endif
	},

......

static int mt_soc_snd_probe(struct platform_device *pdev)
{
	struct snd_soc_card *card = &mt_snd_soc_card_mt;
	struct device_node *btcvsd_node;
	int ret;
	int daiLinkNum = 0;

//begin mod by fuhua.wang for task 8740652 on 2020.01.19
#if 0
	ret = mtk_spk_update_dai_link(mt_soc_extspk_dai, pdev);
	if (ret) {
		dev_err(&pdev->dev, "%s(), mtk_spk_update_dai_link error\n",
			__func__);
		return -EINVAL;
	}
#endif
//end   mod by fuhua.wang for task 8740652 on 2020.01.19

	get_ext_dai_codec_name();
```

modified:   sound/soc/mediatek/common_int/mtk-soc-pcm-dl1-i2s0.c  
modified:   sound/soc/soc-core.c  
剩下的这两个文件都是添加的调试log，主要是看i2s3有没有通、codec有没有注册上。  
```
Audio_i2s0_SideGen_Set(struct snd_kcontrol *kcontrol,
	unsigned int u32Audio2ndI2sIn = 0;
	AudDrv_Clk_On();
+       pr_err("fuhua---check Audio_i2s0_SideGen_Set---begin!!!\n");
	if (ucontrol->value.enumerated.item[0] > ARRAY_SIZE(i2s0_SIDEGEN)) {
			pr_err("return -EINVAL\n");
			return -EINVAL;
static int Audio_i2s0_SideGen_Set(struct snd_kcontrol *kcontrol,
	/* Config smart pa I2S pin */
	AudDrv_GPIO_SMARTPA_Select(mi2s0_sidegen_control > 0 ? 1 : 0);

+       pr_err("fuhua---check Audio_i2s0_SideGen_Set---begin!!!mi2s0_sidegen_control = %d\n",mi2s0_sidegen_control);
	pr_debug(
			"%s(), sidegen = %d, hdoutput = %d, extcodec_echoref = %d, always_hd = %d\n",
			__func__, mi2s0_sidegen_control, hdoutput_control,
static int Audio_i2s0_SideGen_Set(struct snd_kcontrol *kcontrol,
	}

	if (mi2s0_sidegen_control) {
+               pr_err("fuhua---check in if mi2s0_sidegen_control = 1\n");
			/* Phone call echo ref, speaker mode connection*/
			switch (extcodec_echoref_control) {
			case 1:
static int Audio_i2s0_SideGen_Set(struct snd_kcontrol *kcontrol,

			EnableAfe(true);
	} else {
+               pr_err("fuhua---check in else mi2s0_sidegen_control = 0\n");
			if (extcodec_echoref_control > 0) {
					SetIntfConnection(
							Soc_Aud_InterCon_DisConnect,


------

struct snd_soc_dai *snd_soc_find_dai(
	const struct snd_soc_dai_link_component *dlc)
{
	struct snd_soc_component *component;
	struct snd_soc_dai *dai;
    ......
		if (dlc->of_node && component_of_node != dlc->of_node)
			continue;
		pr_err("fuhua snd_soc_find_dai  component->name = %s,dlc->name = %s\n",component->name,dlc->name);
		if (dlc->name && strcmp(component->name, dlc->name))
        ......
	}

	return NULL;
}
```

### hal部分  
vendor\mediatek\proprietary\hardware\audio：  
	modified:   Android.mk
    modified:   common/V3/include/AudioALSAPlaybackHandlerNormal.h
	modified:   common/V3/aud_drv/AudioALSAPlaybackHandlerNormal.cpp
	modified:   common/V3/aud_drv/AudioSmartPaController.cpp

Untracked files:
  (use "git add <file>..." to include in what will be committed)

	common/sktune/

common/sktune/：
里面添加算法固件。  



	modified:   Android.mk
Android.mk中添加了aw的算法库，由于还未进行调试，所以还有宏没有打开：  
```
#begin mod by fuhua.wang for task 8740652 on 2020.01.19
#ifeq ($(findstring aw881xx, $(MTK_AUDIO_SPEAKER_PATH)), aw881xx)
LOCAL_C_INCLUDES += $(LOCAL_PATH)/common/sktune
LOCAL_STATIC_LIBRARIES += libsktune
#LOCAL_LDFLAGS := $(LOCAL_PATH)/$(AUDIO_COMMON_DIR)/sktune/libsktune.a
#endif
#ifeq ($(strip $(TARGET_BUILD_VARIANT)),eng)
#  LOCAL_CFLAGS += -DAUDIODSP_ENABLE_DUMP_PCM
#endif
#end  mod by fuhua.wang for task 8740652 on 2020.01.19
```

	modified:   common/V3/include/AudioALSAPlaybackHandlerNormal.h  
    这其中添加对算法的函数定义什么的。  


```
//begin mod by fuhua.wang for task 8740652 on 2020.01.19
#define AUDIODSP_ENABLE 1
#if defined (AUDIODSP_ENABLE) && ((AUDIODSP_ENABLE) == 1)
//#if defined AUDIODSP_ENABLE_DUMP_PCM
#define AUDIODSP_DUMP_PCM 0
#endif

#include "OptimSpeakerV2Public.h"
//end  mod by fuhua.wang for task 8740652 on 2020.01.19

//begin mod by fuhua.wang for task 8740652 on 2020.01.19
#if defined (AUDIODSP_ENABLE) && ((AUDIODSP_ENABLE) == 1)
    char *audio_buffer;
    void *audiodsp_module;
    status_t open_audiodsp(void);
    status_t open_audiodsp_preset(const char *preset_file_name,
                                  void *audiodsp_module);
    void audiodsp_process(void *pBufferAfterPending,
                          uint32_t bytesAfterpending);
    void close_audiodsp(void);
#if defined (AUDIODSP_DUMP_PCM) && ((AUDIODSP_DUMP_PCM) == 1)
    void audiodsp_dump_pcm_init(void);
    void audiodsp_dump_pcm_free(void);
    void audiodsp_dump_pcm(const void *in, const void *out, size_t nbSamples);
    FILE *audiodsp_dump_pcm_fin = NULL;
    FILE *audiodsp_dump_pcm_fout = NULL;
    bool audiodsp_dump_pcm_init_done;
#endif
#endif
//end  mod by fuhua.wang for task 8740652 on 2020.01.19
```

	modified:   common/V3/aud_drv/AudioALSAPlaybackHandlerNormal.cpp:  

```
int AudioSmartPaController::speakerOn(unsigned int sampleRate, unsigned int device) {

    ......

	//begin mod by fuhua.wang for task 8740652 on 2020.01.19
	#if 0
        ret = mixer_ctl_set_enum_by_string(mixer_get_ctl_by_name(mMixer, mSmartPa.attribute.codecCtlName), "On");
        if (ret) {
            ALOGE("Error: %s invalid value, ret = %d", mSmartPa.attribute.codecCtlName, ret);
        }
	#endif
		ret = mixer_ctl_set_enum_by_string(mixer_get_ctl_by_name(mMixer, "aw881xx_speaker_switch"), "On");
		if(ret){
			ALOGE("fuhua---check aw881xx_speaker_switch Fail ret = %d,codecCylName = %s",ret,mSmartPa.attribute.codecCtlName);
		}
		/*
		ret = mixer_ctl_set_enum_by_string(mixer_get_ctl_by_name(mMixer, "aw881xx_receiver_switch"), "On");
		if(ret){
			ALOGE("fuhua---check aw881xx_speaker_switch Fail ret = %d,codecCylName = %s",ret,mSmartPa.attribute.codecCtlName);
		}
		*/
	//end  mod by fuhua.wang for task 8740652 on 2020.01.19

    ......


    int AudioSmartPaController::speakerOff() {
    ALOGV("%s()", __FUNCTION__);

    ......

    if (strlen(mSmartPa.attribute.codecCtlName)) {
	//begin mod by fuhua.wang for task 8740652 on 2020.01.19
    #if 0
        ret = mixer_ctl_set_enum_by_string(mixer_get_ctl_by_name(mMixer, mSmartPa.attribute.codecCtlName), "Off");
        if (ret) {
            ALOGE("Error: %s invalid value, ret = %d", mSmartPa.attribute.codecCtlName, ret);
        }
    #endif
		ret = mixer_ctl_set_enum_by_string(mixer_get_ctl_by_name(mMixer, "aw881xx_speaker_switch"), "Off");
		if(ret){
			ALOGE("fuhua---check aw881xx_speaker_switch Fail ret = %d,codecCylName = %s",ret,mSmartPa.attribute.codecCtlName);
		}
		/*
		ret = mixer_ctl_set_enum_by_string(mixer_get_ctl_by_name(mMixer, "aw881xx_receiver_switch"), "Off");
		if(ret){
			ALOGE("fuhua---check aw881xx_speaker_switch Fail ret = %d,codecCylName = %s",ret,mSmartPa.attribute.codecCtlName);
		}
		*/
	//end  mod by fuhua.wang for task 8740652 on 2020.01.19
    ......

```

	modified:   common/V3/aud_drv/AudioALSAPlaybackHandlerNormal.cpp  
    这里的代码就比较多了，主要都是供应商提供的算法代码。这里只展示部分关键代码，其余篇幅太多，就不了  

```
//begin mod by fuhua.wang for task 8740652 on 2020.01.19

#if defined (AUDIODSP_ENABLE) && ((AUDIODSP_ENABLE) == 1)
#include <OptimSpeakerV2Public.h>
#include <stdlib.h>
#define AUDIODSP_BUFFER_SIZE 1024*16 /* Why 1024*16 ? */
#define	STEREO_CH 2
#endif

//end  mod by fuhua.wang for task 8740652 on 2020.01.19


#if defined(MTK_AUDIODSP_SUPPORT)
#include "AudioDspStreamManager.h"
......


AudioALSAPlaybackHandlerNormal::AudioALSAPlaybackHandlerNormal(const stream_attribute_t *stream_attribute_source) :
    AudioALSAPlaybackHandlerBase(stream_attribute_source),
    mHpImpeDancePcm(NULL),
    mForceMute(false),
    mCurMuteBytes(0),
    mStartMuteBytes(0),
    mAllZeroBlock(NULL)
//begin mod by fuhua.wang for task 8740652 on 2020.01.19
#if defined (AUDIODSP_ENABLE) && ((AUDIODSP_ENABLE) == 1)
    ,
    audio_buffer(NULL),
    audiodsp_module(NULL)
#if defined (AUDIODSP_DUMP_PCM) && ((AUDIODSP_DUMP_PCM) == 1)
        ,
    audiodsp_dump_pcm_fin(NULL),
    audiodsp_dump_pcm_fout(NULL),
    audiodsp_dump_pcm_init_done(false)
#endif
#endif
//end  mod by fuhua.wang for task 8740652 on 2020.01.19


......


    AL_UNLOCK(AudioALSADriverUtility::getInstance()->getStreamSramDramLock());
//begin mod by fuhua.wang for task 8740652 on 2020.01.19
#if defined (AUDIODSP_ENABLE) && ((AUDIODSP_ENABLE) == 1)
    if (open_audiodsp() != 0) {
        ALOGE("fuhua ---check %s() Could not open audio DSP lib", __FUNCTION__);
    }
    //ALOGE("fuhua ---check %s() in there 111", __FUNCTION__);
#endif
//end  mod by fuhua.wang for task 8740652 on 2020.01.19

......


//begin mod by fuhua.wang for task 8740652 on 2020.01.19
#if defined (AUDIODSP_ENABLE) && ((AUDIODSP_ENABLE) == 1)
    close_audiodsp();
    //ALOGE("fuhua---check %s()",__FUNCTION__);
#endif
//end  mod by fuhua.wang for task 8740652 on 2020.01.19

......


//begin mod by fuhua.wang for task 8740652 on 2020.01.19
#if defined (AUDIODSP_ENABLE) && ((AUDIODSP_ENABLE) == 1)
    audiodsp_process(pBufferAfterPending, bytesAfterpending);
    //ALOGE("fuhua--check after audiodsp_process %s()",__FUNCTION__);
#endif
//end  mod by fuhua.wang for task 8740652 on 2020.01.19
```

### 其他patch目录
device\mediatek\vendor\common:  
	modified:   audio_param_smartpa/SmartPa_AudioParam.xml
	modified:   audio_param_smartpa/SmartPa_ParamUnitDesc.xml  

```
		<Param path="smartpa_awinic_aw881xx" param_id="7"/>

        ......

        <ParamUnit param_id="7">
			<Param name="have_dsp" value="1"/>
			<Param name="is_alsa_codec" value="1"/>
			<Param name="chip_delay_us" value="22000"/>
			<Param name="supported_rate_list" value="8000,11025,12000,16000,22050,24000,32000,44100,48000"/>
			<Param name="spk_lib_path" value=""/>
			<Param name="spk_lib64_path" value=""/>
			<Param name="codec_ctl_name" value="aw881xx_speaker_switch"/>
			<Param name="is_apll_needed" value="1"/>
			<Param name="i2s_set_stage" value="5"/>
		</ParamUnit>



------
			<Category name="smartpa_awinic_aw881xx"/>


```

device\mediateksample\k62v1_64_bsp:  
	modified:   ProjectConfig.mk
	modified:   device.mk

```

#begin mod by fuhua.wang for task 8740652 on 2020.01.19
PRODUCT_COPY_FILES += vendor/mediatek/proprietary/custom/k62v1_64_bsp/audio_param/common/aw_smartpa_firmware/aw881xx_pid_01_rcv_cfg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_01_rcv_cfg.bin
PRODUCT_COPY_FILES += vendor/mediatek/proprietary/custom/k62v1_64_bsp/audio_param/common/aw_smartpa_firmware/aw881xx_pid_01_rcv_fw.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_01_rcv_fw.bin
PRODUCT_COPY_FILES += vendor/mediatek/proprietary/custom/k62v1_64_bsp/audio_param/common/aw_smartpa_firmware/aw881xx_pid_01_spk_cfg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_01_spk_cfg.bin
PRODUCT_COPY_FILES += vendor/mediatek/proprietary/custom/k62v1_64_bsp/audio_param/common/aw_smartpa_firmware/aw881xx_pid_01_spk_fw.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_01_spk_fw.bin
PRODUCT_COPY_FILES += vendor/mediatek/proprietary/custom/k62v1_64_bsp/audio_param/common/aw_smartpa_firmware/aw881xx_pid_03_rcv_cfg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_03_rcv_cfg.bin
PRODUCT_COPY_FILES += vendor/mediatek/proprietary/custom/k62v1_64_bsp/audio_param/common/aw_smartpa_firmware/aw881xx_pid_03_rcv_fw.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_03_rcv_fw.bin
PRODUCT_COPY_FILES += vendor/mediatek/proprietary/custom/k62v1_64_bsp/audio_param/common/aw_smartpa_firmware/aw881xx_pid_03_spk_cfg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_03_spk_cfg.bin
PRODUCT_COPY_FILES += vendor/mediatek/proprietary/custom/k62v1_64_bsp/audio_param/common/aw_smartpa_firmware/aw881xx_pid_03_spk_fw.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_03_spk_fw.bin
PRODUCT_COPY_FILES += vendor/mediatek/proprietary/custom/k62v1_64_bsp/audio_param/common/aw_smartpa_firmware/aw881xx_pid_xx_rcv_reg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_xx_rcv_reg.bin
PRODUCT_COPY_FILES += vendor/mediatek/proprietary/custom/k62v1_64_bsp/audio_param/common/aw_smartpa_firmware/aw881xx_pid_xx_spk_reg.bin:$(TARGET_COPY_OUT_VENDOR)/firmware/aw881xx_pid_xx_spk_reg.bin
#PRODUCT_COPY_FILES += vendor/mediatek/proprietary/hardware/audio/common/sktune/sktPreset.sat:vendor/firmware/sktPreset.sat
PRODUCT_COPY_FILES += vendor/mediatek/proprietary/hardware/audio/common/sktune/sktPreset.sat:$(TARGET_COPY_OUT_VENDOR)/firmware/sktPreset.sat
#end   mod by fuhua.wang for task 8740652 on 2020.01.19

------

#begin mod by fuhua.wang for task 8740652 on 2020.01.19
MTK_AUDIO_SPEAKER_PATH = smartpa_awinic_aw881xx
#end   mod by fuhua.wang for task 8740652 on 2020.01.19

```

上面的地方也顺便把固件给传进去，目录如下：  
vendor\mediatek\proprietary\custom\k62v1_64_bsp






## 调试  
cat codec节点：
cat /sys/kernel/debug/asoc/codecs

使用minicom进行调试：
因为鼠标经常失灵，putty要点鼠标，很麻烦
minicom 首先要配置串口，然后配置波特率，设置保存的文件启动，然后就开始串口输出了，可以在另一个终端筛选文件的关键字`grep -rnsi fuhua tokyo-lite-boot.log`

在没有硬件的时候如何验证codec的注册和加载：
(1)注册platform_driver，并且和dts进行绑定。(这是kernel为了驱动移植做的框架，所以必须遵循)。  
(2)在绑定成功的之后会执行probe函数，在probe函数中注册codec，这个主要是为了platform和codec进行连接。  
(3)在machine的代码中添加dai链接，mtk的代码在`mt_soc_extspk_dai`结构体当中。  
(4)这样就能在soc-core.c中打印出log`fuhua snd_soc_find_dai  component->name = aw881xx_smartpa,dlc->name = aw881xx_smartpa`。  
(5)开机查看`cat /sys/kernel/debug/asoc/codecs`目录是否有`aw881xx_smartpa`或者自己设置的名字的codec则表明一个软件上的codec已经注册完毕了。  


### 参数要怎么调，调哪些。
除了正常的SmartPA参数之外，还需要在意codec本身的固件，以及算法文件，这些可能都需要供应商支持。  
而我们可以控制的估计就是MTK提供的接口部分，以及供应商提供的部分了。  
详细调试内容待添加。  

## 总结：
这一块MTK的软件就是没有分层的概念，混成了一坨~。  


## PS  
在没有SmartPA器件的时候，有要求先把软件给弄好，按照正常流程没有I2C设备进行注册是走不到声卡注册的，所以只能注册一个虚拟的设备与dai接口进行连接，即将I2C设备改为platform_driver ，代码如下即可：  
```
//begin add by fuhua 
static int dummy_codec_probe(struct snd_soc_codec *codec)
{

	return 0;
}

static int dummy_codec_remove(struct snd_soc_codec *codec)
{

	return 0;
}

static struct snd_soc_codec_driver soc_mtk_codec = {
	.probe = dummy_codec_probe, .remove = dummy_codec_remove,
};


static int aw881xx_test_probe(struct platform_device *pdev)
{
	int ret = 0;
	pr_err("fuhua---check---aw881xx_test_probe\n");
	ret = snd_soc_register_codec(&pdev->dev, &soc_mtk_codec,
				      aw881xx_dai,
				      ARRAY_SIZE(aw881xx_dai));
	pr_err("fuhua---check ret = %d",ret);
	return ret;
}

static int aw881xx_test_remove(struct platform_device *pdev)
{
	pr_err("fuhua---check---aw881xx_test_remove\n");
	snd_soc_unregister_codec(&pdev->dev);
	return 0;
}
//end add by fuhua
static const struct i2c_device_id aw881xx_i2c_id[] = {
	{AW881XX_I2C_NAME, 0},
	{}
};

//MODULE_DEVICE_TABLE(i2c, aw881xx_i2c_id);

static struct of_device_id aw881xx_dt_match[] = {
	{.compatible = "awinic,aw881xx_smartpa"},
	{},
};

static struct i2c_driver aw881xx_i2c_driver = {
	.driver = {
		.name = AW881XX_I2C_NAME,
		.owner = THIS_MODULE,
		.of_match_table = of_match_ptr(aw881xx_dt_match),
		},
	.probe = aw881xx_i2c_probe,
	.remove = aw881xx_i2c_remove,
	.id_table = aw881xx_i2c_id,
};

static struct platform_driver aw881xx_test_driver = {
	.driver = {
	.name = AW881XX_I2C_NAME,
	.owner = THIS_MODULE,
	.of_match_table = of_match_ptr(aw881xx_dt_match),
	},
	.probe = aw881xx_test_probe,
	.remove = aw881xx_test_remove,
	//.id_table = aw881xx_i2c_id,
};

static int __init aw881xx_i2c_init(void)
{
	int ret = -1;
	int use_i2c = 0;	

	pr_err("fuhua--aw881xx_i2c_init---begin\n");
	pr_err("%s: fuhua--check aw881xx driver version %s\n", __func__, AW881XX_VERSION);
	if(use_i2c)
	{
		pr_err("fuhua--check i2c_add_driver\n");
		ret = i2c_add_driver(&aw881xx_i2c_driver);
	}
	else
	{
		pr_err("fuhua--check platform_driver_register111111\n");
		ret = platform_driver_register(&aw881xx_test_driver);
	}
	
	if (ret)
		pr_err("%s: fail to add aw881xx device into i2c\n", __func__);

	pr_err("fuhua--aw881xx_i2c_init---end");
	return ret;
}
module_init(aw881xx_i2c_init);

static void __exit aw881xx_i2c_exit(void)
{
	int use_i2c = 0;
	pr_err("fuhua--aw881xx_i2c_exit---begin\n");
	if(use_i2c)
		i2c_del_driver(&aw881xx_i2c_driver);
	else
		platform_driver_unregister(&aw881xx_test_driver);
	pr_err("fuhua--aw881xx_i2c_exit---end\n");
}

module_exit(aw881xx_i2c_exit);

```  

其余的都差不多，主要是一些log添加的地方比较重要，方便查看是否注册成功。主要是以下文件加log：  
```  
	modified:   arch/arm/boot/dts/k62v1_32_bsp.dts
	modified:   arch/arm/boot/dts/mt6765.dts
	modified:   drivers/base/driver.c
	modified:   drivers/i2c/i2c-core.c
	modified:   sound/soc/codecs/Kconfig
	modified:   sound/soc/codecs/Makefile
	modified:   sound/soc/mediatek/codec/mt6357/mtk-soc-codec-6357.c
	modified:   sound/soc/mediatek/common_int/mtk-soc-codec-dummy.c
	modified:   sound/soc/mediatek/common_int/mtk-soc-machine.c
	modified:   sound/soc/mediatek/common_int/mtk-soc-speaker-amp.c
	modified:   sound/soc/mediatek/vendor/mtk-hw-component.c
	modified:   sound/soc/soc-core.c

Untracked files:
  (use "git add <file>..." to include in what will be committed)

	sound/soc/codecs/aw88194.c
	sound/soc/codecs/aw88194.h
	sound/soc/codecs/aw88194_reg.h
	sound/soc/codecs/aw88195.c
	sound/soc/codecs/aw88195.h
	sound/soc/codecs/aw88195_reg.h
	sound/soc/codecs/aw881xx.c
	sound/soc/codecs/aw881xx.h

```
其中，主要注意如下位置：  
```
--- a/drivers/i2c/i2c-core.c
+++ b/drivers/i2c/i2c-core.c
@@ -2188,7 +2188,7 @@ int i2c_register_driver(struct module *owner, struct i2c_driver *driver)
        if (res)
                return res;
 
-       pr_debug("driver [%s] registered\n", driver->driver.name);
+       pr_err("fuhua---check  driver [%s] registered\n", driver->driver.name);



diff --git a/sound/soc/soc-core.c b/sound/soc/soc-core.c
index b920887..576a7b1 100644
--- a/sound/soc/soc-core.c
+++ b/sound/soc/soc-core.c
@@ -958,8 +958,12 @@ struct snd_soc_dai *snd_soc_find_dai(
 
                if (dlc->of_node && component_of_node != dlc->of_node)
                        continue;
+               pr_err("fuhua snd_soc_find_dai  component->name = %s,dlc->name = %s\n",component->name,dlc->name);
                if (dlc->name && strcmp(component->name, dlc->name))
+               {
+                       pr_err("fuhua snd_soc_find_dai find name in continue component->name = %s,dlc->name = %s\n",component->name,dlc->name);
                        continue;
+               }
                list_for_each_entry(dai, &component->dai_list, list) {
                        if (dlc->dai_name && strcmp(dai->name, dlc->dai_name))
                                continue;

```


