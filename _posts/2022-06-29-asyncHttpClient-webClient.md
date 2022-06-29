---
layout: post
author: Joe Ko
title: AsyncHttpClient, WebClient
date: 2022-06-29 14:20 +0800
categories:
- spring boot
tags:
- webFlux
toc:  true
---

本文將介紹 Spring MVC 中 Thread, Future, AsyncHttpClient，在 Spring Boot 對應的寫法。

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


## AsyncHttpClient (Spring MVC)

{% highlight java linenos %}

private CompletableFuture<Intro> getIntroFuture(String prodNo, String deptDt) {

  List<Param> paramList = new ArrayList<Param>();
  paramList.add(new Param("deptDt", deptDt));

  return asyncHttpClient
      .prepareGet("http://3w.eztravel.com.tw:18090/thread-provider/provider/get/{prodNo}"
          .replace("{prodNo}", prodNo))
      .addQueryParams(paramList)
      .execute(new AsyncCompletionHandler<Intro>() {

        @Override
        public Intro onCompleted(Response response) throws Exception {

          String body = response.getResponseBody();
          Intro intro = objectMapper.readValue(body, new TypeReference<Intro>() {});

          return intro;
        }
      })
      .toCompletableFuture();
}
  
HsrHolidayOrderInfo result = future.get();   
          
{% endhighlight %}



## Thread (Spring Boot)

{% highlight java linenos %}

@Async
public void ezOrdError(OrderQueryRes req, String otsOrderNo);   
              
{% endhighlight %}

## Future (Spring Boot)

{% highlight java linenos %}

@Async
public CompletableFuture<Intro> getIntroFuture(String prodNo, String deptDt) {

  try {
    log.info("start prodNo: {}, deptDt: {}", prodNo, deptDt);

    if ("GRT0000004724".equals(prodNo)) TimeUnit.SECONDS.sleep(10);

    log.info("end prodNo: {}, deptDt: {}", prodNo, deptDt);

  } catch (InterruptedException e) {
    e.printStackTrace();
  }

  return CompletableFuture.completedFuture(Intro.builder()
      .prodNo(prodNo)
      .deptDt(deptDt)
      .prodNm("清靜農場二日遊")
      .upYn("Y")
      .build());
}        
          
{% endhighlight %}


## WebClient (Spring Boot)

{% highlight java linenos %}

public Mono<ServerResponse> webClient(ServerRequest request) {

  String prodNo = request.pathVariable("prodNo");

  WebClient webClient = WebClient.create("http://3w.eztravel.com.tw:18090/thread-provider/");

  // 查詢指定使用者
  Mono<Intro> introMono = webClient.get()
      .uri("provider/get/{prodNo}?deptDt=20220101", prodNo)
      .accept(MediaType.APPLICATION_JSON)
      .retrieve()
      .bodyToMono(Intro.class)
      .map(intro -> intro.setProdNm(intro.getProdNm() + ", success"));

  return ServerResponse.ok()
      .contentType(MediaType.APPLICATION_JSON)
      .body(introMono, Intro.class);
}
            
{% endhighlight %}









-----
