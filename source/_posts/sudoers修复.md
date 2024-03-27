---
title: sudoers修复
tags:
- problem
- linux
---
### 背景

![img](https://static.dingtalk.com/media/lQLPJxgQw9cSyHzMn80CcrBbOT-LeJvcbQUfeuB7FicA_626_159.png)

为了修改User权限，手动用vim 修改了/etc/sudoers文件，结果导致报错



### 解决方法

1、打开两个ssh终端，都是用同一个ubuntu用户登录

2、在第一个终端输入以下命令，获取pid

```shell
echo $$
```

3、在第二个终端，输入：

```shell
pkttyagent --process 刚刚得到的pid
```

4、这个时候，第二个终端会卡住，在第一个终端输入：

```shell
pkexec visudo
```

5、然后，第二个终端也卡主，回到第一个终端，会提示输入当前用户密码，输入

6、好吧，输入完密码，第一个终端卡主了，回到第二个终端，会发现，出现了sudoers的内容，编辑出错的地方，保存即可。

7、完成任务，修改完成，发现就可以继续使用sudo命令了，over

PS：这里用的编辑器是nano，以下是nano简单的保存方式：

linux下在编辑状态下退出请按Ctrl+X，会有两种情形：
①、如果文件未修改，直接退出；
②、如果修改了文件，下面会询问是否需要保存修改。输入Y确认保存，输入N不保存，按Ctrl+C取消返回。如果输入了Y，下一步会提示输入想要保存的文件名。如果不需要修改文件名直接回车就行；若想要保存成别的名字（也就是另存为）则输入新名称然后确定，这个时候也可用Ctrl+C来取消返回



### 参考链接

[错误：/etc/sudoers: syntax error near line-CSDN博客](https://blog.csdn.net/li_ellin/article/details/108666014)