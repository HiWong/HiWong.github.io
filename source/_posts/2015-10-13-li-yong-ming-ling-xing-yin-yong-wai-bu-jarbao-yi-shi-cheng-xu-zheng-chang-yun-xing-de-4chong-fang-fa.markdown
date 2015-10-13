---
layout: post
title: "利用命令行引用外部jar包以使程序正常运行的4种方法"
date: 2015-10-13 17:37:20 +0800
comments: true
categories: java
---
平时写一些小的Java Demo时我比较喜欢用UltraEdit或记事本写完后，直接利用命令行进行编译和运行。这样的好处就是方便快捷。相信有这个习惯的人应该还大有人在。但是如果要引用外部jar包，应该如何操作呢？在写JDBC的一些Demo时，由于要利用jar包来加载相应的数据库，每个Demo都用到了外部jar包，所以特地总结了一下利用命令行引用外部jar包的方法，归纳起来有以下4种<!--more-->（为方便说明，此处将jar包路径写为jarPath,包含main方法的类名为JdbcSample):   

1.编译时不需要额外处理，只要运行时使用 java -cp .;jarPath JdbcSample即可正常运行；  

2.编译时使用javac -Djava.ext=jarPath JdbcSample.java；这样运行时跟平常一样即可；  

3.设置过环境变量的肯定都知道这种方法，就是在运行时使用java -classpath jarPath;jrePath JdbcSample命令，也就是说把jar包和jre路径包含进去，肯定可以运行；  

4.由于一般来说我们已经设置了环境变量，所以可以采用比第3种方法更简单一点的方法，即先设置好路径：  

	set classPath=%CLASSPATH%;jarPath

然后编译运行即可