---
title: wlan crash【bind报错ENOSPC】
tags:
- problem
---
## wlan crash

```c
[   66.175350] cnss-daemon: interop issues ap: read_iot_ap_file: No such file /data/vendor/wifi/iotap_ps.bin
[   66.175356] cnss-daemon: interop issues ap: read_iot_ap_ps_from_file, read file error
[   66.175469] cnss-daemon: Fail to bind user socket No space left on device
[   66.175477] cnss-daemon: Failed to init user interface
[   66.175517] cnss-daemon: interop issues ap: save_iot_ap_file: Failed to open file /data/vendor/wifi/iotap_ps.bin
[   66.175520] cnss-daemon: interop issues ap: save_iot_ap_ps_to_file, save file error
...
[   70.558379] init: Could not store persistent property: Could not open temporary properties file: No space left on device
[   70.584538] msm_qti_pp_get_rms_value_control, back not active to query rms be_idx:3
[   70.596279] core_get_license_status: cmdrsp_license_result.result = 0x15 for module 0x131ff
[   70.596637] msm_ext_disp_update_audio_ops: Display not found (EXT_DISPLAY_TYPE_HDMI) ctld (0) stream (0)
[   70.607535] msm-ext-disp-audio-codec-rx soc:qcom,msm-ext-disp:qcom,msm-ext-disp-audio-codec-rx: msm_ext_disp_audio_device_get: invalid dai id: 4
[   70.624021] cnss: fatal: Timeout waiting for FW ready indication
[   70.630204] cnss: Update driver status: 4
[   70.630216] cnss: Posting event: RECOVERY(9), state: 0x11 flags: 0x0
[   70.630217] cnss: PM stay awake, state: 0x11, count: 1
[   70.630226] cnss: PM relax, state: 0x11, count: 0
[   70.630230] cnss: PM stay awake, state: 0x11, count: 1
[   70.630233] cnss: Processing driver event: RECOVERY(9), state: 0x11
[   70.630235] cnss: Driver recovery is triggered with reason: TIMEOUT(3)
[   70.630240] subsys-restart: subsystem_restart_dev(): Restart sequence requested for wlan, restart_level = SYSTEM.
[   70.630246] cnss: PM relax, state: 0x611, count: 0
[   70.640767] msm_adsp_stream_callback_get: ASM Stream PP event queue is empty.
[   70.655599] msm_pcm_volume_ctl_get substream not found
[   70.667346] binder: 6681:6681 ioctl 40046210 7fc5227c24 returned -22
[   70.732033] Kernel panic - not syncing: subsys-restart: Resetting the SoC - wlan crashed.
[   70.740438] CPU: 5 PID: 78 Comm: kworker/5:1 Tainted: G S      W  O      4.19.157-perf #1
[   70.748834] Hardware name: Qualcomm Technologies, Inc. kona-xr-overlay Standalone (DT)
[   70.756982] Workqueue: events device_restart_work_hdlr
[   70.762272] Call trace:
[   70.763745] msm_qti_pp_get_rms_value_control, back not active to query rms be_idx:3
[   70.810796]  dump_backtrace+0x0/0x208
[   70.810798]  show_stack+0x14/0x20
[   70.810801]  dump_stack+0xb8/0xf4
[   70.810802]  panic+0x158/0x2d8
[   70.810805]  device_restart_work_hdlr+0x44/0x48
[   70.823290] core_get_license_status: cmdrsp_license_result.result = 0x15 for module 0x131ff
[   70.825875]  process_one_work+0x278/0x440
[   70.825876]  worker_thread+0x260/0x4a8
[   70.825877]  kthread+0x140/0x150
[   70.825878]  ret_from_fork+0x10/0x18
[   70.825881] SMP: stopping secondary CPUs
```

日志推断空间不足导致cnss超时，最终restart soc导致crash



### 考虑data分区内存不足

看到ENOSPC首先想到是空间不足，但是kmem看free很多

