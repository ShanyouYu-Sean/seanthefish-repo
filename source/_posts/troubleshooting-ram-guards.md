---
layout: post
title: 一次艰难的troubleshooting
date: 2020-5-11 10:00:00
tags: 
- OAuth
categories:
- OAuth
---

# 一次艰难的troubleshooting

## 起因

先说一下什么是ram-guards，这是我们的一个微服务的权限处理系统，接受一个oauth idtoken并认证，然后返回一个能代表权限的token。

起因是最近对ram-guards做升级，增加了切换application id的功能，使ram-guards可以通过ram系统为不同的app提供不同的权限服务（以前只服务于一个项目）。

本身这只是个微小的改动，加一个参数就好，但是在spring security的设计体系中，下面的这个简单的接口承担着装配用户以及权限的核心

```java
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

但这个接口有个很大的缺点，就是只有一个string做参数，为了满足我们多个参数的需求，又不想丑陋的在string里split，我决定把这个接口用我自定义的接口代替。简单来说就是写一个能接收自定义object的接口，并为这个自定义接口提供一个AuthenticationProvider，注册到spring security的AuthenticationManager中，同时重写DefaultTokenServices中refreshtoken所用的PreAuthenticatedAuthenticationProvider，并将tokenservice和AuthenticationManager注册到AuthorizationServerEndpointsConfigurer中。

因为这部分改动不是本文的重点，在这里不展开说了。

从代码角度来说我对这部分改动还是很自信的，因为自己经过多轮测试并且其他team也一直在使用这套代码进行权限获取，但就是这些很自信的改动，在dev环境上跑出了问题。

## troubleshooting

新版本的ram-guards上了dev环境之后，测试在dev上跑了bdd，结果大批量的fail掉了，报的错误是前端没有拿到用户的权限。但是奇怪的是，手工测试却并没有出现问题，而且老版本的ram-guards bdd也从来没有fail过。因为对自己的盲目自信以及function测试并没有出现问题，我最先把怀疑指向了bdd的代码会不会有问题，但经过一番查验，并没有发现bdd的代码有问题。

此时我便陷入了僵局，一般情况下，系统有问题通常都会要求有稳定的复现，但这是一个复现不了，只会在bdd中才发生的问题。在不断的查验中，终于意识到bdd场景是并发进行的，十多个bdd脚本在一秒内并发，这是手工测试所不能复现的场景，会不会是新版本代码在并发场景下会出问题？带着这种怀疑，我决定做一次压测。

压测工具为ab，进行API压测，以单秒并发100次请求为标准进行压测，果然，dev上的新版本出现了大批量的API fail，test上的老版本则全部通过。

既然是压测出的问题，首先想到的肯定是不同环境上的资源分配是不是影响因素，于是便在dev上也部署了一份老版本ram-guards的代码到同一个node上，在进行压测，发现并没有问题。

那这样来看就不是环境资源配置的问题了，那代码会哪里出问题呢，功能都OK，但一上并发就挂。我便仔细分析了下log，发现压测中，最先挂的是数据库连接池，总会报出数据库连接失败的错误。先说明下，ram-guards本身是不需要数据库的支持的，这里的数据库只是用来记录audit log用的。我们处于成本控制，使用了最大连结数只能有100个的cloud db，而每个service都有记录audit log的需求，所以我限制了每个service在log db上只能开2个active连接的连接池。那么会不会是连接池数量太少引起的性能问题呢？

如果仅仅是连接池数量太少导致的，那么老版本的代码在并发场景下应该也会出问题，虽然增加连接数也可能解决问题，但这并不像是rootcause。那会不会是代码优化后执行速度过快，导致单秒写入数据库的请求增多，进而把连接池压垮？亦或者是代码重构后，写入数据库的次数和内容增多，导致单个commit处理时间过长，把数据库连接池压垮？翻来覆去没想出问题所在，我决定开debug log看一下，因为全局的debug log太多，反而不好定位问题，这里我只开启跟log db有关的debug：

```yaml
logging:
  level:
    ROOT: info
    com.ibm.ram.guards: debug
    org.springframework.orm.jpa: debug
    org.hibernate: debug
    org.postgresql: debug
    com.zaxxer.hikari: debug
