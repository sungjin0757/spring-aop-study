## AOP - Aspect Oriented Programming
***

### ๐ ์ ์  ์๋น์ค๊ธฐ๋ฅ์ Transaction ๊ฒฝ๊ณ ์ค์  ๊ธฐ๋ฅ์ ๋ถ์ฌํ๋ฉด์, ๋จ๊ณ๋ณ๋ก AOP์ ๋ฑ์ฅ ๋ฐฐ๊ฒฝ๊ณผ ์ฅ์ ์ ์์๋ด์๋ค!

**๐ ๋จผ์ , ๋ค์์ ์ฝ๋๋ฅผ ๊ธฐ๋ฐ์ผ๋ก ์ ์ฐจ์ ์ผ๋ก ๋ฐ์ ์์ผ ๋๊ฐ๋ณด๊ฒ ์ต๋๋ค.** 

**User.Java(Domain)**
```java
@Getter
@NoArgsConstructor(access= AccessLevel.PROTECTED)
public class User {

    private String id;
    private String name;
    private String password;

    private Level level;
    private int login;
    private int recommend;
    private String email;
    private LocalDateTime createdAt;
    private LocalDateTime lastUpgraded;

    public void updateLevel(Level level){
        this.level=level;
    }

    public void updateLogin(int login){
        this.login=login;
    }

    public void updateRecommend(int recommend){
        this.recommend=recommend;
    }

    public void upgradeLevel(){
        Level next=this.level.getNext();
        if(next==null){
            throw new IllegalStateException(this.level+"์ ํ์ฌ ์๊ทธ๋ ์ด๋๊ฐ ๋ถ๊ฐ๋ฅํฉ๋๋ค.");
        }
        updateLevel(next);

        this.lastUpgraded=LocalDateTime.now();
    }

    @Builder(builderMethodName = "createUser")
    public User(String id, String name, String password,Level level,int login,int recommend, String email
            ,LocalDateTime createdAt,LocalDateTime lastUpgraded){
        this.id=id;
        this.name=name;
        this.password=password;
        this.level=level;
        this.login=login;
        this.recommend=recommend;
        this.email=email;
        this.createdAt=createdAt;
        this.lastUpgraded=lastUpgraded;
    }
}

```

**UserDao.java**
```java
public interface UserDao {
    void add(User user);
    Optional<User> get(String id);
    List<User> getAll();
    void deleteAll();
    int getCount();
    void update(User user);
}
```

**UserDaoImpl.java**
```java
@RequiredArgsConstructor
public class UserDaoImpl implements UserDao {

    private final JdbcOperations jdbcOperations;

    private RowMapper<User> userMapper=new RowMapper<User>() {
        @Override
        public User mapRow(ResultSet rs, int rowNum) throws SQLException {
            return User.createUser()
                    .id(rs.getString("id"))
                    .password(rs.getString("password"))
                    .name(rs.getString("name"))
                    .level(Level.valueOf(rs.getInt("level")))
                    .login(rs.getInt("login"))
                    .recommend(rs.getInt("recommend"))
                    .email(rs.getString("email"))
                    .createdAt(rs.getTimestamp("createdAt").toLocalDateTime())
                    .lastUpgraded(rs.getTimestamp("lastUpgraded").toLocalDateTime())
                    .build();
        }
    };

    @Override
    public void add(User user) {
        jdbcOperations.update("insert into users(id,name,password,level,login,recommend,email,createdAt,lastUpgraded) " +
                        "values(?,?,?,?,?,?,?,?,?)",
                user.getId(),user.getName(),user.getPassword()
                ,user.getLevel().getValue(),user.getLogin(),user.getRecommend(), user.getEmail()
                , Timestamp.valueOf(user.getCreatedAt()),Timestamp.valueOf(user.getLastUpgraded()));
    }

    @Override
    public Optional<User> get(String id) {
        return Optional.ofNullable(
                jdbcOperations.queryForObject("select * from users u where u.id=?", userMapper,new Object[]{id}));
    }

    @Override
    public List<User> getAll() {
        return jdbcOperations.query("select * from users u order by id", userMapper);
    }

    @Override
    public void deleteAll() {
        jdbcOperations.update("delete from users");
    }

    @Override
    public int getCount() {
        return jdbcOperations.queryForObject("select count(*) from users",Integer.class);
    }

    @Override
    public void update(User user) {
        jdbcOperations.update("update users set name=?,password=?,level=?,login=?,recommend=?,email=?, createdAt=?," +
                        "lastUpgraded=? where id=?",
                user.getName(), user.getPassword(),user.getLevel().getValue()
                ,user.getLogin(),user.getRecommend(),user.getEmail(),user.getCreatedAt(),user.getLastUpgraded(),user.getId());
    }
}

```

**UserService.java**
```java
public interface UserService {
    void add(User user);
    
    void upgradeLevels();

    User get(String id);
    List<User> getAll();
    int getCount();
    
    void deleteAll();
    
    void update(User user);
}

```

**UserServiceImpl.java**
```java
package study.aop.service;

import lombok.RequiredArgsConstructor;
import org.springframework.mail.MailSender;
import org.springframework.mail.SimpleMailMessage;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.TransactionStatus;
import org.springframework.transaction.support.DefaultTransactionDefinition;
import study.aop.domain.Level;
import study.aop.dao.UserDao;
import study.aop.domain.User;

import java.util.List;

@RequiredArgsConstructor
public class UserServiceImpl implements UserService {

    private final UserDao userDao;
    private final PlatformTransactionManager transactionManager;
    private final MailSender mailSender;

    public static final int LOG_COUNT_FOR_SILVER=50;
    public static final int REC_COUNT_FOR_GOLD=30;

    public void upgradeLevels(){
        TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());

        try{
            upgradeLevelsInternal();
            transactionManager.commit(status);
        }catch(Exception e){
            transactionManager.rollback(status);
            throw e;
        }
    }

    public void add(User user){
        if(user.getLevel()==null)
            user.updateLevel(Level.BASIC);
        userDao.add(user);
    }

    protected void upgradeLevel(User user){
        user.upgradeLevel();
        userDao.update(user);
        sendUpgradeMail(user);
    }

    @Override
    public User get(String id) {
        return userDao.get(id).orElseThrow(()->{
            throw new RuntimeException();
        });
    }

    @Override
    public List<User> getAll() {
        return userDao.getAll();
    }

    @Override
    public int getCount() {
        return userDao.getCount();
    }

    @Override
    public void deleteAll() {
        userDao.deleteAll();
    }

    @Override
    public void update(User user) {
        userDao.update(user);
    }
    

    private void sendUpgradeMail(User user){
        SimpleMailMessage mailMessage=new SimpleMailMessage();
        mailMessage.setTo(user.getEmail());
        mailMessage.setFrom("admin");
        mailMessage.setSubject("Upgrade ์๋ด");
        mailMessage.setText("์ฌ์ฉ์๋์ ๋ฑ๊ธ์ด "+user.getLevel()+"๋ก ์๊ทธ๋ ์ด๋ ๋ ์์ต๋๋ค.");

        mailSender.send(mailMessage);
    }

    private boolean checkUpgrade(User user){
        Level currentLevel=user.getLevel();

        if(currentLevel==Level.BASIC)
            return (user.getLogin()>=LOG_COUNT_FOR_SILVER);
        else if(currentLevel==Level.SILVER)
            return (user.getRecommend()>=REC_COUNT_FOR_GOLD);
        else if(currentLevel==Level.GOLD)
            return false;
        throw new IllegalArgumentException("Unknown Level : "+currentLevel);
    }


}

```

**AppConfig.java**
```java
@Configuration
@RequiredArgsConstructor
public class AppConfig {

    private final Environment env;

    @Bean
    public DataSource dataSource(){
        SimpleDriverDataSource dataSource=new SimpleDriverDataSource();
        dataSource.setDriverClass(org.h2.Driver.class);
        dataSource.setUrl(env.getProperty("spring.datasource.url"));
        dataSource.setUsername(env.getProperty("spring.datasource.username"));
        dataSource.setPassword(env.getProperty("spring.datasource.password"));

        return dataSource;
    }

    @Bean
    public PlatformTransactionManager transactionManager(){
        return new DataSourceTransactionManager(dataSource());
    }

    @Bean
    public JdbcOperations jdbcOperations(){
        return new JdbcTemplate(dataSource());
    }

    @Bean
    public UserDao userDao(){
        return new UserDaoImpl(jdbcOperations());
    }

    @Bean
    public MailSender mailSender(){
        return new DummyMailSender();
    }

    @Bean
    public UserService userService(){
        return new UserServiceImpl(userDao(),mailSender());
    }
}
```

***
### ๐ Transaction ์ฝ๋ ๋ถ๋ฆฌ

**๋จผ์ , UserServiceImpl์ ์ฝ๋์ UpgradeLevels ๋ฉ์๋๋ฅผ ์ดํด๋ด์๋ค.**
```java
public void upgradeLevels(){
        TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());

        try{
            upgradeLevelsInternal();
            transactionManager.commit(status);
        }catch(Exception e){
            transactionManager.rollback(status);
            throw e;
        }
    }
```

์ง๊ธ ์ด ๋ฉ์๋๋ Transaction ๊ฒฝ๊ณ์ค์  ์ฝ๋์ ๋น์ฆ๋์ค ๋ก์ง ์ฝ๋๊ฐ ์๋ก ๊ณต์กดํ๋ ๊ฒ์ ๋ณผ ์ ์์ต๋๋ค.

๊ทธ๋ ๋ค๊ณ  ํ๋๋ผ๋ ์ฝ๋๋ฅผ ์ ์ดํด๋ณด์๋ฉด ์๋ก๊ฐ์ ์ฐ๊ด๊ด๊ณ๊ฐ ๊ฑฐ์ ์์ผ๋ฉฐ, ๋๋ ท์ด ๊ฐ์์ ๊ธฐ๋ฅ๋ค์ ํ๊ณ  ์๋ค๋ ๊ฒ์ ์์๋ณผ ์ ์์ต๋๋ค.
1. Transaction ๊ฒฝ๊ณ์ค์  ์ฝ๋๋ ์์๊ณผ ์ข๋ฃ๋ง์ ๋ด๋น.
2. ์คํ๋ง์ด ์ ๊ณตํ๋ Transaction (์๋น์ค ์ถ์ํ๊ฐ ๋์ด์์!)๊ณผ ๊ณ์ธต ๋ถ๋ฆฌ๊ฐ ์๋์ด์ง `Repository` ์ `Service`๋ฅผ ์ฌ์ฉํ๊ธฐ ๋๋ฌธ์ ๋น์ฆ๋์ค ๋ก์ง ์ฝ๋๋
์ง์  DB๋ฅผ ๋ค๋ฃจ์ง ์์ ๋ฟ๋๋ฌ Transaction ๊ฒฝ๊ณ๊ธฐ๋ฅ์ ํ๋ ์ฝ๋์ ์ ๋ณด๋ฅผ ์ฃผ๊ณ  ๋ฐ์ง ์์ต๋๋ค.

**๊ฒฐ๋ก ์ ์ผ๋ก, ์ด ๋๊ฐ์ง ๊ธฐ๋ฅ์ ์ฝ๋๋ ์๋ก ์ฑ๊ฒฉ์ด ๋ค๋ฅด๋ค๋ ๊ฒ์ ๋ณผ ์ ์์ต๋๋ค!**

**๊ทธ๋ ๋ค๋ฉด, ํ์ฌ๋ก์๋ Refactoring์ ํตํ์ฌ ์๋ก๋ฅผ ๋ถ๋ฆฌ์์ผ ๋๋ ๊ฒ์ด ์ต๊ณ ์ ๋ฐฉ๋ฒ์ด ๋  ๊ฒ์๋๋ค! ๐**

#### โ๏ธ ๊ธฐ๋ฅ ๋ถ๋ฆฌ - Refactoring
๋น์ฆ๋์ค ๋ก์ง ์ฝ๋์ Transaction ๊ฒฝ๊ณ ์ค์  ์ฝ๋๋ฅผ ๋ถ๋ฆฌํ๊ธฐ ์ํด ๊ฐ์ฅ ๋จผ์  ์๊ฐํด๋ณผ ๋ฐฉ๋ฒ์ ๋ค์ ๋ ๊ฐ์ง ์๋๋ค.
1. ๋ฉ์๋๋ฅผ ์ด์ฉํ ๋ถ๋ฆฌ
2. DI๋ฅผ ์ด์ฉํ ๋ถ๋ฆฌ 

๋ค์ ํ๋ฒ ์๊ฐํด๋ณด๊ฒ ์ต๋๋ค. ์ด๋ป๊ฒ ํ๋ฉด ๋์ฑ ๊น๋ํ ๊น?!

`์ด์ฐจํผ ์๋ก ์ฃผ๊ณ ๋ฐ์ ์ ๋ณด๊ฐ ์๋ค๋ฉด, ํธ๋์ญ์ ์ฝ๋๋ฅผ ์์จ ์ ์์ผ๋ ๋น์ฆ๋์ค ๋ก์ง์ ํธ๋์ญ์ ์ฝ๋๊ฐ ์์ ์๋ณด์ด๋ ๊ฒ์ฒ๋ผ ์ฌ์ฉํ์ฌ๋ณด์!`

์ด๋ฐ ๋ฐฉ์์ ํด๊ฒฐ๋ฒ์ ์๊ฐํด๋ณด๋ฉด, ๋ฉ์๋๋ฅผ ์ด์ฉํ ๋ถ๋ฆฌ๋ ์ฒดํ๋  ์ ์์ต๋๋ค. ์๋ํ๋ฉด, ๋ ๊ฐ๋ฐํ ๋ฉ์๋๊ฐ ์๋์ ํด๋์ค์ ์จ์ ํ ๋จ์์๊ธฐ ๋๋ฌธ์๋๋ค.