```shell
crash> mount | grep data
ffffffc70cd3ad80 ffffffc59e09f800 ext4   /dev/block/by-name/metadata /metadata 
ffffffc70cd3b640 ffffffc59e3c7000 ext4   /dev/block/dm-10 /apex/com.android.tzdata
ffffffc736581180 ffffffc70bdfe000 f2fs   /dev/block/dm-15 /data     
ffffffc713b48380 ffffffc70bdfe000 f2fs   /dev/block/dm-15 /data/user/0
ffffffc736582840 ffffffc70bdff800 tmpfs  tmpfs     /data_mirror
ffffffc736582680 ffffffc70bdfe000 f2fs   /dev/block/dm-15 /data_mirror/data_ce/null
ffffffc736583640 ffffffc70bdfe000 f2fs   /dev/block/dm-15 /data_mirror/data_ce/null/user/0
ffffffc736583800 ffffffc70bdfe000 f2fs   /dev/block/dm-15 /data_mirror/data_de/null
ffffffc713b48fc0 ffffffc70bdfe000 f2fs   /dev/block/dm-15 /data_mirror/cur_profiles
ffffffc736580000 ffffffc70bdfe000 f2fs   /dev/block/dm-15 /data_mirror/ref_profiles
```



```shell
crash> dev -d
MAJOR GENDISK            NAME       REQUEST_QUEUE      TOTAL ASYNC  SYNC   DRV
  254 ffffffc712f53800   zram0      ffffffc712e26b40       0     0     0     0
    8 ffffffc7120f6800   sda        ffffffc59e17b0c0       0     0     0     0
    8 ffffffc59e1f6800   sdb        ffffffc59e17c440       0     0     0     0
    8 ffffffc59e1f7000   sdc        ffffffc59e17f500       0     0     0     0
    8 ffffffc59e1f0800   sdf        ffffffc59e17ba80       0     0     0     0
    8 ffffffc59e1f5000   sdd        ffffffc59e17ce00       0     0     0     0
    8 ffffffc59e1f2800   sde        ffffffc59e17e180       0     0     0     0
  252 ffffffc59e099800   dm-0       ffffffc59dd4ba80       0     0     0     0
  252 ffffffc59dd7d800   dm-1       ffffffc59dd48000       0     0     0     0
  252 ffffffc59dd7f800   dm-2       ffffffc59dd49380       0     0     0     0
  252 ffffffc59dd7c000   dm-3       ffffffc59dd4eb40       0     0     0     0
  252 ffffffc59dd79800   dm-4       ffffffc59dd4a700       0     0     0     0
  252 ffffffc59dd7b000   dm-5       ffffffc59dd49d40       0     0     0     0
  252 ffffffc59dd79000   dm-6       ffffffc59dd4d7c0       0     0     0     0
  252 ffffffc59dd9f000   dm-7       ffffffc59dd489c0       0     0     0     0
  252 ffffffc59dd9d000   dm-8       ffffffc59dd4b0c0       0     0     0     0
  252 ffffffc59dd9a800   dm-9       ffffffc59dd4c440       0     0     0     0
  252 ffffffc59dd98800   dm-10      ffffffc59dd4f500       0     0     0     0
  252 ffffffc59e30d000   dm-11      ffffffc59e178000       0     0     0     0
  252 ffffffc59d82d800   dm-12      ffffffc59e179380       0     0     0     0
  252 ffffffc59d82c000   dm-13      ffffffc59e17eb40       0     0     0     0
  252 ffffffc59d9dd800   dm-14      ffffffc59e17a700       0     0     0     0
  252 ffffffc70c5cc000   dm-15      ffffffc707cace00       0     0     0     0
```



```
crash> kmem -i
                 PAGES        TOTAL      PERCENTAGE
    TOTAL MEM  3007109      11.5 GB         ----
         FREE  1941920       7.4 GB   64% of TOTAL MEM
         USED  1065189       4.1 GB   35% of TOTAL MEM
       SHARED   114800     448.4 MB    3% of TOTAL MEM
      BUFFERS     1407       5.5 MB    0% of TOTAL MEM
       CACHED   208340     813.8 MB    6% of TOTAL MEM
         SLAB    42807     167.2 MB    1% of TOTAL MEM

   TOTAL HUGE        0            0         ----
    HUGE FREE        0            0    0% of TOTAL HUGE

   TOTAL SWAP        0            0         ----
    SWAP USED        0            0    0% of TOTAL SWAP
    SWAP FREE        0            0    0% of TOTAL SWAP

 COMMIT LIMIT  1503554       5.7 GB         ----
    COMMITTED  1772605       6.8 GB  117% of TOTAL LIMIT
```



