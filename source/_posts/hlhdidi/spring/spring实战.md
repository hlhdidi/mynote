---
title: spring实战
date: 2017-07-21 15:59:10
tags: spring
---

<!-- MDTOC maxdepth:6 firsth1:1 numbering:0 flatten:0 bullets:1 updateOnSave:1 -->

- [spring实战](#spring实战)
   - [springsecurity](#springsecurity)
<!-- /MDTOC -->

# spring实战

## springsecurity

* pom.xml

```xml
<dependency>
      <groupId>org.springframework.security</groupId>
      <artifactId>spring-security-core</artifactId>
      <version>RELEASE</version>
    </dependency>
    <dependency>
      <groupId>org.springframework.security</groupId>
      <artifactId>spring-security-config</artifactId>
      <version>RELEASE</version>
    </dependency>
    <dependency>
      <groupId>org.springframework.security</groupId>
      <artifactId>spring-security-web</artifactId>
      <version>RELEASE</version>
    </dependency>
    <dependency>
      <groupId>org.springframework.security</groupId>
      <artifactId>spring-security-taglibs</artifactId>
      <version>RELEASE</version>
    </dependency>
    <dependency>
      <groupId>com.alibaba</groupId>
      <artifactId>druid</artifactId>
      <version>1.0.11</version>
    </dependency>
    <dependency>
      <groupId>org.thymeleaf.extras</groupId>
      <artifactId>thymeleaf-extras-springsecurity4</artifactId>
      <version>2.1.2.RELEASE</version>
    </dependency>
```

* 设置filterproxy

  filterproxy是一个特殊的拦截器,用于拦截请求.在springsecurity中,我们需要注册DelegatingFilterProxy.只需要一个继承了
AbstractSecurityWebApplicationInitializer的类即可,spring会自动找到它,并且在web容器注册

```java
/**
 * 开启springsecurity
 */
public class SecurityWebInitializer extends AbstractSecurityWebApplicationInitializer{

}
```

* 编写安全性配置

  我们需要安全性配置来配置对应的安全属性.注意,该配置类应该在继承AbstractSecurityWebApplicationInitializer的类
 的子包下.

 全部的安全配置见下:
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter{

    @Autowired
    DataSource dataSource;
    @Autowired
    UserRepository userRepository;

    @Bean
    public SaltSource saltSource() {
        //设置盐为hlhdidi
        return new SaltSource() {
            @Override
            public Object getSalt(UserDetails userDetails) {
                return "hlhdidi";
            }
        };
    }

    // 设置盐,用户服务service和md5编码
    @Bean
    public AuthenticationProvider authenticationProvider(){
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(new UserInfoService(userRepository));
        provider.setPasswordEncoder(new Md5PasswordEncoder());
        provider.setSaltSource(saltSource());
        return provider;
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.authenticationProvider(authenticationProvider());
    }



    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // 对/user进行认证,对于其他请求允许通过
        http.authorizeRequests().antMatchers("/user").authenticated()
                .antMatchers("/item/**").authenticated()
                .antMatchers("/item/**").hasAuthority("ADMIN") //item/**的请求需要权限验证.
                .anyRequest().permitAll(); // 其他请求允许通过
       // loginPage是调用/login返回的接口,登出成功访问的url为/home,开通rememberMe.
        http.formLogin().loginPage("/login").and().logout().logoutSuccessUrl("/home")
                .and().rememberMe()
                .tokenValiditySeconds((2419200)).key("userKey");//设置token时间和私钥名称.
        // 关闭csrf,否则logout会有问题
        http.csrf().disable();
    }

}
```

* rememberMe

  上面的安全配置开启了rememberMe,这里我们需要的就是提供rememberMe的选项,可以看看login.html页面,它提供了rememberMe的
  checkbox,它在Login.html页面里.

  ```html
  <!--springsecurity会自动识别这个Login..-->
     <form name="f" th:action="@{/login}" method="post">
         username : <input type="text" name="username" /><br/>
         password : <input type="password" name="password"/> <br/>
         <input type="submit" value="LOGIN"/>
         <!--checkbox会被识别-->
         remember me :<input id="remember_me" name="remember-me" type="checkbox"/>
     </form>
  ```

* 自定义的用户服务

    即UserDetailService.如下:

```java
public class UserInfoService  implements UserDetailsService {

    UserRepository userRepository;

    public UserInfoService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username)
            throws UsernameNotFoundException {
        UserInfo user = userRepository.findUserByUsername(username);

        if(user != null) {
            List<GrantedAuthority> authorities = new ArrayList<>();
            // 设置用户的权限.
            authorities.add(new SimpleGrantedAuthority("ROLE_USER"));
            return new User(user.getUsername(),user.getPassword(),authorities);
        }
        // 抛出异常不会打印在日志里面,会调回登录页面
        throw new UsernameNotFoundException("can not found username !");
    }


}
```

* 保护视图

  采用thymeleaf的一些标签可以去保证某些连接不被没权限的用户访问,如下所示:

  首先需要引入thymeleaf对应的springsecurity标签库

```html
<html lang="en" xmlns = "http://www.w3.org/1999/xhtml"
      xmlns:th = "http://www.thymeleaf.org"
      xmlns:sec = "http://www.thymeleaf.org/thymeleaf-extras-springsecurity4">
```
  接着使用标签:
```html
<span sec:authorize-url="/item/detail">
            <td align="center" width="150px"><a th:href="@{'/item/detail/' + ${i.id}}">详情</a></td>
</span>
```

* 配置错误页面

需要注意的是springsecurity如果没有权限访问会跳转到一个默认的错误页面.因此我们需要配置错误页面,只需在web.xml里面
声明即可.

```xml
<error-page>
		<error-code>403</error-code>
		<location>/WEB-INF/templates/error/error_403.html</location>
	</error-page>
```