**์ฆ, DI๋ฅผ ์ด์ฉํ ๋ถ๋ฆฌ๋ฅผ ์๊ฐํด๋ด์ผ ํฉ๋๋ค!**

**๐ DI๋ฅผ ์ด์ฉํ ๋ถ๋ฆฌ**

์ด๋ค ์๋ฆฌ๋ฅผ ํตํด DI๋ฅผ ์ด์ฉํ ๋ถ๋ฆฌ๊ฐ ๊ฐ๋ฅํ์ง ์ดํด๋ด์๋ค.

์ผ๋จ์, `UserService` ํด๋์ค๋ `UserService`์ธํฐํ์ด์ค๋ฅผ ๊ตฌํํ ํด๋์ค๋ผ๋ ๊ฒ์ ์๊ฐํด๋ณผ ์ ์์ต๋๋ค. 

์ฆ, `UserServiceImpl`์ ์ด๋ ํ ํด๋ผ์ด์ธํธ์๋ ๊ฐ๋ ฅํ, ์ง์ ์ ์ธ
๊ฒฐํฉ์ด ๋์ด์์ง ์๊ณ  ์ ์ฐํ ํ์ฅ์ด ๊ฐ๋ฅํ ์ํ์๋๋ค.

ํ์ง๋ง, ์ผ๋ฐ์ ์ธ DI๋ฅผ ์ฐ๋ ์ด์ ๋ฅผ ์๊ฐํด ๋ณธ๋ค๋ฉด, ์ผ๋ฐ์ ์ผ๋ก ๋ฐํ์์ ํ๊ฐ์ง ๊ตฌํ ํด๋์ค๋ฅผ ๊ฐ์๊ฐ๋ฉฐ ์ฌ์ฉํ๊ธฐ ์ํ์ฌ ์ฌ์ฉํ๋ค๊ณ  ๋ณด์๋ฉด ๋ฉ๋๋ค.

**์ฌ๊ธฐ์, ์ผ๋ฐ์ ์ด๋ผ๋ ๋ง์ ๊ฒฐ์ฝ ์ ํด์ง ์ ์ฝ์ด ์๋๋๊น, ์ผ๋ฐ์ ์ธ DI๊ฐ ์๋ ํ ๋ฒ์ ๋ ๊ฐ์ง ๊ตฌํํด๋์ค๋ฅผ ๋์์ ์ด์ฉํด๋ณธ๋ค๋ฉด ์ด๋จ๊น๋ผ๋ ์๊ฐ์ ํ  ์ ์์ต๋๋ค.**

**๋ค์๊ณผ ๊ฐ์ ๊ตฌ์กฐ๋ฅผ ์๊ฐํด๋ณผ ์ ์์ต๋๋ค.**
- `UserService`๋ฅผ ๊ตฌํํ๋ ๋๋ค๋ฅธ ๊ตฌํ ํด๋์ค`UserServiceTx`๋ฅผ ๋ง๋ ๋ค.
  - ์ด ๊ตฌํ ํด๋์ค๋ ํธ๋์ญ์ ๊ฒฝ๊ณ ์ค์  ๊ธฐ๋ฅ๋ง์ ์ฑ์์ผ๋ก ํ๋ค
- `UserServiceTx`๋ ํธ๋์ญ์ ๊ฒฝ๊ณ ์ค์ ์ฑ์๋ง์ ๋งก๊ณ , ๋น์ฆ๋์ค ๋ก์ง์ ๋ ๋ค๋ฅธ ๊ตฌํ ํด๋์ค์ธ `UserServiceImpl`์๊ฒ ๋งก๊ธด๋ค.

๊ฒฐ๊ณผ์ ์ผ๋ก, ์ด๋ฐ ๊ตฌ์กฐ๋ฅผ ํตํด ์์์ ์ํ ํธ์ถ ์์ ์ด์  ์ดํ์ ํธ๋์ญ์ ๊ฒฝ๊ณ๋ฅผ ์ค์ ํด์ฃผ๊ฒ ๋๋ค๋ฉด, 
ํด๋ผ์ด์ธํธ ์์ฅ์์๋ ํธ๋์ญ์ ๊ฒฝ๊ณ์ค์ ์ด ๋ถ์ฌ๋ ๋น์ฆ๋์ค ๋ก์ง์ ์ฌ์ฉํ๋ ๊ฒ๊ณผ ๋๊ฐ์ด ๋ฉ๋๋ค.

์ด๋ฌํ ๋ฐฉ๋ฒ์ ์ด์ฉํด ๋ค์๊ณผ ๊ฐ์ ์ฝ๋๋ฅผ ๊ตฌ์ฑํ  ์ ์์ต๋๋ค.

**UserServiceTx.java**
```java
@RequiredArgsConstructor
public class UserServiceTx implements UserService{

    private final UserService userService;
    private final PlatformTransactionManager transactionManager;

    @Override
    public void add(User user) {
        userService.add(user);
    }

    @Override
    public void upgradeLevels() {
        TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            userService.upgradeLevels();
            transactionManager.commit(status);
        }catch(RuntimeException e){
            transactionManager.rollback(status);
            throw e;
        }
    }
}

```

**UserServiceImpl.java - UpgradeLevels๋ฉ์๋**
```java
    public void upgradeLevels(){
        List<User> users=userDao.getAll();
        for(User user:users){
            if(checkUpgrade(user)){
                upgradeLevel(user);
            }
        }
    }
```

**AppConfig.java - DI**
```java
@Bean
public UserService userServiceTx(){
        return new UserServiceTx(userService(),transactionManager());
}
    
@Bean
public UserService userService(){
        return new UserServiceImpl(userDao(),mailSender());
}
```

์ฝ๋๋ฅผ ์ดํด ๋ณด์๋ฉด, ๋จผ์  ํธ๋์ญ์์ ๋ด๋นํ๋ ์ค๋ธ์ ํธ๊ฐ ์ฌ์ฉ๋จ์ผ๋ก ํธ๋์ญ์ ๊ฒฝ๊ณ ์ค์ ์ ํด์ค ๋ค์ ์ค์  ๋น์ฆ๋์ค ๋ก์ง์
`UserServiceImpl`์ ์์๋ ๊ฒ์ ๋ณผ ์ ์์ต๋๋ค.

์ต์ข์ ์ผ๋ก ๋ค์ ๊ทธ๋ฆผ๊ณผ ๊ฐ์ ์์กด ๊ด๊ณ๊ฐ ๊ตฌ์ฑ๋์๋ค๊ณ  ๋ณผ ์ ์์ต๋๋ค.

<img width="1017" alt="แแณแแณแแตแซแแฃแบ 2022-01-14 แแฉแแฅแซ 4 08 04" src="https://user-images.githubusercontent.com/56334761/149393742-e28f4891-00ae-4d9e-a718-1d1511fa02c3.png">

์ด์  ํด๋ผ์ด์ธํธ๋ `UserServiceTx` ๋น์ ํธ์ถํ๋ฉด ํธ๋์ญ์ ๊ฒฝ๊ณ์ค์ ์ด ๋ ๋น์ฆ๋์ค ๋ก์ง์ ์ฌ์ฉํ  ์ ์๊ฒ ๋ฉ๋๋ค.


***

### ๐ Dynamic Proxy & Factory Bean

์ง๊ธ ๊น์ง ํด์๋ ๊ณผ์ ์ ํน์ง์ ์ดํด๋ด์๋ค.

ํ์ฌ, `UserServiceTx`์ `UserServiceImpl` ์ํ๋ก ๋ถ๋ฆฌ๋์ด ์์ต๋๋ค. 

๋ถ๊ฐ ๊ธฐ๋ฅ์ ๋ด๊ณ ์๋ ํด๋์ค์ธ `UserServiceTx`๋ ๋ถ๊ฐ๊ธฐ๋ฅ์ธ์ ๋๋จธ์ง ํต์ฌ ๋ก์ง์ ๋ชจ๋ `UserServiceImpl`์ ์์ํ๋ ๊ตฌ์กฐ๊ฐ ๋ฉ๋๋ค.

๋ฐ๋ผ์ `UserServiceImpl`์ `UserServiceTx`์ ์กด์ฌ ์์ฒด๋ฅผ ๋ชจ๋ฅด๋ฉฐ, ๋ถ๊ฐ๊ธฐ๋ฅ์ด ํต์ฌ ๊ธฐ๋ฅ์ ์ฌ์ฉํ๋ ๊ตฌ์กฐ๊ฐ ๋ฉ๋๋ค.

>**๋ฌธ์ ์ **
> 
>ํด๋ผ์ด์ธํธ๊ฐ `UserServiceImpl`๋ฅผ ์ง์  ์ฌ์ฉํด๋ฒ๋ฆฌ๋ฉด?? ๋ถ๊ฐ๊ธฐ๋ฅ์ด ์ ์ฉ๋์ง๋ฅผ ๋ชปํฉ๋๋ค.

๋ฐ๋ผ์, ์ด๋ฌํ ์ฌํ๋ฅผ ๋ฐฉ์งํ๊ธฐ ์ํ์ฌ ๋ถ๊ฐ๊ธฐ๋ฅ์ ๋ด์ ํด๋์ค์ธ `UserServiceTx`๋ฅผ ๋ง์น ํต์ฌ๊ธฐ๋ฅ์ ๊ฐ์ง ํด๋์ค์ฒ๋ผ ๊พธ๋ฉฐ ํด๋ผ์ด์ธํธ๊ฐ
์ด ํด๋์ค๋ง์ ์ฌ์ฉํ๋๋ก ๋ง๋ค์ด์ผ ํฉ๋๋ค.

๊ทธ๋ฌ๊ธฐ ์ํด์๋ ํด๋ผ์ด์ธํธ๋ ์ง์  ํด๋์ค๋ฅผ ์ฌ์ฉํ๋ ๊ฒ์ด ์๋ ์ธํฐํ์ด์ค๋ฅผ ํตํด์๋ง ๊ธฐ๋ฅ์ ์ฌ์ฉํ๋๋ก ํ์ฌ์ผ ํฉ๋๋ค.

์ฆ, ์ฌ์ฉ์๋ ์ ํฌ๊ฐ ๋ง๋ค์ด๋์
<img width="1017" alt="แแณแแณแแตแซแแฃแบ 2022-01-14 แแฉแแฅแซ 4 08 04" src="https://user-images.githubusercontent.com/56334761/149393742-e28f4891-00ae-4d9e-a718-1d1511fa02c3.png">
์ด ๊ตฌ์กฐ๋ฅผ ๋ชจ๋ฅด๋๋ผ๋ ์ด๋ฐ ๊ตฌ์กฐ๋ฅผ ์ฌ์ฉํ๊ฒ๋ ์ ๋ํ๋ ๊ฒ์๋๋ค.

**ํ์ด์ ์ด์ผ๊ธฐํ์๋ฉด ํด๋ผ์ด์ธํธ๋ ์ธํฐํ์ด์ค๋ง์ ๋ณด๊ณ  ์ฌ์ฉํ๊ธฐ ๋๋ฌธ์, ๊ทธ ์ธํฐํ์ด์ค๊ฐ ๋น์ฆ๋์ค ๋ก์ง์ ๊ฐ์ง ์ฝ๋์ธ์ค ์ํ์ง๋ง ์ฌ์ค์ ๋ถ๊ฐ๊ธฐ๋ฅ ์ฝ๋๋ฅผ ํตํด ์ ๊ทผํ๊ฒ ๋๋ ๊ฒ์๋๋ค.**

๋ด์๋ ๊ฒ์ฒ๋ผ ํด๋ผ์ด์ธํธ๊ฐ ์ฌ์ฉํ๋ ค๊ณ  ํ๋ ์ค์  ๋์์ธ๊ฒ ์ฒ๋ผ ์์ฅํด์ ํด๋ผ์ด์ธํธ์ ์์ฒญ์ ๋ฐ์์ฃผ๋ ๊ฒ,
์ง๊ธ์ `UserServiceTx`๊ฐ ํ๋ ์ญํ ์ ๋๋ฆฌ์๋ผ๋ ๋ป์ผ๋ก <span style="color:red; font-weight:bold;">Proxy</span>๋ผ๊ณ  ๋ถ๋ฅด๊ฒ ๋ฉ๋๋ค.

์ด๋ฐ ํ๋ก์๋ฅผ ํตํด ์์๋ฐ์ ํต์ฌ๊ธฐ๋ฅ์ ์ฒ๋ฆฌํ๋ ์ค์  ์ค๋ธ์ ํธ๋ฅผ <span style="color:red; font-weight:bold;">Target</span>์ด๋ผ๊ณ  ๋ถ๋ฆ๋๋ค.

**Proxy์ ํน์ง & ์ฌ์ฉ์ด์ **
1. Target๊ณผ ๊ฐ์ ์ธํฐํ์ด์ค๋ฅผ ๊ตฌํ, ํ๋ก์๋ Target์ ์ ์ดํ  ์์น์ ์์ผํฉ.
2. ํด๋ผ์ด์ธํธ๊ฐ ํ๊น์ ์ ๊ทผ ๋ฐฉ๋ฒ ์ ์ด.
3. ํ๊น์ ๋ถ๊ฐ๊ธฐ๋ฅ ๋ถ์ฌ ๊ฐ๋ฅ.

#### ๐ ๋ฐ์ฝ๋ ์ดํฐ ํจํด

๋ฐ์ฝ๋ ์ดํฐ ํจํด์ด๋ ์ผ๋ฐ์ ์ผ๋ก ๋ฐํ์์์ ํ๊น์๊ฒ ๋ถ๊ฐ์ ์ธ ๊ธฐ๋ฅ์ ๋ถ์ฌํ๊ธฐ ์ํด ํ๋ก์๋ฅผ ์ฌ์ฉํ๋ ํจํด์ ๋งํฉ๋๋ค.

