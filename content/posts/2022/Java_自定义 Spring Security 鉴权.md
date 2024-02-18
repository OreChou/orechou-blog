---
title: 自定义 Spring Security 鉴权
date: 2022-06-01 17:00:00
tags: [Java, Spring]
---

# 背景
当我们使用 Spring 技术栈搭建单体的系统或服务时，若系统涉及到登录鉴权等功能，一般会使用 SpringSecurity 搭配一个 RBAC 的权限模型，很容易实现一套 OAuth2 鉴权、授权的流程。
随着业务扩展，单体的服务作为一个微服务并入一个大的系统之后。我们为保证其他业务能够调用该单体服务，但是又无法让其他系统来使用该服务已有鉴权，从而引入其他微服务都用基础的鉴权服务。根据业务场景的区分，当单体服务内部使用的时候走 SpringSecurity 的鉴权，单体服务外部调用的时候使用基础鉴权服务鉴权。

# 基础
要根据不同的场景启用或绕过 SpringSecurity。我们可以首先想到能否通过 URL 来区分不同的场景，使用 `HttpSecurity` 的 `permitAll()` 和 `authenticated()` 来让不同路径的接口是否需要走鉴权。除此之外，可以去修改 SpringSecurity 的 Filter ，使用自定义 Filter 或者用动态代理对 Filter 进行增强。

# 实现
```Java
@Aspect
@Component
@Slf4j
public class AuthorizationHeaderAspect {

    /**
     * 基础鉴权服务 Feign 调用业务
     */
    @Autowired
    private OauthFeignBiz oauthFeignBiz;

    @Pointcut("execution(* org.springframework.security.oauth2.provider.authentication.OAuth2AuthenticationProcessingFilter.doFilter(..))")
    public void securityOauth2DoFilter() {
    }

    @Pointcut("execution(* org.springframework.security.web.access.intercept.FilterSecurityInterceptor.doFilter(..))")
    public void securitySecurityInterceptor2DoFilter() {

    }

    @Around("securityOauth2DoFilter()")
    public void enhanceSecurityOauth2DoFilter(ProceedingJoinPoint joinPoint) throws Throwable {
        Object[] args = joinPoint.getArgs();
        if (args == null || args.length != 3 || !(args[0] instanceof HttpServletRequest && args[1] instanceof javax.servlet.ServletResponse && args[2] instanceof FilterChain)) {
            joinPoint.proceed();
            return;
        }

        HttpServletRequest request = (HttpServletRequest) args[0];
        String accessToken = request.getHeader("Authorization");

        if (StringUtils.isNotBlank(request.getParameter("sceneId")) && StringUtils.isNotBlank(request.getParameter("sceneType"))) {
            // 这里我们可以根据业务场景确定走哪种鉴权方式
            SecurityUtils.setBaseAuth(true);
        } else {
            SecurityUtils.setBaseAuth(false);
        }

        if (SecurityUtils.isBaseAuth()) {
            Response<CheckTokenResponse> checkTokenResponse = oauthFeignBiz.checkToken(accessToken);
            if (checkTokenResponse.getCode() == 0) {
                SecurityUtils.setBaseAuth((true));
                ((FilterChain) args[2]).doFilter((ServletRequest) args[0], (ServletResponse) args[1]);
            } else {
                throw new Exception("鉴权失败");
            }
        } else {
            // 让原 Filter 的逻辑继续执行
            joinPoint.proceed();
        }
    }

    @Around("securitySecurityInterceptor2DoFilter()")
    public void enhanceSecuritySecurityInterceptor2DoFilter(ProceedingJoinPoint joinPoint) throws Throwable {
        Object[] args = joinPoint.getArgs();
        if (args == null || args.length != 3 || !(args[0] instanceof HttpServletRequest && args[1] instanceof javax.servlet.ServletResponse && args[2] instanceof FilterChain)) {
            joinPoint.proceed();
            return;
        }

        if (!SecurityUtils.isBaseOauth()) {
            joinPoint.proceed();
            return;
        }
        ((FilterChain) args[2]).doFilter((ServletRequest) args[0], (ServletResponse) args[1]);
    }

}
```