---
title: anr
date: 2024-03-27 16:26:37
tags:
---
## 分析日志

1.分析main主线程

![.image-20240327163049722](D:\blog\source\_posts\anr\image-20240327163049722.png)

2.存在binder跨进程通信，看binder_info日志

	行 1057:incoming transaction 540396: 00000000f34d0d1d from 1019:1019 to 977:1211 code 23 flags 10 pri 1:98 r1 node 1294 size 140:16 data 000000004f5a8a4a
	行 1117:outgoing transaction 540396: 00000000f34d0d1d from 1019:1019 to 977:1211 code 23 flags 10 pri 1:98 r1

可以看出当前进程1019正在等待977进程的1211线程执行操作



3.查看1211线程

----- pid 977 at 2024-03-18 17:41:15.690162978+0800 -----
Cmd line: /vendor/bin/hw/vendor.qti.hardware.display.composer-service

![.image-20240327163053842](D:\blog\source\_posts\anr\image-20240327163053842.png)

## gdb

### 1.symbol path

LINUX/android/out/target/product/kona/symbols/vendor/bin/hw/vendor.qti.hardware.display.composer-service



### 2.add mutex owner in android bionic libc

```c
diff --git a/core/libutils/include/utils/Mutex.h b/core/libutils/include/utils/Mutex.h
index 1325bf3..4481d82 100644
--- a/core/libutils/include/utils/Mutex.h
+++ b/core/libutils/include/utils/Mutex.h
@@ -160,10 +160,20 @@ class CAPABILITY("mutex") Mutex {
 #if !defined(_WIN32)
 
 inline Mutex::Mutex() {
-    pthread_mutex_init(&mMutex, nullptr);
+    //pthread_mutex_init(&mMutex, nullptr);
+    pthread_mutexattr_t attr;
+    pthread_mutexattr_init(&attr);
+    pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_ERRORCHECK);
+    pthread_mutex_init(&mMutex, &attr );
+    pthread_mutexattr_destroy(&attr);
 }
 inline Mutex::Mutex(__attribute__((unused)) const char* name) {
-    pthread_mutex_init(&mMutex, nullptr);
+    //pthread_mutex_init(&mMutex, nullptr);
+    pthread_mutexattr_t attr;
+    pthread_mutexattr_init(&attr);
+    pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_ERRORCHECK);
+    pthread_mutex_init(&mMutex, &attr );
+    pthread_mutexattr_destroy(&attr);
 }
 inline Mutex::Mutex(int type, __attribute__((unused)) const char* name) {
     if (type == SHARED) {
@@ -173,7 +183,12 @@ inline Mutex::Mutex(int type, __attribute__((unused)) const char* name) {
         pthread_mutex_init(&mMutex, &attr);
         pthread_mutexattr_destroy(&attr);
     } else {
-        pthread_mutex_init(&mMutex, nullptr);
+        //pthread_mutex_init(&mMutex, nullptr);
+       pthread_mutexattr_t attr;
+       pthread_mutexattr_init(&attr);
+       pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_ERRORCHECK);
+       pthread_mutex_init(&mMutex, &attr );
+       pthread_mutexattr_destroy(&attr);
     }
 }
 inline Mutex::~Mutex() {
```



### 3.compile libutils and get libutils.so

> 1.cd to LINUX/android and run source build/envsetup.sh
>
> 2.lunch and choose qssi-userdebug
>
> 3.cd to system/core/libutils and run mm

libutils.so file path is follows:

out/target/product/qssi/system/lib64/libutils.so



### 4.start gdbserver and bind to the display process

```
adb shell gdbserver :8888 --attach pid
```



### 5.map the pc port to specified port in android system

```
adb forward tcp:8888 tcp:8888
```



### 6.start gdb client in wins

```
prebuilts/gdb/linux-x86/bin/gdb.exe out/target/product/kona/symbols/vendor/bin/hw/vendor.qti.hardware.display.composer-service
```



### 7.set symbols path

```
(gdb) set solib-absolute-prefix out/target/product/kona/symbols/vendor/bin/hw/vendor.qti.hardware.display.composer-service
(gdb) set solib-search-path out/target/product/kona/symbols/vendor/bin/hw/vendor.qti.hardware.display.composer-service
```



### 8.connect gdb to gdbserver

```
(gdb)target remote :8888
```



## perfetto

### 1.record trace

adb shell perfetto -o /data/misc/perfetto-traces/trace_file.perfetto-trace -t 20s sched freq idle am wm view binder_driver hal dalvik input res memory gfx

adb pull /data/misc/perfetto-traces/trace_file.perfetto-trace



2.



参考链接：

1.[native 解析死锁方法___futex_wait_ex-CSDN博客](https://blog.csdn.net/aa787282301/article/details/104693152)

2.[binder ANR案例-CSDN博客](https://blog.csdn.net/liu362732346/article/details/80406772)