์ฆ, ๋ฐํ์ ์์ ์ ๋ถ๊ฐ์ ์ธ ๊ธฐ๋ฅ์ ๋ถ์ฌํ๊ธฐ ๋๋ฌธ์ ์ฝ๋์์์๋ ์ด๋ค ๋ฐฉ์์ผ๋ก ํ๋ก์์ ํ๊น์ด ์ฐ๊ฒฐ๋์ด์๋์ง ์ ์ ์์ต๋๋ค.

๋ฐ์ฝ๋ ์ดํฐ ํจํด์์๋ ๊ฐ์ ์ธํฐํ์ด์ค๋ฅผ ๊ตฌํํ ํ๊ฒ๊ณผ ์ฌ๋ฌ๊ฐ์ ํ๋ก์๋ฅผ ๋ง๋ค ์ ์์ต๋๋ค. ์ด๋ ๋ถ๊ฐ๊ธฐ๋ฅ์ ์๋ฅผ ๋ค์ด Transaction๋ฟ๋ง์๋๋ผ
ํ๊ฒ์ ์ฌ๋ฌ๊ฐ์ง์ ๊ธฐ๋ฅ์ ํ ๋ฒ์ ๋ถ์ฌ์ํฌ ์ ์๋ค๋ ๊ฒ์ ๋ปํฉ๋๋ค.

ํ๋ก์๋ก์ ๋์ํ๋ ๊ฐ ๋ฐ์ฝ๋ ์ดํฐ๋ ๋ค์ ๋จ๊ณ๊ฐ ๋ฐ์ฝ๋ ์ดํฐ ํ๋ก์์ธ์ง ์ต์ข ํ๊ฒ์ธ์ง๋ฅผ ๋ชจ๋ฅด๊ธฐ ๋๋ฌธ์
๋ค์ ์์ ๋์์ ์ธํฐํ์ด์ค๋ก ์ ์ธํ๋ฉฐ ์ธ๋ถ์์ ๋ฐํ์ ์์ ์ฃผ์๋ฐ์ ์ ์๋๋ก ํฉ๋๋ค.

์ฆ, ์คํ๋ง์ DI๋ ๋ฐ์ฝ๋ ์ดํฐ ํจํด์ ์ ์ฉ์ํค๊ธฐ์ ์์ฃผ ํธ๋ฆฌํ๋ค๊ณ  ํ  ์ ์์ต๋๋ค.

**์๋ฅผ ๋ค์ด,** ์ง๊ธ ๊ตฌ์ฑํ `UserService`์ ํธ๋์ญ์ ๊ธฐ๋ฅ๊ณผ ๋๋ถ์ด ๋ ๋ค๋ฅธ ๊ธฐ๋ฅ์ ์ถ๊ฐํ๋ค๊ณ  ํ๋ฉด,

```java
@Bean
public UserService userServiceAnother(){
    return new UserServiceAnother(userServiceTx());
        }
        
@Bean
public UserService userServiceTx(){
        return new UserServiceTx(userService(),transactionManager());
        }

@Bean
public UserService userService(){
        return new UserServiceImpl(userDao(),mailSender());
        }
```

์ด๋ฐ ์์ผ๋ก ํ์ํ๋ฉด ์ธ์ ๋ ์ง ๋ฐ์ฝ๋ ์ดํฐ๋ฅผ ์ถ๊ฐ์ํฌ ์ ์๊ฒ ๋ฉ๋๋ค.

#### ๐ ํ๋ก์ ํจํด

๋ฐ์ฝ๋ ์ดํฐ ํจํด์ ์ต์ข ํ๊ฒ์ ๊ธฐ๋ฅ์ ๋ถ์ฌํ๋ ค๋ ๋ชฉ์ ์ผ๋ก ์ค๊ฐ์ ํ๋ก์๊ฐ ๋ผ์ด๋ค์๋ค๋ฉด, ์ผ๋ฐ์ ์ผ๋ก ํ๋ก์ ํจํด์
์ต์ข ํ๊ฒ์ ๋ํ ์ ๊ทผ ์ ์ด๋ฅผ ์ํด ๋ง๋ค์ด์ง ๊ฒฝ์ฐ๋ฅผ ๋ปํฉ๋๋ค.

๋ ์์ธํ, ํ๋ก์ ํจํด์ ํ๊น์ ๊ธฐ๋ฅ์ ์ถ๊ฐํ๋ ๊ฒ์ด ์๋ ํด๋ผ์ด์ธํธ๊ฐ ํ๊น์ ์ ๊ทผํ๋ ๋ฐฉ๋ฒ์ ๋ฐ๊ฟ์ค๋ค๊ณ  ์๊ฐํ์๋ฉด ํธํฉ๋๋ค.
์ฆ, ํ๊ฒ์ ์ค๋ธ์ ํธ๋ฅผ ์์ฑํ๊ธฐ๊ฐ ๋ณต์กํ๊ฑฐ๋ ๋น์ฅ ํ์ํ์ง ์์ ๊ฒฝ์ฐ ๋ฐ๋ก ์ค๋ธ์ ํธ๋ฅผ ์์ฑํ๋ ๊ฒ์ด ์๋ ํ๋ก์๋ฅผ ์์ฑํด ๋์๋ค๊ฐ,
ํด๋ผ์ด์ธํธ๊ฐ ๋ฉ์๋๋ฅผ ์์ฒญํ ์์ ์ ํ๋ก์๊ฐ ํ๊น ์ค๋ธ์ ํธ๋ฅผ ๋ง๋ค๊ณ , ์์ฒญ์ ์์ํด์ฃผ๋ ๊ฒ์๋๋ค.

>๋ง์น **JPA์ ํ๋ก์ ํจํด๊ณผ๋ ์ ์ฌํฉ๋๋ค.**
> 
> <a href="https://velog.io/@sungjin0757/JPA-%ED%94%84%EB%A1%9D%EC%8B%9C%EC%A6%89%EC%8B%9C%EB%A1%9C%EB%94%A9-VS-%EC%A7%80%EC%97%B0%EB%A1%9C%EB%94%A9">JPA ํ๋ก์ ํจํด ๋ณด๋ฌ๊ฐ๊ธฐ (์ฆ์๋ก๋ฉ, ์ง์ฐ๋ก๋ฉ)</a>

๊ตฌ์กฐ์ ์ผ๋ก ๋ณด์๋ฉด, ํ๋ก์ ํจํด ๋ํ ๋ค์ ๋์์ ์ธํฐํ์ด์ค๋ฅผ ํตํด ์์ ๊ฐ๋ฅํ๊ธฐ ๋๋ฌธ์ ๋ฐ์ฝ๋ ์ดํฐ ํจํด๊ณผ
 ์ ์ฌํ๋ค๊ณ  ๋ณผ ์ ์์ต๋๋ค. ๋ค๋ง, ๋ฐ์ฝ๋ ์ดํฐ ํจํด์ ๋ค์ ๋์์ด ๋ฌด์์ธ์ง๋ฅผ ๋ชฐ๋ผ๋ ๋์์ง๋ง ํ๋ก์ ํจํด์ ํ๋ก์๋
์ฝ๋์์ ์์ ์ด ๋ง๋ค๊ฑฐ๋ ์ ๊ทผํ  ํ๊ฒ์ ์ ๋ณด๋ฅผ ์์์ผํ๋ ๊ฒฝ์ฐ๊ฐ ๋ง์ต๋๋ค. ์๋ํ๋ฉด, ํ๊ฒ์ ์ค๋ธ์ ํธ๋ฅผ ๋ง๋ค์ด์ผ ํ๋ ํ๋ก์์ผ ๊ฒฝ์ฐ
 ํ๊ฒ์ ๋ํ ์ง์ ์ ์ธ ์ ๋ณด๋ฅผ ์์์ผํ๊ธฐ ๋๋ฌธ์๋๋ค.

์ธํฐํ์ด์ค๋ฅผ ํตํด ๋ค์ ๋์์ ์์ํ๋ฏ๋ก ๊ฒฐ๊ตญ์ ๋ฐ์ฝ๋ ์ดํฐ ํจํด๊ณผ ํ๋ก์ ํจํด ๋ ๊ฐ์ง ๊ฒฝ์ฐ๋ฅผ ํผํฉํ์ฌ๋ ์ฌ์ฉํ  ์๋ ์์ต๋๋ค.

<img width="627" alt="แแณแแณแแตแซแแฃแบ 2022-01-16 แแฉแแฎ 10 24 32" src="https://user-images.githubusercontent.com/56334761/149661818-42ed498a-da6e-4c09-ad2f-276427b2e44c.png">

์ด๋ฐ ๋๋์ผ๋ก ๋ง์ด์ฃ ..

#### ๐ Dynamic Proxy

ํ๋ก์๊ฐ ์ด๋ค ์ด์ ๋ก ๋ง๋ค์ด ์ก๋์ง ๋ํ ํ๋ก์๊ฐ ์ด๋ค ๋ฐฉ์์ผ๋ก ๋ง๋ค์ด ์ง๋ ์ง๋ฅผ ์ง๊ธ๊น์ง ์์๋ณด์์ต๋๋ค.

๊ทธ ์ด์ ๋ 
- **์ฒซ๋ฒ์งธ๋ก๋,** ํ๋ก์๋ฅผ ๊ตฌ์ฑํ๊ณ  ๋ ๋ค์ ํ๊น์๊ฒ ์์ํ๋ ์ฝ๋๋ฅผ ์์ฑํ๊ธฐ ๋ฒ๊ฑฐ๋กญ๋ค๋ ์ ์๋๋ค.
  
    ์๋ํ๋ฉด, ํด๋ผ์ด์ธํธ๋ ๊ฒฐ๊ตญ์๋ ํ๋ก์ ๊ฐ์ฒด๋ฅผ ์ด์ฉํ์ฌ ํ๊น์๊ฒ ์ ๊ทผ์ด ๊ฐ๋ฅํ  ํฐ์ธ๋ฐ, ํ๊น์ ๋ฉ์๋๊ฐ ๋ง์์ง์๋ก ์์ํด์ค์ผํ๋ ์ฝ๋์ ์์ ๊ธธ์ด์ง ๊ฒ์ด๋ฉฐ,
    ๊ธฐ๋ฅ์ด ์ถ๊ฐ๊ฑฐ๋ ์์ ๋  ๋ ๋ํ ํจ๊ป ๊ณ ์ณ์ค์ผํ๋ค๋ ๋ฌธ์ ์ ์ด ์์ต๋๋ค.
- **๋๋ฒ์งธ๋ก๋,** ๋ถ๊ฐ๊ธฐ๋ฅ ์ฝ๋ ์์ฑ์ด ์ค๋ณต๋  ๊ฒฝ์ฐ๊ฐ ๋ง๋ค๋ ์ ์๋๋ค. ์๋ํ๋ฉด, ๋ชจ๋  ๋ฉ์๋๋ง๋ค ๋๊ฐ์ด ์ ์ฉ์์ผ์ผ ํ  ์ง๋ ๋ชจ๋ฅด๊ธฐ ๋๋ฌธ์๋๋ค.

์ด๋ฐ ๋ฌธ์ ์ ์ ํด๊ฒฐํ  ์ ์๋๊ฒ์ด ๋ฐ๋ก **Dynamic Proxy**์๋๋ค.

Dynamic Proxy๋ฅผ ๊ตฌ์ฑํ๊ธฐ ์ ์ ๋จผ์  **๋ฆฌํ๋ ์**์ ๋ํด์ ์์๋ด์๋ค.

๋ฆฌํ๋ ์ API๋ฅผ ํ์ฉํด ๋ฉ์๋์ ๋ํ ์ ์๋ฅผ ๋ด์ Method ์ธํฐํ์ด์ค๋ฅผ ํ์ฉํด ๋ฉ์๋๋ฅผ ํธ์ถํ๋ ๋ฐฉ๋ฒ์ ์์๋ด์๋ค.

**ArrayList**์ **size**๋ผ๋ ๋ฉ์๋๋ฅผ ์ถ์ถํ ๋ค **invoke**๋ฅผ ํตํด ์ถ์ถํด๋ธ ๋ฉ์๋๋ฅผ ์คํ์์ผ ๋ด์๋ค.

```java
@Test
    @DisplayName("Reflect - Method Test")
    void ๋ฆฌํ๋ ํธ_๋ฉ์๋_์ถ์ถ_ํ์คํธ() throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {
        Method sizeMethod= ArrayList.class.getMethod("size");

        List<Integer> testList=new ArrayList<>();
        testList.add(1);
        testList.add(2);
        testList.add(3);
        
        Assertions.assertThat(testList.size()).isEqualTo(3);
        Assertions.assertThat(testList.size()).isEqualTo(sizeMethod.invoke(testList));
        Assertions.assertThat(sizeMethod.invoke(testList)).isEqualTo(3);
    }
```

**ํ์คํธ ๊ฒฐ๊ณผ**

<img width="50%" alt="แแณแแณแแตแซแแฃแบ 2022-01-19 แแฉแแฎ 7 17 59" src="https://user-images.githubusercontent.com/56334761/150111020-ae021db9-597d-4b2d-aa3e-abbd3326b006.png">

๋ณด์๋ ๊ฒ๊ณผ ๊ฐ์ด Reflect๋ฅผ ํ์ฉํด์ ๋ฉ์๋์ ๋ํ ์ ๋ณด๋ฅผ ์ถ์ถํด๋ผ ์ ์์๊ณ , ์ด๋ฅผ ์ด์ฉํ์ฌ ์ง์ ํ ์ค๋ธ์ ํธ์ ๋ํ์ฌ ๋ฉ์๋๋ฅผ ์คํ์ํฌ ์ ์๋ค๋ ๊ฒ์ 
ํ์ธํ์์ต๋๋ค.

Dynamic Proxy์ ๋์ ๋ฐฉ๋ฒ๋ถํฐ ์ดํด๋ด์๋ค.

