>工欲善其事必先利其器

###1. 调试常见手段(从bring up开始)
#####(1) 查看设备树，看是否有注册上
cd /sys/firmware/
其中有"devicetree/  fdt"
一个是可以直接查看的，一个是fdt，可以导出来之后用fdtdump fdt > irvine.txt查看。
#####(2) 查看声卡、pcm等状态。
声卡：
cd /sys/kernel/debug/asoc/
439系列的基线是有张声卡的，但是现在的6350用的另外的方式，直接加了自己的东西。
或者用如下命令查看声卡
```
lito:/data/local/tmp # cat /proc/asound/cards                                 
 0 [litolagoonmtpsn]: lito-lagoonmtp- - lito-lagoonmtp-snd-card
                      lito-lagoonmtp-snd-card
```
pcm:
```
cd /sys/class/sound
```
codec:
```
cd /sys/kernel/debug/asoc/lito-lagoonmtp-snd-card
```
里面会有wcd937x的codec
dais:
```
cat /d/asoc/dais
```
platform:
应该是变成了components:
```
cat /d/asoc/compinents
```
#####(3) wcd937x寄存器：
```
cat /sys/kernel/debug/regmap/wcd937x-slave.a01170223-wcd937x_csr/registers
```
查mic的时候查过，00e和00f
#####(4) i2c设备：
Irvine:/sys/bus/i2c/devices/3-0034
4.19的Qcom平台注册的是
这个根据项目引脚来配置，根据Irvine项目，我们可以在如下文件中配置
`qualcomm/platform/vendor/qcom/proprietary/devicetree-4.19 / qcom/irvine-lagoon-qrd.dtsi`
```
&qupv3_se8_i2c {
        status = "ok";
        st21nfc@08 {
                compatible = "st,st21nfc";
                reg = <0x08>;
                interrupt-parent = <&tlmm>;
		        interrupts = <9 0>;
                st,reset_gpio = <&tlmm 6 0x00>;
                st,irq_gpio = <&tlmm 9 0x00>;
                st,clkreq_gpio = <&tlmm 7 0x00>;
                clocks = <&rpmhcc RPMH_LN_BB_CLK2>;
                clock-names = "nfc_ref_clk";
                st,clk_pinctrl;
                status = "ok";
        };
//begin add by fuhua.wang for task 9554561 on 2020.07.10
	aw881xx_smartpa@34 {
		compatible = "awinic,aw881xx_smartpa";
		reg = <0x34>;
		reset-gpio = <&tlmm 12 0>;
		irq-gpio = <&tlmm 11 0>;
		monitor-flag = <1>;
		monitor-timer-val = <30000>;
		status = "okay";
	};
//end add by fuhua.wang for task 9554561 on 2020.07.10
};
```
overlay位置：`/vendor/qcom/proprietary/devicetree-4.19 / qcom/lagoon-audio-overlay.dtsi`
```
	&bolero {
		qcom,num-macros = <3>;
	};
	&wsa_macro {
		status = "disabled";
	};
	&wsa_swr_gpios {
		status = "disabled";
	};
	&wsa_spkr_en1 {
		status = "disabled";
	};
	&wsa_spkr_en2 {
		status = "disabled";
	};
	&cdc_dmic01_gpios{
		status = "disabled";
	};
	&cdc_dmic23_gpios{
		status = "disabled";
	};
	&cdc_dmic45_gpios{
		status = "disabled";
	};

	&dai_mi2s4 {
		pinctrl-names = "default", "sleep";
		pinctrl-0 = <&lpi_i2s1_sck_active &lpi_i2s1_ws_active &lpi_i2s1_sd0_active &lpi_i2s1_sd1_active>;
		pinctrl-1 = <&lpi_i2s1_sck_sleep  &lpi_i2s1_ws_sleep  &lpi_i2s1_sd0_sleep  &lpi_i2s1_sd1_sleep>;
	};
```
当然，最好的方法是另外加overlay，1是可以和其他模块区分，比如nfc，2是可以和基线区分开，方便基线升级，3是可以将audio的修改部分独立，查看也方便。
因此添加了如下文件。
`platform/vendor/qcom/proprietary/devicetree-4.19 / qcom/lito-irvine-audio-overlay.dtsi`，其中的内容如下：
```
	//begin add by fuhua.wang for task 9554561 on 2020.07.21
	&lagoon_snd {
		qcom,model = "lito-irvine-snd-card";
		qcom,msm-mi2s-master = <1>, <1>, <1>, <1>, <1>, <1>;
		qcom,wcn-btfm = <1>;
		qcom,ext-disp-audio-rx = <1>;
		qcom,audio-routing =
```
			"AMIC1", "MIC BIAS1",
			"MIC BIAS1", "Analog Mic1",
			"AMIC2", "MIC BIAS2",
			"MIC BIAS2", "Analog Mic2",
			"AMIC3", "MIC BIAS3",
			"MIC BIAS3", "Analog Mic3",
			"IN1_HPHL", "HPHL_OUT",
			"IN2_HPHR", "HPHR_OUT",
			"IN3_AUX", "AUX_OUT",
			"TX SWR_ADC0", "ADC1_OUTPUT",
			"TX SWR_ADC2", "ADC2_OUTPUT",
			"TX SWR_ADC3", "ADC3_OUTPUT",
			"RX_TX DEC0_INP", "TX DEC0 MUX",
			"RX_TX DEC1_INP", "TX DEC1 MUX",
			"RX_TX DEC2_INP", "TX DEC2 MUX",
			"RX_TX DEC3_INP", "TX DEC3 MUX",
			"VA_AIF1 CAP", "VA_SWR_CLK",
			"VA_AIF2 CAP", "VA_SWR_CLK",
			"VA_AIF3 CAP", "VA_SWR_CLK",
			"TX_AIF1 CAP", "VA_MCLK",
			"TX_AIF2 CAP", "VA_MCLK",
			"RX AIF1 PB", "VA_MCLK",
			"RX AIF2 PB", "VA_MCLK",
			"RX AIF3 PB", "VA_MCLK",
			"RX AIF4 PB", "VA_MCLK",
			"VA SWR_ADC0", "ADC1_OUTPUT",
			"VA SWR_ADC1", "ADC2_OUTPUT",
			"VA SWR_ADC2", "ADC3_OUTPUT";
