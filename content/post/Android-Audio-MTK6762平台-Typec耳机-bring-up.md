+++
author = "fuhua"
title = "Android - Audio - MTK6762平台 - Typec耳机 bring up"
date = "2020-09-18 16:45:16"
description = "手机技术笔记"
tags = [
    "markdown",
    "text",
]
weight = 10
+++



- [背景](#背景)  
- [耳机的分类和识别](#耳机的分类)  
- [TypeC耳机的识别细节](#调试Qcom耳机功能时常用修改)  
	- [(1)CC pin模式判断是否是耳机模式](#1-qcom-msm-mbhc-hphl-swh-lt-1-gt)  
	- [(2)耳机识别中断](#2-qcom-msm-hs-micbias-type-”internal”)  
	- [(3)识别逻辑通了，剩下具体实现了，dts修改](#3-耳机的micbias-常用micbias2-是否有外部电容)  
	- [(4)dts操作做](#4-耳机是否支持欧美标转换)  
- [综述](#综述)  





## 背景
>与MTK的工程师交流，他们说没有TypeC耳机的feature？为什么我们就要这么做呢？既然没有方案可以参考，岂不是又要靠自己？





## TypeC耳机识别细节
#####(1)CC pin模式判断是否是耳机模式
从原理图和硬件同事的设计来看，CC pin = 0的时候会触发耳机识别，所以我们需要在usb驱动部分添加log先把识别的第一步给确认。
[mtk/alps/kernel-4.9.git] / drivers / power / supply / mediatek / charger / mtk_chg_type_det.c
```
@@ -360,6 +360,12 @@ static void plug_in_out_handler(struct chg_type_info *cti, bool en, bool ignore)
        mutex_unlock(&cti->chgdet_lock);
 }
 
+/*begin mod by fuhua task id: 9528749 on 2020-6-29*/
+#ifdef CONFIG_TCT_DAVINCI
+extern irqreturn_t ex_eint_handler(int irq, void *data);
+#endif
+
+
 static int pd_tcp_notifier_call(struct notifier_block *pnb,
                                unsigned long event, void *data)
 {
@@ -367,6 +373,9 @@ static int pd_tcp_notifier_call(struct notifier_block *pnb,
        struct chg_type_info *cti = container_of(pnb,
                struct chg_type_info, pd_nb);
        int vbus = 0;
+    #ifdef CONFIG_TCT_DAVINCI
+       int ret_accdet = 0;
+    #endif
 
        switch (event) {
        case TCP_NOTIFY_TYPEC_STATE:
@@ -400,6 +409,20 @@ static int pd_tcp_notifier_call(struct notifier_block *pnb,
                        noti->typec_state.new_state == TYPEC_ATTACHED_SRC) {
                        pr_info("%s Sink_to_Source\n", __func__);
                        plug_in_out_handler(cti, false, true);
+        #ifdef CONFIG_TCT_DAVINCI
+               } else if(noti->typec_state.new_state == TYPEC_ATTACHED_AUDIO ||
+               noti->typec_state.old_state == TYPEC_ATTACHED_AUDIO){// change
+                       ret_accdet = ex_eint_handler((int)NULL,(void *)NULL);
+                       if(ret_accdet)
+                       {
+                               printk("fuhua-check accdet if ret_accdet = %d",ret_accdet);
+                       }
+                       else
+                       {
+                               printk("fuhua-check accdet else ret_accdet = %d",ret_accdet);
+                       }
+        #endif
+/*end   mod by fuhua task id: 9528749 on 2020-6-29*/
                }
                break;
        }
```
其中，中断处理函数' EXPORT_SYMBOL(ex_eint_handler);'是已经在accdet.c中extern出来了的。
所以直接用就好（其实在采用充电的这个接口之前，还使用过两个方式，一个是在充电的callback里直接加，一个是注册充电callback；但这两个修改的代码太多，而且还有延时）

#####(2)耳机识别中断
这里就需要在accdet.c里面改了：
[mtk/alps/kernel-4.9.git] / drivers / misc / mediatek / accdet / mt6357 / accdet.c
改得还蛮多

```
@@ -113,7 +113,14 @@ static struct pwm_deb_settings *cust_pwm_deb;
 #ifdef CONFIG_ACCDET_EINT
 static struct pinctrl *accdet_pinctrl;
 static struct pinctrl_state *pins_eint;
+/* begin mod by fuhua task id: 9528749 on 2020-6-29 */
+#ifdef CONFIG_TCT_DAVINCI
+//static u32 gpiopin, gpio_headset_deb;
+static u32 gpio_headset_deb;
+#else
 static u32 gpiopin, gpio_headset_deb;
+#endif
+/* end   mod by fuhua task id: 9528749 on 2020-6-29 */
 static u32 accdet_irq;
 #endif
 
@@ -135,7 +142,13 @@ static struct wakeup_source *accdet_timer_lock;
 static DEFINE_MUTEX(accdet_eint_irq_sync_mutex);
 static int s_button_status;
 
+/* begin mod by fuhua task id: 9528749 on 2020-6-29 */
+#ifdef CONFIG_TCT_DAVINCI
+//static u32 accdet_eint_type = IRQ_TYPE_LEVEL_LOW;
+#else
 static u32 accdet_eint_type = IRQ_TYPE_LEVEL_LOW;
+#endif
+/* end   mod by fuhua task id: 9528749 on 2020-6-29 */
 static u32 button_press_debounce = 0x400;
 static atomic_t accdet_first;
 #ifdef HW_MODE_SUPPORT
@@ -858,7 +871,8 @@ static inline void clear_accdet_eint_check(u32 eintid)
                AUD_TOP_INT_STATUS0, pmic_read(AUD_TOP_INT_STATUS0));
 
 }
-
+/*begin mod by fuhua task id: 9528749 on 2020-6-29*/
+#if defined (CONFIG_ACCDET_SUPPORT_EINT0) || defined (CONFIG_ACCDET_SUPPORT_EINT1) || defined (CONFIG_ACCDET_SUPPORT_BI_EINT)
 static void eint_debounce_set(u32 eint_id, u32 debounce)
 {
        u32 reg_val;
@@ -880,11 +894,16 @@ static void eint_debounce_set(u32 eint_id, u32 debounce)
                        reg_val | debounce);
        }
 }
+#endif
 
 static u32 get_triggered_eint(void)
 {
        u32 eint_ID = NO_PMIC_EINT;
+
+#if defined (CONFIG_ACCDET_SUPPORT_EINT0) || defined (CONFIG_ACCDET_SUPPORT_EINT1) || defined (CONFIG_ACCDET_SUPPORT_BI_EINT)
        u32 irq_status = pmic_read(ACCDET_IRQ_STS);
+#endif
+/*end   mod by fuhua task id: 9528749 on 2020-6-29*/
 
 #ifdef CONFIG_ACCDET_SUPPORT_EINT0
        if ((irq_status & ACCDET_EINT0_IRQ_B2) == ACCDET_EINT0_IRQ_B2)
@@ -1094,6 +1113,61 @@ static void accdet_set_debounce(int state, unsigned int debounce)
        }
 }
 
+/* begin mod by fuhua task id: 9528749 on 2020-6-29 */
+#ifdef CONFIG_TCT_DAVINCI
+static int change_smg3798_hpmic_sel()
+{
+       int retval = 0;
+       int sgm3798_pin_id = -1;
+       int sgm3798_pin_state = -1;
+       struct device_node *node = NULL;
+
+       node = of_find_matching_node(node, accdet_of_match);
+       sgm3798_pin_id = of_get_named_gpio(node, "sgm3798_hpmic_sel_state", 0);
+       sgm3798_pin_state = gpio_get_value(sgm3798_pin_id);
+
+       printk("sgm3798_pin_state = %d",sgm3798_pin_state);
+
+       //change to low
+       if(sgm3798_pin_state == 1)
+       {
+               if (typec_sel_gpios[GPIO_SGM3798_HPMIC_SEL_LOW].gpio_prepare) {
+                       retval = pinctrl_select_state(
+                               accdet_pinctrl,
+                               typec_sel_gpios[GPIO_SGM3798_HPMIC_SEL_LOW].gpioctrl);
+                       if (retval)
+                               pr_err("fuhua--could not set typec_sel_gpios[GPIO_SGM3798_HPMIC_SEL_LOW] pins\n");
+               }
+               else{
+                       pr_err("fuhua--gpio no prepare\r\n");
+               }
+       }
+
+       //change to high
+       if(sgm3798_pin_state == 0)
+       {
+               if (typec_sel_gpios[GPIO_SGM3798_HPMIC_SEL_HIGH].gpio_prepare) {
+                       retval = pinctrl_select_state(
+                               accdet_pinctrl,
+                               typec_sel_gpios[GPIO_SGM3798_HPMIC_SEL_HIGH].gpioctrl);
+                       if (retval)
+                               pr_err("fuhua--could not set typec_sel_gpios[GPIO_SGM3798_HPMIC_SEL_HIGH] pins\n");
+               }
+               else{
+                       pr_err("fuhua--gpio no prepare\r\n");
+               }
+       }
+
+
+       sgm3798_pin_state = gpio_get_value(sgm3798_pin_id);
+       printk("fuhua-check-sgm3798_pin_state = %d",sgm3798_pin_state);
+
+
+       return retval;
+}
+#endif
+
+
 static inline void check_cable_type(void)
 {
        u32 cur_AB;
@@ -1102,6 +1176,25 @@ static inline void check_cable_type(void)
        cur_AB = cur_AB & ACCDET_STATE_AB_MASK;
        pr_notice("accdet %s(), cur_status:%s current AB = %d\n", __func__,
                     accdet_status_str[accdet_status], cur_AB);
+       #ifdef CONFIG_TCT_DAVINCI
+       // u32 cur_AB;
+       // cur_AB = pmic_read(ACCDET_STATE_RG) >> ACCDET_STATE_MEM_IN_OFFSET;
+       // cur_AB = cur_AB & ACCDET_STATE_AB_MASK;
+       //printk("fuhua-check cur_AB = %d\r\n",cur_AB);
+       //change_smg3798_hpmic_sel();
+       pr_err("fuhua-check 000 accdet %s(), cur_status:%s current AB = %d\n", __func__,
+                   accdet_status_str[accdet_status], cur_AB);
+       if(cur_AB == ACCDET_STATE_AB_00 && accdet_status == PLUG_OUT)
+       {
+               change_smg3798_hpmic_sel();
+               mdelay(200);
+               cur_AB = pmic_read(ACCDET_STATE_RG) >> ACCDET_STATE_MEM_IN_OFFSET;
+               cur_AB = cur_AB & ACCDET_STATE_AB_MASK;
+       }
+       pr_err("fuhua-check 111 accdet %s(), cur_status:%s current AB = %d\n", __func__,
+                   accdet_status_str[accdet_status], cur_AB);
+       #endif
+/* end   mod by fuhua task id: 9528749 on 2020-6-29 */
 
        s_button_status = 0;
        pre_status = accdet_status;
@@ -1485,8 +1578,10 @@ static void accdet_int_handler(void)
        pr_debug("%s()\n", __func__);
        accdet_irq_handle();
 }
-
-#ifdef CONFIG_ACCDET_EINT_IRQ
+/*begin mod by fuhua task id: 9528749 on 2020-6-29*/
+#if (defined (CONFIG_ACCDET_EINT_IRQ) && defined (CONFIG_ACCDET_SUPPORT_EINT0)) ||\
+       (defined (CONFIG_ACCDET_EINT_IRQ) && defined (CONFIG_ACCDET_SUPPORT_EINT1)) ||\
+       (defined (CONFIG_ACCDET_EINT_IRQ) && defined (CONFIG_ACCDET_BI_EINT))
 static void accdet_eint_handler(void)
 {
        pr_info("%s() enter\n", __func__);
@@ -1496,48 +1591,109 @@ static void accdet_eint_handler(void)
 #endif
 
 #ifdef CONFIG_ACCDET_EINT
-/*begin mod by fuhua task id: 9528749 on 2020-6-29*/
 #ifdef CONFIG_TCT_DAVINCI
+//extern int ex_eint_handler(int irq, void *data);
 //static irqreturn_t ex_eint_handler(int irq, void *data)
-static int ex_eint_handler(int irq, void *data)
+static int flag_probe_ok = 0;
+irqreturn_t ex_eint_handler(int irq, void *data)
+//int ex_eint_handler(void)
 {
        int ret = 0;
-
+       int retval = 0;
+       //mutex_lock(&accdet_eint_irq_sync_mutex);
+       printk("fuhua  ex_eint_handler begin!!!11111\r\n");
        if (cur_eint_state == EINT_PIN_PLUG_IN) {
-               /* To trigger EINT when the headset was plugged in
-                * We set the polarity back as we initialed.
-                */
-               // if (accdet_eint_type == IRQ_TYPE_LEVEL_HIGH)
-               //      irq_set_irq_type(accdet_irq, IRQ_TYPE_LEVEL_HIGH);
-               // else
-               //      irq_set_irq_type(accdet_irq, IRQ_TYPE_LEVEL_LOW);
-               // gpio_set_debounce(gpiopin, gpio_headset_deb);
-
+               printk("fuhua  ex_eint_handler in if!!!\r\n");
+               if (typec_sel_gpios[GPIO_WAS47681_ALP_SEL_LOW].gpio_prepare) {
+                       retval = pinctrl_select_state(
+                               accdet_pinctrl,
+                               typec_sel_gpios[GPIO_WAS47681_ALP_SEL_LOW].gpioctrl);
+                       if (retval)
+                               pr_err("fuhua--could not set typec_sel_gpios[GPIO_WAS47681_ALP_SEL_LOW] pins\n");
+               }
+               else{
+                       pr_err("fuhua--gpio no prepare\r\n");
+               }
                cur_eint_state = EINT_PIN_PLUG_OUT;
        } else {
-               /* To trigger EINT when the headset was plugged out
-                * We set the opposite polarity to what we initialed.
-                */
-               // if (accdet_eint_type == IRQ_TYPE_LEVEL_HIGH)
-               //      irq_set_irq_type(accdet_irq, IRQ_TYPE_LEVEL_LOW);
-               // else
-               //      irq_set_irq_type(accdet_irq, IRQ_TYPE_LEVEL_HIGH);
-
-               // gpio_set_debounce(gpiopin, accdet_dts.plugout_deb * 1000);
-
+               printk("fuhua  ex_eint_handler in else!!!\r\n");
+               if (typec_sel_gpios[GPIO_WAS47681_ALP_SEL_HIGH].gpio_prepare) {
+                       retval = pinctrl_select_state(
+                               accdet_pinctrl,
+                               typec_sel_gpios[GPIO_WAS47681_ALP_SEL_HIGH].gpioctrl);
+                       if (retval)
+                               printk("fuhua--could not set typec_sel_gpios[GPIO_WAS47681_ALP_SEL_HIGH] pins\n");
+               }
+               else{
+                       printk("fuhua--gpio no prepare\r\n");
+               }
                cur_eint_state = EINT_PIN_PLUG_IN;
 
                mod_timer(&micbias_timer, jiffies + MICBIAS_DISABLE_TIMER);
        }
-
+    //mutex_unlock(&accdet_eint_irq_sync_mutex);
        // disable_irq_nosync(accdet_irq);
        pr_info("accdet %s(), cur_eint_state=%d\n", __func__, cur_eint_state);
-       ret = queue_work(eint_workqueue, &eint_work);
+       if(flag_probe_ok == 1)
+               ret = queue_work(eint_workqueue, &eint_work);
+       else
+               ret = -1;
        //return IRQ_HANDLED;
+       pr_debug("fuhua  ex_eint_handler end!!! ret = %d\r\n",ret);
        return ret;
 }
 EXPORT_SYMBOL(ex_eint_handler);
 
+
+// static int ex_eint_handler_callback(struct notifier_block *pnb,
+//                             unsigned long event, void *data)
+// {
+//     int ret = 0;
+//     int retval = 0;
+//     //mutex_lock(&accdet_eint_irq_sync_mutex);
+//     printk("fuhua  ex_eint_handler begin!!!11111\r\n");
+//     if (cur_eint_state == EINT_PIN_PLUG_IN) {
+//             printk("fuhua  ex_eint_handler in if!!!\r\n");
+//             if (typec_sel_gpios[GPIO_WAS47681_ALP_SEL_LOW].gpio_prepare) {
+//                     retval = pinctrl_select_state(
+//                             accdet_pinctrl,
+//                             typec_sel_gpios[GPIO_WAS47681_ALP_SEL_LOW].gpioctrl);
+//                     if (retval)
+//                             pr_err("fuhua--could not set typec_sel_gpios[GPIO_WAS47681_ALP_SEL_LOW] pins\n");
+//             }
+//             else{
+//                     pr_err("fuhua--gpio no prepare\r\n");
+//             }
+//             cur_eint_state = EINT_PIN_PLUG_OUT;
+//     } else {
+//             printk("fuhua  ex_eint_handler in else!!!\r\n");
+//             if (typec_sel_gpios[GPIO_WAS47681_ALP_SEL_HIGH].gpio_prepare) {
+//                     retval = pinctrl_select_state(
+//                             accdet_pinctrl,
+//                             typec_sel_gpios[GPIO_WAS47681_ALP_SEL_HIGH].gpioctrl);
+//                     if (retval)
+//                             pr_err("fuhua--could not set typec_sel_gpios[GPIO_WAS47681_ALP_SEL_HIGH] pins\n");
+//             }
+//             else{
+//                     pr_err("fuhua--gpio no prepare\r\n");
+//             }
+//             cur_eint_state = EINT_PIN_PLUG_IN;
+
+//             mod_timer(&micbias_timer, jiffies + MICBIAS_DISABLE_TIMER);
+//     }
+//     //mutex_unlock(&accdet_eint_irq_sync_mutex);
+//     // disable_irq_nosync(accdet_irq);
+//     pr_info("accdet %s(), cur_eint_state=%d\n", __func__, cur_eint_state);
+//     if(flag_probe_ok == 1)
+//             ret = queue_work(eint_workqueue, &eint_work);
+//     else
+//             ret = -1;
+//     //return IRQ_HANDLED;
+//     printk("fuhua  ex_eint_handler end!!! ret = %d\r\n",ret);
+//     return ret;
+// }
+
+
 #else
 static irqreturn_t ex_eint_handler(int irq, void *data)
 {
@@ -1576,7 +1732,11 @@ static irqreturn_t ex_eint_handler(int irq, void *data)
        return IRQ_HANDLED;
 }
 #endif
-/*end   mod by fuhua task id: 9528749 on 2020-6-29*/
+// #ifdef CONFIG_TCPC_CLASS
+// #ifdef CONFIG_TCT_DAVINCI
+// extern int register_tcp_dev_notifier(struct tcpc_device *tcp_dev, struct notifier_block *nb, uint8_t flags);
+// #endif
+// #endif
 
 static inline int ext_eint_setup(struct platform_device *platform_device)
 {
@@ -1584,7 +1744,13 @@ static inline int ext_eint_setup(struct platform_device *platform_device)
        u32 ints[4] = { 0 };
        struct device_node *node = NULL;
        struct pinctrl_state *pins_default = NULL;
-
+#ifdef CONFIG_TCT_DAVINCI
+       int i = 0;
+// #ifdef CONFIG_TCPC_CLASS
+//     int ret_typec_accdet = 0;
+//     struct typec_accdet_cb *cti = NULL;
+// #endif
+#endif
        pr_info("accdet %s()\n", __func__);
        accdet_pinctrl = devm_pinctrl_get(&platform_device->dev);
        if (IS_ERR(accdet_pinctrl)) {
@@ -1612,7 +1778,31 @@ static inline int ext_eint_setup(struct platform_device *platform_device)
                pr_notice("accdet %s can't find compatible node\n", __func__);
                return -1;
        }
-
+#ifdef CONFIG_TCT_DAVINCI
+       // gpiopin = of_get_named_gpio(node, "deb-gpios", 0);
+       // ret = of_property_read_u32(node, "debounce", &gpio_headset_deb);
+       // if (ret < 0) {
+       //      pr_notice("accdet %s gpiodebounce not found,ret:%d\n",
+       //              __func__, ret);
+       //      return ret;
+       // }
+       // gpio_set_debounce(gpiopin, gpio_headset_deb);
+       pr_err("fuhua--ext_eint_setup-gpio-begin  \r\n");
+       for (i = 0; i < ARRAY_SIZE(typec_sel_gpios); i++) {
+               typec_sel_gpios[i].gpioctrl =
+                       pinctrl_lookup_state(accdet_pinctrl, typec_sel_gpios[i].name);
+               if (IS_ERR(typec_sel_gpios[i].gpioctrl)) {
+                       ret = PTR_ERR(typec_sel_gpios[i].gpioctrl);
+                       pr_err("%s fuhua ---check pinctrl_lookup_state %s fail ret=%d,i=%d\n", __func__,
+                              typec_sel_gpios[i].name, ret,i);
+               } else {
+                       typec_sel_gpios[i].gpio_prepare = true;
+                       pr_err("%s fuhua ---check pinctrl_lookup_state %s success ret=%d,i=%d\n", __func__,
+                              typec_sel_gpios[i].name, ret,i);
+               }
+       }
+       pr_err("fuhua--ext_eint_setup-gpio-end   \r\n");
+#else
        gpiopin = of_get_named_gpio(node, "deb-gpios", 0);
        ret = of_property_read_u32(node, "debounce", &gpio_headset_deb);
        if (ret < 0) {
@@ -1621,10 +1811,53 @@ static inline int ext_eint_setup(struct platform_device *platform_device)
                return ret;
        }
        gpio_set_debounce(gpiopin, gpio_headset_deb);
+#endif
 
        accdet_irq = irq_of_parse_and_map(node, 0);
        ret = of_property_read_u32_array(node, "interrupts", ints,
                        ARRAY_SIZE(ints));
+#ifdef CONFIG_TCT_DAVINCI                      
+       // if (ret) {
+       //      pr_notice("accdet %s interrupts not found,ret:%d\n",
+       //              __func__, ret);
+       //      return ret;
+       // }
+       // accdet_eint_type = ints[1];
+       // pr_info("accdet set gpio EINT, gpiopin=%d, accdet_eint_type=%d\n",
+       //              gpiopin, accdet_eint_type);
+       // ret = request_irq(accdet_irq, ex_eint_handler, IRQF_TRIGGER_NONE,
+       //      "accdet-eint", NULL);
+       // if (ret) {
+       //      pr_notice("accdet %s request_irq fail, ret:%d.\n", __func__,
+       //              ret);
+       //      return ret;
+       // }
+
+// #ifdef CONFIG_TCPC_CLASS
+//     cti = devm_kzalloc(&platform_device->dev, sizeof(*cti), GFP_KERNEL);
+//     if (!cti) {
+//             ret_typec_accdet = -ENOMEM;
+//             pr_err("fuhua-- typec_accdet setup error 111 !!!\r\n");
+//     }
+//     //cti->dev = &pdev->dev;
+//     cti->dev = &platform_device->dev;
+//     cti->tcpc_dev = tcpc_dev_get_by_name("type_c_port0");
+//     if (cti->tcpc_dev == NULL) {
+//             pr_info("%s: tcpc device not ready, defer\n", __func__);
+//             ret_typec_accdet = -EPROBE_DEFER;
+//             pr_err("fuhua-- typec_accdet setup error 222 !!!\r\n");
+//     }
+//     cti->pd_nb.notifier_call = ex_eint_handler_callback;
+//     ret_typec_accdet = register_tcp_dev_notifier(cti->tcpc_dev,
+//             &cti->pd_nb, TCP_NOTIFY_TYPE_ALL);
+//     if (ret_typec_accdet < 0) {
+//             pr_info("%s: register tcpc notifer fail\n", __func__);
+//             ret_typec_accdet = -EINVAL;
+//             pr_err("fuhua-- typec_accdet setup error 333 !!!\r\n");
+//     }
+
+// #endif
+#else
        if (ret) {
                pr_notice("accdet %s interrupts not found,ret:%d\n",
                        __func__, ret);
@@ -1640,6 +1873,8 @@ static inline int ext_eint_setup(struct platform_device *platform_device)
                        ret);
                return ret;
        }
+#endif
+/*end   mod by fuhua task id: 9528749 on 2020-6-29*/
 
        pr_info("accdet set gpio EINT finished, irq=%d, gpio_headset_deb=%d\n",
                        accdet_irq, gpio_headset_deb);
@@ -1933,6 +2168,12 @@ void accdet_late_init(unsigned long data)
                accdet_init_once();
        } else
                pr_info("%s err:inited already done/get dts fail\n", __func__);
+       
+       /*begin mod by fuhua task id: 9528749 on 2020-6-29*/
+       #ifdef CONFIG_TCT_DAVINCI
+       flag_probe_ok = 1;
+       #endif
+       /*end   mod by fuhua task id: 9528749 on 2020-6-29*/
 }
 EXPORT_SYMBOL(accdet_late_init);
 
@@ -2055,20 +2296,20 @@ int mt_accdet_probe(struct platform_device *dev)
 
        /* register pmic interrupt */
        pmic_register_interrupt_callback(INT_ACCDET, accdet_int_handler);
-#ifdef CONFIG_ACCDET_EINT_IRQ
-#ifdef CONFIG_ACCDET_SUPPORT_EINT0
+/*begin mod by fuhua task id: 9528749 on 2020-6-29*/
+#if defined (CONFIG_ACCDET_SUPPORT_EINT0) && defined (CONFIG_ACCDET_EINT_IRQ)
        pmic_register_interrupt_callback(INT_ACCDET_EINT0,
                        accdet_eint_handler);
-#elif defined CONFIG_ACCDET_SUPPORT_EINT1
+#elif defined (CONFIG_ACCDET_SUPPORT_EINT1) && defined (CONFIG_ACCDET_EINT_IRQ)
        pmic_register_interrupt_callback(INT_ACCDET_EINT1,
                        accdet_eint_handler);
-#elif defined CONFIG_ACCDET_BI_EINT
+#elif defined (CONFIG_ACCDET_BI_EINT) && defined (CONFIG_ACCDET_EINT_IRQ)
        pmic_register_interrupt_callback(INT_ACCDET_EINT0,
                        accdet_eint_handler);
        pmic_register_interrupt_callback(INT_ACCDET_EINT1,
                        accdet_eint_handler);
 #endif
-#endif
+/*end   mod by fuhua task id: 9528749 on 2020-6-29*/
 
        ret = accdet_get_dts_data();
        if (ret) {
```
核心逻辑就是检测到中断了再判断设备状态，再进行处理(check_cable_type)

[mtk/alps/kernel-4.9.git] / drivers / misc / mediatek / accdet / mt6357 / accdet.h
为了可维护性以及规范，还是在h里面做好明确的定义
```
@@ -113,7 +113,13 @@ struct pwm_deb_settings {
        /* auxadc debounce */
        unsigned int debounce4;
 };
-
+/*begin mod by fuhua task id: 9528749 on 2020-6-29*/
+struct typec_sel_gpio_attr {
+       const char *name;
+       bool gpio_prepare;
+       struct pinctrl_state *gpioctrl;
+};
+       
 struct head_dts_data {
        /* set mic bias voltage set: 0x02,1.9V;0x07,2.7V */
        unsigned int mic_vol;
@@ -130,7 +136,34 @@ struct head_dts_data {
        struct pwm_deb_settings pwm_deb;
        struct three_key_threshold three_key;
        struct four_key_threshold four_key;
+
+       //struct typec_sel_gpio_attr typec_sel_gpios;
+
+};
+//sgm3798_hpmic_sel_low
+#ifdef CONFIG_TCT_DAVINCI
+enum typec_sel_gpio_type {
+       GPIO_WAS47681_ALP_SEL_HIGH,
+       GPIO_WAS47681_ALP_SEL_LOW,
+       GPIO_SGM3798_HPMIC_SEL_HIGH,
+       GPIO_SGM3798_HPMIC_SEL_LOW,
+       GPIO_NUM
 };
+static struct typec_sel_gpio_attr typec_sel_gpios[GPIO_NUM] = {
+               [GPIO_WAS47681_ALP_SEL_HIGH] = {"was4768q_alp_sel_high", false, NULL},
+               [GPIO_WAS47681_ALP_SEL_LOW]  = {"was4768q_alp_sel_low", false, NULL},
+               [GPIO_SGM3798_HPMIC_SEL_HIGH] = {"sgm3798_hpmic_sel_high", false, NULL},
+               [GPIO_SGM3798_HPMIC_SEL_LOW]  = {"sgm3798_hpmic_sel_low", false, NULL},
+};
+
+struct typec_accdet_cb {
+       struct device *dev;
+       struct tcpc_device *tcpc_dev;
+       struct notifier_block pd_nb;
+};
+#endif
+
+/*end   mod by fuhua task id: 9528749 on 2020-6-29*/
 
 enum {
        accdet_state000 = 0,
```

#####(3)识别逻辑通了，剩下具体实现了，dts修改
[mtk/alps/kernel-4.9.git] / drivers / misc / mediatek / dws / mt6765 / da_vinci.dws
```
@@ -354,10 +354,10 @@
             <gpio28>
                 <eint_mode>false</eint_mode>
                 <def_mode>0</def_mode>
-                <inpull_en>true</inpull_en>
+                <inpull_en>false</inpull_en>
                 <inpull_selhigh>false</inpull_selhigh>
                 <def_dir>OUT</def_dir>
-                <out_high>false</out_high>
+                <out_high>true</out_high>
                 <smt>false</smt>
                 <ies>true</ies>
             </gpio28>
@@ -1531,11 +1531,10 @@
             <gpio171>
                 <eint_mode>false</eint_mode>
                 <def_mode>0</def_mode>
-                <inpull_en>true</inpull_en>
+                <inpull_en>false</inpull_en>
                 <inpull_selhigh>false</inpull_selhigh>
                 <def_dir>OUT</def_dir>
                 <out_high>false</out_high>
-                <varName0>GPIO_LCD_BIAS_ENN_PIN</varName0>
                 <smt>false</smt>
                 <ies>true</ies>
             </gpio171>
```

[mtk/alps/kernel-4.9.git] / arch / arm64 / boot / dts / mediatek / da_vinci.dts

```
  /*Begin Modified by fuhua.wang for audio-bring-up on 2019.07.31 task_id: 8188551 */ 
  /* accdet start */
  &accdet {
          accdet-mic-vol = <6>;
          headset-mode-setting = <0x500 0x500 1 0x1f0 0x800 0x800 0x20 0x44>;
          accdet-plugout-debounce = <1>;
          accdet-mic-mode = <1>;
          headset-eint-level-pol = <8>;
@@ -298,16 +298,58 @@
        headset-three-key-threshold = <0 80 220 400>;
        headset-three-key-threshold-CDD = <0 121 192 600>;
        headset-four-key-threshold = <0 58 121 192 400>;
-       pinctrl-names = "default", "state_eint_as_int";
+/*Begin Modified by fuhua.wang for audio-bring-up on 20200.07.07 task_id: 9528749 */
+       pinctrl-names = "default", "state_eint_as_int","was4768q_alp_sel_high","was4768q_alp_sel_low","sgm3798_hpmic_sel_high","sgm3798_hpmic_sel_low";
        pinctrl-0 = <&accdet_pins_default>;
        pinctrl-1 = <&accdet_pins_eint_as_int>;
+       pinctrl-2 = <&was4768q_alp_sel_high>;
+       pinctrl-3 = <&was4768q_alp_sel_low>;
+       pinctrl-4 = <&sgm3798_hpmic_sel_high>;
+       pinctrl-5 = <&sgm3798_hpmic_sel_low>;
+
+       was4768q_sel_state = <&pio 171 0x0>;
+       sgm3798_hpmic_sel_state = <&pio 28 0x0>;
        status = "okay";
 };
 &pio {
        accdet_pins_default: accdetdefault {
        };
        accdet_pins_eint_as_int: accdeteint@0 {
+               pins_cmd_dat {
+                       pinmux = <PINMUX_GPIO9__FUNC_GPIO9>;
+                       slew-rate = <0>;
+                       bias-disable;
+               };
        };
+       was4768q_alp_sel_high: was4768q_alp_sel_high {
+        pins_cmd_dat {
+            pinmux = <PINMUX_GPIO171__FUNC_GPIO171>;
+            slew-rate = <1>;
+            output-high;
+        };
+    };
+    was4768q_alp_sel_low: was4768q_alp_sel_low {
+        pins_cmd_dat {
+            pinmux = <PINMUX_GPIO171__FUNC_GPIO171>;
+            slew-rate = <1>;
+            output-low;
+        };
+    };
+       sgm3798_hpmic_sel_high: sgm3798_hpmic_sel_high {
+        pins_cmd_dat {
+            pinmux = <PINMUX_GPIO28__FUNC_GPIO28>;
+            slew-rate = <1>;
+            output-high;
+        };
+    };
+    sgm3798_hpmic_sel_low: sgm3798_hpmic_sel_low {
+        pins_cmd_dat {
+            pinmux = <PINMUX_GPIO28__FUNC_GPIO28>;
+            slew-rate = <1>;
+            output-low;
+        };
+/*End   Modified by fuhua.wang for audio-bring-up on 20200.07.07 task_id: 9528749 */
+    };
 };
 /* accdet end */
 /*End   Modified by fuhua.wang for audio-bring-up on 2019.07.31 task_id: 8188551 */ 
```

#####(4)dts操作做
在这之前有个宏控制
[mtk/alps/kernel-4.9.git] / arch / arm64 / configs / da_vinci_debug_defconfig
[mtk/alps/kernel-4.9.git] / arch / arm64 / configs / da_vinci_defconfig
```
@@ -217,9 +217,12 @@ CONFIG_MTK_ECCCI_DRIVER=y
 CONFIG_MTK_ECCCI_C2K=y
 CONFIG_MTK_BTIF=y
 CONFIG_MTK_SECURITY_SW_SUPPORT=y
+#begin mod by fuhua task id: 9528749 on 2020-6-29
 CONFIG_MTK_ACCDET=y
-CONFIG_ACCDET_EINT_IRQ=y
-CONFIG_ACCDET_SUPPORT_EINT0=y
+CONFIG_ACCDET_EINT=y
+#CONFIG_ACCDET_EINT_IRQ=y
+#CONFIG_ACCDET_SUPPORT_EINT0=y
+#end   mod by fuhua task id: 9528749 on 2020-6-29
 CONFIG_MTK_DRAM_LOG_STORE=y
 CONFIG_MTK_DRAM_LOG_STORE_ADDR=0x0010E700
 CONFIG_MTK_DRAM_LOG_STORE_SIZE=0x100
```
宏控制做好之后用wifi 进行adb调试。
然后就可以看到log了，然后再针对调试；
接下来详细说一下dts的操作，主要是针对上述accdet.c里面的代码。