<img width="1095" alt="แแณแแณแแตแซแแฃแบ 2022-01-19 แแฉแแฎ 7 36 48" src="https://user-images.githubusercontent.com/56334761/150113847-ff0dc65e-cdc3-46be-9276-84563a949d15.png">

**Dynamic Proxy**๋? ๋จผ์  ํ๋ก์ ํฉํ ๋ฆฌ์ ์ํด ๋ฐํ์ ์ ๋ค์ด๋ด๋ฏนํ๊ฒ ๋ง๋ค์ด์ง๋ ํ๋ก์ ์๋๋ค. ํ๋ก์ ํฉํ ๋ฆฌ์๊ฒ `Interface`์ ์ ๋ณด๋ง
 ๋๊ฒจ์ฃผ๋ฉด ํ๋ก์๋ฅผ ์ ์ฉํ ์ค๋ธ์ ํธ๋ฅผ ์๋์ผ๋ก ๋ง๋ค์ด์ฃผ๊ฒ ๋ฉ๋๋ค.

์ด ๊ณผ์ ์์, ์ถ๊ฐ์ํค๊ณ ์ ํ๋ ๋ถ๊ฐ๊ธฐ๋ฅ์ `Invocation Handler`์ ๋ฃ์ด์ฃผ๊ธฐ๋ง ํ๋ฉด ๋ฉ๋๋ค.

**InvocationHandler.java**
```java
public interface InvocationHandler {

    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
}

```

`InvocationHancler` ์ธํฐํ์ด์ค์๋๋ค. `invoke`๋ผ๋ ๋ฉ์๋๋ ์์์ ์งํํด๋ณด์๋ ๋ฆฌํ๋ ์ API์ Method ์ธํฐํ์ด์ค์ ํ๊น ๋ฉ์๋์ ํ๋ผ๋ฏธํฐ๋ฅผ ํ๋ผ๋ฏธํฐ๋ก ์ ๋ฌ ๋ฐ์ต๋๋ค.

์ฆ, ํด๋ผ๋ฆฌ์ธํธ์ ๋ชจ๋  ์์ฒญ ๋ฉ์๋๋ `Dynamic Proxy`๋ฅผ ํตํ์ฌ `InvocationHandler`์ `Invoke`๋ฉ์๋์ ํ๋ผ๋ฏธํฐ๋ก ์ ๋ฌ๋๋ฉฐ 
ํ๊น ๋ฉ์๋์ ๋ถ๊ฐ๊ธฐ๋ฅ์ ์ ์ฉ์์ผ ๊ทธ ๊ฒฐ๊ณผ๋ฅผ ๋ฆฌํดํด์ค๋๋ค.

์ด๋ ์์์ ๋ดค๋ ๋๋ฒ์งธ ๋ฌธ์ ์ ์ธ ์ค๋ณต๋ ์ฝ๋๋ฅผ ํด๊ฒฐํ  ์ ์์ต๋๋ค. `Invoke`๋ผ๋ ๋ฉ์๋ ํ๋๋ก ํ๊น ์ค๋ธ์ ํธ์ ๋ฉ์๋์ ๋ถ๊ฐ๊ธฐ๋ฅ์ ์ ์ฉ์์ผ ์คํํ  ์ ์๊ธฐ ๋๋ฌธ์๋๋ค.

์ด์ ๋, Transaction ๋ถ๊ฐ๊ธฐ๋ฅ์ Dynamic Proxy๋ฅผ ํตํ์ฌ ์ฝ๋๋ก ์์ฑํด๋ด์๋ค.

**TransactionHandler.java**
```java
@RequiredArgsConstructor
public class TransactionHandler implements InvocationHandler {

    private final Object target;
    private final PlatformTransactionManager transactionManager;
    private final String pattern;

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if(method.getName().startsWith(pattern))
            return invokeWithTransaction(method,args);
        return method.invoke(target,args);
    }

    private Object invokeWithTransaction(Method method,Object[] args) throws Throwable{
        TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
        try{
            Object invoke = method.invoke(target, args);
            transactionManager.commit(status);
            return invoke;
        }catch(InvocationTargetException e){
            transactionManager.rollback(status);
            throw e.getTargetException();
        }
    }
}

```

์ด ์ฝ๋์์๋
1. Target Object
2. Transaction Manager
3. Method Pattern (๋ถ๊ฐ๊ธฐ๋ฅ์ ์ง์ ๋ ๋ฉ์๋์๋ง ์ ์ฉ์ํค๊ธฐ ์ํด)

๋ค์ DI ์์ผ์ฃผ๋ ๋ถ๋ถ์ ์ ์ํ๋ฉด ๋ฉ๋๋ค.

**Dynamic Proxy - Client์์ ์ง์  ์์ฑ**
```java
TransactionHandler txHandler=new TranscationHandler();

UserService userService=(UserService)Proxy.newProxyInstance(
        getClass().getClassLoader(),new Class[]{UserService.class},txHndler
        );
```

์ด๋ ๊ฒ Dynamic Proxy๋ฅผ ์ง์  ์์ฑํด์ค ์๋ ์์ต๋๋ค.

์ง๊ธ๋ถํฐ๋, `TransactionHandler`์ `Dynamic Proxy`๋ฅผ ์คํ๋ง DI๋ฅผ ํตํด ์ฌ์ฉํ  ์ ์๋๋ก ๋ง๋ค๋ฉด ๋ฉ๋๋ค.

ํ์ง๋ง, `Dynamic Proxy`๋ ๋ฐํ์ ์์ ๋์ ์ผ๋ก ๋ง๋ค์ด์ง๊ธฐ ๋๋ฌธ์ ์ผ๋ฐ์ ์ธ ์คํ๋ง Bean์ผ๋ก ๋ฑ๋กํ  ์ ์๋ค๋ ๊ฒ์๋๋ค.

#### ๐ Factory Bean
์คํ๋ง์ ์์ฑ์๋ฅผ ํตํด ์ค๋ธ์ ํธ๋ฅผ ๋ง๋๋ ๋ฐฉ๋ฒ ์ธ์๋ ๋ค์ํ ๋ฐฉ๋ฒ์ด ์์ต๋๋ค. ๊ทธ ์ค ํ๋๊ฐ ์ด Factory Bean์๋๋ค.

Factory Bean ์ ์คํ๋ง์ ๋์ ํด์ ์ค๋ธ์ ํธ์ ์์ฑ๋ก์ง์ ๋ด๋นํ๋๋ก ๋ง๋ค์ด์ง ํน๋ณํ ๋น์ ๋งํฉ๋๋ค.

```java
public interface FactoryBean<T> {
    String OBJECT_TYPE_ATTRIBUTE = "factoryBeanObjectType";

    @Nullable
    T getObject() throws Exception;

    @Nullable
    Class<?> getObjectType();

    default boolean isSingleton() {
        return true;
    }
}
```
์ด ์ธํฐํ์ด์ค๋ฅผ ๊ตฌํํ๊ธฐ๋ง ํ๋ฉด ๋ฉ๋๋ค.
- getObject() ๋ฉ์๋ ๋ด๋ถ์์ Dynamic Proxy๋ฅผ ์์ฑํ ํ ๋ฐํ์์ผ์ค๋๋ค.

๊ฒฐ๋ก ์ ์ผ๋ก ์ด ์ธํฐํ์ด์ค๋ฅผ ๊ตฌํํ ํด๋์ค๋ฅผ ์คํ๋ง์ ๋น์ผ๋ก ๋ฑ๋กํด์ฃผ๋ฉด ๋๋ ๊ฒ์๋๋ค.

์ถ๊ฐ๋ก, ์คํ๋ง์ `FactoryBean`์ธํฐํ์ด์ค๋ฅผ ๊ตฌํํ ํด๋์ค๊ฐ ๋น์ ํด๋์ค๋ก ์ง์ ๋๋ฉด, ํฉํ ๋ฆฌ ๋น ํด๋์ค์ getObject()๋ฅผ ํตํ์ฌ ์ค๋ธ์ ํธ๋ฅผ ๊ฐ์ ธ์ค๊ณ ,
 ์ด๋ฅผ ๋น ์ค๋ธ์ ํธ๋ก ์ฌ์ฉํฉ๋๋ค. ๋น์ ํด๋์ค๋ก ๋ฑ๋ก๋ ํฉํ ๋ฆฌ๋น์ ๋น ์ค๋ธ์ ํธ๋ฅผ ์์ฑํ๋ ๊ณผ์ ์์๋ง ์ฌ์ฉ๋ฉ๋๋ค.

`FactoryBean` ์ธํฐํ์ด์ค๋ฅผ ๊ตฌํํ ํด๋์ค๋ฅผ ์คํ๋ง ๋น์ผ๋ก ๋ง๋ค์ด๋๋ฉด getObject() ๋ผ๋ ๋ฉ์๋๊ฐ ์์ฑํด์ฃผ๋ ์ค๋ธ์ ํธ๊ฐ ์ค์  ๋น์
์ค๋ธ์ ํธ๋ก ๋์ฒด ๋๋ค๊ณ  ๋ณด์๋ฉด ๋  ๊ฒ ๊ฐ์ต๋๋ค.

**์ฝ๋๋ฅผ ํตํด ์ดํด๋ด์๋ค.**

**TransactionFactoryBean.java**
```java
@RequiredArgsConstructor
@Getter
public class TransactionFactoryBean implements FactoryBean<Object> {

    private final Object target;
    private final PlatformTransactionManager transactionManager;
    private final String pattern;
    private final Class<?> interfaces;

    @Override
    public Object getObject() throws Exception {
        return Proxy.newProxyInstance(getClass().getClassLoader(),new Class[]{interfaces},new TransactionHandler(target,
                transactionManager,pattern));
    }

    @Override
    public Class<?> getObjectType() {
        return interfaces;
    }

    @Override
    public boolean isSingleton() {
        return false;
    }
}
```

**AppConfig.java - ์คํ๋ง ๋น ๋ฑ๋ก**
```java
    @Bean
    public TransactionFactoryBean userService(){
        return new TransactionFactoryBean(userServiceImpl(),transactionManager()
        ,"upgradeLevels()",UserService.class);
    }
```

**์ง๊ธ ๊น์ง, `Dynamic Proxy`์ `Factory Bean`์ ์ ์ฉํด ๋ณด์์ต๋๋ค. ์ฅ์ ๊ณผ ๋จ์  ๋ํ ์์๋ด์๋ค.**

**์ฅ์ **
- ์ฌ์ฌ์ฉ์ด ๊ฐ๋ฅํฉ๋๋ค.
  - `Factory Bean`์ ๋ค์ํ ํด๋์ค์ ์ ์ฉ๊ฐ๋ฅํฉ๋๋ค. ๋ํ ํ๋ ์ด์์ ๋น์ ๋ฑ๋กํด๋ ์๊ด ์์ต๋๋ค.
- ์ธํฐํ์ด์ค๋ฅผ ๊ตฌํํ๋ ํ๋ก์ ํด๋์ค๋ฅผ ์ผ์ผ์ด ๋ง๋ค์ด์ผ ํ๋ค๋ ๋ฒ๊ฑฐ๋ก์์ ํด๊ฒฐํด์ค๋๋ค.
- ๋ถ๊ฐ์ ์ธ ๊ธฐ๋ฅ์ด ์ฌ๋ฌ ๋ฉ์๋์ ๋ฐ๋ณต์ ์ผ๋ก ๋ํ๋๊ฒ ๋๋ ๊ฒ์ ํด๊ฒฐํด์ค๋๋ค.

**๋จ์ **
- ํ ๋ฒ์ ์ฌ๋ฌ๊ฐ์ ํด๋์ค์ ๊ณตํต์ ์ธ ๋ถ๊ฐ๊ธฐ๋ฅ์ ๋ถ์ฌํ๋ ๊ฒ์ ๋ถ๊ฐ๋ฅํฉ๋๋ค. (`Factory Bean`์ ์ค์ ์ ์ค๋ณต์ ๋ง์ ์ ์๋ค๋ ๊ฒ์ ๋ปํฉ๋๋ค.)
- ํ๋์ ํ๊น์ ์ฌ๋ฌ๊ฐ์ง ๋ถ๊ฐ๊ธฐ๋ฅ์ ๋ถ์ฌํ ์๋ก ์ค์  ํ์ผ์ด ๋ณต์กํด์ง๋๋ค.
  - ์๋ฅผ ๋ค์ด, Transaction ๊ธฐ๋ฅ ์ธ์ ์ ๊ทผ ์ ํ ๊ธฐ๋ฅ๊น์ง ์ถ๊ฐํ๊ณ  ์ถ๊ณ  ์ด ๊ธฐ๋ฅ๋ค์ ๊ณตํต์ ์ผ๋ก ์ฌ์ฉํ๋ ํ๊น์ด ์ ๋ฐฑ๊ฐ๋ผ๋ฉด ๊ทธ ๊ฐฏ์๋งํผ ์ค์  ํ์ผ์์ 
  ์ถ๊ฐ๋ก ์ค์ ํด ์ค์ผ ๋๊ธฐ ๋๋ฌธ์๋๋ค.
- `TransactionHandler` ์ค๋ธ์ ํธ๋ `FactoryBean`์ ๊ฐ์๋งํผ ๋ง๋ค์ด ์ง๋๋ค. ์์ ์ฝ๋์์ ๋ณด์จ๋ค ์ํผ ํ๊ฒ์ด ๋ฌ๋ผ์ง ๋๋ง๋ค,
๊ณตํต ๊ธฐ๋ฅ์์๋ ๋ถ๊ฐํ๊ณ  ์๋ก `TransactionHandler`๋ฅผ ๋ง๋ค์ด ์ค์ผ ํ์ต๋๋ค.

๋ค์๋ถํฐ๋, ์ด ๋จ์ ๋ค์ ํด๊ฒฐํด๋๊ฐ ๋ด์๋ค!

***