高通回复：

![image-20231109143700131](C:\Users\rokid\AppData\Roaming\Typora\typora-user-images\image-20231109143700131.png)



```c
static inline int utilization(struct f2fs_sb_info *sbi)
{
        return div_u64((u64)valid_user_blocks(sbi) * 100,
                                        sbi->user_block_count);
}

u64 div_u64(u64 dividend, u32 divisor)
{
    return dividend / divisor;
}

static inline block_t valid_user_blocks(struct f2fs_sb_info *sbi)
{
        return sbi->total_valid_block_count;
}

total_valid_block_count/user_block_count
```



```c
crash> f2fs_sb_info  0xffffffc707cca000 | grep block_count
  user_block_count = 26074112,
  total_valid_block_count = 26074112,
  last_valid_block_count = 26074112,
  unusable_block_count = 0,
  alloc_valid_block_count = {
  block_count = {87, 267},
```



查看源码报错点发现是bind失败，继续分析bind内核源码

### 用户态cnss报错函数

```c
int cnss_user_socket_init(int sock_type)
{       
        int sockfd = -1;
        struct sockaddr_un un_address;
        int ret = 0;
        
        switch (sock_type) {
        case AF_UNIX:
                if (strlen(CNSS_USER_SERVER) >
                    (sizeof(un_address.sun_path) - 1)) {
                        wsvc_printf_err("Invalid server path %s\n",
                                        CNSS_USER_SERVER);
                        ret = -EINVAL;
                        break;
                }
                
                if (access(CNSS_USER_SERVER, F_OK) == 0) {
                        wsvc_printf_info("server %s exists, remove it \n",
                                         CNSS_USER_SERVER);
                        remove(CNSS_USER_SERVER);
                }
                
                sockfd = socket(AF_UNIX, SOCK_DGRAM, 0);
                if (sockfd < 0) {
                        wsvc_printf_err("Fail to create user socket %s\n",
                                        strerror(errno));
                        ret = -errno;
                        break;
                }
                
                memset(&un_address, 0, sizeof(struct sockaddr_un));
                un_address.sun_family = AF_UNIX;
                //CNSS_USER_SERVER赋值给sun_path
                strlcpy(un_address.sun_path, CNSS_USER_SERVER,
                        (sizeof(un_address.sun_path) - 1));
                
                if (bind(sockfd, (struct sockaddr *)&un_address,
                         sizeof(un_address)) < 0) {
                        wsvc_printf_err("Fail to bind user socket %s\n",
                                        strerror(errno)); //ERROR
                        close(sockfd);
                        sockfd = -1;
                        ret = -errno;
                }
                break;
        default:
                wsvc_printf_err("%s: Unknown sock type %d\n",
                                __func__, sock_type);
                ret = -EINVAL;
        }
        
        cnss_user_sock = sockfd;
        return ret;
}
```



#### sun_path -> CNSS_USER_SERVER

```c
/* LINUX/android/vendor/qcom/proprietary/wlan/cnss-daemon/cnss_cli.h */
#ifdef ANDROID
#define CNSS_USER_SERVER "/data/vendor/wifi/sockets/cnss_user_server"
#define CNSS_USER_CLIENT "/data/vendor/wifi/sockets/cnss_user_client"
#else
#define CNSS_USER_SERVER "/var/run/cnss_user_server"
#define CNSS_USER_CLIENT "/var/run/cnss_user_client"
#endif
```



正常环境：

![image-20231030201034484](C:\Users\rokid\AppData\Roaming\Typora\typora-user-images\image-20231030201034484.png)

```shell
crash> ps | grep cnss
     1392       1   5  ffffffcdafdbbb00  IN   0.1 12939508     9736  cnss-daemon
     1446       1   2  ffffffcda12dd880  IN   0.1 12939508     9736  cnss-daemon
     3676       1   6  ffffffcd8b220ec0  UN   0.1 12939508     9736  cnss-daemon
     3904       1   1  ffffffcd7677e740  IN   0.1 12939508     9736  cnss-daemon
     3978       1   5  ffffffcd72796740  UN   0.1 12939508     9736  cnss-daemon
```



异常环境：没有cnss-daemon守护进程

```shell
crash> ps | grep cnss
     1297       1   5  ffffffc6eaf83b00  IN   0.1 12927572     6560  cnss_diag
```



