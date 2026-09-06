---
comments: true
layout: post
title: unmappable character (0xE3) for encoding x-windows-949 오류
date: 2026-07-11 18:56:12 +0900
category: error
---

인텔리제이 사용 중 한글 인코딩이 안되는 오류가 있었다.

error: unmappable character (0xE3) for encoding x-windows-949

ctrt + alt + s 로 설정을 연 후 Build Tools에서 Gradle 설정 중 'Build and run using', 'Run tests using'을 'IntelliJ IDEA'로 수정해주었더니 해결되었다.

![](https://velog.velcdn.com/images/y00913/post/3bb52f57-e7d3-41f9-a4ef-f33f38f520d7/image.png)
