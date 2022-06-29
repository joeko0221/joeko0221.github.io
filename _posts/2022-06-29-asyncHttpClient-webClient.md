---
layout: post
author: Joe Ko
title: k8s 安裝 (docker, containerd, cri-o)
date: 2022-06-29 14:20 +0800
categories:
- spring boot
tags:
- webFlux
toc:  true
---

本文將介紹 Spring MVC 中的 Thread, Future, AsyncHttpClient，在 Spring Boot 對應的寫法。

## Thread (Spring MVC)

{% highlight java linenos %}
  @Autowired
  private ThreadPoolTaskExecutor threadPoolTaskExecutor;
  
  threadPoolTaskExecutor.execute(() -> your method);
  
{% endhighlight %}


-----
