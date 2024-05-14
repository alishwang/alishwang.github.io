+++
author = "fuhua"
title = "Android - Audio - MT67平台 - 单双spk方案共基线"
date = "2020-04-08 09:21:35"
description = "手机技术笔记"
tags = [
    "markdown",
    "text",
]
weight = 10
+++


- [背景](#背景)  
- [修改位置](#修改位置) 
- [思路](#思路)  
- [验证](#验证)


## 背景
>“是日也，天朗气清，惠风和畅，仰观宇宙之大，俯察品类之盛，所以游目驰怀，足以极视听之娱，信可乐也。”  
Aquaman项目设计有单spk和双spk两种配置，硬件方案不一样，却要求共基线。根据以往的mtk编译方案来看，spk_path都是编译的时候选择的，现在做了参数定制之后，应该有机会实现兼容。  

## 修改位置  
vendor/mediatek/proprietary/hardware/audio/common/V3/aud_drv/AudioALSAHardwareResourceManager.cpp  
```
 static const char *PROPERTY_KEY_EXTDAC = "vendor.audiohal.resource.extdac.suppo
+static int companyname_custom_spktype = -1;
 #define A2DP_DEFAULT_LANTENCY (100)
 

    if (!mSmartPaController->isSmartPAUsed() &&
-        mSmartPaController->isSmartPADynamicDetectSupport()) {
+        !mSmartPaController->isSmartPADynamicDetectSupport()) {


int AudioALSAHardwareResourceManager::setNonSmartPAType() {
     AppOps *appOps = appOpsGetInstance();
+    ALOGE("fuhua---check--setNonSmartPAType--begin");
     if (appOps == NULL) {
         ALOGE("%s(), Error: AppOps == NULL", __FUNCTION__);
         return -ENOENT;



int AudioALSAHardwareResourceManager::setNonSmartPAType() {
 
     if (strstr(spkType, "int_spk_amp")) {
         nonSmartPAType = AUDIO_SPK_INTAMP;
+        companyname_custom_spktype = AUDIO_SPK_INTAMP;
     } else if (strstr(spkType, "int_lo_buf")) {
         nonSmartPAType = AUDIO_SPK_EXTAMP_LO;
+        companyname_custom_spktype = AUDIO_SPK_EXTAMP_LO;
     } else if (strstr(spkType, "int_hp_buf")) {
         nonSmartPAType = AUDIO_SPK_EXTAMP_HP;
+        companyname_custom_spktype = AUDIO_SPK_EXTAMP_HP;
     } else if (strstr(spkType, "2_in_1_spk")) {
         nonSmartPAType = AUDIO_SPK_2_IN_1;
+        companyname_custom_spktype = AUDIO_SPK_2_IN_1;
     } else if (strstr(spkType, "3_in_1_spk")) {
         nonSmartPAType = AUDIO_SPK_3_IN_1;
+        companyname_custom_spktype = AUDIO_SPK_3_IN_1;
     } else {
         ALOGW("%s(), invalid spkType:%s", __FUNCTION__, spkType);
         nonSmartPAType = AUDIO_SPK_INVALID;
+        companyname_custom_spktype = AUDIO_SPK_INVALID;
     }
     ALOGD("%s(), nonSmartPAType: %d", __FUNCTION__, nonSmartPAType);
-
+    ALOGE("fuhua---check--setNonSmartPAType--end");
     return 0;
 }

int AudioALSAHardwareResourceManager::getNonSmartPAType() {
+    ALOGE("fuhua---check--getNonSmartPAType--begin");
     if (mSmartPaController->isSmartPADynamicDetectSupport()) {
+        ALOGE("fuhua---check --- getNonSmartPAType 1111");
         return nonSmartPAType;
     } else {
+        ALOGE("fuhua---check--getNonSmartPAType nonSmartPAType = %d companyname_custom_spktype = %d",nonSmartPAType,companyname_cus
+        return companyname_custom_spktype;
+/*
 #if defined(USING_EXTAMP_HP)
+        ALOGE("fuhua---check --- getNonSmartPAType 2222");
         return AUDIO_SPK_EXTAMP_HP;
 #elif defined(USING_EXTAMP_LO)
+        ALOGE("fuhua---check --- getNonSmartPAType 3333");
         return AUDIO_SPK_EXTAMP_LO;
 #else
+        ALOGE("fuhua---check --- getNonSmartPAType 4444");
         return AUDIO_SPK_INTAMP;
 #endif
+*/
     }
+    ALOGE("fuhua---check--getNonSmartPAType--end");
 }


status_t AudioALSAHardwareResourceManager::OpenSpeakerPath(const uint32_t Sample
         mSmartPaController->speakerOn(SampleRate, mOutputDevices);
         int spkType = getNonSmartPAType();
+        ALOGE("fuhua---AudioALSAHardwareResourceManager::OpenSpeakerPath--in else spkType = %d",spkType);
         switch (spkType) {
         case AUDIO_SPK_INTAMP:

```

## 思路  
1. 将宏定义配置改为从audio参数读取配置，可做开机动态调整，前提是做好参数定制。  

2. 考虑到每次`OpenSpeakerPath`等函数都会调用`getNonSmartPAType();`，为了缩短时间提升效率，开机就写牢spk的硬件配置，往后不再通过读取配置文件再来获取spk的通路配置。  

3. 尽量用mtk已有的框架，所以找到setNonSmartPAType()进行修改即可。  


## 验证  
通过morgan4g的基线和机器已经验证能生效，为了规范和兼容其他项目，`audio_device.xml`和`common/V3/include/AudioALSADeviceConfigManager.h`也改了一下。  