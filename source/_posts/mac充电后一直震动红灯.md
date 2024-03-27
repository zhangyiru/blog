---
title: 使用mac充电后一直震动红灯
tags:
- problem
---
## 背景

mac充电器充电（应该是插好的），早上来看电量没增加，插拔一次，震动了几下后红灯



## 日志

```c
[ 3209.332272] ---[ end trace c8d1c26a5a3e68b1 ]---
[ 3209.332286] Unable to handle kernel NULL pointer dereference at virtual address 0000000000000020
[ 3209.341529] Mem abort info:
[ 3209.344474]   ESR = 0x96000005
[ 3209.347662]   Exception class = DABT (current EL), IL = 32 bits
[ 3209.353795]   SET = 0, FnV = 0
[ 3209.357033]   EA = 0, S1PTW = 0
[ 3209.360358] Data abort info:
[ 3209.363410]   ISV = 0, ISS = 0x00000005
[ 3209.367457]   CM = 0, WnR = 0
[ 3209.370537] user pgtable: 4k pages, 39-bit VAs, pgdp = 000000000943e194
[ 3209.377427] [0000000000000020] pgd=0000000000000000, pud=0000000000000000
[ 3209.384468] Internal error: Oops: 96000005 [#1] PREEMPT SMP
[ 3209.390195] Modules linked in: wlan(O) machine_dlkm(O) wcd938x_slave_dlkm(O) wcd938x_dlkm(O) wcd9xxx_dlkm(O) mbhc_dlkm(O) tx_macro_dlkm(O) rx_macro_dlkm(O) va_macro_dlkm(O) wsa_macro_dlkm(O) swr_ctrl_dlkm(O) bolero_cdc_dlkm(O) wsa881x_dlkm(O) wcd_core_dlkm(O) stub_dlkm(O) hdmi_dlkm(O) swr_dlkm(O) pinctrl_lpi_dlkm(O) pinctrl_wcd_dlkm(O) usf_dlkm(O) native_dlkm(O) platform_dlkm(O) q6_dlkm(O) adsp_loader_dlkm(O) apr_dlkm(O) snd_event_dlkm(O) q6_notifier_dlkm(O) q6_pdr_dlkm(O) msm_11ad_proxy
[ 3209.434453] Process init (pid: 1, stack limit = 0x00000000aa1f60b5)
[ 3209.440886] CPU: 4 PID: 1 Comm: init Tainted: G S      W  O      4.19.157-perf #1
[ 3209.448560] Hardware name: Qualcomm Technologies, Inc. kona-xr-overlay Standalone (DT)
[ 3209.456681] pstate: 60400005 (nZCv daif +PAN -UAO)
[ 3209.461609] pc : klist_del+0x20/0x88
[ 3209.465292] lr : device_del+0xa4/0x4b0
[ 3209.469148] sp : ffffff800805bb90
[ 3209.470791] aw210xx_led 1-0020: usb: 0, pc_port: 0
[ 3209.472558] x29: ffffff800805bb90 x28: ffffffde408d9d80
[ 3209.472559] x27: 0000000000000000 x26: 0000000000000000
[ 3209.472560] x25: 0000000056000000 x24: ffffffdfcc8254b8
[ 3209.472561] x23: ffffffdfe99e2b20 x22: ffffffdfcc335008
[ 3209.472561] x21: 0000000000000000 x20: 0000000000000000
[ 3209.472562] x19: ffffffdf22ac2628 x18: 000000000000004c
[ 3209.472563] x17: ffffff9313a66000 x16: 0000000000000068
[ 3209.472563] x15: 0000000000000068 x14: 0000000000000086
[ 3209.472564] x13: 000000000000004c x12: 0000000000000000
[ 3209.472565] x11: 0000000000000000 x10: 0000000000000007
[ 3209.472565] x9 : b8e204bfccbe9500 x8 : 0000000000000000
[ 3209.472566] x7 : 0000000000000000 x6 : ffffffdfffaa6ba9
[ 3209.472567] x5 : 0000000000000000 x4 : 0000000000000006
[ 3209.472568] x3 : 0000000000000027 x2 : 0000000000000007
[ 3209.472568] x1 : 0000000000000007 x0 : 0000000000000000
[ 3209.472570] Call trace:
[ 3209.472572]  klist_del+0x20/0x88
[ 3209.472572]  device_del+0xa4/0x4b0
[ 3209.472574]  device_unregister+0x14/0x30
[ 3209.472576]  typec_unregister_partner+0x18/0x20
[ 3209.472579]  handle_disconnect+0x24c/0x2a0
[ 3209.581559]  disconnect_store+0x58/0x88
[ 3209.585496]  dev_attr_store+0x18/0x30
[ 3209.589266]  sysfs_kf_write+0x38/0x50
[ 3209.593034]  kernfs_fop_write+0x130/0x1d0
[ 3209.597151]  __vfs_write+0x44/0x148
[ 3209.600741]  vfs_write+0xe0/0x1a0
[ 3209.604152]  ksys_write+0x6c/0xd0
[ 3209.607562]  __arm64_sys_write+0x18/0x20
[ 3209.611588]  el0_svc_common+0x98/0x160
[ 3209.615446]  el0_svc_handler+0x64/0x80
```



