---
title: spring security实现动态配置url权限的两种方法
tags: ["Spring Boot","java","Spring Security","jqpeng"]
categories: ["博客","jqpeng"]
date: 2018-06-07 15:33
---
文章作者:jqpeng
原文链接: [spring security实现动态配置url权限的两种方法](https://www.cnblogs.com/xiaoqi/p/spring-security-rabc.html)

## 缘起

标准的RABC, 权限需要支持动态配置，spring security默认是在代码里约定好权限，真实的业务场景通常需要可以支持动态配置角色访问权限，即在运行时去配置url对应的访问角色。

基于spring security，如何实现这个需求呢？

最简单的方法就是自定义一个Filter去完成权限判断，但这脱离了spring security框架，如何基于spring security优雅的实现呢？

## spring security 授权回顾

spring security 通过FilterChainProxy作为注册到web的filter，FilterChainProxy里面一次包含了内置的多个过滤器，我们首先需要了解spring security内置的各种filter：


| Alias | Filter Class | Namespace Element or Attribute |
| --- | --- | --- |
| CHANNEL\_FILTER | ChannelProcessingFilter | http/intercept-url@requires-channel |
| SECURITY\_CONTEXT\_FILTER | SecurityContextPersistenceFilter | http |
| CONCURRENT\_SESSION\_FILTER | ConcurrentSessionFilter | session-management/concurrency-control |
| HEADERS\_FILTER | HeaderWriterFilter | http/headers |
| CSRF\_FILTER | CsrfFilter | http/csrf |
| LOGOUT\_FILTER | LogoutFilter | http/logout |
| X509\_FILTER | X509AuthenticationFilter | http/x509 |
| PRE\_AUTH\_FILTER | AbstractPreAuthenticatedProcessingFilter Subclasses | N/A |
| CAS\_FILTER | CasAuthenticationFilter | N/A |
| FORM\_LOGIN\_FILTER | UsernamePasswordAuthenticationFilter | http/form-login |
| BASIC\_AUTH\_FILTER | BasicAuthenticationFilter | http/http-basic |
| SERVLET\_API\_SUPPORT\_FILTER | SecurityContextHolderAwareRequestFilter | http/@servlet-api-provision |
| JAAS\_API\_SUPPORT\_FILTER | JaasApiIntegrationFilter | http/@jaas-api-provision |
| REMEMBER\_ME\_FILTER | RememberMeAuthenticationFilter | http/remember-me |
| ANONYMOUS\_FILTER | AnonymousAuthenticationFilter | http/anonymous |
| SESSION\_MANAGEMENT\_FILTER | SessionManagementFilter | session-management |
| EXCEPTION\_TRANSLATION\_FILTER | ExceptionTranslationFilter | http |
| FILTER\_SECURITY\_INTERCEPTOR | FilterSecurityInterceptor | http |
| SWITCH\_USER\_FILTER | SwitchUserFilter | N/A |


最重要的是`FilterSecurityInterceptor`，该过滤器实现了主要的鉴权逻辑，最核心的代码在这里：


    protected InterceptorStatusToken beforeInvocation(Object object) {    // 获取访问URL所需权限	Collection<ConfigAttribute> attributes = this.obtainSecurityMetadataSource()			.getAttributes(object);
    	Authentication authenticated = authenticateIfRequired();
    	// 通过accessDecisionManager鉴权	try {		this.accessDecisionManager.decide(authenticated, object, attributes);	}	catch (AccessDeniedException accessDeniedException) {		publishEvent(new AuthorizationFailureEvent(object, attributes, authenticated,				accessDeniedException));
    		throw accessDeniedException;	}
    	if (debug) {		logger.debug("Authorization successful");	}
    	if (publishAuthorizationSuccess) {		publishEvent(new AuthorizedEvent(object, attributes, authenticated));	}
    	// Attempt to run as a different user	Authentication runAs = this.runAsManager.buildRunAs(authenticated, object,			attributes);
    	if (runAs == null) {		if (debug) {			logger.debug("RunAsManager did not change Authentication object");		}
    		// no further work post-invocation		return new InterceptorStatusToken(SecurityContextHolder.getContext(), false,				attributes, object);	}	else {		if (debug) {			logger.debug("Switching to RunAs Authentication: " + runAs);		}
    		SecurityContext origCtx = SecurityContextHolder.getContext();		SecurityContextHolder.setContext(SecurityContextHolder.createEmptyContext());		SecurityContextHolder.getContext().setAuthentication(runAs);
    		// need to revert to token.Authenticated post-invocation		return new InterceptorStatusToken(origCtx, true, attributes, object);	}}


从上面可以看出，要实现动态鉴权，可以从两方面着手：

- 自定义SecurityMetadataSource，实现从数据库加载ConfigAttribute
- 另外就是可以自定义accessDecisionManager，官方的UnanimousBased其实足够使用，并且他是基于AccessDecisionVoter来实现权限认证的，因此我们只需要自定义一个AccessDecisionVoter就可以了


下面来看分别如何实现。

## 自定义AccessDecisionManager

官方的三个AccessDecisionManager都是基于AccessDecisionVoter来实现权限认证的，因此我们只需要自定义一个AccessDecisionVoter就可以了。

自定义主要是实现`AccessDecisionVoter`接口，我们可以仿照官方的RoleVoter实现一个：


    
    public class RoleBasedVoter implements AccessDecisionVoter<Object> {
    
        @Override
        public boolean supports(ConfigAttribute attribute) {
            return true;
        }
    
        @Override
        public int vote(Authentication authentication, Object object, Collection<ConfigAttribute> attributes) {
            if(authentication == null) {
                return ACCESS_DENIED;
            }
            int result = ACCESS_ABSTAIN;
            Collection<? extends GrantedAuthority> authorities = extractAuthorities(authentication);
    
            for (ConfigAttribute attribute : attributes) {
                if(attribute.getAttribute()==null){
                    continue;
                }
                if (this.supports(attribute)) {
                    result = ACCESS_DENIED;
    
                    // Attempt to find a matching granted authority
                    for (GrantedAuthority authority : authorities) {
                        if (attribute.getAttribute().equals(authority.getAuthority())) {
                            return ACCESS_GRANTED;
                        }
                    }
                }
            }
    
            return result;
        }
    
        Collection<? extends GrantedAuthority> extractAuthorities(
            Authentication authentication) {
            return authentication.getAuthorities();
        }
    
        @Override
        public boolean supports(Class clazz) {
            return true;
        }
    }


如何加入动态权限呢？

`vote(Authentication authentication, Object object, Collection<ConfigAttribute> attributes) `里的`Object object`的类型是`FilterInvocation`，可以通过`getRequestUrl`获取当前请求的URL:


      FilterInvocation fi = (FilterInvocation) object;
      String url = fi.getRequestUrl();


因此这里扩展空间就大了，可以从DB动态加载，然后判断URL的ConfigAttribute就可以了。

如何使用这个RoleBasedVoter呢？在configure里使用accessDecisionManager方法自定义，我们还是使用官方的`UnanimousBased`，然后将自定义的RoleBasedVoter加入即可。


    @EnableWebSecurity
    @EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true)
    public class SecurityConfiguration extends WebSecurityConfigurerAdapter {
    
     
        @Override
        protected void configure(HttpSecurity http) throws Exception {
            http
                .addFilterBefore(corsFilter, UsernamePasswordAuthenticationFilter.class)
                .exceptionHandling()
                .authenticationEntryPoint(problemSupport)
                .accessDeniedHandler(problemSupport)
            .and()
                .csrf()
                .disable()
                .headers()
                .frameOptions()
                .disable()
            .and()
                .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
                .authorizeRequests()
                // 自定义accessDecisionManager
                .accessDecisionManager(accessDecisionManager())
              
            .and()
                .apply(securityConfigurerAdapter());
    
        }
    
    
        @Bean
        public AccessDecisionManager accessDecisionManager() {
            List<AccessDecisionVoter<? extends Object>> decisionVoters
                = Arrays.asList(
                new WebExpressionVoter(),
                // new RoleVoter(),
                new RoleBasedVoter(),
                new AuthenticatedVoter());
            return new UnanimousBased(decisionVoters);
        }


## 自定义SecurityMetadataSource

自定义FilterInvocationSecurityMetadataSource只要实现接口即可，在接口里从DB动态加载规则。

为了复用代码里的定义，我们可以将代码里生成的SecurityMetadataSource带上，在构造函数里传入默认的FilterInvocationSecurityMetadataSource。


    public class AppFilterInvocationSecurityMetadataSource implements org.springframework.security.web.access.intercept.FilterInvocationSecurityMetadataSource {
    
        private FilterInvocationSecurityMetadataSource  superMetadataSource;
    
        @Override
        public Collection<ConfigAttribute> getAllConfigAttributes() {
            return null;
        }
    
        public AppFilterInvocationSecurityMetadataSource(FilterInvocationSecurityMetadataSource expressionBasedFilterInvocationSecurityMetadataSource){
             this.superMetadataSource = expressionBasedFilterInvocationSecurityMetadataSource;
    
             // TODO 从数据库加载权限配置
        }
    
        private final AntPathMatcher antPathMatcher = new AntPathMatcher();
        // 这里的需要从DB加载
        private final Map<String,String> urlRoleMap = new HashMap<String,String>(){{
            put("/open/**","ROLE_ANONYMOUS");
            put("/health","ROLE_ANONYMOUS");
            put("/restart","ROLE_ADMIN");
            put("/demo","ROLE_USER");
        }};
    
        @Override
        public Collection<ConfigAttribute> getAttributes(Object object) throws IllegalArgumentException {
            FilterInvocation fi = (FilterInvocation) object;
            String url = fi.getRequestUrl();
    
            for(Map.Entry<String,String> entry:urlRoleMap.entrySet()){
                if(antPathMatcher.match(entry.getKey(),url)){
                    return SecurityConfig.createList(entry.getValue());
                }
            }
    
            //  返回代码定义的默认配置
            return superMetadataSource.getAttributes(object);
        }
    
    
    
        @Override
        public boolean supports(Class<?> clazz) {
            return FilterInvocation.class.isAssignableFrom(clazz);
        }
    }


怎么使用？和`accessDecisionManager`不一样，`ExpressionUrlAuthorizationConfigurer` 并没有提供set方法设置`FilterSecurityInterceptor`的`FilterInvocationSecurityMetadataSource`，how to do?

发现一个扩展方法`withObjectPostProcessor`，通过该方法自定义一个处理`FilterSecurityInterceptor`类型的`ObjectPostProcessor`就可以修改`FilterSecurityInterceptor`。


    @EnableWebSecurity
    @EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true)
    public class SecurityConfiguration extends WebSecurityConfigurerAdapter {
    
     
        @Override
        protected void configure(HttpSecurity http) throws Exception {
            http
                .addFilterBefore(corsFilter, UsernamePasswordAuthenticationFilter.class)
                .exceptionHandling()
                .authenticationEntryPoint(problemSupport)
                .accessDeniedHandler(problemSupport)
            .and()
                .csrf()
                .disable()
                .headers()
                .frameOptions()
                .disable()
            .and()
                .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
                .authorizeRequests()
      			// 自定义FilterInvocationSecurityMetadataSource
                .withObjectPostProcessor(new ObjectPostProcessor<FilterSecurityInterceptor>() {
                    @Override
                    public <O extends FilterSecurityInterceptor> O postProcess(
                        O fsi) {
                        fsi.setSecurityMetadataSource(mySecurityMetadataSource(fsi.getSecurityMetadataSource()));
                        return fsi;
                    }
                })
            .and()
                .apply(securityConfigurerAdapter());
    
        }
    
    
        @Bean
        public AppFilterInvocationSecurityMetadataSource mySecurityMetadataSource(FilterInvocationSecurityMetadataSource filterInvocationSecurityMetadataSource) {
            AppFilterInvocationSecurityMetadataSource securityMetadataSource = new AppFilterInvocationSecurityMetadataSource(filterInvocationSecurityMetadataSource);
            return securityMetadataSource;
    }
    


## 小结

本文介绍了两种基于spring security实现动态权限的方法，一是自定义accessDecisionManager，二是自定义FilterInvocationSecurityMetadataSource。实际项目里可以根据需要灵活选择。

延伸阅读:

[Spring Security 架构与源码分析](http://www.cnblogs.com/xiaoqi/p/spring-security.html)

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


