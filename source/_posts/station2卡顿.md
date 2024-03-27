---
title: station2卡顿
tags:
- problem
---
# 1.问题背景

station2设备滑动屏幕感到明显卡顿



# 2.分析过程

## 2.1 trace信息

adb登录到板端，进入sdcard目录，然后运行：

atrace -b 19200 -o ./sdcard/mytrace gfx input view webview wm am sm hal res dalvik rs ss sched freq power camera -t 20
adb pull将mytrace传到本地，
使用systrace.py转换成html格式

./systrace.py --from-file mytrace



![c7cbc29b0aad4d04e5fbc26d1c2612bb](D:\DingTalkAppData\DingTalk\216604157_v2\resource_cache\c7\c7cbc29b0aad4d04e5fbc26d1c2612bb.png)

### 流程梳理

```c
ops->wait_for_commit_done
	sde_encoder_phys_wb_wait_for_commit_done
		_sde_encoder_phys_wb_wait_for_idle
			wait_info.wq = &phys_enc->pending_kickoff_wq; //Wait queue for blocking until kickoff completes
			_sde_encoder_wait_timeout
```



### 正常vs异常日志对比

1.正常日志

![image-20240326111508593](C:\Users\rokid\AppData\Roaming\Typora\typora-user-images\image-20240326111508593.png)

2.异常日志

异常打印：

