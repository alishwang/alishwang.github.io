+++
author = "fuhua"
title = "Android - Audio - MT67平台 - hac器件bring up"
date = "2019-12-30 23:56:11"
description = "手机技术笔记"
tags = [
    "markdown",
    "text",
]
weight = 10
+++


- [背景](#背景)  
- [正文](#正文)  
    - [助听器构造](#1-助听器构造)  
    - [使用方式](#2-使用方式)  
    - [HAC部分硬件原理图](#3-硬件原理图)  
    - [硬件调试需求](#4-硬件调试需求)  
    - [软件平台及框架](#5-软件平台及框架)  
        - [1:audio_device.xml中添加hac path](#1)
        - [2:alsa用户空间，tinyalsa中注册HAC控件](#2)
        - [3:HAL层中的定制](#3)
        - [4:MMITest软件中添加HAC测试的思路](#4)
- [结语](#结语)  


## __背景__
>公司的项目要求支持助听器（HAC器件，Hearing Aid Compatibility），以此正好再复习一下在android的框架下(基于MTK平台)如何bring up一个类似PA的器件。 

## __正文__
1. 我们首先需要了解助听器的原理，使用方式（以此确定测试手段），硬件原理图（以此确定软件如何适配），硬件调试需求，软件平台以及框架（MTK和Qcom在底层有一些差别）。  

2. 既然已经明确了步骤，那么就开始一步一步来；  
    #### (1) 助听器构造:  
    >从百度百科来看，助听器有5个主要的构造，麦克风，放大器，接收器，电源和外壳。  
    其中对我们这次开发有用的就是接收器，因为mic用手机的mic就够了。  

    ### (2) 使用方式:  
    >按照我拿到的助听器，上了电池之后默认是放大声音的模式，我理解的助听器基本上也是放大器的作用，但是HAC器件实际上还有一个功能，就是电圈耦合，按下助听器上的按钮触发，从以往的功能定义来看在打电话的时候该功能才打开（但是个人疑问助听器不是应该时时刻刻都帮助听障人士[更清楚的听到声音](https://baike.baidu.com/item/HAC/1525675?fr=aladdin)么，难道是怕耗电快？）。  
    知道了使用方式，我在测试效果的时候发现Receiver发出的声音实际上会影响到功能判断，所以我直接将Receiver器件和speaker器件给去掉了（去掉speaker主要是为了测试后面添加path的可行性。）  

    ### (3) 硬件原理图:    
    ```
    +-----------+  INP   +-------------+  VON   +----------+
    |   AU_HSP  | ~~~~~> |             | ~~~~~> | HAC 线圈 |
    +-----------+        |             |        +----------+
                         |             |  VOP     ^
                         |  AW8155FCR  | ~~~~~~~~~*
                         |             |
    +-----------+  INN   |             |
    |   AU_HSN  | ~~~~~> |             |
    +-----------+        +-------------+
                        ^
                        } CTRL
                        }
                        +-------------+
                        | GPIO_HAC_EN |
                        +-------------+
    ```
    从原理图我们能明显的看到有一个艾为的放大器，通过对AU_HSP通路的查询，发现接到了Receiver器件上，所以电信号的数据输出可以理解为既给了Receiver，会给到hac部分。  

    ### (4) 硬件调试需求:  
    >和硬件同事进行沟通，发现有如下需求  
    --1. 器件bring up。  
    --2. 打电话的时候hac器件能听到声音就行，因为只是兼容设计，后面不会贴的。  

    ### (5) 软件平台及框架:  
    根据需求，我们可以大致确定这需要从上到下一起打通，那么就需要像普通的PA那样单独配置一个通路，如果后期要贴，也能留下接口。  

    和上层Dialer通信的接口google已经预留有了HACSetting这样的字符串，但是MTK用了自己定义的。  

    根据以上的软件需求，我将此次的软件开发分为如下步骤：  
    __1__ : audio_device.xml中添加hac path。  
    __2__ : 在alsa用户空间，tinyalsa中注册HAC控件。  
    __3__ : HAC层中添加HAC与Receiver同时打开的接口。  
    __4__ : MMITest软件中添加HAC测试的思路  
 
    #### 1   
    #### audio_device.xml中添加hac path。  
    在`device/mediatek/mt6765/audio_device.xml`中添加HAC_PA的控件声明和path操作。  
    ```
	<!--begin add by fuhua for task 8834760 on 2020-02-19-->
    <path name="hac_output" value="turnon">
        <kctl name="Hac_Switch" value="On" />
    </path>
    <path name="hac_output" value="turnoff">
        <kctl name="Hac_Switch" value="Off" />
    </path>
	<!--end   add by fuhua for task 8834760 on 2020-02-19-->
    <!--receiver output-->
    <path name="receiver_output" value="turnon">
        <kctl name="Voice_Amp_Switch" value="On" />
		<kctl name="Hac_Switch" value="On" />
    </path>
    <path name="receiver_output" value="turnoff">
        <kctl name="Voice_Amp_Switch" value="Off" />
		<kctl name="Hac_Switch" value="Off" />
    </path>
    ```

    由于只是兼容，为了更方便测试，我直接就在BSP版本中打开Receiver的时候打开HAC了，这样方便测试。  

    ## 2   
    ## 在kernel当中注册SOC_ENUM_EXT而不是SOC_ENUM_SINGLE_EXT，然后添加实际引脚操作  

    MTK定制控件的文件路径是`kernel-4.9\sound\soc\mediatek\codec\mt6357\mtk-soc-codec-6357.c`，具体代码位置这个主要和平台定制有关，Qcom就是在vendor下的。
	
```  
    /* Begin modify by fuhua.wang for task 8834760 on 2020-02-19 */
    static int Hac_Switch_Get(struct snd_kcontrol *kcontrol,
                    struct snd_ctl_elem_value *ucontrol)
    {
        /* pr_debug("%s()\n", __func__); */
        pr_err("fuhua--check HAC function!! in Hac_Switch_Get\n");
        ucontrol->value.integer.value[0] =
            mCodec_data->mAudio_Ana_DevicePower
                [AUDIO_ANALOG_DEVICE_OUT_EXTSPKAMP];
        return 0;
    }
    static int Hac_Switch_Set(struct snd_kcontrol *kcontrol,
                    struct snd_ctl_elem_value *ucontrol)
    {
        pr_err("%s() :fuhua -- check gain = %ld\n ", __func__,
            ucontrol->value.integer.value[0]);
        if (ucontrol->value.integer.value[0]) {
            AudDrv_GPIO_HAC_PA_Select(true);
        } else {
            AudDrv_GPIO_HAC_PA_Select(false);
        }
        return 0;
    }
    /* End   modify by fuhua.wang for task 8834760 on 2020-02-19 */

    ......

    /* Begin modify by fuhua.wang for task 8834760 on 2020-02-20*/
    static const char * const hac_switch[] = { "Off", "On"};
    /* End   modify by fuhua.wang for task 8834760 on 2020-02-20*/

    ......

    static const struct soc_enum Audio_DL_Enum[] = {

        ......

        /* Begin modify by fuhua.wang for task 8834760 on 2020-02-19*/
        SOC_ENUM_SINGLE_EXT(ARRAY_SIZE(hac_switch), hac_switch),
        /* End   modify by fuhua.wang for task 8834760 on 2020-02-19 */
    };
    static const struct snd_kcontrol_new mt6357_snd_controls[] = {    

        ......

        /* Begin modify by fuhua.wang for task 8834760 on 2020-02-19*/
        SOC_ENUM_EXT("Hac_Switch", Audio_DL_Enum[15],
                Hac_Switch_Get,
                Hac_Switch_Set),
        /* End   modify by fuhua.wang for task 8834760 on 2020-02-19 */

    ...

```

添加引脚操作的时候，需要注意最好是在dtsi或者mtk提供的dws中添加，然后再在代码中进行操作，保证代码干净整洁易懂易维护。

在`kernel-4.9\arch\arm64\boot\dts\mediatek\k62v1_64_bsp.dts`中添加引脚操作

```

    /* AUDIO GPIO standardization */
    &audgpio {

        ......

        /* Begin modify by fuhua.wang for task 8834760 on 2020-02-19*/
        pinctrl-10 = <&hac_pins_on>;
        pinctrl-11 = <&hac_pins_off>;
        /* End   modify by fuhua.wang for task 8834760 on 2020-02-19 */

        ......

        /* Begin modify by fuhua.wang for task 8834760 on 2020-02-19*/
        hac_pins_on: hac_pins_on {
            pins_cmd_dat {
                pinmux = <PINMUX_GPIO154__FUNC_GPIO154>;
                slew-rate=<1>;
                output-high;
            };
        };

        hac_pins_off: hac_pins_off {
            pins_cmd_dat {
                pinmux = <PINMUX_GPIO154__FUNC_GPIO154>;
                slew-rate=<1>;
                output-low;
            };
        };
        /* End   modify by fuhua.wang for task 8834760 on 2020-02-19 */
```

还有mtk特有的dws文件中修改初始引脚电平，不然会出现开机hac器件就开始工作的情况，会有功耗的损耗。  

```
            <gpio154>
                <eint_mode>false</eint_mode>
                <def_mode>0</def_mode>
                <inpull_en>false</inpull_en>
                <inpull_selhigh>false</inpull_selhigh>
                <def_dir>OUT</def_dir>
                <out_high>false</out_high>
                <smt>false</smt>
                <ies>true</ies>
            </gpio154>

```

然后在实现引脚操作的位置添加具体操作逻辑，`kernel-4.9\sound\soc\mediatek\common_int\mtk-auddrv-gpio.c`。  

```
    /* Begin modify by fuhua.wang for task 8834760 on 2020-02-19 */
	GPIO_HAC_PA_HIGH,
	GPIO_HAC_PA_LOW,
	/* End   modify by fuhua.wang for task 8834760 on 2020-02-19 */

    ......

    /* Begin modify by fuhua.wang for task 8834760 on 2020-02-19 */
    [GPIO_HAC_PA_HIGH] = {"hac_pins_on", false, NULL},
    [GPIO_HAC_PA_LOW] = {"hac_pins_off", false, NULL},
    /* End   modify by fuhua.wang for task 8834760 on 2020-02-19 */

    ......

    /* Begin modify by fuhua.wang for task 8834760 on 2020-02-19 */
    int AudDrv_GPIO_HAC_PA_Select(int bEnable)
    {
    int retval = 0;

    #if MT6755_PIN

    mutex_lock(&gpio_request_mutex);
    if (bEnable == 1) {
        pr_err("fuhua--check in %s hac enable!!!\n",__func__);
        if (aud_gpios[GPIO_HAC_PA_HIGH].gpio_prepare) {
            retval = pinctrl_select_state(
                pinctrlaud,
                aud_gpios[GPIO_HAC_PA_LOW].gpioctrl);
            if (retval)
                pr_info("could not set aud_gpios[GPIO_HAC_PA_LOW] pins\n");
            udelay(2);
            retval = pinctrl_select_state(
                pinctrlaud,
                aud_gpios[GPIO_HAC_PA_HIGH].gpioctrl);
            if (retval)
                pr_info("could not set aud_gpios[GPIO_HAC_PA_HIGH] pins\n");
            udelay(2);
        }
    } else {
        pr_err("fuhua--check in %s hac disenable!!!\n",__func__);
        if (aud_gpios[GPIO_HAC_PA_LOW].gpio_prepare) {
            retval = pinctrl_select_state(
                pinctrlaud,
                aud_gpios[GPIO_HAC_PA_LOW].gpioctrl);
            if (retval)
                pr_info("could not set aud_gpios[GPIO_HAC_PA_LOW] pins\n");
        }
    }
    mutex_unlock(&gpio_request_mutex);
    #endif
    return retval;
    }
    /* End   modify by fuhua.wang for task 8834760 on 2020-02-19 */
```

然后在.h文件中进行定义即可在逻辑代码中用到了(`kernel-4.9\sound\soc\mediatek\common_int\mtk-auddrv-gpio.h`)  

```
    /* Begin modify by fuhua.wang for task 8834760 on 2020-02-19 */
    int AudDrv_GPIO_HAC_PA_Select(int bEnable);
    /* End   modify by fuhua.wang for task 8834760 on 2020-02-19 */
    
```


## 3  
## HAL层中操作
MTK的HAL层中的操作是以设备为重心的，看了代码之后会有很深刻的体会。  
针对HAC的修改我这边主要是在`vendor\mediatek\proprietary\hardware\audio\common\V3\aud_drv\AudioALSAHardwareResourceManager.cpp`进行的。
```
    //add by fuhua
    #include "SpeechEnhancementController.h"

    ......

    status_t AudioALSAHardwareResourceManager::OpenReceiverPath(const uint32_t SampleRate __unused) {
	String8 value_str;
    if (IsAudioSupportFeature(AUDIO_SUPPORT_2IN1_SPEAKER)) {
        mDeviceConfigManager->ApplyDeviceTurnonSequenceByName(AUDIO_DEVICE_2IN1_SPEAKER);
    } else {
        mDeviceConfigManager->ApplyDeviceTurnonSequenceByName(AUDIO_DEVICE_RECEIVER);
    }

	//debug by fuhua
	bool bEnable = SpeechEnhancementController::GetInstance()->GetHACOn();
	ALOGE("fuhua---check --- OpenReceiverPath:HACsetting = %d",bEnable);
	if(bEnable == true)
	{mDeviceConfigManager->ApplyDeviceTurnonSequenceByName(AUDIO_DEVICE_HAC_SWITCH);}
	else
	{;}
    
    ......

    status_t AudioALSAHardwareResourceManager::CloseReceiverPath() {
    

    ......


	//debug by fuhua
	bool bEnable = SpeechEnhancementController::GetInstance()->GetHACOn();
	ALOGE("fuhua---check --- CloseReceiverPath:HACsetting = %d",bEnable);
	if(bEnable == false)
	{mDeviceConfigManager->ApplyDeviceTurnoffSequenceByName(AUDIO_DEVICE_HAC_SWITCH);}
	else
	{;}

```
然后就是在`vendor\mediatek\proprietary\hardware\audio\common\V3\include\AudioALSADeviceConfigManager.h`中定义`AUDIO_DEVICE_HAC_SWITCH`

```
/* Begin modify by fuhua.wang for task 7598168 for audio-bring-up*/
#define AUDIO_DEVICE_SPEAKER_VOICE					"ext_speaker_voice_output"
/* End   modify by fuhua.wang for task 7598168 for audio-bring-up*/
#define AUDIO_DEVICE_2IN1_SPEAKER       "two_in_one_speaker_output"
#define AUDIO_DEVICE_RECEIVER                "receiver_output"
/* Begin modify by fuhua.wang for task 8834760 on 2020-02-19*/
#define AUDIO_DEVICE_HAC_SWITCH		"hac_output"
/* End   modify by fuhua.wang for task 8834760 on 2020-02-19*/

```
通过`ApplyDeviceTurnoffSequenceByName`接口就可以调用path了。而Qcom是直接用的tinymix接口，并不像MTK这样封装了一层又一层，除非MTK想的是这套软件还有其他业务。 相对Qcom而言，MTK在定制这一方面做了很多工作，自己需要写的代码量很少，但是看代码，理逻辑，debug的时间又多做起来又繁琐，虽然Qcom需要写的代码量大，但是算上debug和定制的时间，基本持平，自由度还高一点，只要不怕难，我觉得这样还好一些。     

## 4  
## MMITest软件中添加HAC测试的思路  
理论上我们能在`frameworks/av/media/libaudioclient/` 找到 `AudioSystem.cpp	67 static const char* keyHACSetting = "HACSetting";`  

进一步寻找，可以发现`packages/services/Telephony/src/com/android/phone/PhoneGlobals.java`有  
```
390              AudioManager audioManager = (AudioManager) getSystemService(Context.AUDIO_SERVICE);
391              audioManager.setParameter(SettingsConstants.HAC_KEY,
392                      hac == SettingsConstants.HAC_ENABLED
393                              ? SettingsConstants.HAC_VAL_ON : SettingsConstants.HAC_VAL_OFF);
```
所以可以尝试在MMI中能否使用该接口，理论上应该是可以的。因为MMITest里面也有这样的代码：  
```
HeadsetTest.java:253:            mAudioManager.setParameters("SET_LOOPBACK_TYPE=2,2");
HeadsetTest.java:257:            mAudioManager.setParameters("SET_LOOPBACK_TYPE=0,0");

```

## __结语__  

1. 那么至此，基于MTK平台的hac器件bring up就告一段落了，除了MMITest没有验证之外，剩下的可能就是一些维护工作了。  

2. 不知道这次的项目上hac器件噪声会不会严重，之前Qcom项目上由于布局的原因，加上屏幕的干扰，实际上是有干扰的，因为背光的信号频率很高。  
---




    

