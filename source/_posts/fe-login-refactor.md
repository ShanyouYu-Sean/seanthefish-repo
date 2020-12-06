---
layout: post
title: 前端login重构
date: 2020-6-11 10:00:00
tags: 
- OAuth
categories:
- OAuth
---


# 记录一下前端login部分重构案例

前端是用的angular6搭建，因为需要连接ibmid登录，所以在网上找了一个符合openid connect标准的库[angular-oauth-oidc](https://github.com/manfredsteyer/angular-oauth2-oidc), 结果就开始了长达一年的漫长重构之路。

enable前端的openid connect的登录倒还好说，熟悉openid connect标准的同学应该都清楚前端spa应用应该使用[implicit flow](https://tools.ietf.org/html/rfc6749#section-4.2)（因为oauth协议更新，现在已经不建议用implicit flow，而是[PKCE auth code flow](https://tools.ietf.org/html/rfc7636)，这是后话，我会在后文提及）。因此登录并不是难事，angular-oauth-oidc有详细的文档告诉我们怎么enable implicit flow。但坑体现在登录以外的地方。

## 坑#1 - 有关discovery endpoint的坑

angular-oauth-oidc默认的推荐使用方式是应用自动通过[openid discovery endpoint](https://openid.net/specs/openid-connect-discovery-1_0.html)去加载所需的配置，对于我们开发者来说，只需要配clientid和redirecturl就可以。但是ibmid在之前使用idaas的一年时间里，都不提供discovery endpoint，因此我们只能hardcode在代码里这部分配置，因此也使得在启动框架的时候略显繁琐。代码如下：

```typescript
this.oauthService.configure(authConfig);
this.oauthService.tokenValidationHandler = new JwksValidationHandler();
this.oauthService.setupAutomaticSilentRefresh();
this.oauthService.tryLogin();
```

```typescript
export const authConfig: AuthConfig = {

  clientId: environment.clientId,
  redirectUri: window.location.origin + '/it-infrastructure/z/resources/tools/danube/login',
  postLogoutRedirectUri: '',
  loginUrl: environment.loginUrl,
  scope: 'openid',
  oidc: true,
  requestAccessToken: true,
  issuer: environment.issuer,
  tokenEndpoint: environment.tokenEndpoint,
  userinfoEndpoint: environment.userinfoEndpoint,
  responseType: 'id_token token',
  silentRefreshRedirectUri: window.location.origin + '/it-infrastructure/z/resources/tools/danube/silent-refresh',
  requireHttps: 'remoteOnly',
  jwks: environment.jwks,
  timeoutFactor: 0.9,
  siletRefreshTimeout: 100000,
  useIdTokenHintForSilentRefresh: true,
  clearHashAfterLogin: false,
  silentRefreshShowIFrame: false,
  showDebugInformation: true
};
```

而现在，ibmid移到了cloud identity上，并且加了discovery endpoint，本以为以后就不用hardcode配置了，结果，新的ibmid的discovery endpoint竟然不支持cors，看起来ibmid在设计之初就没有考虑过前端spa或是移动端的使用，十分失望，我只能再继续hardcode。至于cors，我已经跟ibmid team的人说过要加上（一定得加上因为以后如果前端想要用PKCE auth code就必须支持cors），不知道他们什么时候跟进。

## 坑#2 - 有关前端认证id-token的坑

同样是因为以前ibmid的idaas不支持jwkset endpoint，长时间以来我们都没办法在前端认证id-token（即`this.oauthService.tokenValidationHandler = new NullValidationHandler()`），只能靠后端验证，知道后来cloud identity加上jwkset endpoint才得已解决。但是，以为上文提到的cors的问题，我们只能把jwkset hardcode在environment文件里，十分丑陋。

```typescript
    jwks:
      {
        "keys": [
          {
            "kty": "xxx",
            "kid": "xxx",
            "use": "xxx",
            "alg": "xxx",
            "n": "xxx",
            "e": "xxx",
            "x5c": [
              "xxxx"
            ],
            "x5t#S256": "xxx"
          }
        ]
      },
```

## 坑#3 - 关于回调URL的坑

angular-oauth-oidc推荐使用静态页面的url作为回调url（即index.html），但这会导致一个问题，就是我们回调回应用后，都会有一个一闪而过的画面，因就是我们回调回index.html之后，还得route到我们当初访问的url上，这个就会出现index.html一闪而过的现象（一般来说一闪而过很快，肉眼看不见，但我们回调回应用后还有后续操作，所以变得很明显），解决这个坑就是不应静态的index.html来作为回调url，而是给一个route，我们这是/login，angular会捕获这个路由显示一个loading页面，知道后续操作完毕，跳转到当初访问的url。

同样是因为我们做了一些后续操作后再跳转，我们就必须保证这些后续操作无论成功失败都必须跳转，否则就会一直卡在loading页面了，所以必须对回调之后的操作进行异常捕捉，处理后再跳转。

## 坑#4 - 有关同步代码的坑

先说一下什么是ram-guards，这是我们的一个微服务的权限处理系统，接受一个oauth idtoken并认证，然后返回一个能代表权限的token。

这部分不是ibmid的锅，是我们自己的。我们在oauth server redirect回我们的应用后，需要作两件事情，一是去获得我们的权限ram-guards-token，二是跳转到当初我们希望route的url中去，这就要求跳转必须在后去权限之后发生，获取权限就不能异步请求，只能同步请求。
angular httpclient返回的是一个异步observable，把它变成同步很简单，只需要使用observable.toPromise，然后在套上aysnc/await语法糖就可以。但这块有两个问题需要注意，一是我长时间以为aysnc/await语法糖就可以保证代码同步，就像java的synchronized一样，结果并不是，aysnc/await只是一个语法糖，而且不论aysnc方法内是否返回promise，aysnc都会将整个方法包装成一个promise，真正想在同步后执行的代码还得放在.then()里面。因此得出教训，aysnc/await固然好用，但必须完全理解其含义并谨慎使用。

再就是我们在得到ram-guards-token后必须解析它，ram-guards-token是一个jwe所以我们用了另外一个库[node-jose](https://github.com/cisco/node-jose), 这个库推荐我们这样解析jwe：

```typescript
jose.JWK.asKey({
      'kty': 'oct',
      'use': 'enc',
      'k': ramGuardsClientSecret,
      'alg': 'A128CBC-HS256'
    })
    .then(key => {
        jose.JWE.createDecrypt(key)
            .decrypt(ramGuardsResponse.access_token)
            .then(result => {
                const claims = new TextDecoder().decode(result.plaintext);
                localStorage.setItem('Ram-Guards-Details', claims);
            });
    });
```

那么问题来了，promise直接return后.then()就OK，那promise套一个promise呢？这种情况下想要同步必须给两个promise都套上return才行。

## 坑#5 - 有关解析ram-guards-token的坑

其实不算坑，更多是一个bug，以为ram-guards-token用的是一个加密的jwe，前端要想解析据必须使用它的secret，而在spa里报漏secret是非常危险的，因此这一块需要改成使用oauth推荐的introspect endpoint来通过后端解析token，多加了一次请求，但保证了应用的安全性，这一部分已经加到了ram-guards里，后续我们需要改一下。

## 坑#6 - 有关silent-refresh的坑

天坑！！！

首先说一下什么是silent-refresh，以为implicit是不使用clientsecret申请token的方式，因此我们无法通过正常的refresh token的方式刷新token，所以才有了silent-refresh，即在前端临时加一个看不见的iframe，在iframe的src里指向authorize endpoint；然后通过auth server那一端未过期的session进行自动的登录并重定向回我们的iframe，然后iframe向parent window发通知更新应用存储的token，并把iframe自己给干掉。（about [slient-refresh](https://www.scottbrady91.com/OpenID-Connect/Silent-Refresh-Refreshing-Access-Tokens-when-using-the-Implicit-Flow)）

所以说这一连串骚操作都是建立在iframe的基础上的，但是偏偏ibmid也有它的骚操作，不允许从iframe申请的跨域请求，回报Refused to display in a iframe的错，即response里面会有`Content-Security-Policy: frame-ancestors 'self'`([CSP: frame-ancestors](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Security-Policy/frame-ancestors))，这说明ibmid压根就没考虑过silent-refresh的场景啊，再次验证了我之前的推测，ibmid就没想过前端spa和移动端应用会用它。

因此，我们必须想办法进行silent-refresh，我做了一个实验，发现只有在iframe的src中请求的是静态页面（即xxx.html）的时候，浏览器才会报Refused to display in a iframe的错，而非静态页面的url，即使response中包含`Content-Security-Policy: frame-ancestors 'self'`，也不会报错（暂不清楚这是为啥，希望有大神可以帮忙解答），因此我们就可以通过把silent-refresh的回调URL交给angular的route来捕获处理。到这为止，angular-oauth-oidc的slient-refresh部分的文档就完全没啥用了，我们需要按照我们的思路来实验。

既然silent-refresh的回调URL交给angular的route来捕获处理，我们就要给silent-refresh的回调URL一个component，当然这个component啥也不用做，我们要做的是在appcomponent中区分出监听tokenreceived事件，区分出哪些是iframe内部接收的，那些事iframe外部接受的。

最开始是没有iframe的，只有页面进行登录，登录成功后我们要开始一个timer去检查token啥时候会过期，以便我们在token过期之前去开始slient-refresh。

当token过期时，token_expires事件将会被触发，然后angular-oauth-oidc框架会自动创建一个看不见的iframe去申请我们的silent-refresh url /silent-refresh，然后authserver会重定向回iframe，而iframe内部接收的，就是一次silent-refresh的请求，也就不需要开始timer去看它是否会过期（因为iframe内部的只作为refresh token之用），我们在接收到之后需要同样的去refresh ram-guards-token，然后手动通知iframe的parent window token已经刷新成功，再把这个iframe自己干掉。

而iframe外部在接收到token更新成功的事件后，就会出发silently_refreshed的事件，从而使这一次slient-refresh流程成功。

因此，我们代码的关键就是：

1.在appcomponent中区分出哪次token-received事件是第一次申请token的，哪一次是silent-refresh token的。

2.区分出当前appcomponent是在iframe内部还是在iframe外部。

3.处理所有的silent-refresh异常，以避免silent-refresh失败

这里需要重点说一下的是silent-refresh异常，总结了一下，大概分四种：

1.silent_refresh_error，即auth-server返回了login_required的error，通常是因为auth-server那一端session过期的缘故([about1](https://openid.net/specs/openid-connect-session-1_0.html#ChangeNotification), [about2](https://openid.net/specs/openid-connect-core-1_0.html#AuthError))

2.token_error，原因同上，只是silent_refresh_error一定伴随着token_error，但token_error不一定是silent-refresh造成的，也有可能是第一次申请token造成的

3.silent_refresh_timeout，在silent-refresh的timeout时间内，iframe没能通知到parent window refresh成功

4.invalid_nonce_in_state，两次authorize申请同时发生，框架不能正确的验证nonce，通常是因为一次silent_refresh没有成功而又开始了一次authorize请求

按道理来说，我们应该根据这四种不同的error去设计不同的解决方案，silent_refresh_timeout，invalid_nonce_in_state是重启timer，silent_refresh_error，token_error是强制用户重新登录，但angular-oauth-oidc框架无法重启timer，我们就只能粗暴的都强制用户重新登录了。这个可以通过后续更改源码来实现。

还有一点需要注意的是，虽然文档中说angular-oauth-oidc的trylogin的几个回调方法都已经过时了，要使用events来监听事件，但我发现events监听到的事件并不能返回token的info，所以我们还得用trylogin的回调，这个也可以通过改源码来解决。

大概就是这些个坑，还有一些细节优化啥的就不一一概述了，我这里贴一下关键的源码：

app.component.ts:

```typescript
private configureIbmId() {
    this.oauthService.configure(authConfig);
    this.oauthService.tokenValidationHandler = new JwksValidationHandler();
    let iframeOrNot;
    if (window.location !== window.parent.location) {
      iframeOrNot = 'inside-iframe';
    } else {
      iframeOrNot = 'outside-iframe';
      // only need auto silent refresh outside the iframe
      this.oauthService.setupAutomaticSilentRefresh();
    }
    this.oauthService.events.subscribe( event => {
      if (event instanceof OAuthErrorEvent) {
        // normally , we will face four kind of errors:
        //
        // silent_refresh_error: response login_required error, force user to re login
        // token_error: response response login_required error, force user to re login
        // silent_refresh_timeout: iframe failed to notify parent window, kill old RefreshTimer, setup a new RefreshTimer
        // invalid_nonce_in_state: two authorize post happens at the same time, kill old RefreshTimer, setup a new RefreshTimer
        //
        // TODO: when received error, auto silent refresh will stop, we can make it restart by modify setupRefreshTimer() in angular-oauth2-oidc to subscribe a error event for different conditions
        // since setupAutomaticSilentRefresh() cannot produce the same behaviour as setupRefreshTimer() (will cause invalid_nonce_in_state error), for now, we will force user to re login in all error conditions
        console.error(iframeOrNot, event.type, event);
        this.oauthService.logOut();
        this.authorizationService.removeStorage();
      } else {
        console.log(iframeOrNot, event.type, event);
      }
    });
    this.oauthService.tryLogin({
      onTokenReceived: (info) => {
        this.isLogged = this.loginService.isLoggedIn();
        if (!this.authorizationService.neverCallRamGuardsBefore()) {
          // refresh logs will always be:
          //
          // outside-iframe token_expires
          // inside-iframe token_received
          // inside-iframe id token refreshed
          // inside-iframe ram-guards token refreshed
          // outside-iframe token_received
          // outside-iframe silently_refreshed
          //
          // if these logs appear:
          //
          // outside-iframe id token refreshed
          // outside-iframe ram-guards token refreshed
          //
          // this means for some unknown reason, oauth server redirect to out client at outside iframe and ram guards tokens are still in local storage
          // for example: id token expired or not valid while ram guards token still there outside iframe
          // not sure why this happens but we should be able to handle this situation
          //
          // refresh token in a iframe
          console.log(iframeOrNot, 'id token refreshed', info);
          this.authorizationService.syncRefreshAuthorization()
            .then(res => {
              const ramGuardsResponse: RamGuardsResponse = res.body;
              this.authorizationService.setStorage(ramGuardsResponse).then(() => {
                console.log(iframeOrNot, 'ram-guards token refreshed', ramGuardsResponse);
                // notify parent and remove iframe
                window.parent.postMessage(location.hash , location.origin + '/it-infrastructure/z/resources/tools/danube');
                if (iframeOrNot === 'inside-iframe' && frameElement) {
                  frameElement.remove();
                } else {
                  // handle the unexpected situation mentioned before
                  this.router.navigate([info && info.state ? info.state : 'dashboard']);
                }
              });
            })
            .catch(err => {
              // do nothing wait for next time refresh
              console.error(iframeOrNot, 'ram-guards token refreshed error, waiting for next time refresh', err);
              // notify parent and remove iframe
              window.parent.postMessage(location.hash , location.origin + '/it-infrastructure/z/resources/tools/danube');
              if (iframeOrNot === 'inside-iframe' && frameElement) {
                frameElement.remove();
              } else {
                // handle the unexpected situation mentioned before
                this.router.navigate([info && info.state ? info.state : 'dashboard']);
              }
            });
        } else {
          // get token first time
          console.log(iframeOrNot, 'id token received', info);
          this.authorizationService.syncAuthorization()
            .then(res => {
              const ramGuardsResponse: RamGuardsResponse = res.body;
              this.authorizationService.setStorage(ramGuardsResponse).then(() => {
                console.log(iframeOrNot, 'ram-guards token received', ramGuardsResponse);
                this.router.navigate([info && info.state ? info.state : 'dashboard']);
              });
            }).catch(err => {
              this.authorizationService.setRamGuardsError(err.body);
              console.error(iframeOrNot, 'ram-guards token received error', err);
              this.router.navigate(['dashboard']);
            });
        }
      }
    });
  }
```

login.service.ts:

```typescript
import { Injectable } from '@angular/core';
import { OAuthService } from 'angular-oauth2-oidc';
import { RouterStateSnapshot } from '@angular/router';


@Injectable({
  providedIn: 'root'
})
export class LoginService {

  constructor(private oauthService: OAuthService) { }

  public login() {
    this.oauthService.initImplicitFlow();
  }

  public loginWithRedirectUrl(redirectUrl: string) {
    this.oauthService.initImplicitFlow(redirectUrl);
  }

  public getIdToken() {
    return this.oauthService.getIdToken();
  }

  public isLoggedIn() {
    return this.oauthService.hasValidIdToken();
  }

  public async syncRefreshIdToken() {
    return await this.oauthService.silentRefresh();
  }
}
```

authorization.service.ts:

```typescript
declare var TextDecoder: any;

import {Injectable} from '@angular/core';
import {HttpClient, HttpResponse} from '@angular/common/http';
import {Observable} from 'rxjs';
import {timeout} from 'rxjs/operators';
import {RamGuardsResponse} from './RamGuardsResponse';
import * as jose from 'node-jose';
import {environment} from 'src/environments/environment';

const authorizationServiceUrl  = environment.authorizationServiceUrl;
const ramGuardsClientSecret  = environment.ramGuardsClientSecret;

@Injectable({
  providedIn: 'root'
})
export class AuthorizationService {

  constructor(private http: HttpClient) {
  }

  public authorize(): Observable<HttpResponse<RamGuardsResponse>> {
    return this.http.post<RamGuardsResponse>(
      authorizationServiceUrl + '/oauth/token?grant_type=password',
      null,
      {
        observe: 'response'
      }
    ).pipe(timeout(120000));
  }

  public refreshToken(token: String): Observable<HttpResponse<RamGuardsResponse>> {
    return this.http.post<RamGuardsResponse>(
      authorizationServiceUrl + '/oauth/token?grant_type=refresh_token&refresh_token=' + token,
      null,
      {
        observe: 'response'
      }
    ).pipe(timeout(120000));
  }

  public async syncAuthorization() {
    return await this.authorize().toPromise();
  }

  public checkAuthorization(id: number) {
    if (localStorage.getItem('Ram-Guards-Details')) {
      return JSON.parse(localStorage.getItem('Ram-Guards-Details')).authorities.some(auth => {
        return auth.roles.some(role => {
          return role.id === id;
        });
      });
    }
    return false;
  }

  public isAuthorized() {
    if (localStorage.getItem('Ram-Guards-Access-Token') && localStorage.getItem('Ram-Guards-Refresh-Token')) {
      const expiresAt = localStorage.getItem('Ram-Guards-Expire-At');
      const now = new Date();
      return !(expiresAt && parseInt(expiresAt, 10) < now.getTime());
    }
    return false;
  }

  public isRamGuardsError() {
    return !!localStorage.getItem('Ram-Guards-Error');
  }

  public setRamGuardsError(msg: any) {
    localStorage.setItem('Ram-Guards-Error', 'Error: ' + msg);
  }

  public isAnonymous() {
    if (localStorage.getItem('Ram-Guards-Details')) {
      return JSON.parse(localStorage.getItem('Ram-Guards-Details')).authorities[0].name === 'anonymous';
    }
    return true;
  }

  public setStorage (ramGuardsResponse: RamGuardsResponse) {
    this.removeStorage();
    this.setAuthorizationBasicStorage(ramGuardsResponse);
    return jose.JWK.asKey({
      'kty': 'oct',
      'use': 'enc',
      'k': ramGuardsClientSecret,
      'alg': 'A128CBC-HS256'
    })
    .then(key => {
      return jose.JWE.createDecrypt(key)
      .decrypt(ramGuardsResponse.access_token)
      .then(result => {
        const claims = new TextDecoder().decode(result.plaintext);
        localStorage.setItem('Ram-Guards-Details', claims);
      });
    });
  }

  public setAuthorizationBasicStorage(ramGuardsResponse: RamGuardsResponse) {
    localStorage.setItem('Ram-Guards-Access-Token', ramGuardsResponse.access_token);
    localStorage.setItem('Ram-Guards-Refresh-Token', ramGuardsResponse.refresh_token);
    localStorage.setItem(
      'Ram-Guards-Expire-At',
      (new Date().valueOf() + Math.floor(ramGuardsResponse.expires_in * 1000)).toString()
    );
    localStorage.setItem(
      'Ram-Guards-Expire-Time',
      ramGuardsResponse.expires_in.toString()
    );
  }

  public removeStorage () {
    localStorage.removeItem('Ram-Guards-Error');
    localStorage.removeItem('Ram-Guards-Access-Token');
    localStorage.removeItem('Ram-Guards-Refresh-Token');
    localStorage.removeItem('Ram-Guards-Expire-At');
    localStorage.removeItem('Ram-Guards-Expire-Time');
    localStorage.removeItem('Ram-Guards-Details');
  }

  public async syncRefreshAuthorization() {
    return await this.refreshToken(localStorage.getItem('Ram-Guards-Refresh-Token')).toPromise();
  }

  public getRamGuardsToken() {
    return localStorage.getItem('Ram-Guards-Access-Token');
  }

  public getRamGuardsRefreshToken() {
    return localStorage.getItem('Ram-Guards-Refresh-Token');
  }

  public neverCallRamGuardsBefore() {
    return !localStorage.getItem('Ram-Guards-Refresh-Token');
  }
}
```

## 后记 - 解决silent-refresh的更优方案

虽然silent-refresh可以解决refresh token的问题，但只是在oauthserver那一端session不过期的情况下，而且因为在iframe中申请authorize endpoint时添加了prompt=none的参数，ibmid的login的页面不会出现，一旦session过期，silent refresh必然会失败。那么有没有更优的方案呢？很简单，就是用上问提到的pkce auth code flow，这部分我问过ibmid的人了，说他们实现了，但没有默认enable，需要申请，考虑到他们还没加cors，现在就没申请，就看他们什么时候加cors了，到时候在申请使用pkce auth code flow。

关于pkce auth code flow的学习看[这篇教程](https://espressocoder.com/2019/10/28/secure-your-spa-with-authorization-code-flow-with-pkce/)，我就不展开了。

## 总结

实践论，实践出真知，照着文档一知半解的写肯定会出错，要想写好代码就要了解全局，oauth oidc协议得懂，angular-oauth-oidc的源码也得阅读，有了这些认识后还得加以大量的实践来验证，不断的重构和完善，才能搞出好的代码。
