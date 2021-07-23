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

spring boot security 登入加密有四種設定方式

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
### AuthenticationManagerBuilder

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

    /spring-boot-actuator-yml/src/main/resources/application.yml

spring:
  security:
    user:
      name: admin
#      password: "{noop}12345"
#     bcrypt 密碼: 0000
      password: "{bcrypt}$2a$10$JLBkYF9cXfHhYhjQFz7LbuGxJsAolSchQYS2TaCiwmRcsFgEmWVCq"
      roles: ADMIN,ACTRADMIN

    1. {} 裡面放加密方式，{noop} 代表明碼

    2. jasypt 與 bcrypt 不要搞混，見 spring boot - 資料庫

    5. bcrypt 密碼產生器

        https://www.browserling.com/tools/bcrypt

        也可用 java 產生，見 spring boot - security 實做

    6. CL 範例

        https://bitbucket.org/ezecteam/ez-member-gs/src/master/


5. 帳密設定在 db

    /spring-boot-actuator-db/src/main/java/me/joe/conf/SecurityConfig.java

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


        /spring-boot-actuator-db/src/main/java/me/joe/conf/service/AuthService.java

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


    1. 登入失敗

        1. 用 debug 才能看到錯誤訊息

            user account is locked spring security

        2. 解決方式教學

            https://blog.csdn.net/chaojibt888/article/details/84260035

            是因为用户实体类实现UserDetails这个接口时，我默认把所有抽象方法给自动实现了，而自动生成下面这四个方法，默认返回false，

                @Override
                public boolean isAccountNonExpired() {
                    return false;
                }
             
                @Override
                public boolean isAccountNonLocked() {
                    return false;
                }
             
                @Override
                public boolean isCredentialsNonExpired() {
                    return false;
                }
             
                @Override
                public boolean isEnabled() {
                    return false;
                }            

        3. 解決方式實做

            /spring-boot-actuator-db/src/main/java/me/joe/conf/entity/UserInfo.java

            要參考

            /rest-ots-server/src/main/java/eztravel/modal/UserInfo.java

            原本自動產生的 @Override method 要修改

    2. 設定角色

    /spring-boot-actuator-db/src/main/java/me/joe/conf/SecurityConfig.java

      @Override
      protected void configure(HttpSecurity http) throws Exception {

        http.formLogin()
            .and()
            .csrf()
            .and()
            .authorizeRequests()
            // 無限制
            .antMatchers(HttpMethod.GET, "/actuator")
            .permitAll()
            // 限 ADMIN
            .antMatchers(HttpMethod.GET, "/actuator/health")
            .hasRole("ADMIN")
            // 限 TEST
            .antMatchers(HttpMethod.GET, "/actuator/env")
            .hasRole("TEST")
            .anyRequest()
            .authenticated();
      }

          /spring-boot-actuator-db/src/main/java/me/joe/conf/entity/UserInfo.java

            @Override
            public Collection<? extends GrantedAuthority> getAuthorities() {

              List<SimpleGrantedAuthority> authorities = new ArrayList<>();
              // authorities.add(new SimpleGrantedAuthority("ROLE_user"));

              // 這樣不行，一定要 ROLE_ 開頭
              // authorities.add(new SimpleGrantedAuthority(this.role));

              authorities.add(new SimpleGrantedAuthority("ROLE_" + this.role));

              return authorities;
            }        
-----
