---
title: Shiro jwt 构建无状态分布式鉴权体系
date: 2019-06-07 19:57:22
tags: Shiro
---

##概括

![JWT鉴权场景](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/JWI%20Token%E7%AD%BE%E5%8F%91%E5%92%8C%E9%89%B4%E6%9D%83.png)

在分布式系统中，没法将认证、授权信息保存在session或本地内存中，这里使用JWT Token的方式来保存认证信息。需要提供一个令牌签发服务来进行Tokean的签发，每次请求的时候带上该Token，每个被调用的服务都需要有对该Token进行鉴权的功能。

### JWT

JWT(json web token)是可在网络上传输的用于声明某种主张的令牌（token），以JSON 对象为载体的轻量级开放标准（RFC 7519）。

一个JWT令牌的定义包含头信息、荷载信息、签名信息三个部分：

```
Header//头信息
{
    "alg": "HS256",//签名或摘要算法
    "typ": "JWT"//token类型
}
Playload//荷载信息
{
    "iss": "token-server",//签发者
    "exp ": "Mon Nov 13 15:28:41 CST 2017",//过期时间
    "sub ": "wangjie",//用户名
    "aud": "web-server-1"//接收方,
    "nbf": "Mon Nov 13 15:40:12 CST 2017",//这个时间之前token不可用
    "jat": "Mon Nov 13 15:20:41 CST 2017",//签发时间
    "jti": "0023",//令牌id标识
    "claim": {“auth”:”ROLE_ADMIN”}//访问主张
}
Signature//签名信息
签名或摘要算法（
    base64urlencode（Header），
    Base64urlencode（Playload），
    secret-key
）
```

按照JWT规范，对这个令牌定义进行如下操作：

```
base64urlencode（Header）
+"."+
base64urlencode（Playload）
+"."+
signature（
    base64urlencode（Header）
    +"."+
    base64urlencode（Playload）
    ,secret-key
）
```

形成一个完整的JWT:
`eyJhbGciOiJIUzUxMiIsInppcCI6IkRFRiJ9.eNqqVspMLFGyMjQ1NDA1tTA0NNRRKi5NUrJSKk_MS8_KTFXSUUqtKEAoMDKsBQAAAP__.dGLe7BVECKzQ_utZJqk4hbcBZthNhohuEjjue98vmpQSGn_9cCYHq7lPIfwKubW8M553F8Uhk933EJwgI5vbLQ`

需要注意的是：

1：荷载信息(Playload)中的属性可以根据情况进行设置，不要求必须全部填写。

2：由token的生成方式发现，Header和Playload仅仅是base64编码，通过base64解码之后可见，基本相当于是明文传输，所以应避免敏感信息放入Playload。

**2、令牌特点**

紧凑性：体积较小、意味着传输速度快，可以作为POST参数或放置在HTTP头。

自包含性：有效的负载包含用户鉴权所需所有信息，避免多次查询数据库。

安全性：支持对称和非对称方式(HMAC、RSA)进行消息摘要签名。

标准化：开放标准，多语言支持，跨平台。

**3、适用场景**

1：无状态、分布式鉴权，比如rest api系统、微服务系统。

2：方便解决跨域授权的问题，比如SSO单点登陆。

3：JWT只是消息协议，不牵涉到会话管理和存储机制，所以单体WEB应用还是推荐session-cookie机制。

### 典型微服务鉴权架构

客户端（移动端或者pc端）根据口令或者APP KEY到认证服务鉴权并申请签发令牌，如果需要操作服务A，必须先拿着令牌到服务A进行权限问询。如果需要操作服务B，同样先拿着令牌到服务B进行权限问询，一个令牌可以一次，也可以多次使用连续穿梭多个服务系统，直至令牌过期失效或被销毁。