### ๐ Spring Proxy Factory Bean

์คํ๋ง์ ์ผ๊ด๋ ๋ฐฉ๋ฒ์ผ๋ก ํ๋ก์๋ฅผ ๋ง๋ค ์ ์๊ฒ ๋์์ฃผ๋ ์ถ์ ๋ ์ด์ด๋ฅผ ์ ๊ณตํฉ๋๋ค. ์คํ๋ง์ ํ๋ก์ ์ค๋ธ์ ํธ๋ฅผ ์์ฑํด์ฃผ๋ ๊ธฐ์ ์
์ถ์ํํ ํ๋ก์ ํฉํ ๋ฆฌ ๋น์ ์ ๊ณตํ์ฌ ์ค๋๋ค.

์คํ๋ง์ `ProxyFactoryBean`์ ํ๋ก์๋ฅผ ์์ฑํด์ ๋น ์ค๋ธ์ ํธ๋ก ๋ฑ๋กํ๊ฒ ํด์ฃผ๋ ํฉํ ๋ฆฌ ๋น์ด๋ฉฐ,
์์ํ๊ฒ ํ๋ก์๋ฅผ ์์ฑํ๋ ์์๋ง๋ค ๋ด๋นํ๊ฒ ๋ฉ๋๋ค.

๋ถ๊ฐ๊ธฐ๋ฅ๊ณผ ๊ฐ์ ์์์ ๋ณ๋์ ๋น์ ๋ ์ ์์ต๋๋ค.

`ProxyFactoryBean`์ `InvocationHandler`๊ฐ ์๋ `MethodInterceptor`๋ฅผ ์ฌ์ฉํฉ๋๋ค.

๋์ ๊ฐ์ฅ ํฐ ์ฐจ์ด ์ ์
`InvocationHandler`๋ target์ ์ ๋ณด๋ฅผ ์ง์  ์๊ณ  ์์ด์ผ Method๋ฅผ Invokeํ  ์ ์์๋ ๋ฐ๋ฉด์,
```java
@Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if(method.getName().startsWith(pattern))
            return invokeWithTransaction(method,args);
        return method.invoke(target,args);
    }
```

`MethodInterceptor`๋ target์ค๋ธ์ ํธ์ ๋ํ ์ ๋ณด๋ `ProxyFactoryBean`์๊ฒ ์ ๊ณต๋ฐ๊ธฐ ๋๋ฌธ์, ํ๊น์ ๋ํ ์ ๋ณด๋ฅผ ์ง์  ๋ชฐ๋ผ๋ ๋ฉ๋๋ค.
๋ ๋๋ถ์ `MethodInterceptor`๋ ํ๊น๊ณผ ์๊ด ์์ด ๋๋ฆฝ์ ์ผ๋ก ๋ง๋ค ์ ์์ผ๋ฉฐ, ์ฑ๊ธํค ๋น์ผ๋ก๋ ๋ฑ๋ก์ด ๊ฐ๋ฅํฉ๋๋ค.

์ด์ ๊ฐ์ ์ ๋ณด๋ฅผ ๊ธฐ๋ฐ์ผ๋ก ์ฝ๋๋ฅผ ์์ฑํด ๋ด์๋ค

**TransactionAdvice.java**
```java
@RequiredArgsConstructor
public class TransactionAdvice implements MethodInterceptor {
    private final PlatformTransactionManager transactionManager;

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
        try{
            Object ret = invocation.proceed();
            transactionManager.commit(status);
            return ret;
        }catch(RuntimeException e){
            transactionManager.rollback(status);
            throw e;
        }
    }
}

```

**AppConfig.java**
```java
@Configuration
@RequiredArgsConstructor
@EnableTransactionManagement
public class AppConfig {
    private final Environment env;

    //Advice ๋ถ๋ถ ์ค๋ช
    @Bean
    public TransactionAdvice transactionAdvice(){
        return new TransactionAdvice(transactionManager());
    }
    
    //Pointcut ๋ถ๋ถ ์ค๋ช
    //NameMatchMethodPointcut์ ์คํ๋ง ๊ธฐ๋ณธ ์ ๊ณต
    @Bean
    public NameMatchMethodPointcut transactionPointcut(){
        NameMatchMethodPointcut pointcut=new NameMatchMethodPointcut();
        pointcut.setMappedNames("upgrade*");
        return pointcut;
    }

    //Advisor = Advice + Pointcut
    @Bean
    public DefaultPointcutAdvisor transactionAdvisor(){
        return new DefaultPointcutAdvisor(transactionPointcut(),transactionAdvice());
    }
    
    @Bean
    public ProxyFactoryBean userService() {
        ProxyFactoryBean factoryBean = new ProxyFactoryBean();
        factoryBean.setTarget(userServiceImpl());
        factoryBean.setInterceptorNames("transactionAdvisor");
        return factoryBean;
    }
    
    ...
}
```
์์ ์ฝ๋๋ฅผ ๋ณด์๋ค ์ํผ, ํ๊น์๋ํ ์ ๋ณด๋ฅผ ์ง์ ์ ์ผ๋ก ์๊ณ  ์์ง ์์ต๋๋ค. `MethodInvocation`์ด๋ผ๋ ํ๋ผ๋ฏธํฐ๋ก ํ๊น์ ๋ํ
์ ๋ณด์ ๋ฉ์๋์ ๋ํ ์ ๋ณด๊ฐ ํจ๊ป ๋์ด์จ๋ค๊ณ  ์๊ฐํ์๋ฉด ๋ฉ๋๋ค.

#### ๐ Advice ์ด๋๋ฐ์ด์ค
**Target**์ด ํ์ ์๋ ์์ํ ๋ถ๊ฐ๊ธฐ๋ฅ์ ๋ปํฉ๋๋ค.

`MethodInvocation`์ ๋ฉ์๋ ์ ๋ณด์ ํ๊น ์ค๋ธ์ ํธ๊ฐ ๋ด๊ฒจ์๋ ํ๋ผ๋ฏธํฐ์๋๋ค.
`MethodInvocation`์ ํ๊น ์ค๋ธ์ ํธ์ ๋ฉ์๋๋ฅผ ์คํํ  ์ ์๋ ๊ธฐ๋ฅ์ด ์๊ธฐ ๋๋ฌธ์ `MethodInterceptor`๋ ๋ถ๊ฐ๊ธฐ๋ฅ์๋ง
์ง์ค์ ํ  ์ ์์ต๋๋ค.

`MethodInvocation`์ proceed() ๋ฉ์๋๋ฅผ ์คํํ๋ฉด ํ๊ฒ ์ค๋ธ์ ํธ์ ๋ฉ์๋๋ฅผ ๋ด๋ถ์ ์ผ๋ก ์คํํด์ฃผ๋ ๊ธฐ๋ฅ์ด ์์ต๋๋ค.

์ฆ, `MethodInvocation`์ ๊ตฌํํ ํด๋์ค๋ฅผ ํด๋์ค๊ฐ ๊ณต์  ๊ฐ๋ฅํ๊ฒ ์ฌ์ฉ๊ฐ๋ฅํ๋ค๋ ๊ฒ์๋๋ค.

๊ทธ๋ฅ JDK์์์ `ProxyFactoryBean`์ ๋จ์ ์ด์๋ **`TransactionHandler` ์ค๋ธ์ ํธ๋ `FactoryBean`์ ๊ฐ์๋งํผ ๋ง๋ค์ด ์ง๋๋ค. ์์ ์ฝ๋์์ ๋ณด์จ๋ค ์ํผ ํ๊ฒ์ด ๋ฌ๋ผ์ง ๋๋ง๋ค,
๊ณตํต ๊ธฐ๋ฅ์์๋ ๋ถ๊ฐํ๊ณ  ์๋ก `TransactionHandler`๋ฅผ ๋ง๋ค์ด ์ค์ผ ํ์ต๋๋ค.** ์ด ๋ฌธ์ ๋ฅผ ํด๊ฒฐํ  ์ ์๊ฒ ๋์์ต๋๋ค.

๋ํ, `MethodInterceptor`๋ฅผ ๊ตฌํํ `TransactionAdvice`์ ์ด๋ฆ์์ ์ ์ ์๋ฏ์ด 

<span style="color:red; font-weight:bold;">ํ๊ฒ ์ค๋ธ์ ํธ์ ์ ์ฉํ๋ ๋ถ๊ฐ๊ธฐ๋ฅ์ ๋ด์ ์ค๋ธ์ ํธ๋ฅผ ์คํ๋ง์์๋ ์ด๋๋ฐ์ด์ค(Advice)๋ผ๊ณ  ๋ถ๋ฅด๊ฒ ๋ฉ๋๋ค.</span>

๋ง์ง๋ง์ผ๋ก ๋ค๋ฅธ ์ ์ด ์์ต๋๋ค.
`TransactionFactoryBean`์ ์ฌ์ฉํ์ ๋๋ `Dynamic Proxy`๋ฅผ ๋ง๋ค๊ธฐ ์ํด์ ์ธํฐํ์ด์ค ํ์์ ์ ๊ณต๋ฐ์์ผ ํ์ต๋๋ค.

```java
//TransactionFactoryBean
@RequiredArgsConstructor
@Getter
public class TransactionFactoryBean implements FactoryBean<Object> {

    private final Object target;
    private final PlatformTransactionManager transactionManager;
    private final String pattern;
    private final Class<?> interfaces;   //์ด ๋ถ๋ถ

    ...
}

```

ํ์ง๋ง, ์ฐ๋ฆฌ๊ฐ ๊ตฌํํ `Advice`์์๋ ๋ฐ๋ก ์ธํฐํ์ด์ค์ ์ ๋ณด๋ฅผ ์ ๊ณต๋ฐ์ง ์์๋ ๋์์ต๋๋ค. ๊ทธ ์ด์ ๋,
์ธํฐํ์ด์ค์ ์ ๋ณด๋ฅผ ์ ๊ณตํ์ง ์์๋ `ProxyFactoryBean`์๋ ์ธํฐํ์ด์ค๋ฅผ ์๋ ๊ฒ์ถํ๋ ๊ธฐ๋ฅ์ ์ฌ์ฉํ์ฌ ํ๊ฒ ์ค๋ธ์ ํธ๊ฐ
๊ตฌํํ๊ณ  ์๋ ์ธํฐํ์ด์ค ์ ๋ณด๋ฅผ ์์๋ด๊ธฐ ๋๋ฌธ์๋๋ค.

์ด๋ ๊ฒ Advice์ ๋ํด์ ์์๋ณด์์ต๋๋ค. Advice๋ ํ๊ฒ ์ค๋ธ์ ํธ์ ์์ํ ๋ถ๊ฐ๊ธฐ๋ฅ์ ๋ด์ ์ค๋ธ์ ํธ๋ผ๊ณ  ์์๋ฉด ๋ฉ๋๋ค.

#### ๐ Pointcut ํฌ์ธํธ์ปท
**๋ถ๊ฐ๊ธฐ๋ฅ ์ ์ฉ๋์ ๋ฉ์๋ ์ ์  ๋ฐฉ๋ฒ**์ ๋ปํฉ๋๋ค.

`InvocationHandler`๋ฅผ ๊ตฌํํ`TransactionHandler`์์๋ String ๊ฐ์ผ๋ก Pattern์ ์ฃผ์ ๋ฐ์ ๋ถ๊ฐ๊ธฐ๋ฅ์ด ์ ์ฉ๋  ๋์ ๋ฉ์๋๋ฅผ ์ ์  ํ์์ต๋๋ค.

๊ทธ๋ ๋ค๋ฉด `MethodInterceptor`์์๋ ๋๊ฐ์ด pattern์ ์ฃผ์๋ฐ์ ๋ด๋ถ ๋ก์ง์ผ๋ก ์ฒ๋ฆฌํ๋ฉด ๋ ๊น์?? ์๋๋๋ค!!

`MethodInterceptor`๋ ์ฌ๋ฌ ํ๋ก์์์ ๊ณต์ ํด์ ์ฌ์ฉํ  ์ ์์ต๋๋ค. ์ด ๋ง์ ์ฆ, ํ๊ฒ์ ๋ํ ์ ๋ณด๋ฅผ ์ง์ ์ ์ผ๋ก ๊ฐ์ง๊ณ  ์์ง ์๋ค๋ ๋ป๊ณผ ๊ฐ์ต๋๋ค.
๋๋ฌธ์, ์ฑ๊ธํคํํ์ธ ์คํ๋ง ๋น์ผ๋ก๋ ๋ฑ๋กํ  ์ ์์๋ ๊ฒ ์๋๋ค. 

๋ ์์ธํ ๋ณด์๋ฉด, `InvocationHandler`๋ฐฉ์์ ๋ฌธ์ ์ ์ด์๋ `InvocationHandler`๋ฅผ ๊ตฌํํ ํด๋์ค๊ฐ `FactoryBean`์ ๋ง๋ค ๋๋ง๋ค ์๋ก์ด ์ค๋ธ์ ํธ๊ฐ ์์ฑ
๋๋ค๋ ๊ฒ์ด์์ต๋๋ค. ๊ทธ ์ด์ ๋ ํ๊ฒ๋ง๋ค ๋ฉ์๋ ์ ์  ์๊ณ ๋ฆฌ์ฆ์ด๋ ํ๊ฒ ์์ฒด๊ฐ ๋ค๋ฅผ ์ ์๊ธฐ ๋๋ฌธ์ ์ด๋ค ํ๊ฒ์ด๋, ํด๋์ค์ ์ข์๋์ง ์๊ธฐ ์ํด์ ์๋๋ค.

์ด ๋ฌธ์ ๋ฅผ ๊ธฐ๊ป ํ๋ฅญํ ํด๊ฒฐํด ๋จ๋๋ฐ Pattern์ ์ฃผ์ ๋ฐ์ ํ์ฉํ๋ค๋ฉด ๋๋ค์ ์ด๋ค ๋ฉ์๋๋ ํด๋์ค์๋ง ์ข์๋  ์ ๋ฐ์ ์๋ค๋ ๊ฒ์
์๋ฏธํฉ๋๋ค.