`cnss_diag` 和 `cnss-daemon` 是 Qualcomm 芯片上的两个不同的组件，它们在 Wi-Fi 和蓝牙功能的实现中扮演不同的角色。

`cnss_diag` 是一个用于诊断 Qualcomm 芯片上 Wi-Fi 和蓝牙功能的工具，它可以帮助开发人员和工程师查找和解决与 Wi-Fi 和蓝牙相关的问题。它提供了一系列的命令和选项，可以用于获取和分析 Wi-Fi 和蓝牙的日志、统计信息和诊断数据等。

`cnss-daemon` 是一个运行在 Qualcomm 芯片上的守护进程，它负责管理和控制 Wi-Fi 和蓝牙的硬件和软件资源。它与 Android 系统的 HAL 层和框架层进行交互，接收和处理来自应用程序和用户的 Wi-Fi 和蓝牙请求，同时也负责处理 Wi-Fi 和蓝牙的事件、状态和错误等。



### bind unix

bind系统调用

```c
SYSCALL_DEFINE3(bind, int, fd, struct sockaddr __user *, umyaddr, int, addrlen)
{
        return __sys_bind(fd, umyaddr, addrlen);
}

/* net/socket.c */
int __sys_bind(int fd, struct sockaddr __user *umyaddr, int addrlen)
{
        struct socket *sock;
        struct sockaddr_storage address;
        int err, fput_needed;

        sock = sockfd_lookup_light(fd, &err, &fput_needed);
        if (sock) {
                err = move_addr_to_kernel(umyaddr, addrlen, &address);
                if (err >= 0) {
                        err = security_socket_bind(sock,
                                                   (struct sockaddr *)&address,
                                                   addrlen);
                        if (!err)
                                err = sock->ops->bind(sock, //called
                                                      (struct sockaddr *)
                                                      &address, addrlen);
                }
		...
}
```



#### socket结构体

```shell
crash> socket | grep ops
    const struct proto_ops *ops;
```



```c
/* include/linux/socket.h */
#define PF_UNIX         AF_UNIX

static const struct proto_ops unix_stream_ops = {
        .family =       PF_UNIX,
        .owner =        THIS_MODULE,
        .release =      unix_release,
        .bind =         unix_bind, //called
		...
}
```



#### bind底层调用关系

```c
static int unix_bind(struct socket *sock, struct sockaddr *uaddr, int addr_len)
{
	...
    //sun_path来源于uaddr
    struct sockaddr_un *sunaddr = (struct sockaddr_un *)uaddr;
    char *sun_path = sunaddr->sun_path;
    ...
	if (addr_len == sizeof(short)) {
            err = unix_autobind(sock);
            goto out;
    }
    ...
}
```

#### unix_autobind

```c
static int unix_autobind(struct socket *sock)
{
	...
    struct unix_address *addr;
    ...
    addr = kzalloc(sizeof(*addr) + sizeof(short) + 16, GFP_KERNEL);
    addr->name->sun_family = AF_UNIX;
    ...
	if (__unix_find_socket_byname(net, addr->name, addr->len, sock->type,
                                      addr->hash)) {
            spin_unlock(&unix_table_lock);
            /*
             * __unix_find_socket_byname() may take long time if many names
             * are already in use.
             */
            cond_resched();
            /* Give up if all names seems to be in use. */
            if (retries++ == 0xFFFFF) {
                    err = -ENOSPC;
                    kfree(addr);
                    goto out;
            }
            goto retry;
    }
    ...
}

struct unix_address {
    atomic_t refcnt;
    int len;
    unsigned hash;
    struct sockaddr_un name[0];
};

struct sockaddr_un {
        __kernel_sa_family_t sun_family; /* AF_UNIX */
        char sun_path[UNIX_PATH_MAX];   /* pathname */
};
```



#### __unix_find_socket_byname

```c
static struct sock *__unix_find_socket_byname(struct net *net,
                                              struct sockaddr_un *sunname,
                                              int len, int type, unsigned int hash)
{
        struct sock *s;

        sk_for_each(s, &unix_socket_table[hash ^ type]) {
                struct unix_sock *u = unix_sk(s);

                if (!net_eq(sock_net(s), net))
                        continue;
				//sunname包含AF_UNIX和/data/vendor/wifi/sockets/cnss_user_server
            	//表示这个socket名一直被占用
                if (u->addr->len == len &&
                    !memcmp(u->addr->name, sunname, len)) //memcmp为0表示u->addr->name == sunname
                        goto found;
        }
        s = NULL;
found:
        return s;
}
```



