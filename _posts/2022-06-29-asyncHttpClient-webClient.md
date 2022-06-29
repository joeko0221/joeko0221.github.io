---
layout: post
author: Joe Ko
title: AsyncHttpClient, webClient
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

## Future (Spring MVC)

{% highlight java linenos %}

Future<HsrHolidayOrderInfo> future = threadPoolTaskExecutor.submit(new Callable<HsrHolidayOrderInfo>() {

    @Override
    public HsrHolidayOrderInfo call() throws Exception {
      ...
    }
});

HsrHolidayOrderInfo result = future.get();   
          
{% endhighlight %}


-----