```c
crash> dis -l klist_del+0x20
/data/jenkins/workspace/StationPro_new/sxr2130p_repo/emdoor/LINUX/android/kernel/msm-4.19/lib/klist.c: 213
0xffffff93127c2cd8 <klist_del+32>:      ldr     x22, [x20, #32]

klist_del(&dev->p->knode_parent);

void klist_del(struct klist_node *n)
{
        klist_put(n, true);
}

static struct klist *knode_klist(struct klist_node *knode)
{
        return (struct klist *)
                ((unsigned long)knode->n_klist & KNODE_KLIST_MASK);
}


210 static void klist_put(struct klist_node *n, bool kill)
211 {
212         struct klist *k = knode_klist(n);
213         void (*put)(struct klist_node *) = k->put; //put：函数指针，用于链表内的节点减少引用计数。
214 
215         spin_lock(&k->k_lock);
216         if (kill)
217                 knode_kill(n);
218         if (!klist_dec_and_del(n))
219                 put = NULL;
220         spin_unlock(&k->k_lock);
221         if (put)
222                 put(n);
223 }

crash> struct klist -xo
struct klist {
   [0x0] spinlock_t k_lock;
   [0x8] struct list_head k_list;
  [0x18] void (*get)(struct klist_node *);
  [0x20] void (*put)(struct klist_node *);
}
SIZE: 0x28

put在klist_init中初始化
put: The put function for the embedding object (NULL if none)
```

说明klist_node已经被释放，double free导致了空指针访问



下面日志说明/sys/class/typec/port0-partner/没有power这个组信息

```c
[ 3209.332078] ------------[ cut here ]------------
[ 3209.332082] sysfs group 'power' not found for kobject 'port0-partner'
[ 3209.332126] WARNING: CPU: 4 PID: 1 at fs/sysfs/group.c:255 sysfs_remove_group+0x4c/0xe0
[ 3209.332128] Modules linked in: wlan(O) machine_dlkm(O) wcd938x_slave_dlkm(O) wcd938x_dlkm(O) wcd9xxx_dlkm(O) mbhc_dlkm(O) tx_macro_dlkm(O) rx_macro_dlkm(O) va_macro_dlkm(O) wsa_macro_dlkm(O) swr_ctrl_dlkm(O) bolero_cdc_dlkm(O) wsa881x_dlkm(O) wcd_core_dlkm(O) stub_dlkm(O) hdmi_dlkm(O) swr_dlkm(O) pinctrl_lpi_dlkm(O) pinctrl_wcd_dlkm(O) usf_dlkm(O) native_dlkm(O) platform_dlkm(O) q6_dlkm(O) adsp_loader_dlkm(O) apr_dlkm(O) snd_event_dlkm(O) q6_notifier_dlkm(O) q6_pdr_dlkm(O) msm_11ad_proxy
[ 3209.332178] CPU: 4 PID: 1 Comm: init Tainted: G S      W  O      4.19.157-perf #1
[ 3209.332180] Hardware name: Qualcomm Technologies, Inc. kona-xr-overlay Standalone (DT)
[ 3209.332183] pstate: 60400005 (nZCv daif +PAN -UAO)
[ 3209.332186] pc : sysfs_remove_group+0x4c/0xe0
[ 3209.332189] lr : sysfs_remove_group+0x4c/0xe0
[ 3209.332190] sp : ffffff800805bb70
[ 3209.332191] x29: ffffff800805bb70 x28: ffffffde408d9d80
[ 3209.332194] x27: 0000000000000000 x26: 0000000000000000
[ 3209.332196] x25: 0000000056000000 x24: ffffffdfcc8254b8
[ 3209.332198] x23: ffffffdfe99e2b20 x22: ffffffdfcc335008
[ 3209.332199] x21: ffffffdf77aca010 x20: 0000000000000000
[ 3209.332201] x19: ffffff9312be3cd0 x18: 000000000000004c
[ 3209.332203] x17: ffffff9313a66000 x16: 0000000000000068
[ 3209.332204] x15: 0000000000000068 x14: 0000000000000086
[ 3209.332209] x13: 000000000000004c x12: 0000000000000000
[ 3209.332211] x11: 0000000000000000 x10: 0000000000000007
[ 3209.332212] x9 : b8e204bfccbe9500 x8 : b8e204bfccbe9500
[ 3209.332214] x7 : 0000000000000000 x6 : ffffffdfffaa6ba9
[ 3209.332216] x5 : 0000000000000000 x4 : 0000000000000006
[ 3209.332217] x3 : 0000000000000027 x2 : 0000000000000007
[ 3209.332219] x1 : 0000000000000007 x0 : 0000000000000039
[ 3209.332223] Call trace:
[ 3209.332227]  sysfs_remove_group+0x4c/0xe0
[ 3209.332234]  dpm_sysfs_remove+0x64/0x70
[ 3209.332238]  device_del+0x94/0x4b0
[ 3209.332240]  device_unregister+0x14/0x30
[ 3209.332247]  typec_unregister_partner+0x18/0x20
[ 3209.332249]  handle_disconnect+0x24c/0x2a0
[ 3209.332250]  disconnect_store+0x58/0x88
[ 3209.332252]  dev_attr_store+0x18/0x30
[ 3209.332253]  sysfs_kf_write+0x38/0x50
[ 3209.332255]  kernfs_fop_write+0x130/0x1d0
[ 3209.332260]  __vfs_write+0x44/0x148
[ 3209.332261]  vfs_write+0xe0/0x1a0
[ 3209.332262]  ksys_write+0x6c/0xd0
[ 3209.332263]  __arm64_sys_write+0x18/0x20
[ 3209.332268]  el0_svc_common+0x98/0x160
[ 3209.332270]  el0_svc_handler+0x64/0x80
[ 3209.332271]  el0_svc+0x8/0x380
[ 3209.332272] ---[ end trace c8d1c26a5a3e68b1 ]---

```