```
		qcom,msm-mbhc-hphl-swh = <1>;
		qcom,msm-mbhc-gnd-swh = <1>;
		qcom,cdc-dmic01-gpios = <&cdc_dmic01_gpios>;
		qcom,cdc-dmic23-gpios = <&cdc_dmic23_gpios>;
		qcom,cdc-dmic45-gpios = <&cdc_dmic45_gpios>;
		asoc-codec  = <&stub_codec>, <&bolero>, <&ext_disp_audio_codec>;
		asoc-codec-names = "msm-stub-codec.1", "bolero_codec",
```
```
					"msm-ext-disp-audio-codec-rx";
```
```
		qcom,wsa-max-devs = <0>;

		qcom,codec-max-aux-devs = <1>;
		qcom,codec-aux-devs = <&wcd937x_codec>;
		qcom,msm_audio_ssr_devs = <&audio_apr>, <&q6core>,
					  <&lpi_tlmm>, <&bolero>;

		//add by fuhua
		qcom,msm-mbhc-usbc-audio-supported = <0>;
		qcom,disable_wsa_codec = <1>;
```
```
		/delete-property/qcom,cdc-dmic01-gpios;
		/delete-property/qcom,cdc-dmic23-gpios;
		/delete-property/qcom,cdc-dmic45-gpios;
	};
```
```
	&bolero {
		qcom,num-macros = <3>;
	};
	&wsa_macro {
		status = "disabled";
	};
	&wsa_swr_gpios {
		status = "disabled";

	};
	&wsa_spkr_en1 {
		status = "disabled";

	};
	&wsa_spkr_en2 {
		status = "disabled";
	};
	&cdc_dmic01_gpios{
		status = "disabled";
	};
	&cdc_dmic23_gpios{
		status = "disabled";
	};
	&cdc_dmic45_gpios{
		status = "disabled";
	};
```
```
	&dai_mi2s4 {
		pinctrl-names = "default", "sleep";
		pinctrl-0 = <&lpi_i2s1_sck_active &lpi_i2s1_ws_active &lpi_i2s1_sd0_active &lpi_i2s1_sd1_active>;
		pinctrl-1 = <&lpi_i2s1_sck_sleep  &lpi_i2s1_ws_sleep  &lpi_i2s1_sd0_sleep  &lpi_i2s1_sd1_sleep>;
	};

	&qupv3_se8_i2c {
		status = "ok";
		aw881xx_smartpa@34 {
			compatible = "awinic,aw881xx_smartpa";
			reg = <0x34>;
			reset-gpio = <&tlmm 12 0>;
			irq-gpio = <&tlmm 11 0>;
			monitor-flag = <1>;
			monitor-timer-val = <30000>;
			status = "okay";
		};
	};

	//end   add by fuhua.wang for task 9554561 on 2020.07.21
```
通过查看设备树的overlay顺序，我们将文件引入申明放到了如下文件，该文件是最后加载的`qualcomm/platform/vendor/qcom/proprietary/devicetree-4.19 / qcom/irvine-lagoon-qrd.dtsi`，具体代码如下：
```
//begin add by fuhua.wang for task 9554561 on 2020.07.21
#include "lito-irvine-audio-overlay.dtsi"
//end   add by fuhua.wang for task 9554561 on 2020.07.21
```
对比MTK的项目：在`kernel-4.9/sound/soc/mediatek/common_int/mtk-soc-machine.c`中加了如下代码的。
```
666  static struct snd_soc_dai_link mt_soc_extspk_dai[] = {
667  	{
668  		.name = "ext_Speaker_Multimedia",
669  		.stream_name = MT_SOC_SPEAKER_STREAM_NAME,
670  		.cpu_dai_name = "snd-soc-dummy-dai",
671  		.platform_name = "snd-soc-dummy",
672  #ifdef CONFIG_SND_SOC_MAX98926
673  		.codec_dai_name = "max98926-aif1",
674  		.codec_name = "MAX98926_MT",
675  #elif defined(CONFIG_SND_SOC_CS35L35)
676  		.codec_dai_name = "cs35l35-pcm",
677  		.codec_name = "cs35l35.2-0040",
678  		.ignore_suspend = 1,
679  		.ignore_pmdown_time = true,
680  		.dai_fmt = SND_SOC_DAIFMT_I2S | SND_SOC_DAIFMT_CBS_CFS |
681  			   SND_SOC_DAIFMT_NB_NF,
682  		.ops = &cs35l35_ops,
683  //begin mod by fuhua.wang for task 8740652 on 2020.01.19
684  #elif defined(CONFIG_SND_SMARTPA_AW881XX)
685  		.ignore_suspend = 1,
686  		.ignore_pmdown_time = true,
687  		.codec_dai_name = "aw881xx-aif",
688  		.codec_name = "aw881xx_smartpa",
689  		.ops = &mt_machine_audio_ops,
690  #else
691  		.codec_dai_name = "snd-soc-dummy-dai",
692  		.codec_name = "snd-soc-dummy",
693  //end   mod by fuhua.wang for task 8740652 on 2020.01.19
694  #endif
695  	},
696  	{
697  		.name = "I2S1_AWB_CAPTURE",
698  		.stream_name = MT_SOC_I2S2ADC2_STREAM_NAME,
699  		.cpu_dai_name = MT_SOC_I2S2ADC2DAI_NAME,
700  		.platform_name = MT_SOC_I2S2_ADC2_PCM,
701  		.codec_dai_name = "snd-soc-dummy-dai",
702  		.codec_name = "snd-soc-dummy",
703  		.ops = &mt_machine_audio_ops,
704  	},
705  };
```
而Qcom则是在kona.c中将Quin的i2s替换掉：
```
static struct snd_soc_dai_link msm_mi2s_be_dai_links[] = {

    ......

		.name = LPASS_BE_QUIN_MI2S_RX,
		.stream_name = "Quinary MI2S Playback",
		.cpu_dai_name = "msm-dai-q6-mi2s.4",
		.platform_name = "msm-pcm-routing",
/* MODIFIED-BEGIN by fuhua.wang, 2020-07-18,BUG-9554561*/
//#ifdef CONFIG_SND_SMARTPA_AW881XX
		.codec_name = "aw881xx_smartpa",
		.codec_dai_name = "aw881xx-aif",
// #else
		// .codec_name = "msm-stub-codec.1",
		// .codec_dai_name = "msm-stub-rx",
// #endif
/* MODIFIED-END   by fuhua.wang, 2020-07-18,BUG-9554561*/
		.no_pcm = 1,
		.dpcm_playback = 1,
		.id = MSM_BACKEND_DAI_QUINARY_MI2S_RX,
		.be_hw_params_fixup = msm_be_hw_params_fixup,
		.ops = &msm_mi2s_be_ops,
		.ignore_suspend = 1,
		.ignore_pmdown_time = 1,
	},
```
#####(5) 查看加载哪个mixer_path.xml
这个就有点麻烦了，一般来讲有如下三个思路去看：
有多种方案确定具体是编译的哪个platform.c
~~(1)看out下面生成的audio.primary后缀~~
(2)直接打印对应的${TARGET_PLATFORM}/platform.c
(3)寻找${TARGET_PLATFORM}变量，但估计也有很复杂的逻辑，看花眼。
最好直接第一种方法，但是可以看到出来的是 lito的后缀（TARGET_BOARD_PLATFORM），所以没办法看。
用第三种方法查看变量逻辑，似乎也没有看到实际负值的地方。
现在只有第二种方法了。
那么就要涉及到makefile语法，如何打印变量
$(error "TARGET_PLATFORM = $(TARGET_PLATFORM)")
但是实际上也没在makefile.am文件中编译platform，是在Android.mk中编译的，
vendor/qcom/opensource/audio-hal/primary-hal/hal/Android.mk:126: error: "123123  AUDIO_PLATFORM = msm8974".
用的8974
直接查看代码
可以发现用的mixer_path是mixer_paths_lagoonmtp.xml
```
lito:/data/local/tmp # cat /proc/asound/cards                                 
 0 [litolagoonmtpsn]: lito-lagoonmtp- - lito-lagoonmtp-snd-card
                      lito-lagoonmtp-snd-card
```
配置xml的位置在：
E:\code-sdc\Qcom-Pro\Irvine-bsp\vendor\qcom\opensource\audio-hal\primary-hal\configs\lito
可以修改试试
但是发现tinymix出来并没有aw，所以也不是mixer_paths_lagoonmtp.xml？？？
不对，可能是没有注册控件。
要打开route_print查看，位置:
system/media/audio_route/audio_route.c
```
/* Apply an audio route path by name */
int audio_route_apply_path(struct audio_route *ar, const char *name)
{
    struct mixer_path *path;
    if (!ar) {
        ALOGE("invalid audio_route");
        return -1;
    }
    path = path_get_by_name(ar, name);
    if (!path) {
        ALOGE("unable to find path '%s'", name);
        return -1;
    }
#if 1
    path_print(ar, path);
#endif
    path_apply(ar, path);
    return 0;
}
/* Reset an audio route path by name */
int audio_route_reset_path(struct audio_route *ar, const char *name)
{
    struct mixer_path *path;
    if (!ar) {
```
        ALOGE("invalid audio_route");
        return -1;
		}