์ด๋ฐ ๋ฌธ์ ์ ์ ํด๊ฒฐํ๊ธฐ ์ํด์, ์คํ๋ง์ ๋ถ๊ฐ๊ธฐ๋ฅ์ ์ ๊ณตํ๋ ์ค๋ธ์ ํธ์ธ Advice์ ๋ฉ์๋ ์ ์  ์๊ณ ๋ฆฌ์ฆ ์ค๋ธ์ ํธ์ธ Pointcut์ ๋ฐ๋ก ๋๋์์ต๋๋ค.
Advice์ Pointcut์ ๋ชจ๋ ์ฃผ์์ ๋ฐ์ ์ฌ์ฉํ๋ฉฐ, ๋ ๊ฐ์ง ๋ชจ๋ ์ฌ๋ฌ ํ๋ก์์์ ๊ณต์ ๊ฐ ๊ฐ๋ฅํ๋๋ก ๋ง๋ค์ด์ง๊ธฐ ๋๋ฌธ์ ์คํ๋ง ๋น์ผ๋ก ๋ฑ๋กํ  ์ ์์ต๋๋ค.

์ด์ , ํ๋ก์๋ ํด๋ผ์ด์ธํธ๋ก๋ถํฐ ์์ฒญ์ ๋ฐ์ผ๋ฉด ๋จผ์  Pointcut์๊ฒ ์ ์ฉ ๊ฐ๋ฅํ ๋ฉ์๋์ธ์ง ํ์ธ์ ํ ๋ค, Advice๋ฅผ ํธ์ถํด ์ฃผ๋ฉด ๋ฉ๋๋ค.

๊ฒฐ๊ณผ์ ์ผ๋ก, Advice์ Pointcut์ ๋์์ผ๋ก ์ธํด ์ฌ๋ฌ ํ๋ก์๊ฐ ๊ณต์ ํ๋ฉฐ ์ ์ฐํ๊ฒ ์ฌ์ฉํ  ์ ์๊ฒ ๋์๊ณ , ๊ตฌ์ฒด์ ์ธ ๋ถ๊ฐ๊ธฐ๋ฅ ๋ฐฉ์น์ด๋ ๋ฉ์๋ ์ ์  ์๊ณ ๋ฅด์ง์ด ๋ฐ๋๊ฒ ๋๋ฉด
Advice๋ Pointcut๋ง ๋ฐ๊ฟ์ฃผ๋ฉด ํด๊ฒฐ๋๊ฒ ๋์์ต๋๋ค.

`OCP : Open Closed Priciple`์ ์ ์ง์ผฐ๋ค๊ณ  ๋ณผ ์ ์์ต๋๋ค. 
><a href="https://github.com/sungjin0757/spring-dependency-study"><span style="font-weight:bold;">OCP ๋ ์์ธํ ๋ณด๊ธฐ</span></a>

#### ๐ ์ถ๊ฐ๋ก, Advisor๋?
Advisor๋ Advice์ Pointcut์ ๋ฌถ๋๋ค๊ณ  ๋ณด์๋ฉด ๋ฉ๋๋ค.

๋ฌถ๋ ์ด์ ๋, `ProxyFactoryBean`์ ์ฌ๋ฌ๊ฐ์ง Advice์ Pointcut์ด ์ถ๊ฐ ๋  ์ ์์ต๋๋ค.

์ฌ๊ธฐ์, ๊ฐ๊ฐ์ Advice๋ง๋ค ๋ฉ์๋๋ฅผ ์ ์ ํ๋ ๋ฐฉ์์ด ๋ฌ๋ผ์ง ์๋ ์์ผ๋ ์ด๋ค Pointcut์ ์ ์ฉํ ์ง ์ ๋งคํด์ง ์ ์์๋๋ค. ๊ทธ๋ ๊ธฐ ๋๋ฌธ์ Advice์ Pointcut์ ํ๋๋ก
๋ฌถ์ด์ ์ฌ์ฉํฉ๋๋ค.

***

### ๐ ์คํ๋ง AOP

์ง๊ธ๊น์ง ํด์๋ ๋ฐ์  ๊ธฐ์ ์ ๋ค์ ํ๋ฒ ์ดํด ๋ด์๋ค.

1. `Service` ๋ก์ง์์ `Transaction`๋ถ๊ฐ๊ธฐ๋ฅ์ ๋ถ๋ฆฌ๋ฅผ ์ํด `DynamicProxy`์ `FactoryBean`์ ๋์ํ์์ต๋๋ค.
   - ๋ฌธ์ ์ 
     1. ํ ๋ฒ์ ์ฌ๋ฌ๊ฐ์ ํด๋์ค์ ๊ณตํต์ ์ธ ๋ถ๊ฐ๊ธฐ๋ฅ์ ๋ถ์ฌํ๋ ๊ฒ์ ๋ถ๊ฐ๋ฅํฉ๋๋ค. (`Factory Bean`์ ์ค์ ์ ์ค๋ณต์ ๋ง์ ์ ์๋ค๋ ๊ฒ์ ๋ปํฉ๋๋ค.)
     2. ํ๋์ ํ๊น์ ์ฌ๋ฌ๊ฐ์ง ๋ถ๊ฐ๊ธฐ๋ฅ์ ๋ถ์ฌํ ์๋ก ์ค์  ํ์ผ์ด ๋ณต์กํด์ง๋๋ค.
         - ์๋ฅผ ๋ค์ด, Transaction ๊ธฐ๋ฅ ์ธ์ ์ ๊ทผ ์ ํ ๊ธฐ๋ฅ๊น์ง ์ถ๊ฐํ๊ณ  ์ถ๊ณ  ์ด ๊ธฐ๋ฅ๋ค์ ๊ณตํต์ ์ผ๋ก ์ฌ์ฉํ๋ ํ๊น์ด ์ ๋ฐฑ๊ฐ๋ผ๋ฉด ๊ทธ ๊ฐฏ์๋งํผ ์ค์  ํ์ผ์์
           ์ถ๊ฐ๋ก ์ค์ ํด ์ค์ผ ๋๊ธฐ ๋๋ฌธ์๋๋ค.
     3. `TransactionHandler` ์ค๋ธ์ ํธ๋ `FactoryBean`์ ๊ฐ์๋งํผ ๋ง๋ค์ด ์ง๋๋ค. ์์ ์ฝ๋์์ ๋ณด์จ๋ค ์ํผ ํ๊ฒ์ด ๋ฌ๋ผ์ง ๋๋ง๋ค,
       ๊ณตํต ๊ธฐ๋ฅ์์๋ ๋ถ๊ฐํ๊ณ  ์๋ก `TransactionHandler`๋ฅผ ๋ง๋ค์ด ์ค์ผ ํ์ต๋๋ค.
2. ๋ฌธ์ ์ ์ ํด๊ฒฐํ๊ธฐ ์ํด `SpringProxyFactoryBean`์ ์ฌ์ฉํ์ต๋๋ค.
   - Advice์ ๋์
   - Pointcut์ ๋์
   - Advisor์ ๋์

์ด์ ๊ฐ์ ๊ณผ์ ์ผ๋ก ํฌ๋ชํ ๋ถ๊ฐ๊ธฐ๋ฅ์ ์ ์ฉํ  ์ ์์๊ณ , ํ๊ฒ์๋ ๋น์ฆ๋์ค ๋ก์ง๋ง ์ ์งํ ์ฑ๋ก ๋ ์ ์์์ต๋๋ค.
๋ํ, ๋ถ๊ฐ๊ธฐ๋ฅ์ ํ ๋ฒ๋ง ๋ง๋ค์ด ๋ชจ๋  ํ๊ฒ๊ณผ ๋ฉ์๋์์ ์ฌ์ฌ์ฉ์ด ๊ฐ๋ฅํ  ์ ์๋๋ก ํด๋จ์ต๋๋ค.

**But,** ํ๊ฐ์ง ๋ฌธ์ ์ ์ด ๋ ๋จ์์ต๋๋ค.

๊ทธ๊ฒ์ ๋ฐ๋ก ๋ถ๊ฐ๊ธฐ๋ฅ์ ์ ์ฉ์ด ํ์ํ ํ๊ฒ ์ค๋ธ์ ํธ๋ง๋ค ๊ฑฐ์ ๋น์ทํ ๋ด์ฉ์ `ProxyFactoryBean` ๋น ์ค์ ์ ๋ณด๋ฅผ ์ถ๊ฐํด์ฃผ๋
๋ถ๋ถ์๋๋ค.

```java
@Configuration
@RequiredArgsConstructor
@EnableTransactionManagement
public class AppConfig {
    private final Environment env;
    
    //์ด ๋ถ๋ถ์ด ๊ณ์ํด์ ๋์ด๋๊ฒ ๋ฉ๋๋ค.
    @Bean
    public ProxyFactoryBean userService() {
        ProxyFactoryBean factoryBean = new ProxyFactoryBean();
        factoryBean.setTarget(userServiceImpl());
        factoryBean.setInterceptorNames("transactionAdvisor");
        return factoryBean;
    }
    
    ...
}
```

์์ ๊ฐ์ ์ฝ๋๊ฐ ๊ณ์ํด์ ๋์ด๋๊ฒ ๋ฉ๋๋ค. ๋ฌผ๋ก , ๋จ์ํ๊ณ  ์ฌ์ด ๊ณผ์ ์ด์ง๋ง ๋ง์ฝ ์ ๋ฌํ ์ค๋ธ์ ํธ๊ฐ ์ ๋ฐฑ๊ฐ๊ฐ ๋๊ณ  ์ด๋ ๊ฒ ๋๋ฉด
๊ต์ฅํ ๋ฒ๊ฑฐ๋ก์ด ์์์ผ ๋ฟ๋๋ฌ ์ค์ํ๊ธฐ๋ ์ฝ๊ฒ ๋ฉ๋๋ค.

**์ฆ, ํ ๋ฒ์ ์ฌ๋ฌ ๊ฐ์ ๋น์ ํ๋ก์๋ฅผ ์ ์ฉํด์ผํฉ๋๋ค!**

#### โ๏ธ ๋น ํ์ฒ๋ฆฌ๊ธฐ

๋จผ์ , ๋น ํ์ฒ๋ฆฌ๊ธฐ๋ ์ด๋ฆ ๊ทธ๋๋ก ์คํ๋ง ๋น ์ค๋ธ์ ํธ๋ก ๋ง๋ค์ด์ง๊ณ  ๋ ํ์, ๋น ์ค๋ธ์ ํธ๋ฅผ ๋ค์ ๊ฐ๊ณตํ  ์ ์๊ฒ ํด์ฃผ๋ ๊ฒ์๋๋ค.

์ฌ๊ธฐ์ ์ดํด๋ณผ ๊ฒ์ ๋น์ด ์์ฑ๋ ์ดํ์ Advisor๋ฅผ ์ด์ฉํ ์๋ ํ๋ก์ ์์ฑ๊ธฐ์ธ `DefaultAdvisorAutoProxyCreator`๋ฅผ ์ดํด๋ณผ ์ ์์ต๋๋ค.

`DefaultAdvisorAutoProxyCreator`๊ฐ ๋น ํ์ฒ๋ฆฌ๊ธฐ๋ก ๋ฑ๋ก๋์ด ์์ผ๋ฉด ์คํ๋ง์ ๋น ์ค๋ธ์ ํธ๋ฅผ ๋ง๋ค ๋ ๋ง๋ค ํ์ฒ๋ฆฌ๊ธฐ์๊ฒ ๋น์ ๋ณด๋๋๋ค.
๊ทธ ์ดํ, ๋น ํ์ฒ๋ฆฌ๊ธฐ๋ ๋น์ผ๋ก ๋ฑ๋ก๋ ๋ชจ๋  Advisor๋ด์ ํฌ์ธํธ์ปท์ ์ด์ฉํด ์ ๋ฌ๋ฐ์ ๋น์ด ํ๋ก์ ์ ์ฉ ๋์์ธ์ง ํ์ธํฉ๋๋ค.

ํ๋ก์ ์ ์ฉ ๋์์ด๋ผ๋ฉด, ๋ด์ฅ๋ ํ๋ก์ ์์ฑ๊ธฐ์๊ฒ ํ์ฌ ๋น์ ๋ํ ํ๋ก์๋ฅผ ๋ง๋ค๊ฒ ํ๊ณ , ๋ง๋ค์ด์ง ํ๋ก์์ Advisor๋ฅผ  ์ฐ๊ฒฐํด์ค๋๋ค.

์ด์ , ํ๋ก์๊ฐ ์์ฑ๋๋ฉด ์๋ ์ปจํ์ด๋๊ฐ ์ ๋ฌํด์ค ๋น ์ค๋ธ์ ํธ ๋์  ํ๋ก์ ์ค๋ธ์ ํธ๋ฅผ ์ปจํ์ด๋์๊ฒ ๋๋ ค์ฃผ๊ฒ ๋ฉ๋๋ค.

๊ฒฐ๋ก ์ ์ผ๋ก ์ปจํ์ด๋๋ ํ๋ก์ ์ค๋ธ์ ํธ๋ฅผ ๋น์ผ๋ก ๋ฑ๋กํ๊ณ  ์ฌ์ฉํ๊ฒ ๋ฉ๋๋ค.

์์ ์ค๋ช์ ํ ๋๋ก ๋ถ๊ฐ๊ธฐ๋ฅ์ ๋ถ์ฌํ  ๋น์ ์ ์ฅํ๋ Pointcut์ด ์ฐ๊ฒฐ๋ Advisor๋ฅผ ๋ฑ๋กํ๊ณ , ๋น ํ์ฒ๋ฆฌ๊ธฐ๋ฅผ ์ฌ์ฉํ๊ฒ ๋๋ค๋ฉด
๋ณต์กํ ์ค์ ์ ๋ณด๋ฅผ ์ ์ ํ์์์ด ์๋์ผ๋ก ํ๋ก์๋ฅผ ์์ฑํ  ์ ์๊ฒ ๋ฉ๋๋ค.