JWT令牌使用了数字签名可以有效的防止数据篡改和窃取，同样申请令牌时的数据也需要有这样的安全保障，可以使用HMAC(哈希运算消息认证码)进行签名（摘要）和验签（参考：[基于hmac的rest api鉴权处理](https://www.jianshu.com/p/b0a577708a7b)）。

shiro是java业界普遍采用的安全框架，简单、够用、扩展性强。我们可以在shiro中添加对HMAC和JWT这两种鉴权方式的支持。

## 令牌签发服务

签发服务的核心功能是验证客户端是否合法，如果合法则授予其包含特定访问主张的JWT。

shiro Token定义：

```
/**
 * HMAC令牌
 * @author wangjie (http://www.jianshu.com/u/ffa3cba4c604) 
 * @date 2016年6月24日 下午2:55:15
 */
public class HmacToken implements AuthenticationToken{
    private static final long serialVersionUID = -7838912794581842158L;
    
    private String clientKey;// 客户标识（可以是用户名、app id等等）
    private String digest;// 消息摘要
    private String timeStamp;// 时间戳
    private Map<String, String[]> parameters;// 访问参数
    private String host;// 客户端IP
    
    public HmacToken(String clientKey,String timeStamp,String digest
                                ,String host,Map<String, String[]> parameters){
        this.clientKey = clientKey;
        this.timeStamp = timeStamp;
        this.digest = digest;
        this.host = host;
        this.parameters = parameters;
    }
    
    @Override
    public Object getPrincipal() {
        return this.clientKey;
    }
    @Override
    public Object getCredentials() {
        return Boolean.TRUE;
    }
    
    // 省略getters and setters ... ...
}
    
```

shiro Realm即验证逻辑定义：

```
/**
 * 基于HMAC（ 散列消息认证码）的控制域
 * @author wangjie (http://www.jianshu.com/u/ffa3cba4c604) 
 * @date 2016年6月24日 下午2:55:15
 */
public class HmacRealm extends AuthorizingRealm{
    
    private final AccountProvider accountProvider;//账号服务(持久化服务)
    private final CryptogramService cryptogramService;//密码服务

    public HmacRealm(AccountProvider accountProvider,CryptogramService cryptogramService){
        this.accountProvider = accountProvider;
        this.cryptogramService = cryptogramService;
    }
    
    public Class<?> getAuthenticationTokenClass() {
        return HmacToken.class;//此Realm只支持HmacToken
    }
    
    /**
     *  认证
     */
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) 
                                                          throws AuthenticationException {
        HmacToken hmacToken = (HmacToken)token;
        List<String> keys = Lists.newArrayList();
        for (String key:hmacToken.getParameters().keySet()){
            if (!"digest".equals(key))
                keys.add(key);
        }
        Collections.sort(keys);//对请求参数进行排序参数->自然顺序
        StringBuffer baseString = new StringBuffer();
        for (String key : keys) {
            baseString.append(hmacToken.getParameters().get(key)[0]);
        }
        //认证端生成摘要
        String serverDigest = cryptogramService.hmacDigest(baseString.toString());
        //客户端请求的摘要和服务端生成的摘要不同
        if(!serverDigest.equals(hmacToken.getDigest())){
            throw new AuthenticationException("数字摘要验证失败！！！");
        }
        Long visitTimeStamp = Long.valueOf(hmacToken.getTimeStamp());
        Long nowTimeStamp = System.currentTimeMillis();
        Long jge = nowTimeStamp - visitTimeStamp;
        if (jge > 600000) {// 十分钟之前的时间戳，这是有效期可以双方约定由参数传过来
            throw new AuthenticationException("数字摘要失效！！！");
        }
        // 此处可以添加查询数据库检查账号是否存在、是否被锁定、是否被禁用等等逻辑
        return new SimpleAuthenticationInfo(hmacToken.getClientKey(),Boolean.TRUE,getName());
    }
    
    /** 
     * 授权 
     */  
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
        String clientKey = (String)principals.getPrimaryPrincipal();
        SimpleAuthorizationInfo info =  new SimpleAuthorizationInfo();
        // 根据客户标识（可以是用户名、app id等等） 查询并设置角色
        Set<String> roles = accountProvider.loadRoles(clientKey);
        info.setRoles(roles);
      // 根据客户标识（可以是用户名、app id等等） 查询并设置权限
        Set<String> permissions = accountProvider.loadPermissions(clientKey);
        info.setStringPermissions(permissions);
        return info;  
    }
}
```

HMAC认证过滤器定义：

```
/**
 * 基于HMAC（ 散列消息认证码）的无状态认证过滤器
 * @author wangjie (http://www.jianshu.com/u/ffa3cba4c604) 
 * @date 2016年6月24日 下午2:55:15
 */
public class HmacFilter extends AccessControlFilter{
    
    private static final Logger log = LoggerFactory.getLogger(AccessControlFilter.class);
    
    public static final String DEFAULT_CLIENTKEY_PARAM = "clientKey";
    public static final String DEFAULT_TIMESTAMP_PARAM = "timeStamp";
    public static final String DEFAUL_DIGEST_PARAM = "digest";

    /**
     * 是否放行
     */
    protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, 
                                                        Object mappedValue) throws Exception {
        if (null != getSubject(request, response) 
                && getSubject(request, response).isAuthenticated()) {
            return true;//已经认证过直接放行
        }
        return false;//转到拒绝访问处理逻辑
    }

    /**
     * 拒绝处理
     */
    protected boolean onAccessDenied(ServletRequest request, ServletResponse response)  
                                                                          throws Exception {
        if(isHmacSubmission(request)){//如果是Hmac鉴权的请求
            //创建令牌
            AuthenticationToken token = createToken(request, response);
            try {
                Subject subject = getSubject(request, response);
                subject.login(token);//认证
                return true;//认证成功，过滤器链继续
            } catch (AuthenticationException e) {//认证失败，发送401状态并附带异常信息
                log.error(e.getMessage(),e);
                WebUtils.toHttp(response).sendError(HttpServletResponse.SC_UNAUTHORIZED,e.getMessage());
            }
        }
        return false;//打住，访问到此为止
    }

    protected AuthenticationToken createToken(ServletRequest request, ServletResponse response) {
        String clientKey = request.getParameter(DEFAULT_CLIENTKEY_PARAM);
        String timeStamp= request.getParameter(DEFAULT_TIMESTAMP_PARAM);
        String digest= request.getParameter(DEFAUL_DIGEST_PARAM);
        Map<String, String[]> parameters = request.getParameterMap();
        String host = request.getRemoteHost();
        return new HmacToken(clientKey, timeStamp, digest, host,parameters);
    }
    
    protected boolean isHmacSubmission(ServletRequest request) {
        String clientKey = request.getParameter(DEFAULT_CLIENTKEY_PARAM);
        String timeStamp= request.getParameter(DEFAULT_TIMESTAMP_PARAM);
        String digest= request.getParameter(DEFAUL_DIGEST_PARAM);
        return (request instanceof HttpServletRequest)
                            && StringUtils.isNotBlank(clientKey)
                            && StringUtils.isNotBlank(timeStamp)
                            && StringUtils.isNotBlank(digest);
    }
}
```

HMAC鉴权最基础的工作就此完成，需要注意的是鉴权是无状态的不需要创建SESSION,所以需要对shiro的SubjectFactory做一下改造，并设置到SecurityManager ：

```
/**
 * 扩展自DefaultWebSubjectFactory,对于无状态的TOKEN 类型不创建session
 * @author wangjie (http://www.jianshu.com/u/ffa3cba4c604)
 * @date 2016年6月24日 下午2:55:15
 */
public class AgileSubjectFactory extends DefaultWebSubjectFactory { 

    public Subject createSubject(SubjectContext context) { 
        AuthenticationToken token = context.getAuthenticationToken();
        if((token instanceof HmacToken)){
            // 当token为HmacToken时， 不创建 session 
            context.setSessionCreationEnabled(false);
        }
        return super.createSubject(context); 
    }
}
```

JWT签发逻辑定义：

```
@RestController
@RequestMapping("/auth")
public class AuthenticateAction {
    private final String SECRET_KEY = "*(-=4eklfasdfarerf41585fdasf";

    @RequestMapping(value="/apply-token",method=RequestMethod.POST)
    public Map<String,Object> applyToken(@RequestParam(name="clientKey") String clientKey) {
        // 签发一个Json Web Token
        // 令牌ID=uuid，用户=clientKey，签发者=clientKey
        // token有效期=1分钟，用户角色=null,用户权限=create,read,update,delete
        String jwt = issueJwt(UUID.randomUUID().toString(), clientKey, 
                                    "token-server",60000l, null, "create,read,update,delete");
        Map<String,Object> respond = Maps.newHashMap();
        respond.put("jwt", jwt);
        return respond;
    }
    
    /**
     * @param id 令牌ID
     * @param subject 用户ID
     * @param issuer 签发人
     * @param period 有效时间(毫秒)
     * @param roles 访问主张-角色
     * @param permissions 访问主张-权限
     * @param algorithm 加密算法
     * @return json web token 
     */
    private String issueJwt(String id,String subject,String issuer,Long period,String roles
                                            ,String permissions,SignatureAlgorithm algorithm) {
        long currentTimeMillis = System.currentTimeMillis();// 当前时间戳
        byte[] secretKeyBytes = DatatypeConverter.parseBase64Binary(SECRET_KEY);// 秘钥
        JwtBuilder jwt  =  Jwts.builder();
        if(Strings.isNotBlank(id)) jwt.setId(id);
        jwt.setSubject(subject);// 用户名主题
        if(Strings.isNotBlank(issuer)) jwt.setIssuer(issuer);//签发者
        if(Strings.isNotBlank(issuer)) jwt.setIssuer(issuer);//签发者  
        jwt.setIssuedAt(new Date(currentTimeMillis));//签发时间
        if(null != period){
            Date expiration = new Date(currentTimeMillis+period);
            jwt.setExpiration(expiration);//有效时间
        }
        if(Strings.isNotBlank(roles)) jwt.claim("roles", roles);//角色
        if(Strings.isNotBlank(permissions)) jwt.claim("perms", permissions);//权限
        jwt.compressWith(CompressionCodecs.DEFLATE);//压缩，可选GZIP
        jwt.signWith(algorithm, secretKeyBytes);//加密设置
        return jwt.compact();
    }
}
```

添加过滤器：filterChainManager.addFilter( "hmac", new HmacFilter()）;
配置过滤规则：filterChainManager.addToChain("/auth/**", "hmac");
如果有需要可以在规则中添加其他过滤器。
JWT申请测试:

```
    @Test
    public String applyToken(){
        Long current = System.currentTimeMillis() ;
        String url = "http://localhost:8080/tokenServer/auth/apply-token";
        MultiValueMap<String, Object> dataMap = new LinkedMultiValueMap<String, Object>();  
        String clientKey = "administrator";// 客户端标识（用户名）
        String mix = String.valueOf(new Random().nextFloat());// 随机数，进行混淆
        String timeStamp = current.toString();// 时间戳
        dataMap.add("clientKey", clientKey);
        dataMap.add("mix", mix);
        dataMap.add("timeStamp", timeStamp);
        String baseString = clientKey+mix+timeStamp;
        String digest =  hmacDigest(baseString);// 生成HMAC摘要
        dataMap.add("digest", digest);
        Map result = rt.postForObject(url, dataMap, Map.class);
        return (String)result.get("jwt");
    }
```

返回JWT：

> eyJhbGciOiJIUzUxMiIsInppcCI6IkRFRiJ9.eNo8y80KwjAQBOB32XMCTfNj7NvsNluI2jYkWxHEdzf14GUYPmbecJMMEwzXGAjnUdvBR-2cdzpSRE2jiUtwSLQEUNAO6mNMa95yk4qy1665ta6y33nTjeuTf4gCk_HGWBvtxSjgV_mDsx0K1_U8zpVRWPVM6ijp7IkfLAyfLwAAAP__.GK7EJibs7n50uGksvvLK6Y39Ur6ZYXoXI9LOlFwEpIijHGAZjIyDhiYD-1nv1YbPJ46BI-gDTntV3KC0d8NSrA



## 令牌验签

有了JWT签发服务，要使用JWT就需要业务系统有JWT验鉴功能，同样在shiro中集成。
由于JWT是自包含的，令牌中已经声明了访问主张(比如角色、权限等)，验签功能只需要验证令牌合法就行了，不需要访问数据库。

shiro Token定义：

```
/**
 * JWT令牌
 * @author wangjie (http://www.jianshu.com/u/ffa3cba4c604) 
 * @date 2016年6月24日 下午2:55:15
 */
public class JwtToken implements AuthenticationToken{

    private static final long serialVersionUID = -790191688300000066L;
    
    private String jwt;// json web token
    private String host;// 客户端IP
    
    public JwtToken(String jwt,String host){
        this.jwt = jwt;
        this.host = host;
    }

    @Override
    public Object getPrincipal() {
        return this.jwt;
    }

    @Override
    public Object getCredentials() {
        return Boolean.TRUE;
    }
    
    // 忽略getters and setters
}
    
```

JWT Realm即认证逻辑定义：

```
/**
 * 基于JWT（ JSON WEB TOKEN）的认证域
 * 
 * @author wangjie (http://www.jianshu.com/u/ffa3cba4c604)
 * @date 2016年6月24日 下午2:55:15
 */
public class JwtRealm extends AuthorizingRealm {

    public Class<?> getAuthenticationTokenClass() {
        return JwtToken.class;//此Realm只支持JwtToken
    }

    /**
     * 认证
     */
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) 
                                                        throws AuthenticationException {
        JwtToken jwtToken = (JwtToken) token;
        String jwt = (String) jwtToken.getPrincipal();
        JwtPlayload jwtPlayload;
        try {
            Claims claims = Jwts.parser()
                    .setSigningKey(DatatypeConverter.parseBase64Binary(SECRETKEY))
                    .parseClaimsJws(jwt)
                    .getBody();
            jwtPlayload = new JwtPlayload();
            jwtPlayload.setId(claims.getId());
            jwtPlayload.setUserId(claims.getSubject());// 用户名
            jwtPlayload.setIssuer(claims.getIssuer());// 签发者
            jwtPlayload.setIssuedAt(claims.getIssuedAt());// 签发时间
            jwtPlayload.setAudience(claims.getAudience());// 接收方
            jwtPlayload.setRoles(claims.get("roles", String.class));// 访问主张-角色
            jwtPlayload.setPerms(claims.get("perms", String.class));// 访问主张-权限
        } catch (ExpiredJwtException e) {
            throw new AuthenticationException("JWT 令牌过期:" + e.getMessage());
        } catch (UnsupportedJwtException e) {
            throw new AuthenticationException("JWT 令牌无效:" + e.getMessage());
        } catch (MalformedJwtException e) {
            throw new AuthenticationException("JWT 令牌格式错误:" + e.getMessage());
        } catch (SignatureException e) {
            throw new AuthenticationException("JWT 令牌签名无效:" + e.getMessage());
        } catch (IllegalArgumentException e) {
            throw new AuthenticationException("JWT 令牌参数异常:" + e.getMessage());
        } catch (Exception e) {
            throw new AuthenticationException("JWT 令牌错误:" + e.getMessage());
        }
        // 如果要使token只能使用一次，此处可以过滤并缓存jwtPlayload.getId()
        // 可以做签发方验证
        // 可以做接收方验证
        return new SimpleAuthenticationInfo(jwtPlayload, Boolean.TRUE, getName());
    }

    /**
     * 授权,JWT已包含访问主张只需要解析其中的主张定义就行了
     */
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
        JwtPlayload jwtPlayload = (JwtPlayload) principals.getPrimaryPrincipal();
        SimpleAuthorizationInfo info = new SimpleAuthorizationInfo();
        // 解析角色并设置
        Set<String> roles = Sets.newHashSet(StringUtils.split(jwtPlayload.getRoles(), ","));
        info.setRoles(roles);
        // 解析权限并设置
        Set<String> permissions = Sets.newHashSet(StringUtils.split(jwtPlayload.getPerms(), ","));
        info.setStringPermissions(permissions);
        return info;
    }
}
```

处理逻辑中抛出的异常信息很详细，其实这样并不安全只是对调试友好，线上环境不用把异常信息给那么细。

JWT鉴权过滤器定义：

```
/**
 * 基于JWT标准的无状态认证过滤器
 * @author wangjie (http://www.jianshu.com/u/ffa3cba4c604) 
 * @date 2016年6月24日 下午2:55:15
 * 
 */ 
public class JwtFilter extends AccessControlFilter {
    
    private static final Logger log = LoggerFactory.getLogger(AccessControlFilter.class);
    
    public static final String DEFAULT_JWT_PARAM = "jwt";

    @Override
    protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception {
        if (null != getSubject(request, response) 
                && getSubject(request, response).isAuthenticated()) {
            return true;
        }
        return false;
    }

    @Override
    protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception {
        if(isJwtSubmission(request)){
            AuthenticationToken token = createToken(request, response);
            try {
                Subject subject = getSubject(request, response);
                subject.login(token);
                return true;
            } catch (AuthenticationException e) {
                log.error(e.getMessage(),e);
                WebUtils.toHttp(response).sendError(HttpServletResponse.SC_UNAUTHORIZED,e.getMessage());
            } 
        }
        return false;
    }

    protected AuthenticationToken createToken(ServletRequest request, ServletResponse response) {
        String jwt = request.getParameter(DEFAULT_JWT_PARAM);
        String host = request.getRemoteHost();
        log.info("authenticate jwt token:"+jwt);
        System.out.println("jwt:"+jwt);
        return new JwtToken(jwt, host);
    }
    
    protected boolean isJwtSubmission(ServletRequest request) {
        String jwt = request.getParameter(DEFAULT_JWT_PARAM);
        return (request instanceof HttpServletRequest)
                                && StringUtils.isNotBlank(jwt);
    }
    
}
```

资源访问权限过滤器定义：

```
/**
 * 基于JWT（ JSON WEB TOKEN）的无状态资源过滤器
 * @author wangjie (http://www.jianshu.com/u/ffa3cba4c604) 
 * @date 2016年6月24日 下午2:55:15
 */
public class JwtPermFilter extends HmacFilter{
    @Override
    protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, 
                                                    Object mappedValue) throws Exception {
        Subject subject = getSubject(request, response);
        String[] perms = (String[]) mappedValue;
        boolean isPermitted = true;
        if (perms != null && perms.length > 0) {
            if (perms.length == 1) {
                if (!subject.isPermitted(perms[0])) {
                    isPermitted = false;
                }
            } else {
                if (!subject.isPermittedAll(perms)) {
                    isPermitted = false;
                }
            }
        }
        return isPermitted;
    }
    
}
```

添加过滤器：filterChainManager.addFilter( "jwt", new JwtFilter()）;
filterChainManager.addFilter( "jwtPerms", new JwtPermFilter()）;
配置过滤规则：filterChainManager.addToChain("/api/**", "jwt");filterChainManager.addToChain("/api/delete/**", "jwtPerms["api:delete"]");
如果有需要可以在规则中添加其他过滤器。
同令牌申请服务一样，需要设置shiro不创建SESSION。



转自：[shiro jwt 构建无状态分布式鉴权体系](https://wangjie2016.iteye.com/blog/2406870)



## 项目推荐：jsets-shiro-spring-boot-starter

github地址：[jsets-shiro-spring-boot-starter](https://github.com/wj596/jsets-shiro-spring-boot-starter)

**项目说明：**

springboot中使用shiro大都是通过shiro-spring.jar进行的整合的,虽然不是太复杂，但是也无法做到spring-boot-starter风格的开箱即用。

项目中经常用到的功能比如：验证码、密码错误次数限制、账号唯一用户登陆、动态URL过滤规则、无状态鉴权等等，shiro还没有直接提供支持。

jsets-shiro-spring-boot-starter对这些常用的功能进行了封装和自动导入，少量的配置就可以应用在项目中。



**推荐理由：**

- springboot-start开箱即用

- 提供动态URL过滤规则的功能，让你可以从数据库中读取权限信息，修改的权限之后还可动态刷新url过滤规则
- 提供无状态鉴权功能