```c
#define sk_for_each(__sk, list) \
        hlist_for_each_entry(__sk, list, sk_node)

/**
 * hlist_for_each_entry - iterate over list of given type
 * @pos:        the type * to use as a loop cursor.
 * @head:       the head for your list.
 * @member:     the name of the hlist_node within the struct.
 */
#define hlist_for_each_entry(pos, head, member)                         \
        for (pos = hlist_entry_safe((head)->first, typeof(*(pos)), member);\
             pos;                                                       \
             pos = hlist_entry_safe((pos)->member.next, typeof(*(pos)), member))
```



#### unix_socket_table

```c
static inline void unix_insert_socket(struct hlist_head *list, struct sock *sk)
{
        spin_lock(&unix_table_lock);
        __unix_insert_socket(list, sk);
        spin_unlock(&unix_table_lock);
}

static void __unix_insert_socket(struct hlist_head *list, struct sock *sk)
{
        WARN_ON(!sk_unhashed(sk));
        sk_add_node(sk, list);
}

static inline void sk_add_node(struct sock *sk, struct hlist_head *list)
{
        sock_hold(sk);
        __sk_add_node(sk, list);
}

static inline void __sk_add_node(struct sock *sk, struct hlist_head *list)
{
        hlist_add_head(&sk->sk_node, list);
}
#define sk_node                 __sk_common.skc_node

static inline void hlist_add_head(struct hlist_node *n, struct hlist_head *h)
{
        struct hlist_node *first = h->first;
        n->next = first;
        if (first)
                first->pprev = &n->next;
        WRITE_ONCE(h->first, n);
        n->pprev = &h->first;
}
```



```shell
crash> sock_common -xo | grep skc_node
  [0x68]     struct hlist_node skc_node;
crash> sock -xo | grep sock_common
    [0x0] struct sock_common __sk_common;

crash> unix_socket_table | grep -v 0x0 | grep first | head -n 1
    first = 0xffffffc59ddb5568

first - 0x68 = sock
```



解析unix_socket_table 居然没有cnss_user_server 不理解

> first = 0xffffffc59ddb5568 /dev/socket/property_service
> first = 0xffffffc7088004a8 /dev/socket/logd
> first = 0xffffffc59ddb5de8 /dev/socket/traced_consumer
> first = 0xffffffc59ddb4468 /dev/socket/traced_producer
> first = 0xffffffc59ddb48a8 /dev/socket/dpmd
> first = 0xffffffc59ddb7328 /dev/socket/tcm
> first = 0xffffffc59ddb6668 /dev/socket/dpmwrapper
> first = 0xffffffc708801e28 /dev/socket/qwes_ipc
> first = 0xffffffc708a64028 /dev/socket/ims_qmid
> first = 0xffffffc6efa6c028 /dev/socket/location/mq/location-mq-s
> first = 0xffffffc708a61e28 /dev/socket/thermal-send-client
> first = 0xffffffc708a62268 /dev/socket/thermal-recv-client
> first = 0xffffffc708a60068 /dev/socket/thermal-recv-passive-client
> first = 0xffffffc708a67ba8 /dev/socket/thermal-send-rule
> first = 0xffffffc6e3bac468 /dev/socket/ims_datad
> first = 0xffffffc708a66ee8 /dev/socket/ipacm_log_file
> first = 0xffffffc59ddb7ba8 /dev/socket/lmkd
> first = 0xffffffc708a648a8 /dev/socket/ssgqmig
> first = 0xffffffc6dd5a6668 /dev/socket/zygote
> first = 0xffffffc6dd5a6ee8 /dev/socket/usap_pool_primary
> first = 0xffffffc708a65128 /dev/socket/statsdw
> first = 0xffffffc6dd5a2268 /dev/socket/zygote_secondary
> first = 0xffffffc6dd5a37a8 /dev/socket/usap_pool_secondary
> first = 0xffffffc6dd5a48a8 /dev/socket/dnsproxyd
> first = 0xffffffc6dd5a26a8 /dev/socket/mdns
> first = 0xffffffc6dd5a59a8 /dev/socket/fwmarkd
> first = 0xffffffc702bd0068 /dev/socket/qvrservice_vndr
> first = 0xffffffc702bd7ba8 /dev/socket/qvrservice_vndr_camera
> first = 0xffffffc708805de8 /dev/socket/tombstoned_crash
> first = 0xffffffc708807328 /dev/socket/tombstoned_intercept
> first = 0xffffffc708806668 /dev/socket/tombstoned_java_trace
> first = 0xffffffc6e0996228 /dev/socket/mlid
> first = 0xffffffc708a608e8 /dev/socket/pps
> first = 0xffffffc6dd5a1e28 THERMALE_UIs0
> first = 0xffffffc6e0992268 /dev/socket/adbd
> first = 0xffffffc6e3ba84a8 /dev/socket/mdnsd
> first = 0xffffffc70bf4c468 /dev/socket/port-bridge/port_bridge_connect_socket
> first = 0xffffffc6d74ce228 /data/system/unsolzygotesocket
> first = 0xffffffc6e3bad568 jdwp-control
> first = 0xffffffc708803be8 gplisnr