```

在开启debug后，仔细对着了log的不同，发现了蹊跷。

这是老版本代码在并发场景下的log：

```log
2020-06-16 07:13:49.971 DEBUG 1 --- [nio-8080-exec-5] org.postgresql.jdbc.PgConnection         :   setAutoCommit = false
2020-06-16 07:13:49.982 DEBUG 1 --- [nio-8080-exec-5] com.zaxxer.hikari.pool.PoolBase          : HikariPool-1 - Reset (autoCommit) on connection org.postgresql.jdbc.PgConnection@100d71db
2020-06-16 07:13:49.971  INFO 1 --- [nio-8080-exec-5] g.a.o.c.RamGuardsJweAccessTokenConverter : CP01-log user: ...
2020-06-16 07:13:49.981 DEBUG 1 --- [nio-8080-exec-5] org.postgresql.jdbc.PgConnection         :   setAutoCommit = true
```

这是新版本代码在并发场景下的log：

```log
2020-06-16 07:13:31.550 DEBUG 1 --- [l-1 housekeeper] com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Pool stats (total=2, active=0, idle=2, waiting=0)
2020-06-16 07:13:49.886 DEBUG 1 --- [nio-8080-exec-5] o.s.orm.jpa.JpaTransactionManager        : Creating new transaction with name [org.springframework.security.oauth2.provider.token.DefaultTokenServices.createAccessToken]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
2020-06-16 07:13:49.887 DEBUG 1 --- [nio-8080-exec-5] o.s.orm.jpa.JpaTransactionManager        : Opened new EntityManager [SessionImpl(256287368<open>)] for JPA transaction
2020-06-16 07:13:49.887 DEBUG 1 --- [nio-8080-exec-5] o.h.e.t.internal.TransactionImpl         : On TransactionImpl creation, JpaCompliance#isJpaTransactionComplianceEnabled == false
2020-06-16 07:13:49.887 DEBUG 1 --- [nio-8080-exec-5] o.h.e.t.internal.TransactionImpl         : begin
2020-06-16 07:13:49.891 DEBUG 1 --- [nio-8080-exec-5] org.postgresql.jdbc.PgConnection         :   setAutoCommit = false
2020-06-16 07:13:49.891 DEBUG 1 --- [nio-8080-exec-5] o.s.orm.jpa.JpaTransactionManager        : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@1ee8f5f6]
2020-06-16 07:13:49.892  INFO 1 --- [nio-8080-exec-5] c.i.d.s.authorization.aop.AuditLogAop    : CP01-log user: ...
2020-06-16 07:13:49.913 DEBUG 1 --- [nio-8080-exec-5] org.postgresql.jdbc.PgConnection         :   setAutoCommit = false
2020-06-16 07:13:49.934 DEBUG 1 --- [nio-8080-exec-5] org.postgresql.jdbc.PgConnection         :   setAutoCommit = true
2020-06-16 07:13:49.934 DEBUG 1 --- [nio-8080-exec-5] com.zaxxer.hikari.pool.PoolBase          : HikariPool-1 - Reset (autoCommit) on connection org.postgresql.jdbc.PgConnection@100d71db
2020-06-16 07:13:49.908  INFO 1 --- [nio-8080-exec-5] g.a.o.c.RamGuardsJweAccessTokenConverter : user: ...
2020-06-16 07:13:49.935 DEBUG 1 --- [nio-8080-exec-5] org.postgresql.jdbc.PgConnection         :   setAutoCommit = false
2020-06-16 07:13:49.955 DEBUG 1 --- [nio-8080-exec-5] org.postgresql.jdbc.PgConnection         :   setAutoCommit = true
2020-06-16 07:13:49.955 DEBUG 1 --- [nio-8080-exec-5] com.zaxxer.hikari.pool.PoolBase          : HikariPool-1 - Reset (autoCommit) on connection org.postgresql.jdbc.PgConnection@100d71db
2020-06-16 07:13:49.935  INFO 1 --- [nio-8080-exec-5] g.a.o.c.RamGuardsJweAccessTokenConverter : CP01-log user: ...
2020-06-16 07:13:49.958 DEBUG 1 --- [nio-8080-exec-5] org.postgresql.jdbc.PgConnection         :   setAutoCommit = false
2020-06-16 07:13:49.969 DEBUG 1 --- [nio-8080-exec-5] org.postgresql.jdbc.PgConnection         :   setAutoCommit = true
2020-06-16 07:13:49.969 DEBUG 1 --- [nio-8080-exec-5] com.zaxxer.hikari.pool.PoolBase          : HikariPool-1 - Reset (autoCommit) on connection org.postgresql.jdbc.PgConnection@100d71db
2020-06-16 07:13:49.958  INFO 1 --- [nio-8080-exec-5] g.a.o.c.RamGuardsJweAccessTokenConverter : user: ...
2020-06-16 07:13:49.971 DEBUG 1 --- [nio-8080-exec-5] org.postgresql.jdbc.PgConnection         :   setAutoCommit = false
2020-06-16 07:13:49.981 DEBUG 1 --- [nio-8080-exec-5] org.postgresql.jdbc.PgConnection         :   setAutoCommit = true
2020-06-16 07:13:49.982 DEBUG 1 --- [nio-8080-exec-5] com.zaxxer.hikari.pool.PoolBase          : HikariPool-1 - Reset (autoCommit) on connection org.postgresql.jdbc.PgConnection@100d71db
2020-06-16 07:13:49.971  INFO 1 --- [nio-8080-exec-5] g.a.o.c.RamGuardsJweAccessTokenConverter : CP01-log user: ...
2020-06-16 07:13:49.984 DEBUG 1 --- [nio-8080-exec-5] o.s.orm.jpa.JpaTransactionManager        : Initiating transaction commit
2020-06-16 07:13:49.984 DEBUG 1 --- [nio-8080-exec-5] o.s.orm.jpa.JpaTransactionManager        : Committing JPA transaction on EntityManager [SessionImpl(256287368<open>)]
2020-06-16 07:13:49.984 DEBUG 1 --- [nio-8080-exec-5] o.h.e.t.internal.TransactionImpl         : committing
2020-06-16 07:13:49.984 DEBUG 1 --- [nio-8080-exec-5] org.postgresql.jdbc.PgConnection         :   setAutoCommit = true
2020-06-16 07:13:49.985 DEBUG 1 --- [nio-8080-exec-5] o.s.orm.jpa.JpaTransactionManager        : Closing JPA EntityManager [SessionImpl(256287368<open>)] after transaction
```

可以明显的看到，在代码更新后，每次向数据库写log的时候都加上了声明式事务，而在这个事务中我们一共写了5次log，才将数据库连接释放。而在并发场景下，后进来的线程也会开启事务，但是数据库连接池却没有可有连接给支撑，在timeout时间过了之后，就会报大批量连接不上数据库的错误。这是rootcause，那为什么莫名其妙会加上事务呢？

经过仔细研读spring security代码，发现一个以前没有注意到的事情，就是DefaultTokenService重的createAccessToken()方法上有@Transactional注解，而我们写log的代码又正好是在这个方法内执行的：

```java
public class DefaultTokenServices implements AuthorizationServerTokenServices, ResourceServerTokenServices,
		ConsumerTokenServices, InitializingBean {

	@Transactional
	public OAuth2AccessToken createAccessToken(OAuth2Authentication authentication) throws AuthenticationException {

		OAuth2AccessToken existingAccessToken = tokenStore.getAccessToken(authentication);
		OAuth2RefreshToken refreshToken = null;
		if (existingAccessToken != null) {
			if (existingAccessToken.isExpired()) {
				if (existingAccessToken.getRefreshToken() != null) {
					refreshToken = existingAccessToken.getRefreshToken();
					// The token store could remove the refresh token when the
					// access token is removed, but we want to
					// be sure...
					tokenStore.removeRefreshToken(refreshToken);
				}
				tokenStore.removeAccessToken(existingAccessToken);
			}
			else {
				// Re-store the access token in case the authentication has changed
				tokenStore.storeAccessToken(existingAccessToken, authentication);
				return existingAccessToken;
			}
		}

		// Only create a new refresh token if there wasn't an existing one
		// associated with an expired access token.
		// Clients might be holding existing refresh tokens, so we re-use it in
		// the case that the old access token
		// expired.
		if (refreshToken == null) {
			refreshToken = createRefreshToken(authentication);
		}
		// But the refresh token itself might need to be re-issued if it has
		// expired.
		else if (refreshToken instanceof ExpiringOAuth2RefreshToken) {
			ExpiringOAuth2RefreshToken expiring = (ExpiringOAuth2RefreshToken) refreshToken;
			if (System.currentTimeMillis() > expiring.getExpiration().getTime()) {
				refreshToken = createRefreshToken(authentication);
			}
		}

		OAuth2AccessToken accessToken = createAccessToken(authentication, refreshToken);
		tokenStore.storeAccessToken(accessToken, authentication);
		// In case it was modified
		refreshToken = accessToken.getRefreshToken();
		if (refreshToken != null) {
			tokenStore.storeRefreshToken(refreshToken, authentication);
		}
		return accessToken;

    }
}
```

为什么createAccessToken方法要加事务呢？因为在许多场景下，accesstoken是有持久化的需求的。因为ram-guards用的是jwt(jwe)格式的，它能够自解析，所以我们不需要持久化。但如果是一个普通的token，就需要使用JdbcTokenStore来存库了。

那么现在有三个问题：

1. 为什么我们的代码里开启了事务（还是log-db上的事务）

spring boot在引入任何db的datasource时，会自动通过HibernateJpaAutoConfiguration->HibernateJpaConfiguration->JpaBaseConfiguration.transactionManager()来建立一个transactionManager的bean，并通过TransactionAutoConfiguration去进行自动配置。transactionManager会扫描所有bean中有@Transactional注解的方法，并以aop的方式去开启事务。

2. 为什么以前的代码没有使用事务，现在的代码却使用了

这就得直接上代码了。

老版本的代码：

```java
    @Bean
    @Primary
    public DefaultTokenServices tokenServices() throws Exception {
        final DefaultTokenServices defaultTokenServices = new DefaultTokenServices();
        defaultTokenServices.setTokenStore(tokenStore());
        defaultTokenServices.setSupportRefreshToken(true);
        return defaultTokenServices;
    }

    @Override
    public void configure(final AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        final TokenEnhancerChain tokenEnhancerChain = new TokenEnhancerChain();
        tokenEnhancerChain.setTokenEnhancers(Collections.singletonList(accessTokenConverter()));
        endpoints
                .requestValidator(requestValidator())
                .tokenStore(tokenStore())
                .tokenEnhancer(tokenEnhancerChain)
                .authenticationManager(authenticationManagerBean)
                .userDetailsService(refreshUserDetailsService);
    }