```
    path = path_get_by_name(ar, name);
    if (!path) {
        ALOGE("unable to find path '%s'", name);
        return -1;
    }
#if 1
    path_print(ar, path);
#endif
```
```
    path_reset(ar, path);
    return 0;}
....
static int path_apply(struct audio_route *ar, struct mixer_path *path)
{
    unsigned int i;
    unsigned int ctl_index;
    struct mixer_ctl *ctl;
    enum mixer_ctl_type type;
    ALOGD("Apply path: %s", path->name != NULL ? path->name : "none");
    for (i = 0; i < path->length; i++) {
        ctl_index = path->setting[i].ctl_index;
        ctl = index_to_ctl(ar, ctl_index);
        type = mixer_ctl_get_type(ctl);
        if (!is_supported_ctl_type(type))
            continue;
        size_t value_sz = sizeof_ctl_type(type);
        memcpy(ar->mixer_state[ctl_index].new_value.ptr, path->setting[i].value.ptr,
                   path->setting[i].num_values * value_sz);
    }
    return 0;
}
```

确认修改起效之后再开始下一步。  


从log中我们确认了加载的是mixer_paths_lagoonmtp.xml，说明我们改得没有错。


另外就是直接打log：

`vendor/qcom/opensource/audio-hal/primary-hal/hal/msm8974/platform.c`文件中：
```
void *platform_init(struct audio_device *adev)
{
    ......

    if (platform_is_i2s_ext_modem(snd_card_name, my_data) &&
        !is_auto_snd_card(snd_card_name)) {
        ALOGD("%s: Call MIXER_XML_PATH_I2S", __func__);

        adev->audio_route = audio_route_init(adev->snd_card,
                                             MIXER_XML_PATH_I2S);
    } else {
        /* Get the codec internal name from the sound card name
         * and form the mixer paths file name dynamically. This
         * is generic way of picking any codec name based mixer
         * files in future with no code change. This code
         * assumes mixer files are formed with format as
         * mixer_paths_internalcodecname.xml

         * If this dynamically read mixer files fails to open then it
         * falls back to default mixer file i.e mixer_paths.xml. This is
         * done to preserve backward compatibility but not mandatory as
         * long as the mixer files are named as per above assumption.
        */
        snprintf(mixer_xml_file, sizeof(mixer_xml_file), "%s_%s_%s.xml",
                         MIXER_XML_BASE_STRING, snd_split_handle->snd_card,
                         snd_split_handle->form_factor);
        ALOGD("%s: fuhua-check need mixer file: %s", __func__, mixer_xml_file);// add in here

        if (!audio_extn_utils_resolve_config_file(mixer_xml_file)) {
            memset(mixer_xml_file, 0, sizeof(mixer_xml_file));
            snprintf(mixer_xml_file, sizeof(mixer_xml_file), "%s_%s.xml",
                         MIXER_XML_BASE_STRING, snd_split_handle->variant);

            if (!audio_extn_utils_resolve_config_file(mixer_xml_file)) {
                memset(mixer_xml_file, 0, sizeof(mixer_xml_file));
                snprintf(mixer_xml_file, sizeof(mixer_xml_file), "%s_%s.xml",
                             MIXER_XML_BASE_STRING, snd_split_handle->snd_card);

                if (!audio_extn_utils_resolve_config_file(mixer_xml_file)) {
                    memset(mixer_xml_file, 0, sizeof(mixer_xml_file));
                    strlcpy(mixer_xml_file, MIXER_XML_DEFAULT_PATH, MIXER_PATH_MAX_LENGTH);
                    audio_extn_utils_resolve_config_file(mixer_xml_file);
                }
            }
        }

```

