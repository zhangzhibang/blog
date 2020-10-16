---
title: Shiro框架
---

maven依赖

```xml
<!-- https://mvnrepository.com/artifact/org.apache.shiro/shiro-core -->
<dependency>
    <groupId>org.apache.shiro</groupId>
    <artifactId>shiro-core</artifactId>
    <version>1.4.0</version>
</dependency>
```



shiro三大核心：Realm、SecurityManager、Subject

Realm是存储着权限和角色的对象，SecurityManager是管理者，Subject是对应的用户。

1. Relam

是一个需要实现的接口，接口名称AuthorizingRealm。

```java
public class DatabaseRealm extends AuthorizingRealm {
 
    /**
     * 获取账号的角色、资源
     *
     * @param principalCollection
     * @return
     */
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
        //能进入到这里，表示账号已经通过验证了,获取用户名
        String userName = (String) principalCollection.getPrimaryPrincipal();
 
        //根据userName获取该账户拥有的角色（这个逻辑自己用自己的数据库搞定）
        Set<String> setRoles = getRolesByUserName(userName);
        //根据角色获取该账户拥有的资源（这个也是，自己用自己的数据库搞定）
        Set<String> setPermissions = getPermissionsByRoles(setRoles);
 
        //创建授权对象
        SimpleAuthorizationInfo info = new SimpleAuthorizationInfo();
        //把获取到的角色和资源放进去
        info.setStringPermissions(setPermissions);
        info.setRoles(setRoles);
        return info;
    }
 
    /**
     * 根据token验证账户登录
     *
     * @param token
     * @return 认证信息
     * @throws AuthenticationException 如果登录失败，抛出这个错误
     */
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
        //根据token获取认证的账号密码
        UsernamePasswordToken usernamePasswordToken = (UsernamePasswordToken) token;
        String userName = token.getPrincipal().toString();
        String password = new String(usernamePasswordToken.getPassword());
        //判断数据库中是存在该用户（这个逻辑自己从数据库中拿出来）
        //（这个也自己搞定）
        if (账号不存在) {
            throw new AuthenticationException();
        }
        //认证信息里存放账号密码, getName() 是当前Realm的继承方法,通常返回当前类名 :databaseRealm
        SimpleAuthenticationInfo info = new SimpleAuthenticationInfo(userName, password, getName());
        return info;
    }
}
```

​	2.SecurityManager

将获取到的存储有权限和角色的对象授权给管理员进行管理

```Java
public class SecurityManager {
 
    //初始化管理者(用它管控所有的权限判定)，单例模式
    public static final SecurityManager securityManager = SecurityManager.VOID();
 
    //new一个对象 ^_^
    public static SecurityManager VOID() {
        return new SecurityManager();
    }
 
    private SecurityManager() {
        //初始化
        init();
    }
 
    //初始化
    private void init() {
        //初始化管理者
        DefaultSecurityManager defaultSecurityManager = new DefaultSecurityManager();
        //将数据库实现类绑定到管理者
        defaultSecurityManager.setRealm(new DatabaseRealm());
        //将 SecurityManager 实例绑定给 SecurityUtils
        SecurityUtils.setSecurityManager(defaultSecurityManager);
    }
 
    //获取当前线程主题
    private Subject getSubject() {
        return SecurityUtils.getSubject();
    }
 
    //获取登录用的token
    private UsernamePasswordToken getToken(String userName, String password) {
        return new UsernamePasswordToken(userName, password);
    }
 
    //传入用户id，返回用户的Subject
    private Subject getSubject(String username,String password) {
        //获取主题
        Subject subject = getSubject();
        //认证登录
        try {
            subject.login(getToken(username,password));
        } catch (AuthenticationException e) {
            sout回车("用户验证身份时出现异常:" + e);
        }
        return subject;
    }
 
    //外部通用方法-判断用户是否为某个角色
    public boolean isRole(String username,String password, String role) {
        //获取用户身份subject
        Subject subject = getSubject(username,password);
        //判断角色
        boolean isRole = subject.hasRole(role);
        //登出
        subject.logout();
        return isRole;
    }
 
    //外部通用方法-判断用户是否拥有某个资源
    public boolean isPermitted(String username,String password, String permission) {
        //获取用户身份subject
        Subject subject = getSubject(username,password);
        //判断是否拥有资源
        boolean isPermitted = subject.isPermitted(permission);
        //登出
        subject.logout();
        return isPermitted;
    }
 
}
```