本地复现，插上mac充电器后一直震动，并且logcat一直刷以下日志，看上去是收到uevent事件，反复add,remove

<img src="C:\Users\rokid\AppData\Roaming\Typora\typora-user-images\image-20231117145408186.png" alt="image-20231117145408186" style="zoom:150%;" />



华为typec充电器

<img src="C:\Users\rokid\AppData\Roaming\Typora\typora-user-images\image-20231117154224970.png" alt="image-20231117154224970" style="zoom:200%;" />



mac充电器

![image-20231117154257776](C:\Users\rokid\AppData\Roaming\Typora\typora-user-images\image-20231117154257776.png)



另一个crash

日志如下：

```c
[  913.683162] Unable to handle kernel NULL pointer dereference at virtual address 0000000000000070
[  913.719414] Mem abort info:[  913.722622]   ESR = 0x96000005
[  913.725886]   Exception class = DABT (current EL), IL = 32 bits
[  913.732107]   SET = 0, FnV = 0
[  913.735294]   EA = 0, S1PTW = 0 
[  913.738608] Data abort info:
[  913.741712]   ISV = 0, ISS = 0x00000005
[  913.745756]   CM = 0, WnR = 0 
[  913.748943] user pgtable: 4k pages, 39-bit VAs, pgdp = 0000000054553fc4
[  913.755806] [0000000000000070] pgd=0000000000000000, pud=0000000000000000
[  913.762939] Internal error: Oops: 96000005 [#1] PREEMPT SMP
[  913.768679] Modules linked in: wlan(O) machine_dlkm(O) wcd938x_slave_dlkm(O) wcd938x_dlkm(O) wcd9xxx_dlkm(O) mbhc_dlkm(O) tx_macro_dlkm(O) rx_macro_dlkm(O) va_macro_dlkm(O) wsa_macro_dlkm(O) swr_ctrl_dlkm(O) bolero_cdc_dlkm(O) wsa881x_dlkm(O) wcd_core_dlkm(O) stub_dlkm(O) hdmi_dlkm(O) swr_dlkm(O) pinctrl_lpi_dlkm(O) pinctrl_wcd_dlkm(O) usf_dlkm(O) native_dlkm(O) platform_dlkm(O) q6_dlkm(O) adsp_loader_dlkm(O) apr_dlkm(O) snd_event_dlkm(O) q6_notifier_dlkm(O) q6_pdr_dlkm(O) msm_11ad_proxy[  913.812968] Process init (pid: 1, stack limit = 0x000000008e0451e0)
[  913.819426] CPU: 5 PID: 1 Comm: init Tainted: G S      W  O      4.19.157-perf #1[  913.827125] Hardware name: Qualcomm Technologies, Inc. kona-xr-overlay Standalone (DT)[  913.868118] aw210xx_led 1-0020: usb: 0, pc_port: 0
[  913.938910] usbpd usbpd0: Type-C Source (high - 3.0A) connected
[  914.001246] pstate: 60400005 (nZCv daif +PAN -UAO)
[  914.001260] pc : kernfs_find_ns+0x10/0x148
[  914.001262] lr : kernfs_find_and_get_ns+0x38/0x88
[  914.001262] sp : ffffff800805bb10
[  914.001263] x29: ffffff800805bb10 x28: ffffffde008dc9c0 [  914.001264] x27: 0000000000000000 x26: 0000000000000000
[  914.001265] x25: 0000000056000000 x24: ffffffdf8c80dcb8 [  914.001265] x23: ffffffded84ca520 x22: ffffffdf8c80b008 
[  914.001266] x21: 0000000000000000 x20: ffffff9fb5de3848  
[  914.001267] x19: 0000000000000000 x18: 0000000005f5e100
[  914.001268] x17: 00000000002f29cd x16: 0000000000000001  
[  914.001268] x15: fffffffffee3c95c x14: 0000000000000010 [  914.001269] x13: ffffff9fb4604e98 x12: 0000000000000028 
[  914.001270] x11: 0000000000000072 x10: 00000000243fd918 [  914.001271] x9 : 00000000243fd916 x8 : ffffff9fb6a5b4b0 
[  914.001271] x7 : fefefefefefefefe x6 : 0000000000808080 [  914.001272] x5 : 0000000000000000 x4 : ffffffffffffffff 
[  914.001273] x3 : 0000007265776f70 x2 : 0000000000000000 [  914.001273] x1 : ffffff9fb5de3848 x0 : 0000000000000000 
[  914.001276] Call trace:
[  914.001278]  kernfs_find_ns+0x10/0x148
[  914.001279]  kernfs_find_and_get_ns+0x38/0x88
[  914.001282]  sysfs_remove_group+0x30/0xe0
[  914.001287]  dpm_sysfs_remove+0x64/0x70
[  914.001290]  device_del+0x94/0x4b0
[  914.001290]  device_unregister+0x14/0x30
[  914.001295]  typec_unregister_partner+0x18/0x20
[  914.001297]  handle_disconnect+0x24c/0x2a0
[  914.001297]  disconnect_store+0x58/0x88
[  914.001299]  dev_attr_store+0x18/0x30
[  914.001300]  sysfs_kf_write+0x38/0x50
[  914.001301]  kernfs_fop_write+0x130/0x1d0
[  914.001305]  __vfs_write+0x44/0x148
[  914.001305]  vfs_write+0xe0/0x1a0
[  914.001306]  ksys_write+0x6c/0xd0
[  914.001307]  __arm64_sys_write+0x18/0x20
[  914.001312]  el0_svc_common+0x98/0x160
[  914.001313]  el0_svc_handler+0x64/0x80
[  914.001315]  el0_svc+0x8/0x380
[  914.001317] Code: a9bd7bfd a90157f6 910003fd a9024ff4 (7940e008)
```