还有几个其他的文件，audio_platform_sound_info.xml，sound_tragger等，都可以打印出来看。


#####(6) 打开Audio_route:
`system/media/audio_route/audio_route.c`
```
//mod by fuhua.wang for task 9554561 on 2020.07.10
#if 1
static void path_print(struct audio_route *ar, struct mixer_path *path)
{
    unsigned int i;
    unsigned int j;

    ALOGE("Path: %s, length: %d", path->name, path->length);
    for (i = 0; i < path->length; i++) {
        struct mixer_ctl *ctl = index_to_ctl(ar, path->setting[i].ctl_index);

        ALOGE("  id=%d: ctl=%s", i, mixer_ctl_get_name(ctl));
        if (mixer_ctl_get_type(ctl) == MIXER_CTL_TYPE_BYTE) {
            for (j = 0; j < path->setting[i].num_values; j++)
                ALOGE("    id=%d value=0x%02x", j, path->setting[i].value.bytes[j]);
        } else if (mixer_ctl_get_type(ctl) == MIXER_CTL_TYPE_ENUM) {
            for (j = 0; j < path->setting[i].num_values; j++)
                ALOGE("    id=%d value=%d", j, path->setting[i].value.enumerated[j]);
        } else {
            for (j = 0; j < path->setting[i].num_values; j++)
                ALOGE("    id=%d value=%ld", j, path->setting[i].value.integer[j]);
        }
    }
}
#endif
```
unknown:/system # find ./ -name libaudioroute.so                                                                   
./lib/vndk-29/libaudioroute.so
./lib64/vndk-29/libaudioroute.so