#### 对比正常环境的unix_socket_table

```shell
PID: 1405     TASK: ffffffd55bc3d880  CPU: 6    COMMAND: "cnss-daemon"
FD      SOCKET            SOCK       FAMILY:TYPE SOURCE-PORT DESTINATION-PORT
 4 ffffffd5b787cc00 ffffffd55bfba800 NETLINK/ROUTE:RAW 
 5 ffffffd5b787d500 ffffffd588290440 UNIX:STREAM 
 6 ffffffd5b787e700 ffffffd588292200 UNIX:STREAM 
 7 ffffffd5b787e100 ffffffd588290000 UNIX:DGRAM  0
 8 ffffffd5b787c600 ffffffd55bfbd800 NETLINK/ROUTE:RAW 
 9 ffffffd5b787ea00 ffffffd55bfbe000 NETLINK/ROUTE:RAW 
10 ffffffd565be5200 ffffffd5807ea000 NETLINK/ROUTE:RAW 
11 ffffffd565be7600 ffffffd5807ee800 NETLINK/ROUTE:RAW 
12 ffffffd565be6a00 ffffffd588326a40 UNIX:DGRAM   /data/vendor/wifi/sockets/cnss_user_server
13 ffffffd565be7900 ffffffd5807e8000 NETLINK/ROUTE:RAW 
14 ffffffd570f50600 ffffffd581cf9a00 42:DGRAM  
17 ffffffd541213600 ffffffd58b843a80 42:DGRAM  
20 ffffffd53e1e7900 ffffffd581cfa080 42:DGRAM

sock ffffffd588326a40
first = sock + 0x68 = ffffffd588326aa8

crash> unix_socket_table | grep -v 0x0 | grep first | grep ffffffd588326aa8
    first = 0xffffffd588326aa8
    
ffffffd588326a40 + 0x320 = ffffffd588326d60

crash> rd ffffffd588326d60
ffffffd588326d60:  ffffffd564a38b00                    ...d....
crash> unix_address ffffffd564a38b00
struct unix_address {
  refcnt = {
    refs = {
      counter = 1
    }
  },
  len = 45,
  hash = 256,
  name = 0xffffffd564a38b0c
}
crash> sockaddr_un 0xffffffd564a38b0c
struct sockaddr_un {
  sun_family = 1,
  sun_path = "/data/vendor/wifi/sockets/cnss_user_server\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000"
}
```



#### 反推并没有走到unix_autobind函数，再次审视unix_bind函数

