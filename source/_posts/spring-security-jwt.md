---
layout: post
title: 使用oidc id token验证spring security resource server
date: 2018-11-06 14:00:00
tags: 
- spring
categories:
- spring
---

在前后端分离的项目中，前端因为不能保证密钥的安全，通常会使用oidc implicit grant type来获取access token和id token，后端会验证token的合法性并获得用户信息。

后端使用spring，如果后端是验证access token，那好说，使用spring-security-oauth2-autoconfigure，详见[文档](https://docs.spring.io/spring-security-oauth2-boot/docs/current-SNAPSHOT/reference/htmlsingle/)。

如果后端是验证id token，并且提供jwks url，那也好说，详见[文档](https://spring.io/blog/2018/08/21/spring-security-5-1-0-rc1-released#oauth2-resource-server)。

可惜的是IBM ID OAuth server暂时还没有提供jwks url，只给了一个rsa public key的证书，因此记录一下解决方案。

**版本：spring boot 2.1.0.RELEASE**

```gradle
dependencies{
    compile('org.springframework.boot:spring-boot-starter-security')
    compile('org.springframework.security:spring-security-oauth2-jose:5.1.1.RELEASE')
    compile('org.springframework.security:spring-security-oauth2-client:5.1.1.RELEASE')
    compile('org.springframework.security:spring-security-oauth2-resource-server:5.1.1.RELEASE')
    compile('org.bouncycastle:bcprov-jdk15on:1.60')
}
```

```java
import org.bouncycastle.util.io.pem.PemObject;
import org.bouncycastle.util.io.pem.PemReader;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.oauth2.jwt.JwtDecoder;
import org.springframework.security.oauth2.jwt.NimbusReactiveJwtDecoder;

import java.io.ByteArrayInputStream;
import java.io.File;
import java.io.FileReader;
import java.io.InputStream;
import java.security.cert.CertificateFactory;
import java.security.cert.X509Certificate;
import java.security.interfaces.RSAPublicKey;


@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Value("${JWK-certificate}")
    String jwkCertificate;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests()
                .anyRequest().authenticated()
                .and()
                .oauth2ResourceServer()
                .jwt();
    }

    @Bean
    JwtDecoder getJwtDecoder() throws Exception {
        NimbusReactiveJwtDecoder nimbusReactiveJwtDecoder = new NimbusReactiveJwtDecoder(WebSecurityConfig.get(jwkCertificate));
        return token -> nimbusReactiveJwtDecoder.decode(token).block();
    }

    public static RSAPublicKey get(String filePath) throws Exception {
        PemReader pemReader = new PemReader(new FileReader(new File(WebSecurityConfig.class.getResource(filePath).getPath())));
        PemObject pemObject = pemReader.readPemObject();
        pemReader.close();
        CertificateFactory fact = CertificateFactory.getInstance("X.509");
        InputStream input = new ByteArrayInputStream(pemObject.getContent());
        X509Certificate cert = (X509Certificate) fact.generateCertificate(input);
        return (RSAPublicKey)cert.getPublicKey();
    }

}
```

**版本：spring boot 2.0.6.RELEASE**

```gradle
dependencies{
    compile('org.springframework.security:spring-security-oauth2-core:5.1.1.RELEASE')
    compile('org.springframework.security:spring-security-oauth2-jose:5.1.1.RELEASE')
    compile('org.springframework.security:spring-security-oauth2-client:5.1.1.RELEASE')
    compile('org.springframework.security:spring-security-oauth2-resource-server:5.1.1.RELEASE')
    compile('org.springframework.security:spring-security-core:5.1.1.RELEASE')
    compile('org.springframework.security:spring-security-config:5.1.1.RELEASE')
    compile('org.springframework.security:spring-security-web:5.1.1.RELEASE')
    compile('org.springframework.security:spring-security-crypto':5.1.1.RELEASE)
    compile('org.bouncycastle:bcprov-jdk15on:1.60')
}
```

```java
import org.bouncycastle.util.io.pem.PemObject;
import org.bouncycastle.util.io.pem.PemReader;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.config.annotation.web.configurers.oauth2.server.resource.OAuth2ResourceServerConfigurer;
import org.springframework.security.oauth2.jwt.JwtDecoder;
import org.springframework.security.oauth2.jwt.NimbusReactiveJwtDecoder;

import java.io.ByteArrayInputStream;
import java.io.File;
import java.io.FileReader;
import java.io.InputStream;
import java.security.cert.CertificateFactory;
import java.security.cert.X509Certificate;
import java.security.interfaces.RSAPublicKey;



@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Value("${JWK-certificate}")
    String jwkCertificate;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        OAuth2ResourceServerConfigurer<HttpSecurity> oauth2ResourceServerConfigurer = new OAuth2ResourceServerConfigurer<>(http.getSharedObject(ApplicationContext.class));
        oauth2ResourceServerConfigurer.jwt();
        http
                .authorizeRequests()
                .anyRequest().authenticated()
                .and()
                .apply(oauth2ResourceServerConfigurer);
    }

    @Bean
    JwtDecoder getJwtDecoder() throws Exception {
        NimbusReactiveJwtDecoder nimbusReactiveJwtDecoder = new NimbusReactiveJwtDecoder(WebSecurityConfig.get(jwkCertificate));
        return token -> nimbusReactiveJwtDecoder.decode(token).block();
    }

    public static RSAPublicKey get(String filePath) throws Exception {
        PemReader pemReader = new PemReader(new FileReader(new File(WebSecurityConfig.class.getResource(filePath).getPath())));
        PemObject pemObject = pemReader.readPemObject();
        pemReader.close();
        CertificateFactory fact = CertificateFactory.getInstance("X.509");
        InputStream input = new ByteArrayInputStream(pemObject.getContent());
        X509Certificate cert = (X509Certificate) fact.generateCertificate(input);
        return (RSAPublicKey)cert.getPublicKey();
    }

}
```