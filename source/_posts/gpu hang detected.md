---
title: gpu hang detected
tags:
- problem
---

gpu hang detected



### 日志打印

```html
[16799.633153] kgsl kgsl-3d0: MISC: GPU hang detected
[16799.638132] kgsl kgsl-3d0: maku.bilibilihd[22918]: gpu fault ctx 36 ctx_type GL ts 157287 status 00EF0CF5 rb 13e0/1510 ib1 0000000500C90000/0000 ib2 0000000500CA04F8/0000
[16799.653814] kgsl kgsl-3d0: maku.bilibilihd[22918]: gpu fault rb 2 rb sw r/w 13e0/1644
```



### 代码分析

LINUX/android/kernel/msm-4.19/drivers/gpu/msm/adreno.c

```c
/*
 * adreno_hang_int_callback() - Isr for fatal interrupts that hang GPU
 * @adreno_dev: Pointer to device
 * @bit: Interrupt bit
 */
void adreno_hang_int_callback(struct adreno_device *adreno_dev, int bit)
{
        dev_crit_ratelimited(KGSL_DEVICE(adreno_dev)->dev,
                                "MISC: GPU hang detected\n");
        adreno_irqctrl(adreno_dev, 0);

        /* Trigger a fault in the dispatcher - this will effect a restart */
        adreno_set_gpu_fault(adreno_dev, ADRENO_HARD_FAULT);
        adreno_dispatcher_schedule(KGSL_DEVICE(adreno_dev));
}

```



#### adreno_hang_int_callback定义位置

```c
static struct adreno_irq_funcs a6xx_irq_funcs[32] = {
	...
	/* 23 - MISC_HANG_DETECT */
    ADRENO_IRQ_CALLBACK(adreno_hang_int_callback),
    ...
}

#define ADRENO_IRQ_CALLBACK(_c) { .func = _c }
-> .func = adreno_hang_int_callback

static struct adreno_irq a6xx_irq = {
        .funcs = a6xx_irq_funcs, //called
        .mask = A6XX_INT_MASK,
};

struct adreno_gpudev adreno_a6xx_gpudev = {
        .reg_offsets = &a6xx_reg_offsets,
        .start = a6xx_start,
        .snapshot = a6xx_snapshot,
        .irq = &a6xx_irq, //called
        ...
}
```



#### adreno_irq_handler中断处理函数调用adreno_hang_int_callback

```c
#LINUX/android/kernel/msm-4.19/drivers/gpu/msm/adreno.c
static irqreturn_t adreno_irq_handler(struct kgsl_device *device)
{
    ...
    adreno_readreg(adreno_dev, ADRENO_REG_RBBM_INT_0_STATUS, &status);

        /*
         * Clear all the interrupt bits but ADRENO_INT_RBBM_AHB_ERROR. Because
         * even if we clear it here, it will stay high until it is cleared
         * in its respective handler. Otherwise, the interrupt handler will
         * fire again.
         */
    int_bit = ADRENO_INT_BIT(adreno_dev, ADRENO_INT_RBBM_AHB_ERROR);
    adreno_writereg(adreno_dev, ADRENO_REG_RBBM_INT_CLEAR_CMD,
                                status & ~int_bit);
	/* Loop through all set interrupts and call respective handlers */
    for (tmp = status; tmp != 0;) {
        	//1.tmp从低位往左边数第24位非0
        	//#define A3XX_INT_MISC_HANG_DETECT 24 (a3xx_reg.h)
        	//2.i为23会回调adreno_hang_int_callback
            i = fls(tmp) - 1; 

            if (irq_params->funcs[i].func != NULL) {
                    if (irq_params->mask & BIT(i))
                            irq_params->funcs[i].func(adreno_dev, i);
            } else
                    dev_crit_ratelimited(device->dev,
                                            "Unhandled interrupt bit %x\n",
                                            i);

            ret = IRQ_HANDLED;

            tmp &= ~BIT(i);
    }
	...
}
```



```c
kgsl_device_platform_probe
	kgsl_request_irq(device->pdev, device->pwrctrl.irq_name, kgsl_irq_handler, device)
		device->ftbl->irq_handler
			adreno_irq_handler

kgsl_request_irq
	//申请中断
    //devm_request_irq和request_irq区别在于前者是申请的内核“managed”资源，不需要自己手动释放，会自动回收资源，而后者需要手动调用free_irq来释放中断资源
	devm_request_irq(&pdev->dev, num, handler, IRQF_TRIGGER_HIGH, name, data);           
```



#### adreno_fault_header打印gpu fault ctx