![img](https://static.dingtalk.com/media/lQLPJwWXZJoQWFnMrs0D6bA0C31KiBlddwXoUCA_X7gA_1001_174.png)

![image-20240320164642418](C:\Users\rokid\AppData\Roaming\Typora\typora-user-images\image-20240320164642418.png)



## 2.2 使能drm日志

![image-20240319215059045](C:\Users\rokid\AppData\Roaming\Typora\typora-user-images\image-20240319215059045.png)

echo 0x1ff > /sys/module/drm/parameters/debug





### 2.2.1 wait_timeout等待时存在\_sde_fence_trigger

![image-20240320165437477](C:\Users\rokid\AppData\Roaming\Typora\typora-user-images\image-20240320165437477.png)



### 2.2.2 异常日志中存在DRM_IOCTL_WAIT_VBLANK

![image-20240320182713784](C:\Users\rokid\AppData\Roaming\Typora\typora-user-images\image-20240320182713784.png)



正常日志：

![image-20240320182728940](C:\Users\rokid\AppData\Roaming\Typora\typora-user-images\image-20240320182728940.png)

```
sde_crtc_get_vblank_counter
drm_update_vblank_count
sde_crtc_vblank_cb
drm_wait_vblank_ioctl
```



### 2.2.3 drm_vblank

drm用vblank来抽象vsync，vsync是display模块产生的，正常情况下开启后会按照一定时间触发中断。

drm driver中会注册vsync的中断服务程序，便于软件进行处理异常，包括vsync。

```c
#la_vendor/kernel_platform/msm-kernel/drivers/gpu/drm/msm/disp/dpu1/dpu_crtc.c
void dpu_crtc_vblank_callback(struct drm_crtc *crtc)
{
        struct dpu_crtc *dpu_crtc = to_dpu_crtc(crtc);

        /* keep statistics on vblank callback - with auto reset via debugfs */
        if (ktime_compare(dpu_crtc->vblank_cb_time, ktime_set(0, 0)) == 0)
                dpu_crtc->vblank_cb_time = ktime_get();
        else
                dpu_crtc->vblank_cb_count++;
        drm_crtc_handle_vblank(crtc);
        trace_dpu_crtc_vblank_cb(DRMID(crtc));
}
```

![img](https://img-blog.csdnimg.cn/direct/89fa7adc5b2d42f9858d234892151bd6.png)

```
dp_display_request_irq
	devm_request_irq(dp_display->drm_dev->dev, dp->irq,
                     dp_display_irq_handler,
                     IRQF_TRIGGER_HIGH, "dp_display_isr", dp);
		 
irq->name = "vsync_irq";
irq->cb.func = dpu_encoder_phys_vid_underrun_irq;
dpu_encoder_phys_vid_vblank_irq


static const struct dpu_encoder_virt_ops dpu_encoder_parent_ops = {
        .handle_vblank_virt = dpu_encoder_vblank_callback,
        .handle_underrun_virt = dpu_encoder_underrun_callback,
        .handle_frame_done = dpu_encoder_frame_done_callback,
};

dpu_encoder_vblank_callback
    dpu_crtc_vblank_callback
		drm_crtc_handle_vblank
		
drm_crtc_handle_vblank
	drm_handle_vblank
		drm_update_vblank_count(dev, pipe, true);
```

看到store_vblank里会把vblank->count+1; 然后会唤醒等待队列。

中断里把vblank count+1；中断是display模块产生的，如果刷新率为60帧，每16.6ms就会来一个中断，这个操作与user space无关。

drm_wait_vblank_ioctl等到vblank count之后就会唤醒进程，并返回给user。


![img](https://img-blog.csdnimg.cn/direct/dfd6d08eba3a428d92f01be7861d8015.png)

```
[  192.136009] [drm:_sde_encoder_wait_timeout:369] [sde error]zyr wait_event_timeout1 info->atomic_cnt:0, info->count_check:0
[  192.136222] [drm:sde_encoder_phys_vid_vblank_irq:536] [sde error]zyr sde_encoder_phys_vid_vblank_irq1 pending_retire_fence_cnt:0
[  192.136229] [drm:sde_encoder_phys_vid_vblank_irq:543] [sde error]zyr sde_encoder_phys_vid_vblank_irq2 pending_retire_fence_cnt:0
1.第一次drm_handle_vblank
drm_update_vblank_count
	__get_vblank_counter
		crtc->funcs->get_vblank_counter
			sde_crtc_get_vblank_counter
[  192.136892] [drm:sde_crtc_get_vblank_counter [msm_drm]] crtc:115 enc:56 is_built_in:1 vblank_cnt:1397
[  192.137526] [drm:sde_crtc_get_vblank_counter [msm_drm]] crtc:115 enc:56 is_built_in:1 vblank_cnt:1397
[  192.137543] msm_drm ae00000.qcom,mdss_mdp: [drm:drm_update_vblank_count] updating vblank count on crtc 0: current=1397, diff=1, hw=1397 hw_last=1396
drm_handle_vblank_events
[  192.137557] msm_drm ae00000.qcom,mdss_mdp: [drm:drm_handle_vblank_events] vblank event on 1398, current 1398
vblank_disable_fn
[  192.137565] msm_drm ae00000.qcom,mdss_mdp: [drm:drm_vblank_put] disabling vblank on crtc 0
vblank_disable_fn
	drm_vblank_disable_and_save
		drm_update_vblank_count
[  192.138190] [drm:sde_crtc_get_vblank_counter [msm_drm]] crtc:115 enc:56 is_built_in:1 vblank_cnt:1397
[  192.138817] [drm:sde_crtc_get_vblank_counter [msm_drm]] crtc:115 enc:56 is_built_in:1 vblank_cnt:1397
[  192.138831] msm_drm ae00000.qcom,mdss_mdp: [drm:drm_update_vblank_count] updating vblank count on crtc 0: current=1398, diff=0, hw=1397 hw_last=1397
		__disable_vblank
			crtc->funcs->disable_vblank
				sde_crtc_disable_vblank
[  192.139459] [drm:sde_crtc_disable_vblank [msm_drm]] dev=ffffff8049651000, crtc=0
					vblank_ctrl_queue_work
						vblank_ctrl_worker
							sde_crtc_vblank
								_sde_crtc_vblank_enable
									sde_encoder_register_vblank_callback
										sde_crtc_vblank_cb
[  192.140176] [drm:sde_crtc_vblank_cb [msm_drm]] crtc115, ts:192119469
[  192.140814] [drm:sde_encoder_register_vblank_callback [msm_drm]] enc56 
									sde_encoder_register_vblank_callback
										sde_encoder_phys_vid_control_vblank_irq
[  192.142159] [drm:sde_encoder_phys_vid_control_vblank_irq [msm_drm]] enc56 intf1 [sde_encoder_register_vblank_callback+0x13c/0x210 [msm_drm]] enable=0/2

2.DRM_IOCTL_WAIT_VBLANK SDM_EventThread调用

[  443.228239] CPU: 2 PID: 1201 Comm: SDM_EventThread Tainted: G        W  O      5.10.198-qki-consolidate-android12-9-g436c7dd2d5cf-dirty #1
[  443.228272] Hardware name: Qualcomm Technologies, Inc. Parrot WCN6750 QRD (DT)
[  443.228299] Call trace:
[  443.229730] dump_backtrace.cfi_jt+0x0/0x8
[  443.229749] show_stack+0x1c/0x2c
[  443.229768] dump_stack_lvl+0xf0/0x164
[  443.229780] dump_stack+0x1c/0x2c
[  443.231180] sde_crtc_enable_vblank+0x4c/0xe8 [msm_drm]
[  443.231220] drm_vblank_enable+0xb8/0x220
[  443.231235] drm_vblank_get+0x80/0x16c
[  443.231250] drm_wait_vblank_ioctl+0x15c/0x738
[  443.232686] drm_ioctl_kernel+0x100/0x1d4
[  443.232699] drm_ioctl+0x228/0x34c
[  443.232718] __arm64_sys_ioctl+0xa8/0x110
[  443.232736] el0_svc_common.llvm.4263132210126967612+0xd8/0x20c
[  443.232748] do_el0_svc+0x28/0x98
[  443.232765] el0_svc+0x24/0x38
[  443.232777] el0_sync_handler+0x88/0xec
[  443.232793] el0_sync+0x1b8/0x1c0

[  192.142406] [drm:drm_ioctl] comm="SDM_EventThread" pid=1222, dev=0xe200, auth=1, DRM_IOCTL_WAIT_VBLANK
DRM_IOCTL_DEF(DRM_IOCTL_WAIT_VBLANK, drm_wait_vblank_ioctl, DRM_UNLOCKED) //la_vendor/kernel_platform/msm-kernel/drivers/gpu/drm/drm_ioctl.c
	drm_wait_vblank_ioctl
		drm_vblank_get
			drm_vblank_enable
				__enable_vblank
					crtc->funcs->enable_vblank
						sde_crtc_enable_vblank
[  192.143057] [drm:sde_crtc_enable_vblank [msm_drm]] dev=ffffff8049651000, crtc=0
[  192.143126] msm_drm ae00000.qcom,mdss_mdp: [drm:drm_vblank_enable] enabling vblank on crtc 0, ret: 0
				drm_update_vblank_count
					sde_crtc_get_vblank_counter
[  192.143764] [drm:sde_crtc_get_vblank_counter [msm_drm]] crtc:115 enc:56 is_built_in:1 vblank_cnt:1397
[  192.143945] [drm:sde_encoder_register_vblank_callback [msm_drm]] enc56 
[  192.144570] [drm:sde_crtc_get_vblank_counter [msm_drm]] crtc:115 enc:56 is_built_in:1 vblank_cnt:1397
[  192.144915] [drm:sde_encoder_phys_vid_control_vblank_irq [msm_drm]] enc56 intf1 [sde_encoder_register_vblank_callback+0x13c/0x210 [msm_drm]] enable=1/1
[  192.144924] msm_drm ae00000.qcom,mdss_mdp: [drm:drm_update_vblank_count] updating vblank count on crtc 0: current=1398, diff=0, hw=1397 hw_last=1397
[  192.145073] msm_drm ae00000.qcom,mdss_mdp: [drm:drm_wait_vblank_ioctl] event on vblank count 1399, current 1398, crtc 0
[  192.148365] [drm:drm_mode_object_get] OBJ ID: 57 (6)
[  192.148383] [drm:drm_atomic_get_connector_state] Added [CONNECTOR:57:DSI-1] ffffff87c4eea000 state to ffffff87f4783600
[  192.148410] [drm:drm_mode_object_get] OBJ ID: 172 (3)
[  192.148423] [drm:drm_atomic_get_crtc_state] Added [CRTC:115:crtc-0] ffffff80495a8000 state to ffffff87f4783600
[  192.149174] [drm:msm_property_atomic_set [msm_drm]] RETIRE_FENCE_OFFSET - 1
[  192.152909] [drm:sde_encoder_phys_vid_vblank_irq:536] [sde error]zyr sde_encoder_phys_vid_vblank_irq1 pending_retire_fence_cnt:0
[  192.152919] [drm:sde_encoder_phys_vid_vblank_irq:543] [sde error]zyr sde_encoder_phys_vid_vblank_irq2 pending_retire_fence_cnt:0
[  192.153611] [drm:sde_crtc_get_vblank_counter [msm_drm]] crtc:115 enc:56 is_built_in:1 vblank_cnt:1398
[  192.154246] [drm:sde_crtc_get_vblank_counter [msm_drm]] crtc:115 enc:56 is_built_in:1 vblank_cnt:1398
[  192.154266] msm_drm ae00000.qcom,mdss_mdp: [drm:drm_update_vblank_count] updating vblank count on crtc 0: current=1398, diff=1, hw=1398 hw_last=1397
[  192.154279] msm_drm ae00000.qcom,mdss_mdp: [drm:drm_handle_vblank_events] vblank event on 1399, current 1399
[  192.154288] msm_drm ae00000.qcom,mdss_mdp: [drm:drm_vblank_put] disabling vblank on crtc 0
[  192.154922] [drm:sde_crtc_get_vblank_counter [msm_drm]] crtc:115 enc:56 is_built_in:1 vblank_cnt:1398
[  192.155554] [drm:sde_crtc_get_vblank_counter [msm_drm]] crtc:115 enc:56 is_built_in:1 vblank_cnt:1398
[  192.155572] msm_drm ae00000.qcom,mdss_mdp: [drm:drm_update_vblank_count] updating vblank count on crtc 0: current=1399, diff=0, hw=1398 hw_last=1398
[  192.156208] [drm:sde_crtc_disable_vblank [msm_drm]] dev=ffffff8049651000, crtc=0
[  192.156924] [drm:sde_crtc_vblank_cb [msm_drm]] crtc115, ts:192136153
[  192.157566] [drm:sde_encoder_register_vblank_callback [msm_drm]] enc56 
[  192.158975] [drm:sde_encoder_phys_vid_control_vblank_irq [msm_drm]] enc56 intf1 [sde_encoder_register_vblank_callback+0x13c/0x210 [msm_drm]] enable=0/2

[  192.159147] [drm:drm_ioctl] comm="SDM_EventThread" pid=1222, dev=0xe200, auth=1, DRM_IOCTL_WAIT_VBLANK
[  192.159794] [drm:sde_crtc_enable_vblank [msm_drm]] dev=ffffff8049651000, crtc=0
[  192.159874] msm_drm ae00000.qcom,mdss_mdp: [drm:drm_vblank_enable] enabling vblank on crtc 0, ret: 0
[  192.160514] [drm:sde_crtc_get_vblank_counter [msm_drm]] crtc:115 enc:56 is_built_in:1 vblank_cnt:1398
[  192.161157] [drm:sde_crtc_get_vblank_counter [msm_drm]] crtc:115 enc:56 is_built_in:1 vblank_cnt:1398
[  192.161342] [drm:sde_encoder_register_vblank_callback [msm_drm]] enc56 
[  192.161353] msm_drm ae00000.qcom,mdss_mdp: [drm:drm_update_vblank_count] updating vblank count on crtc 0: current=1399, diff=0, hw=1398 hw_last=1398
[  192.161747] [drm:sde_encoder_phys_vid_control_vblank_irq [msm_drm]] enc56 intf1 [sde_encoder_register_vblank_callback+0x13c/0x210 [msm_drm]] enable=1/1
[  192.161760] [drm:_sde_encoder_wait_timeout:373] [sde error]zyr wait_event_timeout2 info->atomic_cnt:0, info->count_check:0

```



![image-20240321161154281](C:\Users\rokid\AppData\Roaming\Typora\typora-user-images\image-20240321161154281.png)

![img](https://static.dingtalk.com/media/lQLPJx6RI95fnPXNA3nNBYKwI_0gmcX9SfQF6YlcuEm2AQ_1410_889.png)

![img](https://static.dingtalk.com/media/lQLPKHTo9YWTLPXNAtvNBuqwk1yOyzWEbAEF6YlcuEm2AA_1770_731.png)

#### 1.内核态

> [  443.228239] CPU: 2 PID: 1201 Comm: SDM_EventThread Tainted: G        W  O      5.10.198-qki-consolidate-android12-9-g436c7dd2d5cf-dirty #1
> [  443.228272] Hardware name: Qualcomm Technologies, Inc. Parrot WCN6750 QRD (DT)
> [  443.228299] Call trace:
> [  443.229730] dump_backtrace.cfi_jt+0x0/0x8
> [  443.229749] show_stack+0x1c/0x2c
> [  443.229768] dump_stack_lvl+0xf0/0x164
> [  443.229780] dump_stack+0x1c/0x2c
> [  443.231180] sde_crtc_enable_vblank+0x4c/0xe8 [msm_drm]
> [  443.231220] drm_vblank_enable+0xb8/0x220
> [  443.231235] drm_vblank_get+0x80/0x16c
> [  443.231250] drm_wait_vblank_ioctl+0x15c/0x738
> [  443.232686] drm_ioctl_kernel+0x100/0x1d4
> [  443.232699] drm_ioctl+0x228/0x34c
> [  443.232718] __arm64_sys_ioctl+0xa8/0x110
> [  443.232736] el0_svc_common.llvm.4263132210126967612+0xd8/0x20c
> [  443.232748] do_el0_svc+0x28/0x98
> [  443.232765] el0_svc+0x24/0x38
> [  443.232777] el0_sync_handler+0x88/0xec
> [  443.232793] el0_sync+0x1b8/0x1c0

```
dp_display_request_irq
	devm_request_irq(dp_display->drm_dev->dev, dp->irq,
                     dp_display_irq_handler,
                     IRQF_TRIGGER_HIGH, "dp_display_isr", dp);
		 
irq->name = "vsync_irq";
irq->cb.func = dpu_encoder_phys_vid_underrun_irq;
dpu_encoder_phys_vid_vblank_irq


static const struct dpu_encoder_virt_ops dpu_encoder_parent_ops = {
        .handle_vblank_virt = dpu_encoder_vblank_callback,
        .handle_underrun_virt = dpu_encoder_underrun_callback,
        .handle_frame_done = dpu_encoder_frame_done_callback,
};

dpu_encoder_vblank_callback
    dpu_crtc_vblank_callback
		drm_crtc_handle_vblank
```



#### 2.用户态

![image-20240321160541228](C:\Users\rokid\AppData\Roaming\Typora\typora-user-images\image-20240321160541228.png)

![image-20240321191611493](C:\Users\rokid\AppData\Roaming\Typora\typora-user-images\image-20240321191611493.png)

```c
VSyncWorker::Routine
	drmWaitVBlank
		ret = ioctl(fd, DRM_IOCTL_WAIT_VBLANK, vbl);
```





![image-20240322143446276](C:\Users\rokid\AppData\Roaming\Typora\typora-user-images\image-20240322143446276.png)

vsync-sf有边沿跳变 但是还是在等待；

---

看下dsi-display.c代码





## 2.3 DisplayBase::PerformHwCommit::耗时长达1s

<img src="C:\Users\rokid\AppData\Roaming\Typora\typora-user-images\image-20240325115224336.png" alt="image-20240325115224336" style="zoom:150%;" />



### 1.HWC框架【CS架构】

[android多媒体框架介绍（四）显示图形系统之hwc叠加器_hwc layers-CSDN博客](https://blog.csdn.net/runafterhit/article/details/118884165)

Android 8.0 及更高版本使用一个名为Composer HAL 的 HIDL 接口，用于在 HWC 和 SurfaceFlinger 之间binderized IPC通信。即将上层Androd和底层HAL分别采用两个不用的进程实现，最终调用到vender的具体实现hwc中，中间采用Binder进行通信。形成如下几个关键部分：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210720080023873.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3J1bmFmdGVyaGl0,size_16,color_FFFFFF,t_70)

**HWC2 Client**在surfaceFlinger的service进程上下文中。**HWC2 Server**表示hwc的hal层服务端有独立的进程process【composer-servic】；

![image-20240326142729911](C:\Users\rokid\AppData\Roaming\Typora\typora-user-images\image-20240326142729911.png)

```
HIDL::IComposerClient::executeCommands_2_2::client

/la_vendor/hardware/qcom/display/composer/QtiComposerClient.cpp
HIDL::IComposerClient::executeCommands_2_2::server(QtiComposerClient)
```



SurfaceFlinger传入HWC的过程，都会调用beginCommand,最终都会通过QtiComposerClient::CommandReader::parseCommonCmd来解析

```cpp
QtiComposerClient::executeCommands_2_2
	QtiComposerClient::CommandReader::parse
		QtiComposerClient::CommandReader::parseCommonCmd
			QtiComposerClient::CommandReader::parsePresentOrValidateDisplay
				HWCDisplayBuiltIn::CommitOrPrepare::
```

如果没有Client合成的话，会尝试parsePresentOrValidateDisplay(PRESENT_OR_VALIDATE_DISPLAY)送显

[Android Qcom Display学习(四)_presentdisplay-CSDN博客](https://blog.csdn.net/qq_40405527/article/details/123395261)

```cpp
HWCDisplayBuiltIn::CommitOrPrepare::
    HWCDisplay::CommitOrPrepare::
        DisplayBase::CommitOrPrepare::
            DisplayBase::SetUpCommit(layer_stack)
                DisplayBase::Commit(display_comp_ctx_, &disp_layer_stack_)
                    DisplayBase::CommitLocked
                        DisplayBase::PerformHwCommit
                            Fence::Wait
```



### 2.函数调用

PerformHwCommit -> Fence::wait -> SyncWait(Fence::Get(fence), 1000)

```cpp
/la_vendor/hardware/qcom/display/sdm/libs/core/display_base.cpp
DisplayError DisplayBase::PerformHwCommit(HWLayersInfo *hw_layers_info) 
    
1554    // TODO(user): Workaround for messenger app flicker issue in CWB idle fallback,
1555    // to be removed when issue is fixed.
1556    if (cwb_fence_wait_ && hw_layers_info->output_buffer &&
1557        (hw_layers_info->output_buffer->release_fence != nullptr)) {
1558      if (Fence::Wait(hw_layers_info->output_buffer->release_fence) != kErrorNone) {
1559        DLOGW("sync_wait error errno = %d, desc = %s", errno, strerror(errno));
1560      }
1561    }


/la_vendor/hardware/qcom/display/sdm/libs/utils/fence.cpp

118  int Fence::Wait(const shared_ptr<Fence> &fence) {
119    ASSERT_IF_NO_BUFFER_SYNC(g_buffer_sync_handler_);
120  
121    return g_buffer_sync_handler_->SyncWait(Fence::Get(fence), 1000);
122  }
```

![image-20240325144727986](C:\Users\rokid\AppData\Roaming\Typora\typora-user-images\image-20240325144727986.png)

### 3.Wait入参

> Fence::Wait(hw_layers_info->output_buffer->release_fence)

```
DisplayError DisplayBase::SetUpCommit(LayerStack *layer_stack)
	disp_layer_stack_.info.output_buffer = layer_stack->output_buffer;
```



```
DisplayBuiltIn::SetUpCommit
	DisplayBase::SetUpCommit
```



![image-20240325144808392](C:\Users\rokid\AppData\Roaming\Typora\typora-user-images\image-20240325144808392.png)

可以看到是在performHWcommit中执行了Fence::Wait，后者调用SyncWait(Fence::Get(fence), 1000)等待了1s，与trace中耗时匹配



# 3.解决方案

注释Fence::wait代码段

![image-20240326112910428](C:\Users\rokid\AppData\Roaming\Typora\typora-user-images\image-20240326112910428.png)

注：fence等待代码看注释是去解决messenger应用（facebook）闪烁问题，目前看来对station2无影响，暂时忽略



# 4.延伸学习

## 4.1 Vsync

### 1.Vsync虚拟化

![img](https://pic1.zhimg.com/80/v2-b506717057826c9e593182544913b17e_720w.webp?source=2c26e567)

SurfaceFlinger中的DisplayVSync（Android S后改名为VsyncController）就是虚拟的VSync源，其需要两个参数来保证与硬件VSync的同步性，第一个是参考点，第二个就是周期。这些都可以开启硬件VSync同步解决。



### 2.Vsync同步

VSync虚拟化的本质就是在软件层面模拟硬件VSync，既然是软件模拟，那么就会存在托盘，如果自己的托盘比较大，那么就需要开启硬件VSync同步来进行安排。

首先是如何发现间隙比较大？答案是通过fence机制。SurfaceFlinger在每一帧交换HWC的时候，同时都会从HWC那里得到这一帧的PresentFence，它就是这一帧开始刷新到屏幕的信号的时候。

那驱动什么时候开始刷新一帧至屏幕呢，答案是屏幕VSync来的时候。根据PresentFence的信号时间就可以知道真实的VSync时间。

![1711080285888_72073935-8679-4e9e-85E7-3E8C95CD867E](D:\DingTalkAppData\DingTalk\216604157_v2\ImageFiles\1711080285888_72073935-8679-4e9e-85E7-3E8C95CD867E.png)

### 3.Vsync分配

![img](https://picx.zhimg.com/80/v2-9ac6437e04a24c407b4b578aea49b07b_720w.webp?source=2c26e567)

## 4.2 Qcom display

### 1.图显系统框架图

![drm](https://img-blog.csdnimg.cn/c03d564c64374a0393762fd3b0a83fd3.png)

libdrm:对底层定义在drm_ioctl.c 中各种IOCTL接口进行封装，向上层提供通用的API接口
GEM:Graphic Execution Manager，主要负责显示buffer的分配和释放，也是GPU唯一用到DRM的地方。
KMS:Kernel Mode Setting，负责显示buffer的切换，多图层的合成方式，显示位置。设置分辨率、刷新率等

KMS和DRM两者都是通过libdrm来进行调用的，但两者是具有相对独立性的，DPU对应的KMS对应于系统侧的HWC Composer，其重点是在于送显，而GPU对应DRM则侧重应用侧相关的渲染绘制。



### 2.KMS流程

![在这里插入图片描述](https://img-blog.csdnimg.cn/f827da455ca14ce48788724e7a99765c.png)



### 3.SurfaceFlinger传入HWC的过程

```cpp
HWCSession::PresentDisplay
	HWCDisplayBuiltIn::Present
		HWCDisplay::CommitLayerStack  // DumpInputBuffers() 高通dump inputbuffer的地方
			DisplayBuiltIn::Commit
				DisplayBase::Commit

	HWPeripheralDRM::Commit
		HWDeviceDRM::AtomicCommit
		    HWDeviceDRM::SetupAtomic // fb id用于对应framebuffer
			DRMAtomicReq::Commit
				drmModeAtomicCommit
					drm_atomic_commit
```



### 4.fence机制

fence：android4.4开始引入的一种资源同步机制，主要用于处理跨硬件场景，如CPU、GPU、HWC之间的buffer资源同步。可以将fence理解为一种资源锁。

举个例子，customer使用producer提供的buffer，使用完成后要还给producer生产，如果没有fence，通常是customer完全使用完成后开始归还转移buffer，拥有权/使用权 一起给到producer，producer获取到后可以马上使用。

从资源并发访问的原则上看其实customer使用完成后producer已经可以写了，在buffer转移这个过程实际上没有任何人在使用这个buffer。

fence的目的就是customer可以在开始用甚至是收到buffer的时候 开始归还转移buffer，同时也转移这个buffer对应的fence，producer收到buffer后拿到拥有权但不一定能使用， 等到需要使用的时候通过wait fence来阻塞查看buffer是否被customer使用完，wait后拿到使用权开始写。

理想的buffer轮转如下，保证buffer始终有人在使用。



![img](https://img-blog.csdnimg.cn/20190414100532761.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3J1bmFmdGVyaGl0,size_16,color_FFFFFF,t_70)



### 5.PLANE

PLANE代表显示图层，每个 CRTC 需要定义一个 primary plane 以及可选的 overlay plane、cursor plane

PLANE 存在的意义主要有两点：增强系统灵活性、提高系统性能
(1)对于背景和光标等变化不频繁的基本图显输出，可用通用 plane 来实现。而那些频繁变化则可由专用的 plane 来实现。
(2)plane 具备图像缩放、剪裁、多图层叠加等基本的图像处理功能，因此，可以让 GPU 来将更多的精力放在图形渲染上。



### 6.CRTC

CRTC从drm_plane 接收RGB像素数据并将其混合到一起，传输给下级显示设备drm_encoder。
CRTC 模块存在的意义主要有以下三点：
(1)统一协调 FB、DRM、用户空间代码之间对图显输出的控制
(2)图显模式的配置权回收到 kernel 空间，避免用户空间代码直接控制图显控制器引起 kernel panic
(3)kernel 空间实现的图显输出暂停与恢复相关代码，可以方便的调用图显控制器的配置函数



### 7.ENCODER

DRM Encoder 和 Connector 模块由图显外设抽象而来，Encoder 属于控制器部分，将内存的像素编码转换为显示器所需要的信号例如VGA、MIPI；Virutal Encoder管理一个逻辑显示，包含一个或多个物理encoder，每个物理encoder对应一个INTF硬件模块。



### 8.CONNECTOR

Connector 包含外设 PHY 或者显示器参数，代表连接的显示设备连接状态，支持的视频模式等。





# 5.疑问点

#### pending_kickoff_cnt

wait_timeout中atomic_cnt是pending_kickoff_cnt

打印值，为啥不为0

![image-20240321222749624](C:\Users\rokid\AppData\Roaming\Typora\typora-user-images\image-20240321222749624.png)



```
static inline int sde_encoder_phys_inc_pending(struct sde_encoder_phys *phys)
{
        SDE_ERROR("zyr sde_encoder_phys_inc_pending &phys_enc->pending_kickoff_cnt 1:%d\n",atomic_read(&phys->pending_kickoff_cnt));
        return atomic_inc_return(&phys->pending_kickoff_cnt);
}
```

这里面加的1



![image-20240321223749901](C:\Users\rokid\AppData\Roaming\Typora\typora-user-images\image-20240321223749901.png)

得看下sde_encoder_phys_vid_vblank_irq这个函数在做啥

1.399889-415972为啥有个_sde_crtc_vblank_enable

2.415972-425203 irq2到irq1为啥有10ms



# 6.参考链接

1.[DRM驱动（九）之drm_vblank_drmwaitvblank-CSDN博客](https://blog.csdn.net/fengchaochao123/article/details/135262216)

2.[vsync从中断到用户态的经历_hmid vsync中断-CSDN博客](https://blog.csdn.net/dshj2007/article/details/106225175)

3.[帧同步依赖技术理解：安卓图形系统及Vsync - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/616697833)

4.[简图记录-android fence机制-CSDN博客](https://blog.csdn.net/runafterhit/article/details/89293013)

5.[Android中的GraphicBuffer同步机制-Fence_android graphicbuffer-CSDN博客](https://blog.csdn.net/jinzhuojun/article/details/39698317)

6.[android多媒体框架介绍（四）显示图形系统之hwc叠加器_hwc layers-CSDN博客](https://blog.csdn.net/runafterhit/article/details/118884165)