```c
unix_bind
{
	...
	    err = unix_mkname(sunaddr, addr_len, &hash);
        if (err < 0)
                goto out;
        addr_len = err;

        if (sun_path[0]) {
                umode_t mode = S_IFSOCK |
                       (SOCK_INODE(sock)->i_mode & ~current_umask());
                err = unix_mknod(sun_path, mode, &path);
                if (err) {
                        if (err == -EEXIST)
                                err = -EADDRINUSE;
                        goto out;
                }
        }

        err = mutex_lock_interruptible(&u->bindlock);
        if (err)
                goto out_put;

        err = -EINVAL;
        if (u->addr)
                goto out_up;

        err = -ENOMEM;
        addr = kmalloc(sizeof(*addr)+addr_len, GFP_KERNEL);
        if (!addr)
                goto out_up;

        memcpy(addr->name, sunaddr, addr_len);
        addr->len = addr_len;
        addr->hash = hash ^ sk->sk_type;
        refcount_set(&addr->refcnt, 1);

        if (sun_path[0]) {
                addr->hash = UNIX_HASH_SIZE;
                hash = d_backing_inode(path.dentry)->i_ino & (UNIX_HASH_SIZE - 1);
                spin_lock(&unix_table_lock);
                u->path = path;
                list = &unix_socket_table[hash];
        } else {
                spin_lock(&unix_table_lock);
                err = -EADDRINUSE;
                if (__unix_find_socket_byname(net, sunaddr, addr_len,
                                              sk->sk_type, hash)) {
                        unix_release_addr(addr);
                        goto out_unlock;
                }

                list = &unix_socket_table[addr->hash];
        }

        err = 0;
        __unix_remove_socket(sk);
        smp_store_release(&u->addr, addr);
        __unix_insert_socket(list, sk);

out_unlock:
        spin_unlock(&unix_table_lock);
out_up:
        mutex_unlock(&u->bindlock);
out_put:
        if (err)
                path_put(&path);
out:
        return err;
}
```



根据返回值 关注unix_mkname unix_mknod

```c
static int unix_mkname(struct sockaddr_un *sunaddr, int len, unsigned int *hashp)
{
        *hashp = 0;

        if (len <= sizeof(short) || len > sizeof(*sunaddr))
                return -EINVAL;
        if (!sunaddr || sunaddr->sun_family != AF_UNIX)
                return -EINVAL;
        if (sunaddr->sun_path[0]) {
                /*
                 * This may look like an off by one error but it is a bit more
                 * subtle. 108 is the longest valid AF_UNIX path for a binding.
                 * sun_path[108] doesn't as such exist.  However in kernel space
                 * we are guaranteed that it is a valid memory location in our
                 * kernel address buffer.
                 */
                ((char *)sunaddr)[len] = 0;
                len = strlen(sunaddr->sun_path)+1+sizeof(short);
                return len;
        }

        *hashp = unix_hash_fold(csum_partial(sunaddr, len, 0));
        return len;
}
```

unix_mkname没有ENOSPC



```c
static int unix_mknod(const char *sun_path, umode_t mode, struct path *res)
{
        struct dentry *dentry;
        struct path path;
        int err = 0;
        /*
         * Get the parent directory, calculate the hash for last
         * component.
         */
        dentry = kern_path_create(AT_FDCWD, sun_path, &path, 0);
        err = PTR_ERR(dentry);
        if (IS_ERR(dentry))
                return err;

        /*
         * All right, let's create it.
         */
        err = security_path_mknod(&path, dentry, mode, 0);
        if (!err) {
                err = vfs_mknod(d_inode(path.dentry), dentry, mode, 0);
                if (!err) {
                        res->mnt = mntget(path.mnt);
                        res->dentry = dget(dentry);
                }
        }
        done_path_create(&path, dentry);
        return err;
}
```



```c
struct dentry *kern_path_create(int dfd, const char *pathname,
                                struct path *path, unsigned int lookup_flags)
{
        return filename_create(dfd, getname_kernel(pathname),
                                path, lookup_flags);
}
EXPORT_SYMBOL(kern_path_create);
```

filename_create会去创建文件或者目录，会在data分区下面尝试写入，但是由于空间不足导致ENOSPC



#### 一句话结论

cnss等待加载fw超时，超时原因又是cnss daemon创建socket失败创建socket，也需要写入 socket fd



### 那么谁占用了data分区？

```shell
crash> mount | grep data
ffffffc70cd3ad80 ffffffc59e09f800 ext4   /dev/block/by-name/metadata /metadata 
ffffffc70cd3b640 ffffffc59e3c7000 ext4   /dev/block/dm-10 /apex/com.android.tzdata
ffffffc736581180 ffffffc70bdfe000 f2fs   /dev/block/dm-15 /data  
```

得到/data的super_block结构体地址



#### super_block与f2fs_sb_info的联系

