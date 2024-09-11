---
title: 在前后端分离的项目中接入oauth2
date: 2024-02-22
author: falser101
tag:
  - OAuth2
category:
  - frontend
---

## OAuth2
OAuth2的概念不再说明了，可查阅官网[OAuth 2.0](https://oauth.net/2/)。

## 使用Authorization Code Flow模式来实现
### 1. 创建OAuth2客户端
在OAuth2服务器端创建客户端，并设置好secret和client_id。以及回调地址。

### 2. 后端重定向接口
java 使用`com.nimbusds:oauth2-oidc-sdk`来实现OAuth2，重定向接口如下：
```java
    @RequestMapping("/authorize")
    public String authorize(HttpServletResponse response) throws URISyntaxException {

        ClientID clientID = new ClientID(clientId);
        // The client callback URL
        URI callbackUri = new URI(callback);

        // Generate random state string to securely pair the callback to this request
        State state = new State();

        // Generate nonce for the ID token
        Nonce nonce = new Nonce();

        // Compose the OpenID authentication request (for the code flow)
        AuthenticationRequest request = new AuthenticationRequest.Builder(
                new ResponseType("code"),
                new Scope("openid", "email", "profile"), clientID, callbackUri)
                .endpointURI(opMetadata.getAuthorizationEndpointURI())
                .state(state)
                .nonce(nonce)
                .build();
        String url = request.toURI().toString();
        Cookie stateCookie = new Cookie("state", state.getValue());
        stateCookie.setHttpOnly(true);
        stateCookie.setMaxAge((int) Timer.ONE_HOUR);
        response.addCookie(stateCookie);
        Cookie nonceCookie = new Cookie("nonce", nonce.getValue());
        nonceCookie.setHttpOnly(true);
        nonceCookie.setMaxAge((int) Timer.ONE_HOUR);
        response.addCookie(nonceCookie);
        return "redirect:"+ url;
    }
```

### 3. 前端跳转
```js
router.beforeEach((to, from, next) => {
  NProgress.start()
  if (getToken()) {
    .....
  } else {
    /* has no token*/
    if (whiteList.indexOf(to.path) !== -1) { // 在免登录白名单，直接进入
      next()
    } else {
      // 重定向到登录页面
      window.location.href = "/oauth2/authorize"
    }
  }
})
```

### 4. 前端的callback处理
登录成功后重定向到以下页面，请求后端获取token并设置到localStorage中
```js
<script>
export default {
  name: 'Index',
  created() {
    //(钩子函数)
    const query = this.$route.query;
    const code = query.code;
    const state = query.code;
    this.$store.dispatch('getToken', code, state).then(() => {
      this.$router.push({ path: '/management/tenants' })
    }).catch(() => {
      console.log('login error!!')
    })
  },
}
</script>
```

### 5. 后端获取token
```java
    @PostMapping("/token")
    @ResponseBody
    public ResponseEntity<Map<String, Object>> callback(@RequestBody OAuth2GetTokenVO vo) throws ParseException, URISyntaxException, IOException {
        AuthorizationCode code = new AuthorizationCode(vo.getCode());
        AuthorizationGrant codeGrant = new AuthorizationCodeGrant(code, new URI(callback));
        ClientID clientID = new ClientID(clientId);
        Secret secret = new Secret(clientSecret);
        ClientAuthentication clientAuth = new ClientSecretPost(clientID, secret);
        // Make the token request
        TokenRequest tokenRequest = new TokenRequest(opMetadata.getTokenEndpointURI(), clientAuth, codeGrant);
        TokenResponse tokenResponse = OIDCTokenResponseParser.parse(tokenRequest.toHTTPRequest().send());
        HTTPResponse httpResponse;
        Map<String, Object> result = new HashMap<>();
        if (!tokenResponse.indicatesSuccess()) {
            // We got an error response...
            TokenErrorResponse errorResponse = tokenResponse.toErrorResponse();
            result.put("error", errorResponse.getErrorObject().getDescription());
        } else {
            OIDCTokenResponse successResponse = (OIDCTokenResponse)tokenResponse.toSuccessResponse();
            result.put("access_token", successResponse.getTokens().getAccessToken().getValue());
            result.put("refresh_token", successResponse.getTokens().getRefreshToken().getValue());
            result.put("id_token", successResponse.getOIDCTokens().getIDToken().getParsedString());
        }
        return ResponseEntity.ok(result);
    }
```