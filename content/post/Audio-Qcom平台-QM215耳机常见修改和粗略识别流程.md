+++
author = "fuhua"
title = "Android - Audio - Qcom平台 - QM215耳机常见修改和粗略识别流程"
date = "2020-04-07 11:40:15"
description = "手机技术笔记"
tags = [
    "markdown",
    "text",
]
weight = 10
+++

- [背景](#背景)  
- [耳机的分类](#耳机的分类)  
- [调试Qcom耳机功能时常用修改](#调试Qcom耳机功能时常用修改)  
	- [(1)qcom,msm-mbhc-hphl-swh = <1>;](#1-qcom-msm-mbhc-hphl-swh-lt-1-gt)  
	- [(2)qcom,msm-hs-micbias-type="internal";](#2-qcom-msm-hs-micbias-type-”internal”)  
	- [(3)耳机的micbias(常用micbias2)是否有外部电容](#3-耳机的micbias-常用micbias2-是否有外部电容)  
	- [(4)耳机是否支持欧美标转换](#4-耳机是否支持欧美标转换)  
	- [(5)更改micbias电压](#5-更改micbias电压)  
	- [(6)修改识别耳机时候的阻抗](#6-修改识别耳机时候的阻抗)  
	- [(7)Lineout设备](#7-Lineout设备)  
	- [(8)如果没有兼容欧标美标开关，但是想播放音乐(不能通话),可以做如下修改](#8-如果没有兼容欧标美标开关，但是想播放音乐-不能通话-可以做如下修改)  
	- [(9)对耳机按键阻值等调控](#9-对耳机按键阻值等调控)  
- [软件逻辑](#软件逻辑)  
	- [在哪儿识别](#在哪儿识别)  
	- [在哪儿上报](#在哪儿上报)  
	- [按键识别](#按键识别)  
- [综述](#综述)  





## 背景
>耳机部分也是在工作范围内，所以有必要整理，方便回顾和查找。  


## 耳机的分类  

3.5mm耳机接口分为三段式和四段式， 三段式耳机即"左右地"，四段式则带麦分为"左右地麦"（美标）和"左右麦地"（欧标）。  
type c和usb耳机则另算。我这边还没有研究，暂时就不写了。  


## 调试Qcom耳机功能时常用修改  
这一部分既是为了方便回顾和查询，也可以列举debug点。  
先说dtsi的位置，常见配置都在dtsi里：  
kernel/msm-4.9/arch/arm64/boot/dts/qcom/qm215-audio.dtsi   
```
/*begin mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
	qcom,msm-hac-pa = <&tlmm 127 0>;
	qcom,msm-hac-pa-gpio = <&cdc_hac_pa_gpio>;
/*end   mod by fuhua for task: 8668750 on 2019-12-03 task id :8668750 */
	qcom,msm-mclk-freq = <9600000>;
		qcom,msm-mbhc-hphl-swh = <1>;
/*Begin Modified by fuhua.wang for defect_id: 8751408 on 2019 02.25 */
	qcom,msm-mbhc-gnd-swh = <0>;
/*End   Modified by fuhua.wang for defect_id: 8751408 on 2019 02.25 */
/*begin mod by fuhua for 4-plog headset on 2019-08-30 task id :8293309 */
		qcom,msm-hs-micbias-type = "internal";
/*end mod by fuhua for 4-plog headset on 2019-08-30 task id :8293309 */
```

### (1)qcom,msm-mbhc-hphl-swh=<1>;  
//0是NC，1是NO  
NO是指耳机的accdet脚默认上拉到1.8v，插入耳机后，accdet脚跟左声道短接到一块,不插入耳机的时候左声道和det就是断开的，电平拉低。而NC是指耳机的accdet脚默认和左声道短接到一块，为低电平，插入耳机后，accdet脚与左声道断开，accdet脚变为高电平。  
seoul项目就是不接的时候断开的，也可以设置为0做实验，发现开机就有耳机图标。  


### (2)qcom,msm-hs-micbias-type="internal";  
这个是设置Handset/Headset micbias有没有外接电容。  
seoul原理图画的是个电阻，但是却没贴。  

如果micbias电压是内部接过去的：
qcom,msm-hs-micbias-type = "internal";
"MIC BIAS Internal2", "Headset Mic",
"AMIC2", "MIC BIAS Internal2",
如果micbias电压是外部接过去的：
qcom,msm-hs-micbias-type = "external";
"MIC BIAS External2", "Headset Mic",
"AMIC2", "MIC BIAS External2", 

但是也有针对Handset进行设置的。  
如果Handset用的是外部micbias，使用如下配置：  
"MIC BIAS External","Handset Mic"，  
"AMIC1","MIC BIAS External",  
如果Handset用的是内部Micbias，使用如下配置：
"MIC BIAS Internal","Handset Mic",
"AMIC1","MIC BIAS Internal",  

上述应该是说的手持模式和耳机模式，针对这一块，seoul设置是对的。  

```
106 		qcom,audio-routing =
107 				"RX_BIAS", "MCLK",
108 				"SPK_RX_BIAS", "MCLK",
109 				"INT_LDO_H", "MCLK",
110 				"RX_I2S_CLK", "MCLK",
111 				"TX_I2S_CLK", "MCLK",
112 				"MIC BIAS External", "Handset Mic",
113 /*begin mod by fuhua for 4-plog headset on 2019-08-30 task id :8293309 */
114 				"MIC BIAS Internal2", "Headset Mic",
115 /*end mod by fuhua for 4-plog headset on 2019-08-30 task id :8293309 */
116 				"MIC BIAS External", "Secondary Mic",
117 				"AMIC1", "MIC BIAS External",
118 /*begin mod by fuhua for 4-plog headset on 2019-08-30 task id :8293309 */
119 				"AMIC2", "MIC BIAS Internal2",
120 /*end mod by fuhua for 4-plog headset on 2019-08-30 task id :8293309 */
121 				"AMIC3", "MIC BIAS External",
122 				"ADC1_IN", "ADC1_OUT",
123 				"ADC2_IN", "ADC2_OUT",
124 				"ADC3_IN", "ADC3_OUT",
125 				"PDM_IN_RX1", "PDM_OUT_RX1",
126 				"PDM_IN_RX2", "PDM_OUT_RX2",
127 				"PDM_IN_RX3", "PDM_OUT_RX3";
```

### (3)耳机的micbias(常用micbias2)是否有外部电容  
如果有，需添加，而seoul是没有的
qcom,msm-micbias2-ext-cap  
针对micbias1也可以检查是否连有外部电容，seoul是有的。  
qcom,msm-micbias1-ext-cap;  

### (4)耳机是否支持欧美标转换  
欧标耳机绝缘环是白色的，美标一般为黑色。  
如果要支持转换，一般硬件要额外添加开关。  
另外要注意如下pinctrl的GPIO口。  
```
cross-conn-det {
qcom,pins = <&gp 97>;
qcom,num-grp-pins = <1>;
qcom,pin-func = <0>;
label = "cross-conn-det-sw";
	cross_conn_det_act: lines_on {
	drive-strength = <8>;
	output-low;
	bias-pull-down;
	};
	cross_conn_det_sus: lines_off {
	drive-strength = <2>;
	bias-disable;
	};
};
```
如果不支持，最好删除对pinctrl的引用。  
```
pinctrl-names = "cdc_lines_act",
"cdc_lines_sus",
//"cross_conn_det_act",
//"cross_conn_det_sus",
"vdd_spkdrv_act",
"vdd_spkdrv_sus";
pinctrl-0 = <&cdc_pdm_lines_act &vdd_spkdrv_act>;
pinctrl-1 = <&cdc_pdm_lines_sus &vdd_spkdrv_sus>;
// pinctrl-2 = <&cross_conn_det_act>;
// pinctrl-3 = <&cross_conn_det_sus>;
// qcom,cdc-us-euro-gpios = <&msm_gpio 97 0>;
```

### (5)更改micbias电压  
如苹果耳机，它的mic的工作电压是大于1.8v，所以为了能正常使用苹果耳机，需要增加micbias电压。    
qcom,cdc-micbias-cfilt-mv = <2700>;    
seoul项目上没有这个属性，但是代码中有去读的代码。  

或者更改代码： kernel/sound/soc/codecs/msm8x16-wcd.c      #define MICBIAS_DEFAULT_VAL 2700000  
seoul项目上路径是：  vendor/qcom/opensource/audio-kernel/asoc/codecs/sdm660_cdc/msm-analog-cdc.c  

### (6)修改识别耳机时候的阻抗  
kernel/sound/soc/codecs/wcd-mbhc-v2.c         #define HS_VREF_MIN_VAL 1400  
1.4v，最大只能识别7700欧阻抗的耳机， 这个阻抗指的是mic对地的阻抗，耳机的后两节之间的阻抗  
1.5v，最大能识别11k  
1.6v，最大能识别17.6k  
1.7v，最大能识别37.4k  
seoul项目是在`vendor/qcom/opensource/audio-kernel/asoc/codecs/wcd-mbhc-v2.h`这个文件里面。  

### (7)Lineout设备  
Qcom平台会上报SND_JACK_LINEOUR设备，但是Android层是不支持LINEOUT设备的，不会对事件做出任何响应，如果非要支持，那么就需要做如下修改：  
kernel/sound/soc/msm/msm8x16.c            .linein_th = 5000,改为 .linein_th = 0,  

seoul项目对应如下修改位置：  
/vendor/qcom/opensource/audio-kernel/asoc/msm8952.c  	.linein_th = 5000,  

### (8)如果没有兼容欧标美标开关，但是想播放音乐(不能通话),可以做如下修改  
```
595  static void wcd_correct_swch_plug(struct work_struct *work)
596  {

......

745  			if ((pt_gnd_mic_swap_cnt == mbhc->swap_thr) &&
746  				(plug_type == MBHC_PLUG_TYPE_GND_MIC_SWAP)) {
747  				/*
748  				 * if switch is toggled, check again,
749  				 * otherwise report unsupported plug
750  				 */
+++					plug_type = MBHC_PLUG_TYPE_HEADPHONE;
751  				if (mbhc->mbhc_cfg->swap_gnd_mic &&
752  					mbhc->mbhc_cfg->swap_gnd_mic(codec,
753  					true)) {
754  					pr_debug("%s: US_EU gpio present,flip switch\n"
755  						, __func__);
756  					continue;
757  				}
758  			}
```

### (9)对耳机按键阻值等调控  
耳机上报的键值定义(和(7)linein_th定义在一个文件中的)：  
```
88  	.key_code[0] = KEY_MEDIA,
89  	.key_code[1] = KEY_VOICECOMMAND,
90  	.key_code[2] = KEY_VOLUMEUP,
91  	.key_code[3] = KEY_VOLUMEDOWN,
92  	.key_code[4] = 0,
93  	.key_code[5] = 0,
94  	.key_code[6] = 0,
95  	.key_code[7] = 0,
```

`#define WCD_MBHC_DEF_BUTTONS 8 ` 
seoul项目耳机按键的数量定义在`vendor/qcom/opensource/audio-kernel/asoc/codecs/wcd-mbhc-v2.h`这个文件里面。   
耳机按键的阈值定义在：static void *def_msm8x16_wcd_mbhc_cal(void)
seoul项目则是定义在`vendor/qcom/opensource/audio-kernel/asoc/msm8952.c`目录，`static void *def_msm8952_wcd_mbhc_cal(void)`函数当中。  
```
1588  	/*
1589  	 * In SW we are maintaining two sets of threshold register
1590  	 * one for current source and another for Micbias.
1591  	 * all btn_low corresponds to threshold for current source
1592  	 * all bt_high corresponds to threshold for Micbias
1593  	 * Below thresholds are based on following resistances
1594  	 * 0-70    == Button 0
1595  	 * 110-180 == Button 1
1596  	 * 210-290 == Button 2
1597  	 * 360-680 == Button 3
1598  	 */
1599  	btn_low[0] = 75;
1600  	btn_high[0] = 75;
1601  /*Begin Modified by fuhua.wang for defect_id: 8697471 on 2019 12.19 */
1602  	btn_low[1] = 120;
1603  	btn_high[1] = 120;
1604  /*End   Modified by fuhua.wang for defect_id: 8697471 on 2019 12.19 */
1605  	btn_low[2] = 225;
1606  	btn_high[2] = 225;
1607  	btn_low[3] = 450;
1608  	btn_high[3] = 450;
1609  	btn_low[4] = 500;
1610  	btn_high[4] = 500;
1611  
1612  	return msm8952_wcd_cal;
```
如果想调试按键的阈值：  
分别配置MBHC为CS(Current Source)和MB(MIC BIAS)模式：  
CS mode :     0x144 = 0x00,  0x151 = 0xB0  
MB mode :    0x144 = 0x80,  0x151 = 0x80  
adb root  
cd sys/kernel/debug/soc/<snd-card>/msm8x16_wcd_codec/  
echo “<Register Address><value>” > codec_reg  
类如： echo “0x121 0xA0” > codec_reg  
按下耳机按键，分别测量两个模式下的耳机mic上的电压值，把测量的值分别填入高通提供的表格，或者按照80-NK808-15的Table3-3和Table3-4计算出最后的阀值。  


以上信息可以参考[其他的blog](https://www.cnblogs.com/wulizhi/p/8289905.html)，但是我认为写得有点啰嗦。  




## 软件逻辑  
识别，按键，上报等关键细节  
接下来都以Seoul项目为模板  
目录位置：vendor/qcom/opensource/audio-kernel/asoc/codecs  
耳机操作主要是带MBHC(Multibutton headset control)字样的文件。  

### 在哪儿识别  
vendor/qcom/opensource/audio-kernel/asoc/codecs/wcd-mbhc-v2.c
`wcd_mbhc_mech_plug_detect_irq`这个函数作为入口，其中的`wcd_mbhc_swch_irq_handler`作为实现。  
根据`mbhc->current_plug`和`detection_type`判断耳机当前状态。  
以插入耳机为例，正常流程则会进入到调用`mbhc->mbhc_fn->wcd_mbhc_detect_plug_type(mbhc);`函数，到`wcd-mbhc-adc.c`的`wcd_mbhc_adc_detect_plug_type`函数中。  
```
/* called under codec_resource_lock acquisition */
static void wcd_mbhc_adc_detect_plug_type(struct wcd_mbhc *mbhc)
{
	struct snd_soc_codec *codec = mbhc->codec;

	pr_debug("%s: enter\n", __func__);
	WCD_MBHC_RSC_ASSERT_LOCKED(mbhc);

	if (mbhc->mbhc_cb->hph_pull_down_ctrl)
		mbhc->mbhc_cb->hph_pull_down_ctrl(codec, false);

	WCD_MBHC_REG_UPDATE_BITS(WCD_MBHC_DETECTION_DONE, 0);

	if (mbhc->mbhc_cb->mbhc_micbias_control) {
		mbhc->mbhc_cb->mbhc_micbias_control(codec, MIC_BIAS_2,
						    MICB_ENABLE);
	} else {
		pr_err("%s: Mic Bias is not enabled\n", __func__);
		return;
	}

	/* Re-initialize button press completion object */
	reinit_completion(&mbhc->btn_press_compl);
	wcd_schedule_hs_detect_plug(mbhc, &mbhc->correct_plug_swch);
	pr_debug("%s: leave\n", __func__);
}
```
正常情况下就能走到`wcd_schedule_hs_detect_plug(mbhc, &mbhc->correct_plug_swch);`开始执行`wcd_correct_swch_plug`。这个函数就是识别设备的主要逻辑了，详细可以看源码。  

### 在哪儿上报  
上述的`wcd_correct_swch_plug`函数就是上报之前的临门一脚。  
其中可以看到有多次检测是否欧美标转换，
```
static void wcd_correct_swch_plug(struct work_struct *work)
{
	......

	do {
		cross_conn = wcd_check_cross_conn(mbhc);
		try++;
	} while (try < mbhc->swap_thr);
	if (cross_conn > 0) {
		plug_type = MBHC_PLUG_TYPE_GND_MIC_SWAP;
		pr_debug("%s: cross connection found, Plug type %d\n",
			 __func__, plug_type);
		goto correct_plug_type;
	}

	//如果设置了交叉检测，即欧美标转换，那么则进入继续确认的步骤，否则直接通过`wcd_mbhc_find_plug_and_report`上报耳机type了。  
	......

	/*
	 * Report plug type if it is either headset or headphone
	 * else start the 3 sec loop
	 */
	if ((plug_type == MBHC_PLUG_TYPE_HEADSET ||
	     plug_type == MBHC_PLUG_TYPE_HEADPHONE) &&
	    (!wcd_swch_level_remove(mbhc))) {
		WCD_MBHC_RSC_LOCK(mbhc);
		wcd_mbhc_find_plug_and_report(mbhc, plug_type);//直接通过`wcd_mbhc_find_plug_and_report`上报耳机type
		WCD_MBHC_RSC_UNLOCK(mbhc);
	}
	......

//进入继续确认的步
correct_plug_type:
	timeout = jiffies + msecs_to_jiffies(HS_DETECT_PLUG_TIME_MS);
	while (!time_after(jiffies, timeout)) {
```

继续确认主要是针对欧美标转换设置的，为了兼容更多的耳机，即使硬件没有转换开关，多半还是选择识别成三段式耳机，不过耳机状态基本都给到了`wcd_mbhc_find_plug_and_report`函数接口，通过它给`wcd_mbhc_report_plug`函数做逻辑判断，然后上报，上报类型为`SND_JACK_UNSUPPORTED`等。  
```
//wcd-mbhc-v2.c
void wcd_mbhc_report_plug(struct wcd_mbhc *mbhc, int insertion,
				enum snd_jack_types jack_type)
{

......

                else{
                    jack_type = SND_JACK_LINEOUT;
                    mbhc->force_linein = true;
                    mbhc->current_plug = MBHC_PLUG_TYPE_HIGH_HPH;
                    if (mbhc->hph_status) {
                        mbhc->hph_status &= ~(SND_JACK_HEADSET |
                                SND_JACK_LINEOUT |
                                SND_JACK_UNSUPPORTED);
                                wcd_mbhc_jack_report(mbhc,
                                &mbhc->headset_jack,
                                mbhc->hph_status,
                                WCD_MBHC_JACK_MASK);
                    }
```

### 按键识别  

中断的注册是在init函数中的`wcd_mbhc_btn_press_handler`。  
```
int wcd_mbhc_init(struct wcd_mbhc *mbhc, struct snd_soc_codec *codec,
		      const struct wcd_mbhc_cb *mbhc_cb,
		      const struct wcd_mbhc_intr *mbhc_cdc_intr_ids,
		      struct wcd_mbhc_register *wcd_mbhc_regs,
		      bool impedance_det_en)
{

	......

	ret = mbhc->mbhc_cb->request_irq(codec,
					 mbhc->intr_ids->mbhc_btn_press_intr,
					 wcd_mbhc_btn_press_handler,
					 "Button Press detect", mbhc);

```
进入`static irqreturn_t wcd_mbhc_btn_press_handler(int irq, void *data)`函数之前，中断已经触发了，其中的  
```
	mbhc->is_btn_press = true;
	msec_val = jiffies_to_msecs(jiffies - mbhc->jiffies_atreport);

	......

	/* If switch interrupt already kicked in, ignore button press */
	if (mbhc->in_swch_irq_handler) {
		pr_debug("%s: Swtich level changed, ignore button press\n",
			 __func__);
		goto done;
	}
	mask = wcd_mbhc_get_button_mask(mbhc);
	if (mask == SND_JACK_BTN_0)
		mbhc->btn_press_intr = true;

        //Begin-modify-by-fuhua.wang-for-task 8384912 on 20190926
        //if (mbhc->current_plug != MBHC_PLUG_TYPE_HEADSET) {
        if (mbhc->current_plug != MBHC_PLUG_TYPE_HEADSET && !mbhc->is_selfie_stick_insert) { 
        //End-add-by-fuhua.wang-for-task 8384912 on 20190926
        pr_err("%s: Plug isn't headset, ignore button press\n",
				__func__);
		goto done;
	}
```
作为开始处理按键的标志。  
`mask = wcd_mbhc_get_button_mask(mbhc);`则是哪个按键的标志。深究可以发现是调用`msm-analog-cdc.c`中的`msm_anlg_cdc_mbhc_map_btn_code_to_num`函数，其中的`snd_soc_read`则是读取codec的寄存器了，这个接口是`kernel/msm-4.9/sound/soc/soc-io.c`中定义。    

当放开按键的时候，则会调用`wcd_mbhc_release_handler`上报jack。  
```
static irqreturn_t wcd_mbhc_release_handler(int irq, void *data)
{

	......

			wcd_mbhc_jack_report(mbhc, &mbhc->button_jack,
					0, mbhc->buttons_pressed);
		} else {
			if (mbhc->in_swch_irq_handler) {
				pr_debug("%s: Switch irq kicked in, ignore\n",
					__func__);
			} else {
				pr_debug("%s: Reporting btn press\n",
					 __func__);
				wcd_mbhc_jack_report(mbhc,
						     &mbhc->button_jack,
						     mbhc->buttons_pressed,
						     mbhc->buttons_pressed);
				pr_debug("%s: Reporting btn release\n",
					 __func__);
				wcd_mbhc_jack_report(mbhc,
						&mbhc->button_jack,
						0, mbhc->buttons_pressed);
```
短按都在这里了，长按则会进入`wcd_btn_lpress_fn:`。  
```
static void wcd_btn_lpress_fn(struct work_struct *work)
{
	struct delayed_work *dwork;
	struct wcd_mbhc *mbhc;
	s16 btn_result = 0;

	pr_debug("%s: Enter\n", __func__);

	dwork = to_delayed_work(work);
	mbhc = container_of(dwork, struct wcd_mbhc, mbhc_btn_dwork);

	WCD_MBHC_REG_READ(WCD_MBHC_BTN_RESULT, btn_result);
	if (mbhc->current_plug == MBHC_PLUG_TYPE_HEADSET) {
		pr_debug("%s: Reporting long button press event, btn_result: %d\n",
			 __func__, btn_result);
		wcd_mbhc_jack_report(mbhc, &mbhc->button_jack,
				mbhc->buttons_pressed, mbhc->buttons_pressed);
	}
	pr_debug("%s: leave\n", __func__);
	mbhc->mbhc_cb->lock_sleep(mbhc, false);
}
```


## 综述  
如果只是为了应付项目里常见修改，文章开头列举的足矣。  

但有可能会遇到更难的情况，或者更棘手的需求，或者需要创新，这个时候就需要深入了解软件接口和框架了，而且具体问题具体分析，本文仅仅针对了重要的接口，详细问题，比如增加自拍杆等重新定义一个特征按键等，需要另起分析，否则篇幅过长，不利于把握。  