```

新版本的代码：

```java
    @Bean
    @Primary
    public DefaultTokenServices tokenServices() throws Exception {
        final DefaultTokenServices defaultTokenServices = new DefaultTokenServices();
        defaultTokenServices.setTokenStore(tokenStore());
        defaultTokenServices.setSupportRefreshToken(true);
        PreAuthenticatedAuthenticationProvider provider = new PreAuthenticatedAuthenticationProvider();
        provider.setPreAuthenticatedUserDetailsService(token -> {
            UsernamePasswordAuthenticationToken authenticationToken = (UsernamePasswordAuthenticationToken) token.getPrincipal();
            RamGuardsUser ramGuardsUser = new RamGuardsUser();
            ramGuardsUser.setUsername(authenticationToken.getPrincipal().toString());
            Map<String, Object> details = (Map<String, Object>) authenticationToken.getDetails();
            if (details!=null){
                ramGuardsUser.setIsSystemPartner(Boolean.parseBoolean(details.get("system_partner").toString()));
            }
            ramGuardsUser.setDetails(details);
            ramGuardsUser.setCredentials(authenticationToken.getCredentials());
            return refreshUserDetailsService.loadUserByRamGuardsUser(ramGuardsUser);
        });
        defaultTokenServices.setAuthenticationManager(new ProviderManager(Collections.singletonList(provider)));
        defaultTokenServices.setTokenEnhancer(accessTokenConverter());
        defaultTokenServices.setTokenStore(tokenStore());
        return defaultTokenServices;
    }

    @Override
    public void configure(final AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        endpoints
                .tokenServices(tokenServices())
                .requestValidator(requestValidator())
                .authenticationManager(authenticationManagerBean);
    }
