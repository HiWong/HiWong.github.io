---
layout: post
title: "ListView.setOnItemClickListener不起作用的原因"
date: 2015-10-13 10:43:55 +0800
comments: true
categories: android
---
ListView.setOnItemClickListener不起作用的原因是item的layout中对以下两个属性设置为true：  

	android:focusable="true"
	android:focusableInTouchMode="true"

将其改为false或者不设置（默认为false)即可：  

	android:focusable="false"
	android:focusableInTouchMode="false"