**Pointcut์ ํ์ฅ**

์คํ๋ง `ProxyFactoryBean`์ ํ  ๋ Pointcut์์๋ ๋ฉ์๋๋ฅผ ์ด๋ป๊ฒ ํ์ ํ ์ง๋ง ์๊ฐํ์ต๋๋ค. ๊ทผ๋ฐ ๋น ํ์ฒ๋ฆฌ๊ธฐ์์ ์ค๋ชํ ๋ฐ๋ก๋
Pointcut์ผ๋ก ์ด๋ค ๋น์ด ์ ์  ๋์์ด ๋  ๊ฒ์ธ์ง ๊ตฌ๋ณํด์ผํ๋ค๊ณ  ํ๊ณ  ์์ต๋๋ค.

์ด๋ป๊ฒ ๋ ๊ฒ์ผ ๊น์??

Pointcut์ ๊ธฐ๋ฅ์ผ๋ก๋ ์๋ ๋ฉ์๋ ์ ์  ๊ธฐ๋ฅ๋ง ์๋ ๊ฒ์ด ์๋ Class Filter๋ํ ๋ฉ์๋๋ก ๊ฐ๊ณ  ์์ต๋๋ค.

์ฆ, Pointcut์ ํ๋ก์๋ฅผ ์ ์ฉํ  ํด๋์ค์ธ์ง ํ๋จ์ ํ๊ณ ๋์, ์ ์ฉ๋์ ํด๋์ค์ ๊ฒฝ์ฐ์๋ Advice๋ฅผ ์ ์ฉํ  ๋ฉ์๋์ธ์ง ํ์ธํ๋ ๋ฐฉ๋ฒ์ผ๋ก ๋์ํฉ๋๋ค.
๊ฒฐ๊ตญ์ ์ด ๋์กฐ๊ฑด ๋ชจ๋๋ฅผ ๋ง์กฑํ๋ ํ๊ฒ์๊ฒ๋ง ๋ถ๊ฐ๊ธฐ๋ฅ์ด ๋ถ์ฌ๋๋ ๊ฒ์๋๋ค.

์์ ์์๋ณด์๋ ๋น ํ์ฒ๋ฆฌ๊ธฐ์ธ `DefaultAdvisorAutoCreator`์์๋ ํด๋์ค์ ๋ฉ์๋ ์ ์ ์ด ๋ชจ๋ ๊ฐ๋ฅํ Pointcut์ด ํ์ํฉ๋๋ค.

**NameMatchMethodPointcut**
์์์ ๋ฑ๋กํ์๋ Pointcut์ ์ดํด๋ด์๋ค.

```java
    @Bean
    public NameMatchMethodPointcut transactionPointcut(){
        NameMatchMethodPointcut pointcut=new NameMatchMethodPointcut();
        pointcut.setMappedNames("upgrade*");
        return pointcut;
    }
```

์์ ๊ฐ์ ๋ฐฉ์์ผ๋ก ๋ฑ๋ก์ ํ์์ต๋๋ค. ์ฌ๊ธฐ์ ์คํ๋ง์ด ๊ธฐ๋ณธ ์ ๊ณตํ๋ `NameMethodPointcut`์
๋ฉ์๋ ์ ์  ๊ธฐ๋ฅ๋ง์ ๊ฐ๊ณ ์์ ๋ฟ, ํด๋์ค ํํฐ์ ๊ธฐ๋ฅ์ ์กด์ฌํ์ง ์์ต๋๋ค.

๋ฐ๋ผ์, ์ด ํด๋์ค๋ฅผ ํ์ฅํ์ฌ ํด๋์ค ํํฐ์ ๊ธฐ๋ฅ์ผ๋ก์๋ ์๋ํ๋๋ก ๋ง๋ค์ด ๋ด์๋ค.

**NameMathClassMethodPointcut.java**
```java
public class NameMatchClassMethodPointcut extends NameMatchMethodPointcut {

    public void setMappedClassName(String mappedClassName){
        this.setClassFilter(new SimpleFilter(mappedClassName));
    }

    @RequiredArgsConstructor
    static class SimpleFilter implements ClassFilter{
        private final String mappedName;

        @Override
        public boolean matches(Class<?> clazz) {
            return PatternMatchUtils.simpleMatch(mappedName, clazz.getSimpleName());
        }
    }
}

```

```java
public void setMappedClassName(String mappedClassName){
        this.setClassFilter(new SimpleFilter(mappedClassName));
    }
```
์ฌ๊ธฐ์ `setClassFilter`๊ฐ ์ด๋ป๊ฒ ๋์๋์ง ์์ํ์ค์๋ ์์ต๋๋ค.

์๋๋ฉด, `NameMatchMethodPointcut`๋ํ `Pointcut`์ด๋ผ๋ ์ธํฐํ์ด์ค๋ฅผ implementsํ๊ธฐ ๋๋ฌธ์๋๋ค.

```java
public interface Pointcut {
    Pointcut TRUE = TruePointcut.INSTANCE;

    ClassFilter getClassFilter();

    MethodMatcher getMethodMatcher();
}
```
```java
public abstract class StaticMethodMatcherPointcut extends StaticMethodMatcher implements Pointcut {
    private ClassFilter classFilter;

    public StaticMethodMatcherPointcut() {
        this.classFilter = ClassFilter.TRUE;
    }

    public void setClassFilter(ClassFilter classFilter) {
        this.classFilter = classFilter;
    }

    public ClassFilter getClassFilter() {
        return this.classFilter;
    }

    public final MethodMatcher getMethodMatcher() {
        return this;
    }
}
```

```java
public class NameMatchMethodPointcut extends StaticMethodMatcherPointcut implements Serializable {
    private List<String> mappedNames = new ArrayList();

    public NameMatchMethodPointcut() {
    }

    public void setMappedName(String mappedName) {
        this.setMappedNames(mappedName);
    }

    public void setMappedNames(String... mappedNames) {
        this.mappedNames = new ArrayList(Arrays.asList(mappedNames));
    }

    public NameMatchMethodPointcut addMethodName(String name) {
        this.mappedNames.add(name);
        return this;
    }

    public boolean matches(Method method, Class<?> targetClass) {
        Iterator var3 = this.mappedNames.iterator();

        String mappedName;
        do {
            if (!var3.hasNext()) {
                return false;
            }

            mappedName = (String)var3.next();
        } while(!mappedName.equals(method.getName()) && !this.isMatch(method.getName(), mappedName));

        return true;
    }

    protected boolean isMatch(String methodName, String mappedName) {
        return PatternMatchUtils.simpleMatch(mappedName, methodName);
    }

    public boolean equals(@Nullable Object other) {
        return this == other || other instanceof NameMatchMethodPointcut && this.mappedNames.equals(((NameMatchMethodPointcut)other).mappedNames);
    }

    public int hashCode() {
        return this.mappedNames.hashCode();
    }

    public String toString() {
        return this.getClass().getName() + ": " + this.mappedNames;
    }
}
```
์ด๋ฐ ์์ผ๋ก ๊ตฌ์ฑ๋์ด ์์ต๋๋ค.  ์ด๋ก์จ, ํด๋์ค ํํฐ๊ธฐ๋ฅ ๋ํ ๊ฐ์ง Pointcut์ ์์ฑ ์๋ฃํ์์ต๋๋ค.

์ด์ , Advisor๋ฅผ ์ด์ฉํ๋ ์๋ ํ๋ก์ ์์ฑ๊ธฐ๊ฐ ์ด๋ค ์์๋ก ์๋์ ํ๋์ง ์์๋ณด๊ณ  ์ฝ๋๋ฅผ ์์ฑํด๋ด์๋ค.

1. `DefaultAdvisorAutoProxyCreator`๋ ๋ฑ๋ก๋ ๋น ์ค์์ Advisor์ธํฐํ์ด์ค๋ฅผ ๊ตฌํํ ๊ฒ์ ๋ชจ๋ ์ฐพ์ต๋๋ค.
2. Advisor์ Pointcut์ ์ ์ฉํด๋ณด๋ฉด์ ๋ชจ๋  ๋น์ ๋ํ์ฌ ํ๋ก์ ์ ์ฉ ๋์์ ์ ์ ํฉ๋๋ค.
3. ๋น์ด ํ๋ก์ ์ ์ฉ ๋์์ด๋ผ๋ฉด ํ๋ก์๋ฅผ ๋ง๋ค์ด ์๋ ๋น ์ค๋ธ์ ํธ์ ๋ฐ๊ฟ๋๋ค.
4. ์ด์  ์๋ ๋น์ ํ๋ก์๋ฅผ ํตํด ์ ๊ทผ๊ฐ๋ฅํ๋๋ก ์ค์ ์ด ์๋ฃ ๋ฉ๋๋ค.

๐ **์ฐธ๊ณ ๋ก! `DefaultAdvisorAutoProxyCreator`๋ ๋น์ผ๋ก๋ง ๋ฑ๋กํด ๋์๋ฉด ๋ฉ๋๋ค.**

**AppConfig.java**
```java
@Configuration
@RequiredArgsConstructor
@EnableTransactionManagement
public class AppConfig {

    private final Environment env;

    @Bean
    public DataSource dataSource(){
        SimpleDriverDataSource dataSource=new SimpleDriverDataSource();
        dataSource.setDriverClass(org.h2.Driver.class);
        dataSource.setUrl(env.getProperty("spring.datasource.url"));
        dataSource.setUsername(env.getProperty("spring.datasource.username"));
        dataSource.setPassword(env.getProperty("spring.datasource.password"));

        return dataSource;
    }

    @Bean
    public PlatformTransactionManager transactionManager(){
        return new DataSourceTransactionManager(dataSource());
    }

    @Bean
    public JdbcOperations jdbcOperations(){
        return new JdbcTemplate(dataSource());
    }

    @Bean
    public UserDao userDao(){
        return new UserDaoImpl(jdbcOperations());
    }

    @Bean
    public TransactionAdvice transactionAdvice(){
        return new TransactionAdvice(transactionManager());
    }

    @Bean
    public NameMatchClassMethodPointcut transactionPointcut(){
        NameMatchClassMethodPointcut pointcut=new NameMatchClassMethodPointcut();
        pointcut.setMappedClassName("*ServiceImpl");
        pointcut.setMappedNames("upgrade*");
        return pointcut;
    }

    @Bean
    public DefaultPointcutAdvisor transactionAdvisor(){
        return new DefaultPointcutAdvisor(transactionPointcut(),transactionAdvice());
    }
    
    //๋น ํ์ฒ๋ฆฌ๊ธฐ ๋ฑ๋ก
    @Bean
    public DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator(){
        return new DefaultAdvisorAutoProxyCreator();
    }

    @Bean
    public UserService userService(){
        return new UserServiceImpl(userDao(),mailSender());
    }


}
```

์ฝ๋ ์์ฑ๋ํ ์๋ฃํ์ผ๋ฉฐ, ๋น ํ์ฒ๋ฆฌ๊ธฐ๋ฅผ ํตํ ๋ถ๊ฐ๊ธฐ๋ฅ ๋ถ์ฌ ๋ํ ์๋ฒฝํ ์ํํ์์ต๋๋ค.

***

### ๐ Pointcut ํํ์

์ง๊ธ๊น์ง ๋ฐ์ ํด์จ ๊ณผ์ ์ ์ดํด๋ณด๋ฉด, ์ผ์ผ์ด ํด๋์ค ํํฐ์ ๋ฉ์๋ ๋งค์ฒ๋ฅผ ๊ตฌํํ๊ฑฐ๋ ๊ธฐ๋ณธ์ ์ผ๋ก ์คํ๋ง์ด ์ ๊ณตํ๋ ๊ธฐ๋ฅ์ ์ฌ์ฉํด์์ต๋๋ค.

์ง๊ธ๊น์ง๋ ๋จ์ํ ํด๋์ค ์ด๋ฆ์ด๋ ๋ฉ์๋ ์ด๋ฆ์ ๋น๊ตํ๋ ๊ฒ์ด ์ ๋ถ์๋ค๋ฉด, ์ผ์ข์ ํํ์ ์ธ์ด๋ฅผ ์ฌ์ฉํ์ฌ ์ข ๋ ์ธ๋ฐํ๊ฒ ์ ์  ์๊ณ ๋ฆฌ์ฆ์ ์งค ์ ์์ต๋๋ค.

์ด๋ ๊ฒ ๊ณ ์๋ ๊ฒ์ด ํฌ์ธํธ์ปท ํํ์์ด๋ผ๊ณ  ํฉ๋๋ค.

ํฌ์ธํธ์ปท ํํ์์ `AspectJExpressionPointcut` ํด๋์ค๋ฅผ ์ฌ์ฉํ๋ฉด ๋ฉ๋๋ค.

`NameMatchClassMethodPointcut`์ ํด๋์ค์ ๋ฉ์๋์ ์ด๋ฆ์ ๊ฐ๊ฐ ๋๋ฆฝ์ ์ผ๋ก ๋น๊ตํ ๋ฐ๋ฉด์, ํํ์์ผ๋ก๋ ํ๋ฒ์ ์ง์ ๊ฐ๋ฅํ๊ฒ ํด์ค๋๋ค.

`AspectJExpressionPointcut`์ ์ด๋ฆ์ ๋ณด๋ค์ํผ, ์คํ๋ง์ `AspectJ`๋ผ๋ ํ๋ ์์ํฌ์์ ์ ๊ณตํ๋ ๊ฒ์ ์ฌ์ฉํ๊ฒ ๋๋ฉฐ, ์ด๊ฒ์ AspectJํํ์์ด๋ผ ๋ถ๋ฆ๋๋ค.

