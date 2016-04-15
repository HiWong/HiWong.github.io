---
layout: post
title: "代码高亮要注意的一个细节"
date: 2016-04-16 01:23:16 +0800
comments: true
categories: octopress
---

今天在修改完一篇Blog之后，执行 $ rake generate 后去出现如下错误：  

jekyll 2.5.3 | Error: undefined method '[]' for nil:NilClass

搜索出的结果很多，有人说要将_config.yml中的 highlighter: pygments 修改为 highlighter: true ，比如下面这个链接： 

https://github.com/jekyll/jekyll/issues/2678

但是修改之后发现还是会出现错误<!--more-->。

后来是在 https://www.bountysource.com/issues/1422644-plugins-pygments_code-rb-14-in-highlight-undefined-method-for-nil-nilclass-nomethoderror 这个链接下的一个答案中受到了启发，那个答案内容如下：  

	kingiol commented on this issue 2 years ago.
	hello, I also get this error.
	then I find out code format is not right. in ``` tag.

	before:

	view.autoresizingMask = UIViewAutoresizingFlexibleHeight | UIViewAutoresizingFlexibleWidth;
	after:

	view.autoresizingMask = UIViewAutoresizingFlexibleHeight|UIViewAutoresizingFlexibleWidth;
	then the error info disappear, I also don't know why

这时我才意识到，原来在build时会根据代码高亮所指定的语言对代码进行检查和分析，如果出现语法错误，就会导致build出错。仔细看了一下，原来自己是下面这段的高亮部分写错了：  

	service media /system/bin/mediaserver
	user media
	group system audio camera graphics inet net_bt net_bt_admin

它虽然是要在bash中执行的脚本，但是不能将语言指定为 bash,而应该是 sh,即Linux下的可执行文件。修改之后就正常了。  

所以要代码高亮的小伙伴们，代码高亮的语言不能随意指定，否则build会不通过。  