```c
[16799.633153] kgsl kgsl-3d0: MISC: GPU hang detected
[16799.638132] kgsl kgsl-3d0: maku.bilibilihd[22918]: gpu fault ctx 36 ctx_type GL ts 157287 status 00EF0CF5 rb 13e1510 ib1 0000000500C90000/0000 ib2 0000000500CA04F8/0000
[16799.653814] kgsl kgsl-3d0: maku.bilibilihd[22918]: gpu fault rb 2 rb sw r/w 13e0/1644
[16799.661893] kgsl kgsl-3d0: GMU snapshot started at 0xcce4dc5fd ticks

static void adreno_fault_header(struct kgsl_device *device,
                struct adreno_ringbuffer *rb, struct kgsl_drawobj_cmd *cmdobj,
                int fault)
{
	...
	trace_adreno_gpu_fault(drawobj->context->id,
        drawobj->timestamp,
        status, rptr, wptr, ib1base, ib1sz,
        ib2base, ib2sz, drawctxt->rb->id);

    pr_fault(device, drawobj,
            "%s fault ctx %d ctx_type %s ts %d status %8.8X rb %4.4x/%4.4x ib1 %16.16llX/%4.4x ib2 %16.16llX/%4.4x\n",
            type, drawobj->context->id,
            get_api_type_str(drawctxt->type),
            drawobj->timestamp, status,
            rptr, wptr, ib1base, ib1sz, ib2base, ib2sz);

}
```



#### 上层调用do_header_and_snapshot

```c
static void do_header_and_snapshot(struct kgsl_device *device, int fault,
                struct adreno_ringbuffer *rb, struct kgsl_drawobj_cmd *cmdobj)
{
        struct kgsl_drawobj *drawobj = DRAWOBJ(cmdobj);

        /* Always dump the snapshot on a non-drawobj failure */
        if (cmdobj == NULL) {
                adreno_fault_header(device, rb, NULL, fault); //called
                kgsl_device_snapshot(device, NULL, fault & ADRENO_GMU_FAULT);
                return;
        }

        /* Skip everything if the PMDUMP flag is set */
        if (test_bit(KGSL_FT_SKIP_PMDUMP, &cmdobj->fault_policy))
                return;

        /* Print the fault header */
        adreno_fault_header(device, rb, cmdobj, fault); //called

        if (!(drawobj->context->flags & KGSL_CONTEXT_NO_SNAPSHOT))
                kgsl_device_snapshot(device, drawobj->context,
                                        fault & ADRENO_GMU_FAULT);
}
```



```c
adreno_dispatcher_work
    //dispatcher_do_fault() returns 0 if no faults occurred. If that is the case, then clean up preemption and try to schedule more work
	dispatcher_do_fault
    	//if (!(fault & ADRENO_GMU_FAULT_SKIP_SNAPSHOT))
		do_header_and_snapshot
    		//dmesg打印
			adreno_fault_header
    		//kgsl_worker_thr PID290
			kgsl_device_snapshot
    			//打快照
    			adreno_snapshot
```



#### dmesg打印完再打快照

kgsl_worker_thr内核线程正好在循环里面的usleep_range

```shell
crash> ps | grep UN
      290       2   7  ffffffc4133a5880  UN   0.0        0        0  [kgsl_worker_thr]
      540       2   0  ffffffc412223b00  UN   0.0        0        0  [hdcp_2x]
      541       2   1  ffffffc4122249c0  UN   0.0        0        0  [dp_hdcp2p2]
crash> 
crash> 
crash> bt 290
PID: 290      TASK: ffffffc4133a5880  CPU: 7    COMMAND: "kgsl_worker_thr"
 #0 [ffffff80146839d0] __switch_to at ffffffacb348ac54
 #1 [ffffff8014683a20] __schedule at ffffffacb48d33c8
 #2 [ffffff8014683a80] schedule at ffffffacb48d368c
 #3 [ffffff8014683b00] schedule_hrtimeout_range_clock at ffffffacb48d8160
 #4 [ffffff8014683b30] schedule_hrtimeout_range at ffffffacb48d81f4
 #5 [ffffff8014683b50] usleep_range at ffffffacb48d7e90
 #6 [ffffff8014683bb0] a6xx_snapshot at ffffffacb3c07544
 #7 [ffffff8014683c50] adreno_snapshot at ffffffacb3beff18
 #8 [ffffff8014683ce0] kgsl_device_snapshot at ffffffacb3bd6a80
 #9 [ffffff8014683da0] adreno_dispatcher_work at ffffffacb3becc40
#10 [ffffff8014683e10] kthread_worker_fn at ffffffacb34ebdd8
#11 [ffffff8014683e60] kthread at ffffffacb34ece6c
```