```shell
crash> dis -l kernfs_find_ns+0x10
/data/jenkins/workspace/StationPro_new/sxr2130p_repo/emdoor/LINUX/android/kernel/msm-4.19/include/linux/kernfs.h: 315
```



```c
static inline bool kernfs_ns_enabled(struct kernfs_node *kn)
{
        return kn->flags & KERNFS_NS; //空指针访问
}
```



```c
crash> struct kernfs_node -xo
struct kernfs_node {
   [0x0] atomic_t count;
   [0x4] atomic_t active;
   [0x8] struct kernfs_node *parent;
  [0x10] const char *name;
  [0x18] struct rb_node rb;
  [0x30] const void *ns;
  [0x38] unsigned int hash;
         union {
  [0x40]     struct kernfs_elem_dir dir;
  [0x40]     struct kernfs_elem_symlink symlink;
  [0x40]     struct kernfs_elem_attr attr;
         };
  [0x60] void *priv;
  [0x68] union kernfs_node_id id;
  [0x70] unsigned short flags; //called
  [0x72] umode_t mode;
  [0x78] struct kernfs_iattrs *iattr;
}
SIZE: 0x80
```



说明kernfs_node为null



```c
void sysfs_remove_group(struct kobject *kobj,
			const struct attribute_group *grp)
{
	struct kernfs_node *parent = kobj->sd;
	struct kernfs_node *kn;

	if (grp->name) {
		kn = kernfs_find_and_get(parent, grp->name);
		if (!kn) {
			WARN(!kn, KERN_WARNING
			     "sysfs group '%s' not found for kobject '%s'\n",
			     grp->name, kobject_name(kobj));
			return;
		}
	} else {
		kn = parent;
		kernfs_get(kn);
	}

	remove_files(kn, grp);
	if (grp->name)
		kernfs_remove(kn);

	kernfs_put(kn);
}
EXPORT_SYMBOL_GPL(sysfs_remove_group);

static inline struct kernfs_node *
kernfs_find_and_get(struct kernfs_node *kn, const char *name)
{
        return kernfs_find_and_get_ns(kn, name, NULL);
}
```



```c
kernfs_find_and_get_ns
	kn = kernfs_find_ns(parent, name, ns);
		bool has_ns = kernfs_ns_enabled(parent);
```

