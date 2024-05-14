+++
author = "fuhua"
title = "Android - Audio - Qcom平台 - hac器件bring up"
date = "2019-03-09"
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
    - [HAC部分硬件原理图](#3-HAC部分硬件原理图)  
    - [硬件调试需求](#4-硬件调试需求)  
    - [软件平台及框架](#5-软件平台及框架)  
        - [a:mixer_path.xml中添加hac path](#a)
        - [b:alsa用户空间，tinyalsa中注册HAC控件](#b)
        - [c:GPIO_HAC_EN](#c)
        - [d:添加acdb_id](#d)
        - [e:为MMITest添加测试项](#e)
- [结语](#结语)  


## __背景__
>公司的项目要求支持助听器（HAC器件，Hearing Aid Compatibility），以此正好再复习一下在android的框架下如何bring up一个类似PA的器件。 

## __正文__
1. 我们首先需要了解助听器的原理，使用方式（以此确定测试手段），硬件原理图（以此确定软件如何适配），硬件调试需求，软件平台以及框架（MTK和Qcom在底层有一些差别）。  

2. 既然已经明确了步骤，那么就开始一步一步来；  
    ##### (1)助听器构造:  
    >从百度百科来看，助听器有5个主要的构造，麦克风，放大器，接收器，电源和外壳。  
    其中对我们这次开发有用的就是接收器，因为mic用手机的mic就够了。  

    #### (2)使用方式:  
    >按照我拿到的助听器，上了电池之后默认是放大声音的模式，我理解的助听器基本上也是放大器的作用，但是HAC器件实际上还有一个功能，就是电圈耦合，按下助听器上的按钮触发，从以往的功能定义来看在打电话的时候该功能才打开（但是个人疑问助听器不是应该时时刻刻都帮助听障人士[更清楚的听到声音](https://baike.baidu.com/item/HAC/1525675?fr=aladdin)么，难道是怕耗电快？）。  
    知道了使用方式，我在测试效果的时候发现Receiver发出的声音实际上会影响到功能判断，所以我直接将Receiver器件和speaker器件给去掉了（去掉speaker主要是为了测试后面添加path的可行性。）  

    #### (3)HAC部分硬件原理图:  
    ```
    +-----------+  INP   +-------------+  VON   +----------+
    | CDC_EAR_M | ~~~~~> |             | ~~~~~> | HAC 线圈 |
    +-----------+        |             |        +----------+
                         |             |  VOP     ^
                         |  AW8155FCR  | ~~~~~~~~~*
                         |             |
    +-----------+  INN   |             |
    | CDC_EAR_P | ~~~~~> |             |
    +-----------+        +-------------+
                        ^
                        } INP
                        }
                        +-------------+
                        | GPIO_HAC_EN |
                        +-------------+
    ```
    从原理图我们能明显的看到有一个艾薇的放大器，通过对CDC_EAR通路的查询，发现接到了Receiver器件上，所以电信号的数据输出可以理解为既给了Receiver，会给到hac部分。  

    #### (4)硬件调试需求:  
    >和硬件同事进行沟通，发现有如下需求  
    --1. 器件bring up。  
    --2. 单独一条acdb id。  
    --3. dialer界面有设置HAC功能打开的接口。  
    --4. MMITest里面能进行测试。  

    #### (5)软件平台及框架:  
    根据需求，我们可以大致确定这需要从上到下一起打通，那么就需要像普通的PA那样单独配置一个path，而且还需要额外加一条acdb_id，以供调试。  

    然后就是和上层通信的接口，可以按照属性值的方式来，也可以按照其他方式，暂时先通过属性值来比较方便。  

    MMITest当中则需要通过path的开关来控制，所以配置一个单独的path是必须的。  

    根据以上的软件需求，我将此次的软件开发分为如下步骤：  
    __a__ : mixer_path.xml中添加hac path。  
    __b__ : 在alsa用户空间，tinyalsa中注册HAC控件。  
    __c__ : tinyalsa中操作mixer，达到想调用hac path的时候能操作GPIO_HAC_EN，数据信号则不在此处控制。  
    __d__ : hal层添加一个RX 的 acdb_id，TX已经确定是4，RX为31。  
    __e__ : MMITest测试调用的audio_test.sh中添加 HAC PATH 的开关。  
 
    ## a  
    在`hardware/qcom/audio/configs/msm8937/mixer_path_mtp.xml`中添加HAC_PA的控件声明和path操作。  
    ```
    <ctl name="EAR PA Boost" value="ENABLE" />
    <!-- begin mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 -->
    <ctl name="HAC_PA" value="On" />
    <ctl name="HAC_PA" value="Off" />
    <!-- end   mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 -->
    <ctl name="MI2S_RX Channels" value="One" />
    ```
    >在需要打开PA的场景中打开PA，此处应个人测试需求，在speaker的path中也打开了HAC_PA。  
    ```
    <path name="voice-call">
        <ctl name="PRI_MI2S_RX_Voice Mixer CSVoice" value="1" />
        <ctl name="Voice_Tx Mixer TERT_MI2S_TX_Voice" value="1" />
        <!-- begin mod by fuhua for task: 8668750 on 2019-12-03 task id:8668750 -->
        <ctl name="HAC_PA" value="On" />
        <!-- end   mod by fuhua for task: 8668750 on 2019-12-03 task id:8668750 -->
    </path>
    <path name="speaker">
        <ctl name="RX3 MIX1 INP1" value="RX1" />
        <ctl name="SPK" value="Switch" />
        <!-- begin mod by fuhua for task: 8668750 on 2019-12-03 task id:8668750 -->
        <ctl name="HAC_PA" value="On" />
        <!-- end   mod by fuhua for task: 8668750 on 2019-12-03 task id:8668750 -->
    </path>
    ```

    这个时候我们能在mixer.xml中看到配置了path，但是tinyalsa并没有注册HAC_PA这条path，在我的理解这个并不是针对注册的接口。    
    至于为什么选择的是 mixer_paths_mtp.xml ，这个可以查看`/hardware/qcom/audio/hal/msm8916/platform.c`对声卡的判断，然后选择对应的mixer_path.xml文件。  

    ## b
    __在alsa用户空间，tinyalsa中注册HAC控件。__  

    注册的时候需要用到alsa提供出来的接口`SOC_ENUM_SINGLE_EXT`，当然也有其他的接口，比如`SOC_DAPM_ENUM`、`SOC_SINGLE_TLV`；这两个分别是需要注册到动态音频电源管理模块，另外一个的注释是“Convenience kcontrol builders”，主要用于定义那些有增益控制的控件，例如音量控制器，EQ均衡器等等。  
    我的代码如下( 路径：vendor/qcom/opensource/audio-kernel/asoc/codecs/sdm660_cdc/msm-analog-cdc.c )：  
    ```  
    /*begin mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
    static const char * const hac_pa_text[] = {
    "Off", "On"};
    static const struct soc_enum hac_enum[] = {
    SOC_ENUM_SINGLE_EXT(2, hac_pa_text),
    };
    /*end   mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */

    ...

    /*begin mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
    SOC_ENUM_EXT("HAC_PA", hac_enum[0],
        hac_pa_gpio_get,hac_pa_gpio_set),
    /*end   mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
    ```
    ## c 
    实际上这里已经注册好了HAC_PA了，只要申明一下set函数和get函数，然后编译就能用tinymix看到HAC_PA的path了，但显然是不能工作的。  
    所以接下来就是定义`hac_pa_gpio_get`以及`hac_pa_gpio_set`，其中`hac_pa_set`需要额外在`sdm660_cdc_priv`结构体中添加，  
    ```
    
    /*begin mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
    static int hac_pa_gpio_get(struct snd_kcontrol *kcontrol,
                        struct snd_ctl_elem_value *ucontrol)
    {
        struct snd_soc_codec *codec = snd_soc_kcontrol_codec(kcontrol);
        struct sdm660_cdc_priv *sdm660_cdc =
                        snd_soc_codec_get_drvdata(codec);

        ucontrol->value.integer.value[0] =
            (sdm660_cdc->hac_pa_set ? 1 : 0);
        dev_err(codec->dev, "%s: fuhua  sdm660_cdc->hac_pa_set = %d\n",
                __func__, sdm660_cdc->hac_pa_set);
        return 0;
    }

    static int hac_pa_gpio_set(struct snd_kcontrol *kcontrol,
                        struct snd_ctl_elem_value *ucontrol)
    {
        struct snd_soc_codec *codec = snd_soc_kcontrol_codec(kcontrol);
        struct sdm660_cdc_priv *sdm660_cdc =
                        snd_soc_codec_get_drvdata(codec);


        dev_err(codec->dev, "%s: fuhua  ucontrol->value.integer.value[0] = %ld\n",
            __func__, ucontrol->value.integer.value[0]);
        sdm660_cdc->hac_pa_set =
            (ucontrol->value.integer.value[0] ? true : false);
        if(sdm660_cdc->hac_pa_set)
        {
            //msm_cdc_pinctrl_select_active_state(sdm660_cdc->hac_pa_gpio_p);
            //enable_hac_pa(codec,true);
                    if (sdm660_cdc->codec_hac_pa_cb)
                            sdm660_cdc->codec_hac_pa_cb(codec, 1);
        }
        else
        {
            //msm_cdc_pinctrl_select_sleep_state(sdm660_cdc->hac_pa_gpio_p);
            //enable_hac_pa(codec,false);
            if (sdm660_cdc->codec_hac_pa_cb)
                sdm660_cdc->codec_hac_pa_cb(codec, 0);
        }
        return 0;
    }


    /*end   mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */

    ```
    其中的set函数会读取HAC_PA的控件值，然后去调用`sdm660_cdc->codec_hac_pa_cb`函数，而这个函数指针在另一个地方已经赋值成了另一个函数，如下`sdm660_cdc->codec_hac_pa_cb = codec_hac_pa;`：  
    ```
    /*begin mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
    void msm_anlg_cdc_hac_pa_cb(
                int (*codec_hac_pa)(struct snd_soc_codec *codec,
                        int enable), struct snd_soc_codec *codec)
    {
        struct sdm660_cdc_priv *sdm660_cdc;

        if (!codec) {
                pr_err("%s: NULL codec pointer!\n", __func__);
                return;
        }

        sdm660_cdc = snd_soc_codec_get_drvdata(codec);

        dev_err(codec->dev, "%s: fuhua  Enter\n", __func__);
        sdm660_cdc->codec_hac_pa_cb = codec_hac_pa;
    }
    EXPORT_SYMBOL(msm_anlg_cdc_hac_pa_cb);
    /*end   mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
    ```
    这个msm_anlg_cdc_hac_pa_cb已经在msm-analog-cdc.h中已经声明成extern。刚刚我们在sdm660_cdc这个目录下完成了驱动的注册，现在来到`vendor/qcom/opensource/audio-kernel/asoc/msm8952.c`继续完成实现，这些都是在alsa的用户空间进行操作的，也就是用了接口之后的部分。我们首先要在`msm_audrx_init`函数中调用`msm_anlg_cdc_hac_pa_cb(enable_hac_pa, ana_cdc);`，以注册enable_hac_pa函数。  
    ```
        static int msm_audrx_init(struct snd_soc_pcm_runtime *rtd)
    {
        struct snd_soc_codec *dig_cdc = rtd->codec_dais[DIG_CDC]->codec;
        struct snd_soc_codec *ana_cdc = rtd->codec_dais[ANA_CDC]->codec;

        ...

        snd_soc_dapm_ignore_suspend(dapm, "EAR");
        snd_soc_dapm_ignore_suspend(dapm, "HEADPHONE");
        snd_soc_dapm_ignore_suspend(dapm, "SPK_OUT");

        ...
        
        /*begin mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
        msm_anlg_cdc_hac_pa_cb(enable_hac_pa, ana_cdc);
        /*end   mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */

    ```
    这个init函数中我们还能看到`snd_soc_dapm_ignore_suspend`函数，这就是在哪里忽略path的地方。那么接下来就是具体实现` enable_hac_pa ` ,代码如下：  
    ```
    
    /*begin mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
    static int enable_hac_pa(struct snd_soc_codec *codec, int enable)
    {
        struct snd_soc_card *card = codec->component.card;
        struct msm_asoc_mach_data *pdata = snd_soc_card_get_drvdata(card);
        int ret = 0;
        int i = 0;
        int extamp_mode = 2;  //ext pa default mode 2

        if (!gpio_is_valid(pdata->hac_pa_gpio)) {
                pr_err("%s: Invalid gpio: %d\n", __func__,
                        pdata->hac_pa_gpio);
                return false;
        }

        pr_err("%s: %s fuhua HAC PA\n", __func__,
                enable ? "Enable" : "Disable");

        if (enable) {
                for( i=0 ; i<extamp_mode ; i++)
                {
                        ret = msm_cdc_pinctrl_select_sleep_state(pdata->hac_pa_gpio_p);
                        if (ret) {
                                pr_err("%s: gpio set cannot be de-sleeped %s\n ",__func__,"lo_spk_gpio");
                                return ret;
                        }
                        udelay(2);
                        ret =  msm_cdc_pinctrl_select_active_state(
                                                pdata->hac_pa_gpio_p);
                        if (ret) {
                                pr_err("%s: gpio set cannot be de-activated %s\n",
                                                __func__, "lo_spk_gpio");
                                return ret;
                        }

                        pr_err("%s: %s i=%d\n ",__func__,
                                enable ? "Enable" : "Disable",i);
                }
                gpio_set_value_cansleep(pdata->hac_pa_gpio, enable);
        } else {
                gpio_set_value_cansleep(pdata->hac_pa_gpio, enable);
                    pr_err("%s: %s\n ",__func__,
                                enable ? "Enable" : "Disable");
                ret = msm_cdc_pinctrl_select_sleep_state(
                                pdata->hac_pa_gpio_p);
                if (ret) {
                        pr_err("%s: gpio set cannot be de-activated %s\n",
                                        __func__, "lo_spk_gpio");
                        return ret;
                }
        }
        return 0;
    }
    /*end   mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */

    ```
    其中定义了pa的模式，extamp_mode，剩下的主要操作就是通过Qcom的接口开关引脚了。  
    `msm_cdc_pinctrl_select_active_state(pdata->hac_pa_gpio_p);`  
    `msm_cdc_pinctrl_select_sleep_state(pdata->hac_pa_gpio_p);`  
    `gpio_set_value_cansleep(pdata->hac_pa_gpio, enable);`  
    但是控制的时候肯定不能直接指定端口嘛，这种太不便于管理了，我们是有管理的公司，要为后来的产品造福，不要写些烂代码，虽然繁琐。所以需要在probe的时候遍历一遍：  
    ```
    static int msm8952_asoc_machine_probe(struct platform_device *pdev)
    {
        struct snd_soc_card *card;
        struct msm_asoc_mach_data *pdata = NULL;
        const char *hs_micbias_type = "qcom,msm-hs-micbias-type";
        const char *ext_pa = "qcom,msm-ext-pa";
        const char *mclk = "qcom,msm-mclk-freq";
        const char *wsa = "asoc-wsa-codec-names";
        const char *wsa_prefix = "asoc-wsa-codec-prefixes";
        const char *type = NULL;
        const char *ext_pa_str = NULL;
        const char *wsa_str = NULL;
        const char *wsa_prefix_str = NULL;
        const char *spk_ext_pa = "qcom,msm-spk-ext-pa";
        /*begin mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
        const char *hac_pa = "qcom,msm-hac-pa";
        const char *hac_pa_p = "qcom,msm-hac-pa-gpio";
        /*end   mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
        int num_strings;

        ...

        pr_debug("%s: ext_pa = %d\n", __func__, pdata->ext_pa);
        /*begin mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
        pdata->hac_pa_gpio = of_get_named_gpio(pdev->dev.of_node, hac_pa, 0);
        if (pdata->hac_pa_gpio < 0) {
                dev_err(&pdev->dev, "%s: missing %s in dt node\n", __func__, hac_pa);
        }

        pdata->hac_pa_gpio_p = of_parse_phandle(pdev->dev.of_node, hac_pa_p, 0);
        if (pdata->hac_pa_gpio_p < 0) {
                dev_err(&pdev->dev, "%s: missing %s in dt node\n", __func__, hac_pa_p);
        }
        /*end   mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
        pdata->spk_ext_pa_gpio = of_get_named_gpio(pdev->dev.of_node,
                                spk_ext_pa, 0);
        if (pdata->spk_ext_pa_gpio < 0) {

    ```
    以上代码就是读取dts中的定义，那么dts中又添加了什么呢？(我是在kernel/msm-4.9/arch/arm64/boot/dts/qcom/qm215-audio.dtsi中添加的，当然可以选择其他地方)  
    ```
    
    /*begin mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
    &tlmm {
        hac_pa {
                hac_pa_high: hac_pa_high {
                        mux {
                                pins = "gpio127";
                                function = "gpio";
                        };

                        config {
                                pins = "gpio127";
                                drive-strength = <8>; /* 8mA */
                                bias-disable = <0>; /* no pull */
                                output-high;
                        };
                };

                hac_pa_low: hac_pa_low {
                        mux {
                                pins = "gpio127";
                                function = "gpio";
                        };

                        config {
                                pins = "gpio127";
                                drive-strength = <2>;
                                output-low;
                                bias-pull-down;
                        };
                };
        };
    };
    /*end   mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
    &soc {
        qcom,msm-audio-apr {
            compatible = "qcom,msm-audio-apr";
            msm_audio_apr_dummy {
                compatible = "qcom,msm-audio-apr-dummy";
            };
        };

        qcom,avtimer@c0a300c {
            compatible = "qcom,avtimer";
            reg = <0x0c0a300c 0x4>,
                <0x0c0a3010 0x4>;
            reg-names = "avtimer_lsb_addr", "avtimer_msb_addr";
            qcom,clk-div = <27>;
        };
        /*begin mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
        cdc_hac_pa_gpio: msm_cdc_pinctrl@127 {
            compatible = "qcom,msm-cdc-pinctrl";
            pinctrl-names = "aud_active", "aud_sleep";
            pinctrl-0 = <&hac_pa_high>;
            pinctrl-1 = <&hac_pa_low>;
        };
        /*end   mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */

        int_codec: sound {
            status = "okay";
            compatible = "qcom,msm8952-audio-codec";
            qcom,model = "msm8952-snd-card-mtp";
            reg = <0xc051000 0x4>,
                <0xc051004 0x4>,
                <0xc055000 0x4>,
                <0xc052000 0x4>;
            reg-names = "csr_gp_io_mux_mic_ctl",
                "csr_gp_io_mux_spkr_ctl",
                "csr_gp_io_lpaif_pri_pcm_pri_mode_muxsel",
                "csr_gp_io_mux_quin_ctl";

            qcom,msm-ext-pa = "primary";
    /*begin mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
            qcom,msm-hac-pa = <&tlmm 127 0>;
            qcom,msm-hac-pa-gpio = <&cdc_hac_pa_gpio>;
    /*end   mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
            qcom,msm-mclk-freq = <9600000>;
            qcom,msm-mbhc-hphl-swh = <1>;
            qcom,msm-mbhc-gnd-swh = <1>;
    ```

    这些添加完成之后，基本上就能通过tinyalsa去控制“HAC_PA”的On/Off了，达到想打开GPIO_HAC_EN的时候就能打开。  

    ## d  
    那么硬件的同事还有两个需求，第一个就是配对一个acdb_id,这样能方便他们配对调试，那么在hal层需要做什么呢?(目录：hardware/qcom/audio)由于比较零散，也比较简单，知识点不多，所以直接贴上diff文件吧，这一份是比较早的代码备份，有一点逻辑是有问题的。文件直接通过`git diff ./ > diff.txt`导出来就可以了。  

    ```
    diff --git a/hal/audio_hw.c b/hal/audio_hw.c
    index 3404dd6..bed0ec3 100644
    --- a/hal/audio_hw.c
    +++ b/hal/audio_hw.c
    @@ -6866,6 +6866,23 @@ static int adev_set_parameters(struct audio_hw_device *dev, const char *kvpairs)
        status = platform_set_parameters(adev->platform, parms);
        if (status != 0)
            goto done;
    +/*begin mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
    +    ret = str_parms_get_str(parms, "odm.hac.enable", value, sizeof(value));
    +    if(ret >= 0)
    +    {
    +        ALOGE("fuhua ---- get odm.hac.enable = %s",value);
    +        if(strcmp(value,AUDIO_PARAMETER_VALUE_ON) == 0)
    +        {adev->hac_enable = true;}
    +        else
    +        {adev->hac_enable = false;}
    +    }
    +    else
    +    {
    +        ALOGE("fuhua ---- not get the odm.hac.enable = %s",value);
    +    }
    +
    +/*end   mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
    +
    
        ret = str_parms_get_str(parms, AUDIO_PARAMETER_KEY_BT_NREC, value, sizeof(value));
        if (ret >= 0) {
    diff --git a/hal/audio_hw.h b/hal/audio_hw.h
    index f791b6a..3865c0f 100644
    --- a/hal/audio_hw.h
    +++ b/hal/audio_hw.h
    @@ -493,6 +493,9 @@ struct audio_device {
        bool allow_afe_proxy_usage;
        bool is_charging; // from battery listener
        bool mic_break_enabled;
    +	/*begin mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
    +    bool hac_enable;
    +	/*end   mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
    
        int snd_card;
        card_status_t card_status;
    diff --git a/hal/msm8916/platform.c b/hal/msm8916/platform.c
    index aeb3abe..7a8d94e 100644
    --- a/hal/msm8916/platform.c
    +++ b/hal/msm8916/platform.c
    @@ -488,7 +488,9 @@ static const char * const device_table[SND_DEVICE_MAX] = {
        [SND_DEVICE_OUT_VOIP_SPEAKER] = "voip-speaker",
        [SND_DEVICE_OUT_VOIP_HEADPHONES] = "voip-headphones",
    #endif
    -
    +	/*begin mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
    +    [SND_DEVICE_OUT_VOICE_HANDSET_HAC] = "voice-handset-hac",
    +	/*end   mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
        /* Capture sound devices */
        [SND_DEVICE_IN_HANDSET_MIC] = "handset-mic",
        [SND_DEVICE_IN_HANDSET_MIC_EXTERNAL] = "handset-mic-ext",
    @@ -668,6 +670,9 @@ static int acdb_device_table[SND_DEVICE_MAX] = {
        [SND_DEVICE_OUT_VOIP_SPEAKER] = 132,
        [SND_DEVICE_OUT_VOIP_HEADPHONES] = 134,
    #endif
    +	/*begin mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
    +    [SND_DEVICE_OUT_VOICE_HANDSET_HAC] = 31,
    +	/*end   mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
    
        [SND_DEVICE_IN_HANDSET_MIC] = 4,
        [SND_DEVICE_IN_HANDSET_MIC_EXTERNAL] = 4,
    @@ -832,6 +837,10 @@ static struct name_to_index snd_device_name_index[SND_DEVICE_MAX] = {
        {TO_NAME_INDEX(SND_DEVICE_OUT_VOIP_SPEAKER)},
        {TO_NAME_INDEX(SND_DEVICE_OUT_VOIP_HEADPHONES)},
    #endif
    +	/*begin mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
    +    {TO_NAME_INDEX(SND_DEVICE_OUT_VOICE_HANDSET_HAC)},
    +	/*end   mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
    +    
        {TO_NAME_INDEX(SND_DEVICE_IN_HANDSET_MIC)},
        {TO_NAME_INDEX(SND_DEVICE_IN_HANDSET_MIC_EXTERNAL)},
        {TO_NAME_INDEX(SND_DEVICE_IN_HANDSET_MIC_AEC)},
    @@ -4307,6 +4316,18 @@ snd_device_t platform_get_output_snd_device(void *platform, struct stream_out *o
                    snd_device = SND_DEVICE_OUT_ANC_HANDSET;
                else
                    snd_device = SND_DEVICE_OUT_VOICE_HANDSET;
    +			/*begin mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
    +            if(adev->hac_enable == true)
    +                {
    +                    ALOGE("fuhua  --- adev->hac_enable == true");
    +                    snd_device = SND_DEVICE_OUT_VOICE_HANDSET_HAC;
    +                }
    +            else
    +                {
    +                    ALOGE("fuhua  --- adev->hac_enable == false or not get value");
    +                    snd_device = SND_DEVICE_OUT_VOICE_HANDSET_HAC;
    +                }
    +			/*end   mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
            } else if (devices & AUDIO_DEVICE_OUT_TELEPHONY_TX)
                snd_device = SND_DEVICE_OUT_VOICE_TX;
    
    diff --git a/hal/msm8916/platform.h b/hal/msm8916/platform.h
    index f37819d..e0dbcfa 100644
    --- a/hal/msm8916/platform.h
    +++ b/hal/msm8916/platform.h
    @@ -150,6 +150,9 @@ enum {
        SND_DEVICE_OUT_VOIP_SPEAKER,
        SND_DEVICE_OUT_VOIP_HEADPHONES,
    #endif
    +	/*begin mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
    +    SND_DEVICE_OUT_VOICE_HANDSET_HAC,
    +	/*end   mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
        SND_DEVICE_OUT_VOICE_SPEAKER_AND_VOICE_HEADPHONES,
        SND_DEVICE_OUT_VOICE_SPEAKER_AND_VOICE_ANC_HEADSET,
        SND_DEVICE_OUT_VOICE_SPEAKER_STEREO_AND_VOICE_HEADPHONES,
    @@ -292,8 +295,10 @@ enum {
    #define MIXER_PATH_MAX_LENGTH 100
    #define SND_CARD_MAX_LENGTH 100
    #define CODEC_VERSION_MAX_LENGTH 100
    -
    -#define MAX_VOL_INDEX 5
    +/*begin mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
    +//#define MAX_VOL_INDEX 5
    +#define MAX_VOL_INDEX 7
    +/*end   mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
    #define MIN_VOL_INDEX 0
    #define percent_to_index(val, min, max) \
                ((val) * ((max) - (min)) * 0.01 + (min) + .5)
    ```
    ## e  
    完成audio_test.sh中添加开关。实际上只要打通了 HAC_PA 的path，剩下的都不难了吧。  
    ```

    ###################### receiver playback test ######################
    if [ $1 == 'handset' ];then
    #   echo $$ >> /data/audio_test_receiver.pid
    #    echo  'log log ' >> /sdcard/log.txt
    #    setprop "tinyplay_status" "on"
        tinymix_v2 'PRI_MI2S_RX Audio Mixer MultiMedia1' '1'
        tinymix_v2 'RX1 MIX1 INP1' 'RX1'
        tinymix_v2 "RDAC2 MUX" 'RX1'
        tinymix_v2 'RX1 Digital Volume' '84'
        tinymix_v2 "EAR PA Gain" "POS_6_DB"
        tinymix_v2 "EAR_S" 'Switch'
        #begin mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750
        tinymix_v2 "HAC_PA" 'On'
        #end   mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750

        while [ "1" == "1" ]
        do
            tinyplay_v2 /system/etc/112.wav -d 0
        done
    fi

    if [ $1 == 'handset-stop' ];then
    #   setprop "tinyplay_status" "off"
    #    if [ -f /data/audio_test_receiver.pid ];then
    #      pid=`cat /data/audio_test_receiver.pid`
    #     kill -9 $pid
    #     rm /data/audio_test_receiver.pid
    #   fi
        #begin mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750
        tinymix_v2 "HAC_PA" 'Off'
        #end   mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750
        tinymix_v2 'RX1 MIX1 INP1' '0'
        tinymix_v2 "RDAC2 MUX" 'ZERO'
        tinymix_v2 'RX1 Digital Volume' '67'
        tinymix_v2 "EAR PA Gain" 'POS_1P5_DB'
        tinymix_v2 "EAR_S" 'ZERO'
        tinymix_v2 'PRI_MI2S_RX Audio Mixer MultiMedia1' '0'
    fi


    ```
## __结语__  

1. 那么至此，整个hac器件的bring up就告一段落了，剩下的可能是一些维护工作？  
我在自己调试的时候发现，单单的没有`tinymix_v2 "EAR PA Gain" "POS_6_DB"`增益的设置，打电话的时候是不会出声的，向硬件同事请教，说是由于电信号转磁信号效率低，而hac器件还得从磁信号又转一遍电信号，那更低了；而电信号转机械信号效率相对较高，即Receiver能出声而hac器件不能出声的道理。  
2. 期间，发现hac器件噪声严重，由于布局的原因，加上屏幕的干扰，实际上是有干扰的，背光的信号频率很高。  
---




    

