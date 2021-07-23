---
layout: post
author: Joe Ko
title: spring boot security 登入帳密設定
date: 2021-07-23 14:20 +0800
categories:
- spring boot
tags:
- spring boot security
- spring boot actuator
toc:  true
---

spring boot security 登入帳密有四種設定方式

- 使用 spring boot security 預設帳密
- 設定在 memory
- 設定在 application.yml
- 設定在 db

## 使用 spring boot security 預設帳密

只須載入 spring-boot-starter-security 即可，完全不用額外設定
  
{% highlight java linenos %}
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-actuator'
  implementation 'org.springframework.boot:spring-boot-starter-security'
}
{% endhighlight %}

專案啟動時，console 會出現登入密碼，預設登入帳號為 user
![placeholder](https://joeko0221.github.io/images/spring-boot-security-pwd-default.png "預設密碼")


## 設定在 memory
設定在 AuthenticationManagerBuilder 或 UserDetailsService 都可以

### AuthenticationManagerBuilder 
[代碼下載](https://github.com/joeko0221/spring-boot-actuator-memory-AuthenticationManagerBuilder)
{% highlight java linenos %}
@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception {
  auth.inMemoryAuthentication()
      .withUser("admin")
      .password("123")
      .roles("ADMIN")
      .and()
      .withUser("user")
      .password("123")
      .roles("USER");
}    
{% endhighlight %}


### UserDetailsService
[代碼下載](https://github.com/joeko0221/spring-boot-actuator-memory-UserDetailsService)
{% highlight java linenos %}
@Bean
@Override
public UserDetailsService userDetailsService() {

  UserDetails user = User.withDefaultPasswordEncoder()
      .username("user")
      .password("password")
      .roles("ADMIN")
      .build();

  return new InMemoryUserDetailsManager(user);
}
{% endhighlight %}


## 設定在 application.yml
[代碼下載](https://github.com/joeko0221/spring-boot-actuator-yml)
{% highlight yaml linenos %}
spring:
  security:
    user:
      name: admin
#      password: "{noop}12345"
#     bcrypt 密碼: 0000
      password: "{bcrypt}$2a$10$JLBkYF9cXfHhYhjQFz7LbuGxJsAolSchQYS2TaCiwmRcsFgEmWVCq"
      roles: ADMIN,ACTRADMIN
{% endhighlight %}

- {} 裡面放加密方式，{noop} 就是存明碼，其餘加密方式可參考 [DelegatingPasswordEncoder](https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/crypto/password/DelegatingPasswordEncoder.html)
- 0000 用 bcrypt 加密後的結果就是 $2a$10$JLBkYF9cXfHhYhjQFz7LbuGxJsAolSchQYS2TaCiwmRcsFgEmWVCq ([bcrypt 密碼產生器](https://www.browserling.com/tools/bcrypt))
- 相同密碼，用 bcrypt 加密後的結果每次都不同


## 設定在 db
[代碼下載](https://github.com/joeko0221/spring-boot-actuator-db)

SecurityConfig.java
{% highlight java linenos %}
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

  @Autowired
  private AuthService authService;

  @Bean
  public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
  }

  @Override
  @Bean
  public AuthenticationManager authenticationManagerBean() throws Exception {
    return super.authenticationManagerBean();
  }

  @Override
  protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    auth.userDetailsService(authService)
        .passwordEncoder(passwordEncoder());
  }
{% endhighlight %}
- 採用 BCryptPasswordEncoder 加密


AuthService.java
{% highlight java linenos %}
@Service
public class AuthService implements UserDetailsService {

  @Autowired
  private UserInfoRepository userInfoRepository;

  @Override
  public UserDetails loadUserByUsername(String userId) throws UsernameNotFoundException {

    // 查詢 UserInfo
    UserInfo userInfo = userInfoRepository.getByUserId(userId);

    if (userInfo == null) throw new UsernameNotFoundException("email not found");

    return userInfo;
  }
{% endhighlight %}
- Override loadUserByUsername，將 UserInfo 的查詢，改向資料庫

-----
