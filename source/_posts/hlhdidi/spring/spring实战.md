---
title: spring实战
date: 2017-07-21 15:59:10
tags: spring
---

<!-- MDTOC maxdepth:6 firsth1:1 numbering:0 flatten:0 bullets:1 updateOnSave:1 -->

- [spring实战](#spring实战)
   - [springsecurity](#springsecurity)
   - [在spring中使用JDBC](#在spring中使用JDBC)
   - [关系映射和持久化数据](#关系映射和持久化数据)
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

## 在spring中使用JDBC

我们使用JDBCTemplate去执行对于传统JDBC的简化.配置JDBCTemplate,需要设置数据库连接池.
```java
  @Bean
    public DataSource dataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setUrl("jdbc:mysql://127.0.0.1:3306/test");
        dataSource.setUsername("root");
        dataSource.setPassword("root");
        dataSource.setDriverClassName("com.mysql.jdbc.Driver");
        return dataSource;
    }
    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
```

通过在类中注入JdbcOpertations去实现访问数据库.JdbcOpertations是一个接口,它定义了JdbcTemplate的操作,这样子就完成了松耦合.
```java
    @Autowired
    JdbcOperations jdbcOperations;
```

下面是使用JdbcTemplate完成插入和查询的操作:
```java
@Override
    public UserInfo findUserByUsername(String username) {
        try {
            UserInfo userInfo = jdbcOperations.queryForObject("select * from userinfo where username = '" + username + "'",
                    new RowMapper<UserInfo>() {
                        @Override
                        public UserInfo mapRow(ResultSet resultSet, int i) throws SQLException {
                            UserInfo userInfo = new UserInfo();
                            userInfo.setUsername(resultSet.getString("username"));
                            userInfo.setPassword(resultSet.getString("password"));
                            return userInfo;
                        }
                    });
            return userInfo;
        } catch (Exception e) {
            return null;
        }

    }

    @Override
    public void save(UserInfo userInfo) {
        Md5PasswordEncoder encoder = new Md5PasswordEncoder();
        String saltPassword = encoder.encodePassword(userInfo.getPassword(),"hlhdidi");
        jdbcOperations.update("insert into userinfo (username,password) values(?,?)",
                userInfo.getUsername(), saltPassword);
    }
```

还可以使用Java8的Lambda表达式:
```java
UserInfo userInfo = jdbcOperations.queryForObject("select * from userinfo where username = '" + username + "'",
     (rs,rowNum) -> {
         return new UserInfo(rs.getString("username"),
                 rs.getString("password"));
     });
```

* 使用命名参数

  有时候可以使用命名参数,对于参数都指定一个名字,这样子在绑定值的时候就不用去修改参数了,非常方便.
  使用命名参数的步骤如下:
  声明模板:
```java
@Bean
    public NamedParameterJdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new NamedParameterJdbcTemplate(dataSource);
    }
```
  进行更新和查询
```java
@Override
    public UserInfo findUserByUsername(String username) {
        try {
            Map<String,String> usernameMap = new HashMap<>();
            usernameMap.put("username",username);
            UserInfo userInfo = jdbcOperations.queryForObject("select * from userinfo where username = :username",
                    usernameMap, new RowMapper<UserInfo>() {
                        @Override
                        public UserInfo mapRow(ResultSet resultSet, int i) throws SQLException {
                            UserInfo userInfo1 = new UserInfo();
                            userInfo1.setUsername(resultSet.getString("username"));
                            userInfo1.setPassword(resultSet.getString("password"));
                            return userInfo1;
                        }
                    });
            return userInfo;
        } catch (Exception e) {
            return null;
        }

    }

    @Override
    public void save(UserInfo userInfo) {
        Md5PasswordEncoder encoder = new Md5PasswordEncoder();
        String saltPassword = encoder.encodePassword(userInfo.getPassword(),"hlhdidi");
        Map<String,Object> map = new HashMap<>();
        map.put("username",userInfo.getUsername());
        map.put("password",saltPassword);
        jdbcOperations.update("insert into userinfo (username,password) values(:username,:password)",
                map);
    }
```

## 关系映射和持久化数据

### 在spring中使用hibernate

* 使用pom.xml

  这里注意需要解决jar包的冲突:

```xml
<dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-orm</artifactId>
      <version>4.3.2.RELEASE</version>
      <exclusions>
        <exclusion>
          <groupId>org.jboss.logging</groupId>
          <artifactId>jboss-logging</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
    <dependency>
      <groupId>org.hibernate</groupId>
      <artifactId>hibernate-core</artifactId>
      <version>5.0.2.Final</version>
      <exclusions>
        <exclusion>
          <groupId>org.jboss.logging</groupId>
          <artifactId>jboss-logging</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
    <dependency>
      <groupId>org.hibernate</groupId>
      <artifactId>hibernate-validator</artifactId>
      <version>5.0.2.Final</version>
      <exclusions>
        <exclusion>
          <groupId>javax.validation</groupId>
          <artifactId>validation-api</artifactId>
        </exclusion>
          <exclusion>
            <groupId>org.jboss.logging</groupId>
            <artifactId>jboss-logging</artifactId>
          </exclusion>
      </exclusions>
    </dependency>
    <!--独立处理jboss的冲突-->
    <dependency>
      <groupId>org.jboss.logging</groupId>
      <artifactId>jboss-logging</artifactId>
      <version>3.2.0.Final</version>
    </dependency>
```

* 建立持久化类.

使用到了@Entity和@Table等注解标识持久化类
```java
@Entity
@Table(name = "dog")
public class Dog {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer did;
    @Column
    private String dname;
    @Column
    private Integer dage;

    public Integer getDid() {
        return did;
    }

    public void setDid(Integer did) {
        this.did = did;
    }

    public String getDname() {
        return dname;
    }

    public void setDname(String dname) {
        this.dname = dname;
    }

    public Integer getDage() {
        return dage;
    }

    public void setDage(Integer dage) {
        this.dage = dage;
    }
}
```

* 设置LocalSessionFactoryBean
```java
@Bean(name = "sessionFactoryBean")
    public LocalSessionFactoryBean sessionFactoryBean(DataSource dataSource) {
        LocalSessionFactoryBean bean = new LocalSessionFactoryBean();
        bean.setDataSource(dataSource);
        bean.setPackagesToScan(new String[]{"com.hlhdidi.springaction.five.main.hb.domain"});
        Properties properties = new Properties();
        properties.setProperty("dialect","org.hibernate.dialect.MySQLDialect ");
        bean.setHibernateProperties(properties);
        return bean;
    }
```

* 设置OpenSessionViewFilter.
```java
public class FilterInit implements WebApplicationInitializer{
    @Override
    public void onStartup(ServletContext servletContext) throws ServletException {
        FilterRegistration.Dynamic filter = servletContext.addFilter("openSessionViewFilter", OpenSessionInViewFilter.class);
        filter.addMappingForUrlPatterns(null,false,"/*");
        filter.setInitParameter("singleSession","true");
        filter.setInitParameter("sessionFactoryBeanName","sessionFactoryBean");
    }
}
```

* 建立持久化类完成持久化方法

```java
@Repository
public class HbDogRepository {
    @Autowired private SessionFactory sessionFactory;

    private Session currentSession() {
        return sessionFactory.getCurrentSession();
    }

    public void save(Dog dog) {
        currentSession().save(dog);
    }

    public Dog findDogById(Integer id) {
        return currentSession().get(Dog.class,id);
    }
}
```

### 在spring中使用Jpa

首先声明pom.xml:
```xml
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-jpa</artifactId>
    <version>1.9.1.RELEASE</version>
  </dependency>
    <dependency>
      <groupId>org.hibernate.common</groupId>
      <artifactId>hibernate-commons-annotations</artifactId>
      <version>4.0.4.Final</version>
    </dependency>
    <dependency>
      <groupId>org.hibernate</groupId>
      <artifactId>hibernate-entitymanager</artifactId>
      <version>5.0.2.Final</version>
    </dependency>
```
在SpringRootConfig中,声明如下配置:
```java
    @Bean
    public JpaTransactionManager transactionManager(LocalSessionFactoryBean bean) {
        JpaTransactionManager manager = new JpaTransactionManager();
        manager.setDataSource(dataSource());
        return manager;
    }
    @Bean
    public EntityManager entityManager() {
        return entityManagerFactory(jpaVendorAdapter()).getObject().createEntityManager();
    }

    @Bean
    public JpaVendorAdapter jpaVendorAdapter() {
        HibernateJpaVendorAdapter adapter = new HibernateJpaVendorAdapter();
        adapter.setShowSql(true);
        adapter.setGenerateDdl(true);

        adapter.setDatabasePlatform("org.hibernate.dialect.MySQLDialect");
        return adapter;
    }

    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(JpaVendorAdapter adapter) {
        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(dataSource());
        em.setJpaVendorAdapter(adapter);
        em.setPackagesToScan("com.hlhdidi.springaction.five.main");
        return em;
    }
```

  在spring配置类处声明注解,开启SpringJpa
```java
@EnableTransactionManagement
@EnableJpaRepositories(basePackages = "com.hlhdidi.springaction.five.main.dao.jpa")
```
  紧接着书写实体类:
```java
@Entity
@Table(name = "teacher")
@JsonIgnoreProperties(value={"hibernateLazyInitializer","handler","fieldHandler"})
public class Teacher implements Serializable{
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer tid;
    @Column
    private String tName;

    public Integer getTid() {
        return tid;
    }

    public void setTid(Integer tid) {
        this.tid = tid;
    }

    public String gettName() {
        return tName;
    }

    public void settName(String tName) {
        this.tName = tName;
    }
}
```

  建立紧接着就是建立Repository接口,使用的是继承Spring提供的JpaRepository接口的方法.需要注意,JpaRepository提供了一些基本的方法,我们可以使用,如下所示:
```java
List<T> findAll();

    List<T> findAll(Sort var1);

    List<T> findAll(Iterable<ID> var1);

    <S extends T> List<S> save(Iterable<S> var1);

    void flush();

    <S extends T> S saveAndFlush(S var1);

    void deleteInBatch(Iterable<T> var1);

    void deleteAllInBatch();

    T getOne(ID var1);
```

  但是可以通过断言机制,去让spring根据方法名去推断对应的sql..
  例如:findTeacherByTNameLike,中find是查询动词,Teacher是主题,ByTNameLike则是断言.
  spring支持四种动词,get,read,find,count.其中,get,read,find是一样的意思,而count则是查询数量.
  主题通常不会明确太多的规范,但是当以distinct开头的时候,则spring会保证不会产生重复的记录.
  断言则必须要指定属性,同时后面也可以指定比较操作,例如isAfter,Between等
  当然我们也可以指定SQL,不使用这种断言机制:
```java
@Query("select count(tid) from Teacher ")
    int findCountTeacher();
```
  当上述两种方法都不符合我们的需求的时候,就只能考虑较低的层面了,首先建立一个TeacherRepositoryImpl类,它不需要实现TeacherRepository,因为Spring在TeacherRepository不知道该方法执的SQL的时候,会自动给类名加上impl,试着去扫描路径下面找.
```java
public class TeacherRepositoryImpl {
    @PersistenceContext
    private EntityManager em;

    public int delAllTeacher() {
        return em.createQuery("delete from Teacher").executeUpdate();
    }
}
```
  当然啦,需要在TeacherRepository中添加同样的方法:
```java
public int delAllTeacher();
```