adb push system/lib64/vndk-29/libaudioroute.so /system/lib64/vndk-29/libaudioroute.so
adb push system/lib/vndk-29/libaudioroute.so /system/lib/vndk-29/libaudioroute.so


###2. 器件部分：
####(1) 一般k类功放会操作引脚，所以需要查看引脚状态。

4系列的直接按照kernel的标准方案搞就行了，不同于mtk封装了gpio。
[可以直接参考别人的blog](https://blog.csdn.net/luckydarcy/article/details/53061901)

####(2) HAC器件
和操作引脚差不多，但是需要重新按照框架搭建一个通路，并注册控件。

本来可以直接操作GPIO的，但是没有export
cat /sys/kernel/debug/gpio也会死机
先解决死机的问题
修改amss_6350_spf1.0/TZ.XF.5.0/trustzone_images/ssg/securemsm/accesscontrol/cfg/bitra/tz/xpu_config.xml
然后发现expor也没有反应，所以只能通过cat引脚来看了。



####(3) tinymix命令检查通路

###3. 编译部分：
make bootimage
make dtboimage

mmma 相关ko文件

out/target/product/lito/dlkm/lib/modules/

手机里的目录：/vendor/lib/modules

adb push ./out/target/product/lito/dlkm/lib/modules/ /vendor/lib/modules/

adb push ./ /vendor/lib/modules


adb reboot bootloader
fastboot flash dtbo dtbo.img
fastboot reboot


###4. 动态检查log
(1)要写file +p
echo -n "file wcd-mbhc-v2.c +p" > /sys/kernel/debug/dynamic_debug/control


###5. 抓取audio dump：
先要打开usb调试端口

setprop sys.usb.config diag,serial_cdev,rmnet,dpl,adb 
然后重启

发现也不行

对比可以连接QPST的现象，发现应该是电脑没有安装对的驱动，所以重新安装了9086的 MDM2 Diagnostics


###6. 添加hal层的audio log
添加位置：
####(1)  `[platform/hardware/qcom/audio.git] / hal / audio_hw.h`
```
+++ b/hal/audio_hw.h
@@ -437,6 +437,7 @@ struct stream_out {
     mix_matrix_params_t pan_scale_params;
     mix_matrix_params_t downmix_params;
     bool set_dual_mono;
+    int pcm_log_fd; // add by fuhua.wang
     bool prev_card_status_offline;

     error_log_t *error_log;
@@ -617,6 +618,7 @@ struct audio_device {
     bool allow_afe_proxy_usage;
     bool is_charging; // from battery listener
     bool mic_break_enabled;
+    int dump_enable; // add by fuhua.wang
     bool enable_hfp;
     bool mic_muted;
     bool enable_voicerx;
 
```


####(2)  `[platform/hardware/qcom/audio.git] / hal / audio_hw.c`
```
diff --git a/hal/audio_hw.c b/hal/audio_hw.c
index 5170a64..7f6e7dd 100644 (file)
--- a/hal/audio_hw.c
+++ b/hal/audio_hw.c
@@ -3904,6 +3904,16 @@ int start_output_stream(struct stream_out *out)
         if (ret < 0)
             ALOGE("%s: audio_extn_ip_hdlr_intf_open failed %d",__func__, ret);
     }
+    /* add begin by fuhua.wang*/
+    char dumpvalue[PROPERTY_VALUE_MAX]={'\0'};
+    property_get("vendor.def.tct.audio.dump.enabled", dumpvalue, "0");
+    if (strcmp(dumpvalue, "1") == 0) {
+        adev->dump_enable = true;
+        ALOGW("out_write dump enable %d", adev->dump_enable);
+    }else{
+        adev->dump_enable = false;
+    }
+    /* add end by fuhua.wang*/
 
     // consider a scenario where on pause lower layers are tear down.
     // so on resume, swap mixer control need to be sent only when
@@ -5592,6 +5602,16 @@ static ssize_t out_write(struct audio_stream_out *stream, const void *buffer,
                 out->is_compr_metadata_avail = false;
             }
         }
+         /* add begin by fuhua.wang*/
+         if (adev->dump_enable) {
+               ALOGE("out_write3 out->pcm_log_fd = %d", out->pcm_log_fd);
+               if (out->pcm_log_fd == -1)
+                   out->pcm_log_fd = open("data/mediaserver/pcm_offload.pcm", O_RDWR|O_CREAT, S_IRWXU|S_IRWXG|S_IRWXO);
+               ALOGE("out_write3 after open out->pcm_log_fd = %d", out->pcm_log_fd);
+               if (out->pcm_log_fd != -1)
+                   write(out->pcm_log_fd, (void*)buffer, bytes);
+         }
+         /* add end by fuhua.wang*/
         if (!(out->flags & AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD) &&
                       (out->convert_buffer) != NULL) {
 
@@ -5681,6 +5701,16 @@ static ssize_t out_write(struct audio_stream_out *stream, const void *buffer,
             ALOGV("%s: frames=%zu, frame_size=%zu, bytes_to_write=%zu",
                      __func__, frames, frame_size, bytes_to_write);
 
+           /* add begin by fuhua.wang*/
+           if (adev->dump_enable) {
+                ALOGE("out_write4 out->pcm_log_fd = %d", out->pcm_log_fd);
+                if (out->pcm_log_fd == -1)
+                    out->pcm_log_fd = open("data/mediaserver/pcm_other.pcm", O_RDWR|O_CREAT, S_IRWXU|S_IRWXG|S_IRWXO);
+                ALOGE("out_write4 after open out->pcm_log_fd = %d", out->pcm_log_fd);
+                if (out->pcm_log_fd != -1)
+                    write(out->pcm_log_fd, (void*)buffer, bytes);
+            }
+            /* add end by fuhua.wang*/
             if (out->usecase == USECASE_INCALL_MUSIC_UPLINK ||
                 out->usecase == USECASE_INCALL_MUSIC_UPLINK2 ||
                 (out->usecase == USECASE_AUDIO_PLAYBACK_VOIP &&

```
####(3) 权限问题：
`[device/qcom/sepolicy.git] / generic / vendor / common / system_app.te`
```
@@ -52,6 +52,8 @@ allow system_app sysfs_display:file { read write open getattr };
 # allow MMITest to read /sys/module/msm_poweroff/parameters/download_mode
 allow system_app sysfs_download_mode:file { read getattr open };
 
+get_prop(system_app, tct_default_prop)
+
 allow system_app usersupport_app_data_file:dir create_dir_perms;
```


###7. QACT无法连接acdb文件：
有蛮多种可能的，
(1)第一次遇到的那种是，Mismatch between number of ACDB files downloaded and workspace file什么的，然后QACT本地重新保存一下，push进去就好了。


###8. QXDM logmask



