---
layout: post
title: "Android中利用命令行进行截屏并导出到电脑上"
date: 2015-10-13 18:53:51 +0800
comments: true
categories: android
---
大多数人最常用的截屏方法可能就是利用手机的快捷按键了，但是那样如果要导入到电脑中效率会比较低。实际上有更好的截屏方式，最简单的当然就是利用Eclipse中的DDMS进行截屏了，点击“Screen Capture"按钮后等待10多秒，然后就可直接利用Save按钮保存到电脑中。  

显然，由于要进行图片显示的原因，在DDMS中会有一定的延迟<!--more-->，效率还不够高。其实效率最高的方式就是利用命令行来截屏了。用于截屏的shell命令及相关参数的含义为：  

	screencap [-hp] [-d display-id] [FILENAME]

	-h:this message(本条信息)

    -p:save the file as a png.(将文件保存为png格式）

    -d:specify the display id to capture,default 0.(为本次截屏指定显示编号，默认为0)

    If FILENAME ends with .png it will be saved as a png.(如果文件名以.png结尾，它会被保存为png图片)

    If FILENAME is not given,the results will be printed to stdout.(如果没有指定文件名(其实是完整的文件路径),那么结果会打印到标准输出中。实际上就是会将图片信息打印到屏幕上，当然是一片乱码。所以最好指定文件名。)

一般来说，-h,-d这两个参数对我们作用不大，-p用到的地方多一些，但是我不建议用-p，原因如下：  

比如我们用这么一个命令截图：screencap -p /mnt/sdcard/Pic01，截取的这个图形文件名就是Pic01而不是Pic01.png，这样导出时的命令就变成了adb pull /mnt/sdcard/Pic01 d:/，其中d:/是我们要导出到电脑上的路径，这样我们还要给它添加上后缀。  

虽然也可以用screencap -p /mnt/sdcard/Pic01.png的命令，但是显然没有screencap /mnt/sdcard/Pic01.png及screencap /mnt/sdcard/Pic01.jpg这样的命令方便。  

另外有几个值得注意的地方是：第一，如果想将截图放在sdcard中，不一定就是我这样的路径(/mnt/sdcard/)，因为这跟底软的实现有关，最好就是到DDMS确认一下；第二，从电脑push APK到手机中是要先remount的，但是从手机中pull文件到电脑上是不需要先remount的；第三，screenshot命令是不能截屏的，我尝试过，导出到电脑上发现是很杂乱很奇怪的图形，有兴趣的童鞋可以验证一下。  

上面所有的命令都是我亲自验证的，还有问题的小伙伴就到下面留言吧!  