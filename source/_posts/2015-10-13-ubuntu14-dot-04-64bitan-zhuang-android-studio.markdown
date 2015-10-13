---
layout: post
title: "Ubuntu14.04 64bit安装Android Studio"
date: 2015-10-13 11:18:13 +0800
comments: true
categories: Linux
---
Google推出的Android Studio目前除了在NDK和插件方面相比Eclipse有所欠缺之外，其他各方面都已经全面超越Eclipse了，作为一名Android开发者，尝试使用更好的IDE，或许能达到事半功倍的效果。好了，废话不多说，下面进入正题。  

我之前使用过Ubuntu12.04 32bit和Ubuntu14.04 32bit时安装Android Studio都很顺利，但是在使用<!--more-->Ubuntu14.04 64bit时却出现了很多问题，所以本文重点讲Ubuntu14.04 64bit下安装Android Studio的过程。  

在安装Android Studio之前要做好两个准备工作。  

第一个准备工作就是安装JDK1.7,注意不要安装JDK1.8，因为JDK1.8出的时间没多久，在很多在Android Developer网站有明确的提示。  

过程如下：  

1.进入Oracel官网下载jdk，建议下载JDK7u75;  

2.下载完之后解压：  

	sudo tarzxvf jdk*.tar.gz

然后将解压到的文件复制到/usr/lib/jvm下(jvm这个目录是需要自己新建的，当然，也可以用别的名称，比如jdk,java之类）;当然也可以直接解压到/usr/lib/jvm下；  

3.进入目录  

	cd /usr/lib/jvm

4.由于jdk的名称太长，建议修改名称或者建立软链接以方便后面建立环境变量时的书写：sudo ln -s jdk1.7.0_75 jdk7

5.配置环境变量，这里可以配置bashrc文件，也可以配置/etc/profile，但是配置/etc/profile更好，因为bashrc实际上是针对bash shell的，如果切换到其他的shell，则不起作用了，而/etc/profile则是真正的全局环境变量。命令如下：  

(1)  

	sudo vim /etc/profile

(2)  

	GG 

（这个命令可直接到文件末尾)  

(3)在文件末尾加上以下  

   export JAVA_HOME=/usr/lib/jvm/jdk7
   export JRE_HOME=${JAVA_HOME}/jre
   export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
   export PATH=${JAVA_HOME}/bin:$PATH

(4)保存并退出  

	wq
6.由于在ubuntu中可能会有默认的JDK，如openjdk,所以为了将我们安装的JDK设置为默认的JDK版本，还要进行如下工作  

	sudo update-alternatives --install /usr/bin/java java /usr/lib/jvm/jdk7/bin/java 300
	sudo update-alternatives --install /usr/bin/javac javac /usr/lib/jvm/jdk7/bin/javac 300
	sudo update-alternatives --install /usr/bin/jar jar /usr/lib/jvm/jdk7/bin/jar 300
	sudo update-alternatives --install /usr/bin/javah javah /usr/lib/jvm/jdk7/bin/javah 300
	sudo update-alternatives --install /usr/bin/javap javap /usr/lib/jvm/jdk7/bin/javap 300

7.

	sudo update-alternatives --config java

8.输入java -version,如果可看到版本描述，则说明jdk安装成功了。  

上面只是说了第一个准备工作，真正关键的是第二个准备工作。  

由于Ubuntu 14.04 64bit上缺少一些库文件，因而要先安装好这些库文件，执行如下命令  

	sudo apt-get install libc6:i386 libncurses5:i386 libstdc++6:i386

9.进入Android Developer网站下载Android Studio for linux，下载后解压到指定的目录，然后进入android-studio/bin中执行studio.sh文件，就会出现开始页面了，后面按它的提示下载相应的组件即可。因为过程很简单，此处不再赘述。  