```

打眼一看好像没什么不同，但仔细看一下就会发现，老版本代码中，DefaultTokenServices虽然注册成为了bean，但却并没有把这个bean提供给AuthorizationServerEndpointsConfigurer，而当AuthorizationServerEndpointsConfigurer中的tokenservice没有值的时候，事实上会调用createDefaultTokenServices方法去创建一个不是bean的DefaultTokenServices实例，代码如下：

```java
   public AuthorizationServerTokenServices getDefaultAuthorizationServerTokenServices() {
		if (defaultTokenServices != null) {
			return defaultTokenServices;
		}
		this.defaultTokenServices = createDefaultTokenServices();
		return this.defaultTokenServices;
	}

	private DefaultTokenServices createDefaultTokenServices() {
		DefaultTokenServices tokenServices = new DefaultTokenServices();
		tokenServices.setTokenStore(tokenStore());
		tokenServices.setSupportRefreshToken(true);
		tokenServices.setReuseRefreshToken(reuseRefreshToken);
		tokenServices.setClientDetailsService(clientDetailsService());
		tokenServices.setTokenEnhancer(tokenEnhancer());
		addUserDetailsService(tokenServices, this.userDetailsService);
		return tokenServices;
	}
```

而我们代码中所使用的tokenservice，恰恰就是AuthorizationServerEndpointsConfigurer中的tokenservice。

所以说，在老版本代码中，我们使用的tokenservice并不是spring的一个bean，所以事务也就不会开启，而新版本代码中使用的是一个bean，所以也就启用了事务。

3. 我们既然不需要事务，如何把这个事务去掉

找到了rootcause之后，解决就变得简单了许多，摆在我面前有两个方案：1、将TransactionAutoConfiguration给exclude出去，不让spring boot自动装配。2、不把DefaultTokenServices注册为bean。

最后选择了方案2，因为我认为既然ram-guards没有被持久化的需求，就不应该被任何transactionManager扫描到。同时，我也把记录auditlog的方法从createAccessToken方法中拿了出来，在TokenEndpoint中以aop的方式切入记log，这样也减少了数据库的写入次数。

至此，这个issue算是解决了。

## 后记

说一下我们记audit log的方式，直接通过db log appender去写入数据库事实上是一个开销很大的方式，尤其是在我们log db可用连接资源还十分有限的情况下。这个我们暂时讨论，会在后期优化中用kafka，批量记入到kafka中，在批量写入db进行持久化，减小db的开销。

以后要多做一些API压测（ab或者jmeter），来保证接口在一个标准下不挂，可以以单实例qps为多少来制定这个标准，来保证我们接口的稳定性。

个人认为很有价值的一次troubleshooting，后记中提出的问题都值得我们进行实践。
