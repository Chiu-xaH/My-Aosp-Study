# 系统启动（4）——Launcher

书接上篇[系统启动（3）——SystemSever](/系统启动（3）——SystemSever.md) 

![图片](/Launcher.png)

Step1：XXX

后续启动Launcher应用可以看[startActivty（1）——进程创建与Application初始化](/startActivty（1）——进程创建与Application初始化.md) 

本篇文章较少，来点扩展内容吧，看一下Launcher3是如何让做到应用开闭动效的，这里面借助了与Framework的Surface层通信，在开闭过程暂时由Launcher接管动画并实现，画面是经过AIDL传输到Launcher的。

【待写 需要等[View 与 Surface（2）——图形绘制](/View%20与%20Surface（2）——图形绘制.md) 完成后】