#### ๐ ํฌ์ธํธ์ปท ํํ์ ๋ฌธ๋ฒ

AspectJ ํฌ์ธํธ์ปท ํํ์์ ํฌ์ธํธ์ปท ์ง์์๋ฅผ ์ด์ฉํด ์์ฑํฉ๋๋ค. ํฌ์ธํธ์ปท ์ง์์์ค์์ ๊ฐ์ฅ ๋ํ์ ์ผ๋ก ์ฌ์ฉ๋๋ ๊ฒ์ execution์๋๋ค.

`execution(์ ๊ทผ์ ํ์ ํ์ํจํด:return ํ์ ํ์ํจํด:ํด๋์ค ํ์.์ด๋ฆํจํด(๋ฉ์๋) (ํ์ํจํด:ํ๋ผ๋ฏธํฐํจํด) throws ์์ธํจํด) `

1. : ๋ ์ค๋ช์ ์๋ฏธํฉ๋๋ค.
2. ์ ๊ทผ์ ํ์, ํด๋์ค ํ์ํจํด, ์์ธํจํด ๋ฑ์ ์๋ต๊ฐ๋ฅํฉ๋๋ค.

**๋ฌธ๋ฒ์ ๋ํ ์์ธํ ์ค๋ช์ ์๋ตํ๋๋ก ํ๊ฒ ์ต๋๋ค!**

์ด์ ๋ Pointcutํํ์์ AspectJ ๋ฉ์๋์ ํ๋ฆฌ๋ฏธํฐ๋ก ํ๋ฉด ์คํ ๊ฐ๋ฅํ๊ฒ ๋ฉ๋๋ค.

**Pointcut์ ์ ์ฉํด ๋ณด๋๋ก ํ๊ฒ ์ต๋๋ค.**

ํฌ์ธํธ์ปท ํํ์์๋ ์์์ ์ ์ ์ธ๊ธํ๋ execution ์ธ์๋ bean์ ์ ํํ์ฌ์ฃผ๋ bean()๋ฉ์๋, ๋ํ ํน์  ์ ๋ธํ์ด์์ด ํ์, ๋ฉ์๋, ํ๋ผ๋ฏธํฐ์ ์ ์ฉ๋์ด ์๋ ๊ฒ์ ๋ณด๊ณ 
๋ฉ์๋๋ฅผ ์ ์ ํ๊ฒ ํ๋ ํฌ์ธํธ์ปท์ ๋ง๋ค ์ ์์ต๋๋ค. ์๋ฅผ ๋ค๋ฉด `@Transactional` ๊ณผ ๊ฐ์ ๊ฒฝ์ฐ๋ฅผ ๋งํฉ๋๋ค.

ํฌ์ธํธ์ปท ํํ์์ AspectJExpressionPointcut๋น์ ๋ฑ๋กํ๊ณ  expression ํ๋กํผํฐ์ ๋ฃ์ด์ฃผ๋ฉด ๋ฉ๋๋ค. ํด๋์ค์ด๋ฆ์ ServiceImpl๋ก ๋๋๊ณ  ๋ฉ์๋ ์ด๋ฆ์
upgrade๋ก ์์ํ๋ ๋ชจ๋  ํด๋์ค์ ์ ์ฉ๋๋๋ก ์ฝ๋๋ฅผ ์ง๋ด์๋ค.

**AppConfig.java**
```java
@Configuration
@RequiredArgsConstructor
@EnableTransactionManagement
public class AppConfig {

    private final Environment env;

    @Bean
    public DataSource dataSource(){
        SimpleDriverDataSource dataSource=new SimpleDriverDataSource();
        dataSource.setDriverClass(org.h2.Driver.class);
        dataSource.setUrl(env.getProperty("spring.datasource.url"));
        dataSource.setUsername(env.getProperty("spring.datasource.username"));
        dataSource.setPassword(env.getProperty("spring.datasource.password"));

        return dataSource;
    }

    @Bean
    public PlatformTransactionManager transactionManager(){
        return new DataSourceTransactionManager(dataSource());
    }

    @Bean
    public JdbcOperations jdbcOperations(){
        return new JdbcTemplate(dataSource());
    }

    @Bean
    public UserDao userDao(){
        return new UserDaoImpl(jdbcOperations());
    }

    @Bean
    public TransactionAdvice transactionAdvice(){
        return new TransactionAdvice(transactionManager());
    }

    //์ถ๊ฐ๋ ๋ถ๋ถ
    @Bean
    public AspectJExpressionPointcut transactionPointcut(){
        AspectJExpressionPointcut pointcut=new AspectJExpressionPointcut();
        pointcut.setExpression("bean(*Service)");
        return pointcut;
    }

    @Bean
    public DefaultPointcutAdvisor transactionAdvisor(){
        return new DefaultPointcutAdvisor(transactionPointcut(),transactionAdvice());
    }
    
    //๋น ํ์ฒ๋ฆฌ๊ธฐ ๋ฑ๋ก
    @Bean
    public DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator(){
        return new DefaultAdvisorAutoProxyCreator();
    }

    @Bean
    public UserService userService(){
        return new UserServiceImpl(userDao(),mailSender());
    }


}
```

์ง๊ธ๊น์ง์ ๊ณผ์ ์ ๊ฑฐ์ณ `AspectJExpressionPointcut`์ ์ ์ฉ๊น์ง ์๋ฃํ์ต๋๋ค.

***

### ๐ ๋ง์ง๋ง์ผ๋ก, AOP๋?! 

์ผ๋ฐ์ ์ธ ๊ฐ์ฒด์งํฅ ๊ธฐ์  ๋ฐฉ๋ฒ์ผ๋ก๋ ๋๋ฆฝ์ ์ธ ๋ชจ๋ํ๊ฐ ๋ถ๊ฐ๋ฅํ๊ฒ ๋ฉ๋๋ค. ๋ฐ๋ผ์, ๋ถ๊ฐ๊ธฐ๋ฅ ๋ชจ๋ํ ์์์
๊ธฐ์กด์ ๊ฐ์ฒด์งํฅ ์ค๊ณ์๋ ๋ค๋ฅธ์๋ฏธ๊ฐ ์๋ค๋ ๋ป์ ๋ฐ์๋ค์ฌ `Aspect`๋ผ๋ ์ด๋ฆ์ผ๋ก ๋ถ๊ฐ๊ธฐ๋ฅ ๋ชจ๋ํ ์์์ ๋ป๋ผ๊ฒ ๋์์ต๋๋ค.

`Aspect`๋ ํต์ฌ ๋น์ฆ๋์ค๋ก์ง ๋ฐ ๊ธฐ๋ฅ์ ๊ฐ์ง๊ณ  ์์ง๋ ์์ง๋ง ์ง๊ธ๊น์ง ํด์๋ ํธ๋์ญ์ ๊ฒฝ๊ณ ์ค์  ๋ถ๊ฐ๊ธฐ๋ฅ์ ์ถ๊ฐํ๋ค๋์ง ํต์ฌ๊ธฐ๋ฅ์ ๋ถ๊ฐ๋๋ ๋ชจ๋์ ์ง์นญํฉ๋๋ค.

์ด๋ ๊ฒ ์ ํ๋ฆฌ์ผ์ด์์ ํต์ฌ์ ์ธ ๊ธฐ๋ฅ์์ ๋ถ๊ฐ๊ธฐ๋ฅ์ ๋ถ๋ฆฌํด์ `Aspect`๋ผ๋ ๋ชจ๋๋ก ๋ง๋ค์ด์ ๊ฐ๋ฐํ๋ ๋ฐฉ๋ฒ๋ก ์ `Aspect Oriented Progreamming` **AOP**๋ผ๊ณ  ๋ถ๋ฆ๋๋ค.

ํ ๊ฐ์ง ์ ์ํ์ค์ ์, AOP๋ OOP์์ ๋ถ๋ฆฌ๋ ์๋ก์ด ๊ฐ๋์ ๊ฐ๋ฐ๋ก ์ด ์๋ OOP๋ฅผ ๋๋ ๋ณด์กฐ์ ์ธ ๊ธฐ์ ์ด๋ผ๊ณ  ๋ณด์๋ฉด ๋  ๊ฒ ๊ฐ์ต๋๋ค.

์ฆ, **AOP**๋ `Aspect`๋ฅผ ๋ถ๋ฆฌํจ์ผ๋ก์จ ํต์ฌ ๋ก์ง์ ๊ตฌํํ๋๋ฐ ๋ถ๋ด์ด ์๋๋ก ๋ํ ์ต๋ํ ๊ฐ์ฒด์งํฅ ๊ธฐ์ ์ ์ ์งํ๋๋ก ๋๋ ๊ฐ๋ฐ๋ก ์ด๋ผ๊ณ  ํ  ์ ์์ต๋๋ค.

#### ๐ AOP ์ ์ฉ๊ธฐ์ 

**ํ๋ก์๋ฅผ ์ด์ฉํ AOP**

์ง๊ธ๊น์ง ์ ํฌ๊ฐ ํด์๋ ๋ฐฉ์์๋๋ค. ํ๋ก์๋ก ๋ง๋ค์ด์ DI๋ก ์ฐ๊ฒฐ๋ ๋น ์ฌ์ด์ ์ ์ฉํด ํ๊ฒ์ ๋ฉ์๋ ํธ์ถ ๊ณผ์ ์ ์ฐธ์ฌํด์ ๋ถ๊ฐ๊ธฐ๋ฅ์ ์ ๊ณตํ์ฌ ์ฃผ๋ ๊ฒ์ ๋งํฉ๋๋ค.

๋๋ฆฝ์ ์ผ๋ก ๊ฐ๋ฐํ ๋ถ๊ฐ๊ธฐ๋ฅ ๋ชจ๋์ ๋ค์ํ ํ๊ฒ ์ค๋ธ์ ํธ์ ๋ฉ์๋์ ๋ค์ด๋ด๋ฏนํ๊ฒ ์ ์ฉํด์ฃผ๊ธฐ ์ํด ๊ฐ์ฅ ์ค์ํ ์ญํ ์ ํฉ๋๋ค. ๋ฐ๋ผ์, ์คํ๋ง AOP๋
ํ๋ก์ ๋ฐฉ์์ AOP๋ผ๊ณ ํ  ์ ์์ต๋๋ค.

**๋ฐ์ดํธ์ฝ๋ ์์ฑ๊ณผ ์กฐ์์ ํตํ AOP**

AOPํ๋ ์์ํฌ์ ๋ํ๊ฒฉ์ธ `AspectJ`๋ ํ๋ก์๋ฅผ ์ฌ์ฉํ์ง ์๋ ๋ํ์ ์ธ AOP ๊ธฐ์ ์๋๋ค.

`AspectJ`๋ ํ๋ก์ ์ฒ๋ผ ๊ฐ์ ์ ์ธ ์ญํ ์ ํ๋ ๊ฒ์ด ์๋๋ผ ์ง์  ํ๊ฒ ์ค๋ธ์ ํธ์ ๋ถ๊ฐ๊ธฐ๋ฅ์ ๋ฃ์ด์ฃผ๋ ๋ฐฉ๋ฒ์ ์ฌ์ฉํฉ๋๋ค. ๊ทธ๋ ๋ค๊ณ  ํ๋๋ผ๋ ๋ถ๊ฐ๊ธฐ๋ฅ ์ฝ๋๋ฅผ ์ง์  ๋ฃ์์๋ ์์ผ๋, 
์ปดํ์ผ๋ ํ๊ฒ์ ํด๋์ค ํ์ผ ์์ฒด๋ฅผ ์์ ํ๊ฑฐ๋ ํด๋์ค๊ฐ JVM์ ๋ก๋ฉ๋๋ ์์ ์ ๊ฐ๋ก์ฑ์ ๋ฐ์ดํธ์ฝ๋๋ฅผ ์กฐ์ํ๋ ๋ณต์กํ ๋ฐฉ๋ฒ์ ์ฌ์ฉํฉ๋๋ค.

์ด์ ๊ฐ์ด ๋ฒ๊ฑฐ๋ก์ด ์ด์ ๋ฅผ ํ๋ ์ด์ ๋ 
1. DI์ ๋์์ ๋ฐ์ง ์์๋ ๋ฉ๋๋ค.
   - ํ๊ฒ ์ค๋ธ์ ํธ๋ฅผ ์ง์  ์์ ํ๋ ๋ฐฉ์์ด๊ธฐ ๋๋ฌธ์๋๋ค.
2. ํจ์ , Detailํ๊ณ  ์ ์ฐํ๊ฒ AOP๋ฅผ ์ ์ฉํ  ์ ์์ต๋๋ค.
   - ๋ฐ์ดํธ ์ฝ๋๋ฅผ ์ง์  ์กฐ์ํจ์ผ๋ก์ AOP๋ฅผ ์ ์ฉํ๋ฉด ์ค๋ธ์ ํธ์ ์์ฑ, ํ๋ ๊ฐ์ ์กฐํ์ ์กฐ์๋ฑ ๋ค์ํ ๋ถ๊ฐ๊ธฐ๋ฅ ๋ถ์ฌ๊ฐ ๊ฐ๋ฅํฉ๋๋ค.

๊ฑฐ์ ์ผ๋ฐ์ ์ธ ๊ฒฝ์ฐ๋ด์์๋ ํ๋ก์๋ฅผ ํตํด์ ๊ฐ๋ฅํ์ง๋ง, ์ข ๋ ํน๋ณํ ์ํฉ์ด ํ์ํ ๊ฒฝ์ฐ์ ์ด์๊ฐ์ ๋ฐฉ๋ฒ์ ์ด๋ค๊ณ  ํฉ๋๋ค.

***
### ๋๋ง์น๋๋ก ํ๊ฒ ์ต๋๋ค. ๐