+++
author = "fuhua"
title = "Android - Audio - MT67平台 - audio参数定制"
date = "2020-03-10 15:55:03"
description = "手机技术笔记"
tags = [
    "markdown",
    "text",
]
weight = 10
+++


代码巨多，谨慎选择性阅读
- [背景](#背景)  
- [tokyo项目参数定制涉及目录](#tokyo项目参数定制涉及目录)  
    - [开机加载参数的流程](#开机加载参数的流程)  
    - [调试软件读取参数的接口](#调试软件读取参数的接口)  
- [针对mt6762项目的改动](#针对mt6762项目的改动)  
	- [参数目录](#参数目录)  
	- [device目录](#device目录)  
	- [实际加载参数目录](#实际加载参数目录)  
	- [mic开关定制目录](#mic开关定制目录)  
	- [权限目录](#权限目录)  
    - [smart PA 特定的参数部分](#smart-PA-特定的参数部分)  
    - [mtk针对参数哪些不能调了](#mtk针对参数哪些不能调了)  
- [确认加载的参数](#确认加载的参数)  
    - [deviceinfo相关代码](#deviceinfo相关代码)  
    - [自检场景](#自检场景)  




## 背景
>公司对MT6762情有独钟，搞了三个项目共基线，各大运营商针对audio参数可能需要不同的，宁波site只做高端机器，可能由于客户量少和性能好的原因不会遇到这些问题。去年针对mt6761做的参数定制已经普遍适用了，但是都是针对一个项目或者两个手机项目，这次来了三个项目，加上了Smart PA，需要写一下，1来可以再次理清软件思路，2来方便回顾参考。  


## tokyo项目参数定制涉及目录  

参数目录：  
vendor/mediatek/proprietary/custom/k62v1_64_bsp/audio_param_custom
一共有三套参数，默认的和两个运营商的，每套参数下面还有一个acdb.info，方便追溯提交的参数版本。  

编译期间拷贝参数和acdb.info文件操作：  
device/mediateksample/k62v1_64_bsp/device.mk  

手机开机加载参数的目录：  
vendor/mediatek/proprietary/external/AudioParamParser  

mic开关定制：  
vendor/mediatek/proprietary/hardware/audio/common/speech_driver  

权限问题：  
device/mediatek/sepolicy/basic/non_plat  
可以通过开机设置selinux权限验证是否是权限问题。  


参数定制是为了迎合一套硬件多套软件的需求，那么audio相关的差异化需求就需要在开机的时候确定，比如哪套参数，是否单mic，也要预防参数是否出错，用smartPA的path就不能写lo_int_out。  


### 开机加载参数的流程  
开机之后，在读取参数的AudioParamParser目录中进行参数读取，`appGetXmlDirFromProperty`此函数通过`property_get(PROPERTY_KEY_XML_DEF_PATH, xmlDir, "");`接口，操作参数。  
原始参数读取接口如此，对比Qcom的实现差异很大，Qcom的参数定制往后再写。  

读取到参数之后，media.audiopolicy服务则可以正常启动，否则会一直卡在启动这个服务。  

### 调试软件读取参数的接口  
通过对电脑端的Audio_Tuning_tool的log解读，我们可以发现连接的接口：  
较新的版本应该是："persist.vendor.audio.tuning.def_path"  
比较旧的版本是："audio.tuning.def_path"  
在参数定制的时候将该prop填写进去即可保证手机读取的参数和调试的参数路径一致了。  


## 针对mt6762项目的改动  
大致的修改目录及思路同tokyo项目差不多。但是针对主要逻辑需要注意不同项目的区分。修改记录如下：  

### 参数目录  
总目录在vendor/mediatek/proprietary/custom
(1) Milan项目：  
k62v1_64_bsp/  
增加audio_param_custom/aw_smartpa_firmware/目录，暂时拿来装固件。  
增加k62v1_64_bsp/audio_param_custom/mtk_default/mtk_default/目录，拿来装默认参数。  
增加k62v1_64_bsp/audio_param_custom/milan_common/目录，装该项目第一套参数  

(2) Apollo项目：  
apollo/  
增加audio_param_custom/aw_smartpa_firmware/目录，暂时拿来装固件。  
增加apollo/audio_param_custom/mtk_default/mtk_default/目录，拿来装默认参数。  
增加apollo/audio_param_custom/apollo_common/目录，装该项目第一套参数。  

(3) Oakland目：  
oakland/  
增加audio_param_custom/aw_smartpa_firmware/目录，暂时拿来装固件。  
增加oakland/audio_param_custom/mtk_default/mtk_default/目录，拿来装默认参数。  
增加oakland/audio_param_custom/oakland_common/目录，装该项目第一套参数。   


### device目录  
(1) Milan项目：  
device/mediateksample/k62v1_64_bsp/device.mk  
```
#Begin Add by fuhua.wang for Audio task 8984095 on 2020/03/10
ifeq ($(TCT_ENABLE_FCM),1)
MTK_AUDIO_PARAM_DIR_LIST += vendor/mediatek/proprietary/custom/$(MTK_TARGET_PROJECT)/audio_param_v/mtk_default/mtk_default
PRODUCT_COPY_FILES += vendor/mediatek/proprietary/custom/$(MTK_TARGET_PROJECT)/audio_param_custom/mtk_default/acdb.info:$(TARGET_COPY_OUT_VENDOR)/etc/acdb.info

PRODUCT_COPY_FILES += vendor/mediatek/proprietary/custom/$(MTK_TARGET_PROJECT)/audio_param_custom/milan_common/acdb.info:$(TARGET_COPY_OUT_VENDOR)/etc/audio_param_custom/milan_common/acdb.info
MTK_AUDIO_PARAM_DIR_LIST_CUSTOM_MILAN_COMMON += vendor/mediatek/proprietary/custom/$(MTK_TARGET_PROJECT)/audio_param_custom/milan_common/audio_param

endif # TCT_ENABLE_FCM
#End   Add by fuhua.wang for Audio task 8984095 on 2020/03/10
```

(2) Apollo项目：  
device/mediateksample/apollo  
```
#Begin Add by fuhua.wang for Audio task 8984095 on 2020/03/10
ifeq ($(TCT_ENABLE_FCM),1)
MTK_AUDIO_PARAM_DIR_LIST += vendor/mediatek/proprietary/custom/$(MTK_TARGET_PROJECT)/audio_param_custom/mtk_default/mtk_default
PRODUCT_COPY_FILES += vendor/mediatek/proprietary/custom/$(MTK_TARGET_PROJECT)/audio_param_custom/mtk_default/acdb.info:$(TARGET_COPY_OUT_VENDOR)/etc/acdb.info

PRODUCT_COPY_FILES += vendor/mediatek/proprietary/custom/$(MTK_TARGET_PROJECT)/audio_param_custom/apollo_common/acdb.info:$(TARGET_COPY_OUT_VENDOR)/etc/audio_param_custom/apollo_common/acdb.info
MTK_AUDIO_PARAM_DIR_LIST_CUSTOM_APOLLO_COMMON += vendor/mediatek/proprietary/custom/$(MTK_TARGET_PROJECT)/audio_param_custom/apollo_common/audio_param

endif # TCT_ENABLE_FCM
#End   Add by fuhua.wang for Audio task 8984095 on 2020/03/10

```

(3) Oakland目：  
device/mediateksample/oakland  
```
#Begin Add by fuhua.wang for Audio task 8984095 on 2020/03/10
ifeq ($(TCT_ENABLE_FCM),1)
MTK_AUDIO_PARAM_DIR_LIST += vendor/mediatek/proprietary/custom/$(MTK_TARGET_PROJECT)/audio_param_custom/mtk_default/mtk_default
PRODUCT_COPY_FILES += vendor/mediatek/proprietary/custom/$(MTK_TARGET_PROJECT)/audio_param_custom/mtk_default/acdb.info:$(TARGET_COPY_OUT_VENDOR)/etc/acdb.info

PRODUCT_COPY_FILES += vendor/mediatek/proprietary/custom/$(MTK_TARGET_PROJECT)/audio_param_custom/oakland_common/acdb.info:$(TARGET_COPY_OUT_VENDOR)/etc/audio_param_custom/oakland_common/acdb.info
MTK_AUDIO_PARAM_DIR_LIST_CUSTOM_OAKLAND_COMMON += vendor/mediatek/proprietary/custom/$(MTK_TARGET_PROJECT)/audio_param_custom/oakland_common/audio_param

#PRODUCT_COPY_FILES += vendor/mediatek/proprietary/custom/$(MTK_TARGET_PROJECT)/audio_param_custom/apollo_common/acdb.info:$(TARGET_COPY_OUT_VENDOR)/etc/audio_param_custom/apollo_common/acdb.info
#MTK_AUDIO_PARAM_DIR_LIST_CUSTOM_APOLLO_COMMON += vendor/mediatek/proprietary/custom/$(MTK_TARGET_PROJECT)/audio_param_custom/apollo_common/audio_param

endif # TCT_ENABLE_FCM
#End   Add by fuhua.wang for Audio task 8984095 on 2020/03/10
```


### 实际加载参数目录  
vendor/mediatek/proprietary/external/AudioParamParser
```
	修改:         AudioParamParser.h
	修改:         AudioUtils.c
	修改:         DeployAudioParam.mk

```
AudioParamParser.h中指定定制的路径：  
```
//Begin Add by fuhua.wang for FCM Audio task 8321678--8179485 on 2019/09/09
#if TCT_ENABLE_FCM
typedef struct AudioPathInfo {
    const char codename[16];
    const char varientid[16];
    const char subvarientid[16];
    const char audiopath[128];
    const char refmicinloudspk[8];
} AudioPathInfo;
/* Begin mod by fuhua.wang for Audio task 8984095 on 2020/03/10 */
static const AudioPathInfo custom_to_chose[] = {
    //Tokyo GL
    {"5028A", "", "", "/vendor/etc/audio_param_custom/tokyo_common/audio_param", "1"},
    {"5028D", "", "", "/vendor/etc/audio_param_custom/tokyo_vdf/audio_param", "1"}, //VDF
    {"5028Y", "", "", "/vendor/etc/audio_param_custom/tokyo_orange/audio_param", "1"}, //Orange
    {"5128Y", "", "", "/vendor/etc/audio_param_custom/tokyo_common/audio_param", "1"},
    {"5128A", "", "", "/vendor/etc/audio_param_custom/tokyo_common/audio_param", "1"},

    //Tokyo Pro
    {"5029D", "", "", "/vendor/etc/audio_param_custom/tokyo_common/audio_param", "1"},
    {"5029Y", "", "", "/vendor/etc/audio_param_custom/tokyo_common/audio_param", "1"},
    {"5029E", "", "", "/vendor/etc/audio_param_custom/tokyo_common/audio_param", "1"},
    {"5129Y", "", "", "/vendor/etc/audio_param_custom/tokyo_common/audio_param", "1"},

    //Tokyo lite
    {"5007U", "", "", "/vendor/etc/audio_param_custom/tokyo_lite_s_mic/audio_param", "0"},
    {"5007A", "", "", "/vendor/etc/audio_param_custom/tokyo_lite_s_mic/audio_param", "0"},
    {"5007G", "", "", "/vendor/etc/audio_param_custom/tokyo_lite_s_mic/audio_param", "0"},
    {"5107G", "", "", "/vendor/etc/audio_param_custom/tokyo_lite_s_mic/audio_param", "0"},
    {"5107J", "", "", "/vendor/etc/audio_param_custom/tokyo_lite_s_mic/audio_param", "0"},

    //Milan
    {"5061K", "", "", "/vendor/etc/audio_param_custom/milan_common/audio_param", "1"},
    {"5061U", "", "", "/vendor/etc/audio_param_custom/milan_common/audio_param", "1"},
    {"5061A", "", "", "/vendor/etc/audio_param_custom/milan_common/audio_param", "1"},
    {"5161A", "", "", "/vendor/etc/audio_param_custom/milan_common/audio_param", "1"},

    //Apollo
    {"9032X", "", "", "/vendor/etc/audio_param_custom/apollo_common/audio_param", "0"},
    {"9032X", "", "", "/vendor/etc/audio_param_custom/apollo_common/audio_param", "0"},
    {"9032T", "", "", "/vendor/etc/audio_param_custom/apollo_common/audio_param", "0"},


    //default
    {"nomatch", "", "", "/vendor/etc/audio_param", "1"}
};
/* End   mod by fuhua.wang for Audio task 8984095 on 2020/03/10 */
```

AudioParamParserPriv.h里面也有重要的定义，由于很早就做过了，所以这次修改就没有体现：  

```
//Begin Mod by fuhua.wang for FCM Audio task 8321678--8179485 on 2019/09/09
#if TCT_ENABLE_FCM
#define PROPERTY_KEY_XML_DEF_NEW_VERSION_TUNING	"persist.vendor.audio.tuning.def_path"
#define PROPERTY_KEY_XML_DEF_TUNING_PATH		"audio.tuning.def_path"
#define PROPERTY_KEY_XML_DEF_PATH     			"ro.vendor.audio.path"
#define PROPERTY_REFMIC_IN_LOUDSPK    			"ro.vendor.audio.refmicinloudspk"
#define PROPERTY_RO_BOOT_VARIANT      			"ro.boot.variant"
#define PROPERTY_RO_BOOT_SUBVARIANT   			"ro.boot.subvariant"
#define PROPERTY_RO_PRODUCT_DEVICE      		"ro.product.device"
#define PROPERTY_RO_PRODUCT_NAME      		    "ro.product.name"
#else
#define PROPERTY_KEY_XML_DEF_PATH               "persist.vendor.audio.tuning.def_path"
#endif
//End  Mod by fuhua.wang for FCM Audio task 8321678--8179485 on 2019/09/09
```
AudioUtils.c 文件则是控制参数选择的逻辑。   
```
/* Begin mod by fuhua.wang for Audio task 8984095 on 2020/03/10 */
#if TCT_ENABLE_FCM
    char devicename[PROPERTY_VALUE_MAX] = {0};
    char codename[PROPERTY_VALUE_MAX] = {0};
    char varient[PROPERTY_VALUE_MAX] = {0};
    char subvarient[PROPERTY_VALUE_MAX] = {0};
    char new_version_tool_audiopath[256];
    int chose_id = 0;
    int size_of_product = 0;

    property_get(PROPERTY_RO_PRODUCT_DEVICE, devicename, "");
    property_get(PROPERTY_RO_PRODUCT_NAME, codename, "");
    property_get(PROPERTY_RO_BOOT_VARIANT, varient, "");
    property_get(PROPERTY_RO_BOOT_SUBVARIANT, subvarient, "");
    //property_get(PROPERTY_KEY_XML_DEF_NEW_VERSION_TUNING,new_version_tool_audiopath,"");
    size_of_product = sizeof(custom_to_chose)/sizeof(AudioPathInfo);
    MUST_LOG("FCM_audio: devicename=%s, codename=%s, variant=%s, subvariant=%s, size_of_product=%d",\
		devicename, codename, varient, subvarient, size_of_product);

    do {
        if(strcmp(codename, custom_to_chose[chose_id].codename)==0)
        {
            /* if needed, add compare here by variant */

            MUST_LOG("FCM_audio: break, chose_id = %d",chose_id);
            break;
        }
        //MUST_LOG("FCM_audio: chose_id = %d, codename=%s", chose_id, custom_to_chose[chose_id].codename) ;
    }while (++chose_id <= size_of_product - 1);

    //if not matched, default common param.
    if(chose_id >= size_of_product)
    {
        chose_id = size_of_product - 1;
    }
    MUST_LOG("FCM_audio: [%d][%s, %s, %s, %s, %s]", chose_id,\
		custom_to_chose[chose_id].codename, \
		custom_to_chose[chose_id].varientid, \
		custom_to_chose[chose_id].subvarientid,\
		custom_to_chose[chose_id].audiopath,\
		custom_to_chose[chose_id].refmicinloudspk);

    property_set(PROPERTY_REFMIC_IN_LOUDSPK, custom_to_chose[chose_id].refmicinloudspk);
    property_set(PROPERTY_KEY_XML_DEF_PATH, custom_to_chose[chose_id].audiopath);
    property_set(PROPERTY_KEY_XML_DEF_TUNING_PATH, custom_to_chose[chose_id].audiopath);
    sprintf(new_version_tool_audiopath,"%s/.", custom_to_chose[chose_id].audiopath);
    property_set(PROPERTY_KEY_XML_DEF_NEW_VERSION_TUNING, new_version_tool_audiopath);

    MUST_LOG("FCM_audio: new_version_tool_audiopath = %s", new_version_tool_audiopath);
#else

#endif
/* End   mod by fuhua.wang for Audio task 8984095 on 2020/03/10 */
```
DeployAudioParam.mk 则是将所有参数拷贝到定制目录的文件，实际上这个文件最主要的作用是是拿来生成audiooption.xml文件的，既然要做参数定制，这个文件就要自己维护了，所以这个文件只保留其拷贝和解压参数的作用。  
```
AUDIO_PARAM_OUT_DIR := $(TARGET_OUT_VENDOR_ETC)/audio_param
#Begin Mod by fuhua.wang for FCM Audio task 8321678--8179485 on 2019/09/09
ifeq ($(TCT_ENABLE_FCM),1)

#Begin Add by fuhua.wang for Orange Audio task 8461838 on 2019/10/22
AUDIO_PARAM_OUT_DIR_CUSTOM_MILAN_COMMON := $(TARGET_OUT_VENDOR_ETC)/audio_param_custom/milan_common/audio_param
#End  Add  by fuhua.wang for Orange Audio task 8461838 on 2019/10/22

#Begin Add by fuhua.wang for Audio task 8984095 on 2020/03/10
AUDIO_PARAM_OUT_DIR_CUSTOM_APOLLO_COMMON := $(TARGET_OUT_VENDOR_ETC)/audio_param_custom/apollo_common/audio_param
#End   Add by fuhua.wang for Audio task 8984095 on 2020/03/10

#Begin Add by fuhua.wang for Audio task 8984095 on 2020/03/10
AUDIO_PARAM_OUT_DIR_CUSTOM_OAKLAND_COMMON := $(TARGET_OUT_VENDOR_ETC)/audio_param_custom/oakland_common/audio_param
#End   Add by fuhua.wang for Audio task 8984095 on 2020/03/10


EXTRACT_FILE_LIST := *_AudioParam.xml *_ParamUnitDesc.xml *_ParamTreeView.xml *ParamOptions.xml
else
EXTRACT_FILE_LIST := *_AudioParam.xml *_ParamUnitDesc.xml *_ParamTreeView.xml
endif #end TCT_ENABLE_FCM
#End  Mod by fuhua.wang for FCM Audio task 8321678--8179485 on 2019/09/09

······


#Begin Add by fuhua.wang for Audio task 8984095 on 2020/03/10
#please notice AUDIO_PARAM_OUT_DIR_CUSTOM_MILAN_COMMON && MTK_AUDIO_PARAM_DIR_LIST_CUSTOM_MILAN_COMMON(this is in device.mk)
ifeq ($(TCT_ENABLE_FCM)_$(MTK_TARGET_PROJECT), 1_k62v1_64_bsp)
LOCAL_AUDIO_PARAM_COPY_FILE_LIST := $(filter $(LOCAL_AUDIO_PARAM_FILE_PATTERN),$(foreach d,$(CUSTOM_AUDIO_PARAM_DIR_LIST) $(MTK_AUDIO_PARAM_DIR_LIST_CUSTOM_MILAN_COMMON),$(wildcard $(d)/*.xml)))
LOCAL_AUDIO_PARAM_COPY_FILE_STEM := $(sort $(filter-out $(notdir $(LOCAL_AUDIO_PARAM_UNZIP_FILE_LIST)),$(notdir $(LOCAL_AUDIO_PARAM_COPY_FILE_LIST))))
$(foreach f,$(LOCAL_AUDIO_PARAM_COPY_FILE_STEM),\
	$(eval chk := $(filter %/$(f),$(LOCAL_AUDIO_PARAM_COPY_FILE_LIST)))\
	$(eval src := $(firstword $(chk)))\
	$(eval exc := $(filter-out $(src),$(chk)))\
	$(if $(strip $(exc)),$(info AudioParam: $(src) overrides $(exc)))\
	$(eval dst := $(AUDIO_PARAM_OUT_DIR_CUSTOM_MILAN_COMMON)/$(f))\
	$(eval LOCAL_AUDIO_PARAM_INSTALLED += $(dst))\
	$(eval $(call copy-one-file,$(src),$(dst)))\
)
endif

ifeq ($(TCT_ENABLE_FCM)_$(MTK_TARGET_PROJECT), 1_apollo)
LOCAL_AUDIO_PARAM_COPY_FILE_LIST := $(filter $(LOCAL_AUDIO_PARAM_FILE_PATTERN),$(foreach d,$(CUSTOM_AUDIO_PARAM_DIR_LIST) $(MTK_AUDIO_PARAM_DIR_LIST_CUSTOM_APOLLO_COMMON),$(wildcard $(d)/*.xml)))
LOCAL_AUDIO_PARAM_COPY_FILE_STEM := $(sort $(filter-out $(notdir $(LOCAL_AUDIO_PARAM_UNZIP_FILE_LIST)),$(notdir $(LOCAL_AUDIO_PARAM_COPY_FILE_LIST))))
$(foreach f,$(LOCAL_AUDIO_PARAM_COPY_FILE_STEM),\
	$(eval chk := $(filter %/$(f),$(LOCAL_AUDIO_PARAM_COPY_FILE_LIST)))\
	$(eval src := $(firstword $(chk)))\
	$(eval exc := $(filter-out $(src),$(chk)))\
	$(if $(strip $(exc)),$(info AudioParam: $(src) overrides $(exc)))\
	$(eval dst := $(AUDIO_PARAM_OUT_DIR_CUSTOM_APOLLO_COMMON)/$(f))\
	$(eval LOCAL_AUDIO_PARAM_INSTALLED += $(dst))\
	$(eval $(call copy-one-file,$(src),$(dst)))\
)
endif


ifeq ($(TCT_ENABLE_FCM)_$(MTK_TARGET_PROJECT), 1_oakland)
LOCAL_AUDIO_PARAM_COPY_FILE_LIST := $(filter $(LOCAL_AUDIO_PARAM_FILE_PATTERN),$(foreach d,$(CUSTOM_AUDIO_PARAM_DIR_LIST) $(MTK_AUDIO_PARAM_DIR_LIST_CUSTOM_OAKLAND_COMMON),$(wildcard $(d)/*.xml)))
LOCAL_AUDIO_PARAM_COPY_FILE_STEM := $(sort $(filter-out $(notdir $(LOCAL_AUDIO_PARAM_UNZIP_FILE_LIST)),$(notdir $(LOCAL_AUDIO_PARAM_COPY_FILE_LIST))))
$(foreach f,$(LOCAL_AUDIO_PARAM_COPY_FILE_STEM),\
	$(eval chk := $(filter %/$(f),$(LOCAL_AUDIO_PARAM_COPY_FILE_LIST)))\
	$(eval src := $(firstword $(chk)))\
	$(eval exc := $(filter-out $(src),$(chk)))\
	$(if $(strip $(exc)),$(info AudioParam: $(src) overrides $(exc)))\
	$(eval dst := $(AUDIO_PARAM_OUT_DIR_CUSTOM_OAKLAND_COMMON)/$(f))\
	$(eval LOCAL_AUDIO_PARAM_INSTALLED += $(dst))\
	$(eval $(call copy-one-file,$(src),$(dst)))\
)
endif
#End   Add by fuhua.wang for Audio task 8984095 on 2020/03/10

······

```

### mic开关定制目录  
vendor/mediatek/proprietary/hardware/audio  

E:\code\xiongbo_server\milan-apollo-dint-0309-fuhua\vendor\mediatek\proprietary\hardware\audio\common\speech_driver\AudioALSASpeechPhoneCallController.cpp  
```
//Begin Mod by fuhua.wang for FCM Audio task 8321678--8179485 on 2019/09/09
#if TCT_ENABLE_FCM
            
            char refmicinloudspk[PROPERTY_VALUE_MAX];
            property_get("ro.vendor.audio.refmicinloudspk", refmicinloudspk, "");
            ALOGD("%s(), in FCM_audio:ro.vendor.audio.refmicinloudsp = %s",__FUNCTION__,refmicinloudspk);
            if (refmicinloudspk[0] == '1') {
                input_device = AUDIO_DEVICE_IN_BACK_MIC;
            } else {
                input_device = AUDIO_DEVICE_IN_BUILTIN_MIC;
            }
#else
            ALOGD("%s(), not in FCM_audio, use USE_REFMIC_IN_LOUDSPK",__FUNCTION__);
            if (USE_REFMIC_IN_LOUDSPK == 1) {
                input_device = AUDIO_DEVICE_IN_BACK_MIC;
            } else {
                input_device = AUDIO_DEVICE_IN_BUILTIN_MIC;
            }       
#endif
//End  Mod by fuhua.wang for FCM Audio task 8321678--8179485 on 2019/09/09
```
这其中有个问题就是 TCT_ENABLE_FCM 可能无效，编译的时候会导致不能跑到FCM定制框架中，需要添加如下代码  
E:\code\xiongbo_server\milan-apollo-dint-0309-fuhua\vendor\mediatek\proprietary\hardware\audio\Android.mk  
```
#Begin Add by fuhua.wang for Audio task 8984095 on 2020/03/10
ifeq ($(strip $(TCT_ENABLE_FCM)), 1)
   LOCAL_CFLAGS += -DTCT_ENABLE_FCM=1
endif
#End   Add by fuhua.wang for Audio task 8984095 on 2020/03/10
```



### 权限目录  
device/mediatek/sepolicy/basic/non_plat  
mtk_hal_audio.te文件好像是设置组别
```
# set_prop(mtk_hal_audio, mtk_core_property_type);
get_prop(mtk_hal_audio, hwservicemanager_prop);
allow mtk_hal_audio tct_default_prop:property_service set;
allow mtk_hal_audio tct_default_prop:file { read open getattr };
```


property_contexts 文件就是定义属性值了。  
```
ro.vendor.audio.path u:object_r:hwservicemanager_prop:s0
persist.vendor.audio.tuning.def_path u:object_r:tct_default_prop:s0

#=============audio refmic_in_loudspk========#
ro.vendor.audio.refmicinloudspk u:object_r:hwservicemanager_prop:s0

```



mk文件的debug会有一些语法小技巧，mark一下  
（1）ifeq 是没有办法嵌套的，只能进行字符串的叠加进行与或者逻辑与，findstring进行或。  
（2）ifeq 是没有办法用else的，编译不过。不知道是不是endif没有加对，但都试过了。  

### smart PA 特定的参数部分  
(1) 算法  
vendor/mediatek/proprietary/hardware/audio/common/sktune  

如果要做定制，sat文件在hal层加载，可以通过属性值进行定制。  
也可以和固件一样的思路。    


(2) 固件  
和参数放在了一个位置：vendor/mediatek/proprietary/custom  

在kernel层加载。  
如果要做定制，需要找到开机参数在firewire_class加载的时候进行定制。  
由于加载固件的时候是用的kernel的接口，默认都存放在vendor/firmware下面，需要做区分

但是，也有另一种方案，就是不同项目用的不同的device.mk，只需要保证不同项目加载不同的device.mk就ok。  




(3) 本身的参数  
已经做好定制。    

PS:实际上这个供应商的固件和算法是否应该定制，可以做了考量之后再决定，如果要针对不同的硬件结构或者喇叭用料等定制，那么就有必要。  


### mtk针对参数哪些不能调了  
暂时只发现speaker的模拟增益不能调了，数字增益似乎都还可以调。  

## 确认加载的参数  
>方式多种多样，我这边直接采用prop进行确认，显示在deviceinfo界面。  


### deviceinfo相关代码  
packages\apps\JrdMMITest\JrdMMITest-main\src\com\jrdcom\mmitest\common\DeviceInfoActivity_A5X.java  

```
/*Begin Add by fuhua.wang for Add audioVersion infomation on 2019 06.12 task_id:7852064*/
import android.os.SystemProperties;
import android.widget.Toast;
/*End   Add by fuhua.wang for Add audioVersion infomation on 2019 06.12 task_id:7852064*/

......

public class DeviceInfoActivity_A5X extends Activity {
    ......

	//Begin Add by fuhua.wang for Add audioVersion infomation on 2019 06.12 task_id:7852064
	String acdb_info_path		= new String();
	//End   Add by fuhua.wang for Add audioVersion infomation on 2019 06.12 task_id:7852064




		/*Begin Add by fuhua.wang for Add audioVersion infomation on 2019 06.12 task_id:7852064*/
		checkAudioParamifdefault_Getacdbinfo();
        getFileInformance(acdb_info_path);
		String codename 	= SystemProperties.get("ro.boot.code");
		FileInformance += "codename :" + codename + "\n";
        text_cen_zone = (TextView) findViewById(R.id.audioVersion);
        text_cen_zone.setText(FileInformance + "\n");
        text_cen_zone.setTextSize(20);		
		/*End   Add by fuhua.wang for Add audioVersion infomation on 2019 06.12 task_id:7852064*/

......


/*Begin Add by fuhua.wang for Add audioVersion infomation on 2019 06.12 task_id:7852064*/
//get acdb_info_path and show text
private synchronized void checkAudioParamifdefault_Getacdbinfo(){
    //setp 1. 
    String audio_param_path  			= SystemProperties.get("ro.vendor.audio.path");
    //String audio_param_path  = SystemProperties.get("persist.vendor.audio.tuning.def_path");
    String audio_param_path_default 	= "/vendor/etc/audio_param";
    String acdb_info_flag 				= "audio_param";

    /*step 2.
    audio_param_path compare to default, if is default,shouw toast;else show normal,
    otherwise you can use lastIndexOf()to get both default and custom acdb.info.
    */
    if(audio_param_path.compareTo(audio_param_path_default) == 0)
    {	
        //audio_param_path is default so show a toast
        Log.e("fuhua","get into default audio_param_path,please check the prop ro.boot.varient !!!!");
        Toast toast = Toast.makeText(this,"Audio Param is default!!! please check the variant !!!",Toast.LENGTH_LONG);
        toast.show();
        acdb_info_path = "/vendor/etc/acdb.info";
    }
    else
    {
        Log.e("fuhua","get into custom audio_param_path.");
        //begin modified by fuhua.wang for task 8347391 on 2019/09/20
        try
        {
            if (audio_param_path != null)
            {
                acdb_info_path = audio_param_path.substring(0,audio_param_path.lastIndexOf(acdb_info_flag));
                acdb_info_path = acdb_info_path + "acdb.info";
            }
            else
            {
                acdb_info_path = "no porp,please check ro.vendor.audio.path";
            }
                
        }
        catch(Exception e)
        {
            Log.e("fuhua","ro.vendor.audio.apth is null");
        }
        //end modified by fuhua.wang for task 8347391 on 2019/09/20
    }
    Log.e("fuhua","check-FCM-audio_param 'ro.vendor.audio.path' = "+audio_param_path+"   acdb_info_path = "+acdb_info_path);	

}

private synchronized void getFileInformance(String filePath){
        String fileString = new String();
        try {
                deviceReadBuffer = new BufferedReader(new FileReader(filePath));
                while((fileString = deviceReadBuffer.readLine()) != null){
                        FileInformance += fileString + "\r\n" ;
                }
        } catch (IOException e) {
                FileInformance = "no data \r\n";
                e.printStackTrace();
        } finally{
                try{
                        if(deviceReadBuffer != null)
                                deviceReadBuffer.close();
                }catch(IOException e){
                        Log.e(TAG,filePath +"can't be closed "+e);
                }
        }
}
/*End   Add by fuhua.wang for Add audioVersion infomation on 2019 06.12 task_id:7852064*/


```



### 自检场景  
(1)setting -> sound -> alarm  
看是否有卡顿或者断续等。  
(2)setting -> sound -> media  
(3)setting -> sound -> reciver  
(4)播放音乐5分钟，看是否有衰弱或者不正常音质。(可能是smart PA没有校准，进行了温度保护，可能是DAPM关断了通路)  
(5)录音，两个mic是否都能录音，且正常播放。  
(6)免提通话是否有声音。  
(7)voip通话是否有声音。  
(8)代码中各类定制是否都正常~(参数定制-特别是默认的时候或者参数刷错的时候，codec定制，DMNR消噪定制，双mic，单mic，speaker path，音效)。  

