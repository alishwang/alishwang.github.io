+++
author = "fuhua"
title = "Android - Audio - Qcom平台 - audio参数定制"
date = "2020-04-02 08:57:15"
description = "手机技术笔记"
tags = [
    "markdown",
    "text",
]
weight = 10
+++


代码相对MTK少很多，也清晰。
- [背景](#背景)  
- [Seoul项目参数定制涉及目录](#Seoul项目参数定制涉及目录)  
    - [开机加载参数的流程](#开机加载参数的流程)  
    - [调试软件读取参数的接口](#调试软件读取参数的接口)   
	- [参数放哪儿](#参数放哪儿)  
	- [单双mic定制开关](#单双mic定制开关)  
- [综述](#综述)  





## 背景
>之前有整理MTK平台的参数定制方案，但公司的项目也有Qcom，整理一下，一来可以再次理清软件思路，二来方便回顾参考。  


## Seoul项目参数定制涉及目录  

参数以及将参数编译进手机的目录：  
vendor/qcom/proprietary/mm-audio-cal/audcal/acdbdata/8937/   

运行期间加载acdb文件操作：  
vendor/qcom/proprietary/mm-audio-cal/audio-acdb-util/acdb-loader/src/acdb-loader.c  

单双mic开关定制：  
hardware/qcom/audio/hal/msm8916/platform.c  

权限问题：  
device/qcom/sepolicy/vendor/common/  
权限问题主要还是看自己的定制code


参数定制是为了迎合一套硬件多套软件的需求，那么audio相关的差异化需求就需要在开机的时候确定，比如哪套参数，是否单mic，也要预防参数是否出错。    


### 开机加载参数的流程  
开机之后，会在acdb-loader.c中加载.acdb后缀的参数文件，然后才会启动audio相关服务。如果Null则会出现死机重启的现象。  
```
//Begin Modified by fuhua.wang for task_id: 8810241 on 2020 01.09
#define PROPERTY_RO_BOOT_CODE "ro.boot.code"
#define PROPERTY_AUDIOVERSIONPATH "ro.vendor.audioversionpath"
//End   Modified by fuhua.wang for task_id: 8810241 on 2020 01.09
//先是定义关键的定制依赖属性。


......

static int get_files_from_device_tree(AcdbInitCmdType *acdb_init_cmd, char *snd_card_name)
{
	int result = 0;
	//Begin Modified by fuhua.wang for task_id: 8810241 on 2020 01.09
	int result_acdbinfo = 0;
	char dir_path[300];
	char board_type[64] = DEFAULT_BOARD;
	FILE *fp = NULL;
	#ifdef TCT_AUDIO_CUSTOMAIZATION_FOR_FCM
	char codename_to_chose[PROPERTY_VALUE_MAX] = {0};
	//End Modified by fuhua.wang for task_id: 8810241 on 2020 01.09
	//begin add by kun.zheng for task 8598614 on 2019/11/15
	char audioVersionPath[PROPERTY_VALUE_MAX];
	//end add by kun.zheng for task 8598614 on 2019/11/15
	#endif

	/* Get Board type */
	fp = fopen("/sys/devices/soc0/hw_platform","r");
	if (fp == NULL)
		fp = fopen("/sys/devices/system/soc/soc0/hw_platform","r");
	if (fp == NULL)
		LOGE("ACDB -> Error: Couldn't open hw_platform\n");
	else if (fgets(board_type, sizeof(board_type), fp) == NULL)
		LOGE("ACDB -> Error: Couldn't get board type\n");
	else if (board_type[(strlen(board_type) - 1)] == '\n')
		board_type[(strlen(board_type) - 1)] = '\0';
	if (fp != NULL)
		fclose(fp);

	if (!strcmp(board_type, "Surf"))
		strlcpy(board_type, "CDP", 4);

	LOGD("ACDB: board_type = %s\n", board_type);

	//Begin Modified by fuhua.wang for task_id: 8952191 on 2020 02.25
	#ifdef TCT_AUDIO_CUSTOMAIZATION_FOR_FCM

	property_get(PROPERTY_RO_BOOT_CODE, codename_to_chose, "");
	LOGD("%s: variant_to_chose:%s\n",__func__, codename_to_chose);

	if(strstr(codename_to_chose,"5002S")!= NULL){
	//if (1) {//signal mic for Seoul all variant 
		result = snprintf(dir_path, sizeof(dir_path), "%s/CAN",ACDB_BIN_PATH);
		result_acdbinfo = snprintf(audioVersionPath, sizeof(audioVersionPath), "%s/can_acdb.info",dir_path);
		LOGD("fuhua---check get can audio_param \n");
	}else if(strstr(codename_to_chose,"5002L")!= NULL)
	{
		result = snprintf(dir_path, sizeof(dir_path), "%s/US",ACDB_BIN_PATH);
		result_acdbinfo = snprintf(audioVersionPath, sizeof(audioVersionPath), "%s/us_acdb.info",dir_path);
		LOGD("fuhua---check get us audio_param \n");
	}
	else{
		result = snprintf(dir_path, sizeof(dir_path), "%s/SMIC",ACDB_BIN_PATH);
		result_acdbinfo = snprintf(audioVersionPath, sizeof(audioVersionPath), "%s/smic_acdb.info",dir_path);
		LOGD("fuhua---check get smic audio_param \n");
	}
	LOGD("ACDB Loader:dir_path:%s for variant \n",dir_path);

	if (result < 0) {
		LOGD("ACDB -> Error: snprintf failed for variant \n");
		result = -ENODEV;
		goto done;
	}

	//begin add by kun.zheng for task 8598614 on 2019/11/15
	if (result_acdbinfo < 0) {
		LOGD("ACDB -> Error: snprintf failed for audioVersionPath \n");
	} else {
		property_set(PROPERTY_AUDIOVERSIONPATH, audioVersionPath);
	}

	//end add by kun.zheng for task 8598614 on 2019/11/15
	//End   Modified by fuhua.wang for task_id: 8810241 on 2020 01.09

         result = get_acdb_files_in_directory(acdb_init_cmd, dir_path);
	 if (result > 0)
	   goto done;
       #endif
//以上就是读取参数的关键流程了。  

```

### 调试软件读取参数的接口  
上述的dir_path会作为接口，在用QCAT验证时能看到读取了对应的参数。  

### 参数放哪儿  
vendor/qcom/proprietary/mm-audio-cal/audcal/acdbdata/8937/   
其中分为CAN、US、QRD、SMIC  
vendor/qcom/proprietary/mm-audio-cal/audcal/acdbdata/8937/CAN  
vendor/qcom/proprietary/mm-audio-cal/audcal/acdbdata/8937/US  
vendor/qcom/proprietary/mm-audio-cal/audcal/acdbdata/8937/QRD  
vendor/qcom/proprietary/mm-audio-cal/audcal/acdbdata/8937/SMIC  
目录中有acdb.info文件，以跟踪提交记录。  

vendor/qcom/proprietary/mm-audio-cal/audcal/acdbdata/8937/Android.mk文件则会编译acdb的模块。  
```
#Begin Modified by fuhua.wang for task_id: 8810241 on 2020 01.09
#----------------------------------------------------------------------------------
#             Populate ACDB data files to file system for TCT Signal Mic
# ---------------------------------------------------------------------------------

include $(CLEAR_VARS)
LOCAL_MODULE            := SMIC_Bluetooth_cal.acdb
LOCAL_MODULE_FILENAME   := Bluetooth_cal.acdb
LOCAL_MODULE_TAGS       := optional
LOCAL_MODULE_CLASS      := ETC
LOCAL_MODULE_PATH       := $(TARGET_OUT_VENDOR_ETC)/acdbdata/SMIC/
LOCAL_SRC_FILES         := SMIC/SMIC_Bluetooth_cal.acdb
include $(BUILD_PREBUILT)

include $(CLEAR_VARS)
LOCAL_MODULE            := SMIC_General_cal.acdb
LOCAL_MODULE_FILENAME   := General_cal.acdb
LOCAL_MODULE_TAGS       := optional
LOCAL_MODULE_CLASS      := ETC
LOCAL_MODULE_PATH       := $(TARGET_OUT_VENDOR_ETC)/acdbdata/SMIC/
LOCAL_SRC_FILES         := SMIC/SMIC_General_cal.acdb
include $(BUILD_PREBUILT)

include $(CLEAR_VARS)
LOCAL_MODULE            := SMIC_Global_cal.acdb
LOCAL_MODULE_FILENAME   := Global_cal.acdb
LOCAL_MODULE_TAGS       := optional
LOCAL_MODULE_CLASS      := ETC
LOCAL_MODULE_PATH       := $(TARGET_OUT_VENDOR_ETC)/acdbdata/SMIC/
LOCAL_SRC_FILES         := SMIC/SMIC_Global_cal.acdb
include $(BUILD_PREBUILT)

include $(CLEAR_VARS)
LOCAL_MODULE            := SMIC_Handset_cal.acdb
LOCAL_MODULE_FILENAME   := Handset_cal.acdb
LOCAL_MODULE_TAGS       := optional
LOCAL_MODULE_CLASS      := ETC
LOCAL_MODULE_PATH       := $(TARGET_OUT_VENDOR_ETC)/acdbdata/SMIC/
LOCAL_SRC_FILES         := SMIC/SMIC_Handset_cal.acdb
include $(BUILD_PREBUILT)


......


include $(CLEAR_VARS)
LOCAL_MODULE            := US_Speaker_cal.acdb
LOCAL_MODULE_FILENAME   := Speaker_cal.acdb
LOCAL_MODULE_TAGS       := optional
LOCAL_MODULE_CLASS      := ETC
LOCAL_MODULE_PATH       := $(TARGET_OUT_VENDOR_ETC)/acdbdata/US/
LOCAL_SRC_FILES         := US/US_Speaker_cal.acdb
include $(BUILD_PREBUILT)
include $(CLEAR_VARS)
LOCAL_MODULE            := us_acdb.info
LOCAL_MODULE_FILENAME   := us_acdb.info
LOCAL_MODULE_TAGS       := optional 
LOCAL_MODULE_CLASS      := ETC
LOCAL_MODULE_PATH       := $(TARGET_OUT_VENDOR_ETC)/acdbdata/US/
LOCAL_SRC_FILES         := US/acdb.info
include $(BUILD_PREBUILT)
#End  Modified by fuhua.wang for task_id: 8952191 on 2020 02.25
```

但是这还不够，还需在vendor/qcom/proprietary/common/config/device-vendor.mk中添加模块编译指令，否则out目录是没办法编译出来的。  
```
#Begin Modified by fuhua.wang for task_id: 8952191 on 2020 02.25
MM_AUDIO += SMIC_Bluetooth_cal.acdb
MM_AUDIO += SMIC_General_cal.acdb
MM_AUDIO += SMIC_Global_cal.acdb
MM_AUDIO += SMIC_Handset_cal.acdb
MM_AUDIO += SMIC_Hdmi_cal.acdb
MM_AUDIO += SMIC_Headset_cal.acdb
MM_AUDIO += SMIC_Speaker_cal.acdb
MM_AUDIO += smic_acdb.info
MM_AUDIO += CAN_Bluetooth_cal.acdb
MM_AUDIO += CAN_General_cal.acdb
MM_AUDIO += CAN_Global_cal.acdb
MM_AUDIO += CAN_Handset_cal.acdb
MM_AUDIO += CAN_Hdmi_cal.acdb
MM_AUDIO += CAN_Headset_cal.acdb
MM_AUDIO += CAN_Speaker_cal.acdb
MM_AUDIO += can_acdb.info
MM_AUDIO += US_Bluetooth_cal.acdb
MM_AUDIO += US_General_cal.acdb
MM_AUDIO += US_Global_cal.acdb
MM_AUDIO += US_Handset_cal.acdb
MM_AUDIO += US_Hdmi_cal.acdb
MM_AUDIO += US_Headset_cal.acdb
MM_AUDIO += US_Speaker_cal.acdb
MM_AUDIO += us_acdb.info
#End  Modified by fuhua.wang for task_id: 8952191 on 2020 02.25
```

### 单双mic定制开关  
并没有像MTK那样涉及免提主副mic拾音的选择，而是针对fluence的业务搞的，关键变量`fluence_cap`。   
hardware/qcom/audio/hal/msm8916/platform.c  
```
void *platform_init(struct audio_device *adev)
{
    char value[PROPERTY_VALUE_MAX];
    struct platform_data *my_data = NULL;
    int snd_card_num = 0;
    const char *snd_card_name;
    char mixer_xml_path[MAX_MIXER_XML_PATH],ffspEnable[PROPERTY_VALUE_MAX];
    const char *mixer_ctl_name = "Set HPX ActiveBe";
    struct mixer_ctl *ctl = NULL;
    int idx;
    int wsaCount =0;
    bool is_wsa_combo_supported = false;
    const char *id_string = NULL;
    int cfg_value = -1;
    
    #ifdef TCT_AUDIO_CUSTOMAIZATION_FOR_FCM
      char variant_to_chose[PROPERTY_VALUE_MAX]={0};
    #endif
    

    ......


    
    #ifdef TCT_AUDIO_CUSTOMAIZATION_FOR_FCM
      property_get("ro.boot.cu", variant_to_chose, "");
      ALOGD("%s: variant_to_chose %s", __func__, variant_to_chose);
      //if((strstr(variant_to_chose,"5002A")!= NULL)||(strstr(variant_to_chose,"5002J")!= NULL)){ //signal mic for Seoul all variant
      if (1) {//signal mic for Seoul all variant 
        strcpy(my_data->fluence_cap ,"none");        
      } else{
        strcpy(my_data->fluence_cap ,"fluence");        
      }
    #else
      property_get("ro.vendor.audio.sdk.fluencetype", my_data->fluence_cap, "");
    #endif
   

```






## 综述  
Qcom的代码逻辑清楚，定制起来方便快捷，相比MTK着实舒服。  
deviceinfo代码基本和MTK的项目差不多，就不赘述了。  

不过自检场景确实应该强调一下：  
(1)setting -> sound -> alarm  
看是否有卡顿或者断续等。  
(2)setting -> sound -> media  
(3)setting -> sound -> reciver  
(4)播放音乐5分钟，看是否有衰弱或者不正常音质。(可能是smart PA没有校准，进行了温度保护，可能是DAPM关断了通路)  
(5)录音，两个mic是否都能录音，且正常播放。  
(6)免提通话是否有声音。  
(7)voip通话是否有声音。  
(8)代码中各类定制是否都正常~(参数定制-特别是默认的时候或者参数刷错的时候，codec定制，DMNR消噪定制，双mic，单mic，speaker path，音效)。  