```c
static inline struct f2fs_sb_info *F2FS_SB(struct super_block *sb)
{
	return sb->s_fs_info;
}

//偏移0x440
crash> super_block -xo | grep s_fs_info
  [0x440] void *s_fs_info;

s_fs_info: ffffffc70bdfe000 + 0x440 = ffffffc70bdfe440

//s_fs_info是指针，需要rd取内容
crash> rd ffffffc70bdfe440
ffffffc70bdfe440:  ffffffc707cca000                    ........

struct f2fs_sb_info : 0xffffffc707cca000 
```



反推可以证明f2fs_sb_info地址的准确性

```c
struct f2fs_sb_info {
    [0x0] struct super_block *sb;

crash> rd 0xffffffc707cca000
ffffffc707cca000:  ffffffc70bdfe000                    ........

struct super_block : ffffffc70bdfe000
```



#### f2fs superblock结构

![sb_layout](https://github.com/RiweiPan/F2FS-NOTES/raw/master/img/F2FS-Layout/sb_layout2.png)

整个磁盘区域被F2FS设计为6个区域：

分别是Superblock，Checkpoint，Segment Info Table，Node Address Table，Segment Summary Area，以及**Main Area**。

前5个区域总称为元数据区域，保存的是跟F2FS直接相关的元信息，而最后一个区域是**保存文件数据的主要区域**。


block: 4KB对齐且连续的物理存储空间
segment: 2M连续的物理存储空间



```shell
struct f2fs_sb_info {
    [0x0] struct super_block *sb;
    [0x8] struct proc_dir_entry *s_proc;
   [0x10] struct f2fs_super_block *raw_super;
   
crash> rd 0xffffffc707cca010
ffffffc707cca010:  ffffffc707cc9000                    ........

crash> f2fs_super_block  ffffffc707cc9000
struct f2fs_super_block {
  magic = 4076150800,
  major_ver = 1,
  minor_ver = 13,
  log_sectorsize = 12,
  log_sectors_per_block = 0,
  log_blocksize = 12,
  log_blocks_per_seg = 9,
  segs_per_sec = 1,
  secs_per_zone = 1,
  checksum_offset = 0,
  block_count = 26520121,
  section_count = 51573,
  segment_count = 51796,
  segment_count_ckpt = 2,
  segment_count_sit = 4,
  segment_count_nat = 116,
  segment_count_ssa = 101,
  segment_count_main = 51573, //51573*2MB=100GB
  segment0_blkaddr = 512,
  cp_blkaddr = 512,
  sit_blkaddr = 1536,
  nat_blkaddr = 3584,
  ssa_blkaddr = 62976,
  main_blkaddr = 114688,
  root_ino = 3,
  node_ino = 1,
  meta_ino = 2,
  ...
```



#### 疑问点：为啥没有gc出空间

垃圾回收在F2FS中，主要作用是回收无效的block，以供用户重新使用

[F2FS源码分析-4.1 [F2FS GC部分\] 垃圾回收机制源码分析-CSDN博客](https://blog.csdn.net/u011649400/article/details/100530006)



#### 【换个思路】从进程打开的文件角度看

foreach files 命令可以看到所有进程打开的文件路径，dentry，inode地址

过滤/data目录后得到

![image.png](https://cdn.nlark.com/yuque/0/2023/png/38919652/1699512190654-1f0f0643-c723-43d9-9797-769df05bbfff.png?x-oss-process=image%2Fresize%2Cw_750%2Climit_0)

根据inode地址推导出文件i_size大小，单位为字节，最大文件为30M左右

![image.png](https://cdn.nlark.com/yuque/0/2023/png/38919652/1699512198798-efb86cd3-7c50-4b8a-baf3-128d965e4e5e.png)



#### 一句话结论

/data目录下打开的文件属于正常，所以导致data分区满的原因不是系统进程创建的文件，推测是外部导入的



### 如何复现

先使用dd命令把data分区塞满，命令如下：

```
dd if=/dev/zero of=/data/test bs=1g count=100
```

若还剩M级别或者k级别的内存空间，将bs=1g换成1m或者1k来继续填满【记得修改of后面的文件名】

之后就让设备放着跑，过会就复现红灯了



### 参考链接

1、[f2fs文件系统（八）文件布局 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/640917510)

