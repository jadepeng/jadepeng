---
title: Spring Security 架构与源码分析
tags: ["Spring Boot","java","jqpeng"]
categories: ["博客","jqpeng"]
date: 2018-06-06 16:03
---
文章作者:jqpeng
原文链接: [Spring Security 架构与源码分析](https://www.cnblogs.com/xiaoqi/p/spring-security.html)

Spring Security 主要实现了Authentication（认证，解决who are you? ） 和 Access Control（访问控制，也就是what are you allowed to do？，也称为Authorization）。Spring Security在架构上将认证与授权分离，并提供了扩展点。

## 核心对象

主要代码在`spring-security-core`包下面。要了解Spring Security，需要先关注里面的核心对象。

### SecurityContextHolder, SecurityContext 和 Authentication

SecurityContextHolder 是 SecurityContext的存放容器，默认使用ThreadLocal 存储，意味SecurityContext在相同线程中的方法都可用。  
 SecurityContext主要是存储应用的principal信息，在Spring Security中用Authentication 来表示。

获取principal：


    Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();
    
    if (principal instanceof UserDetails) {
    String username = ((UserDetails)principal).getUsername();
    } else {
    String username = principal.toString();
    }


在Spring Security中，可以看一下Authentication定义：


    public interface Authentication extends Principal, Serializable {
    Collection<? extends GrantedAuthority> getAuthorities();
    /** * 通常是密码 */Object getCredentials();
    /** * Stores additional details about the authentication request. These might be an IP * address, certificate serial number etc. */Object getDetails();
    /** * 用来标识是否已认证，如果使用用户名和密码登录,通常是用户名  */Object getPrincipal();
    /** * 是否已认证 */boolean isAuthenticated();
    void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException;
    }


在实际应用中，通常使用`UsernamePasswordAuthenticationToken`：


    public abstract class AbstractAuthenticationToken implements Authentication,	CredentialsContainer {	}
    public class UsernamePasswordAuthenticationToken extends AbstractAuthenticationToken {
    }


一个常见的认证过程通常是这样的，创建一个UsernamePasswordAuthenticationToken，然后交给authenticationManager认证（后面详细说明），认证通过则通过SecurityContextHolder存放Authentication信息。


     UsernamePasswordAuthenticationToken authenticationToken =
                new UsernamePasswordAuthenticationToken(loginVM.getUsername(), loginVM.getPassword());
    
    Authentication authentication = this.authenticationManager.authenticate(authenticationToken);
    SecurityContextHolder.getContext().setAuthentication(authentication);


### UserDetails与UserDetailsService

UserDetails 是Spring Security里的一个关键接口，他用来表示一个principal。


    public interface UserDetails extends Serializable {/** * 用户的授权信息，可以理解为角色 */Collection<? extends GrantedAuthority> getAuthorities();
    /** * 用户密码 * * @return the password */String getPassword();
    /** * 用户名  *	 */String getUsername();
    boolean isAccountNonExpired();
    boolean isAccountNonLocked();
    boolean isCredentialsNonExpired();
    boolean isEnabled();
    }


UserDetails提供了认证所需的必要信息，在实际使用里，可以自己实现UserDetails，并增加额外的信息，比如email、mobile等信息。

在Authentication中的principal通常是用户名，我们可以通过UserDetailsService来通过principal获取UserDetails：


    public interface UserDetailsService {UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
    }


### GrantedAuthority

在UserDetails里说了，GrantedAuthority可以理解为角色，例如 `ROLE_ADMINISTRATOR` or `ROLE_HR_SUPERVISOR`。

### 小结

- `SecurityContextHolder`, 用来访问 `SecurityContext`.
- `SecurityContext`, 用来存储`Authentication` .
- `Authentication`, 代表凭证.
- `GrantedAuthority`, 代表权限.
- `UserDetails`, 用户信息.
- `UserDetailsService`,获取用户信息.


## Authentication认证

### AuthenticationManager

实现认证主要是通过AuthenticationManager接口，它只包含了一个方法：


    public interface AuthenticationManager {
      Authentication authenticate(Authentication authentication)
        throws AuthenticationException;
    }


authenticate()方法主要做三件事：

1. 如果验证通过，返回Authentication（通常带上authenticated=true）。
2. 认证失败抛出`AuthenticationException`
3. 如果无法确定，则返回null


`AuthenticationException`是运行时异常,它通常由应用程序按通用方式处理，用户代码通常不用特意被捕获和处理这个异常。

`AuthenticationManager`的默认实现是`ProviderManager`，它委托一组`AuthenticationProvider`实例来实现认证。  
`AuthenticationProvider`和`AuthenticationManager`类似，都包含`authenticate`，但它有一个额外的方法`supports`，以允许查询调用方是否支持给定`Authentication`类型：


    public interface AuthenticationProvider {
    Authentication authenticate(Authentication authentication)		throws AuthenticationException;boolean supports(Class<?> authentication);
    }


ProviderManager包含一组`AuthenticationProvider`，执行authenticate时，遍历Providers，然后调用supports，如果支持，则执行遍历当前provider的authenticate方法，如果一个provider认证成功，则break。


    public Authentication authenticate(Authentication authentication)		throws AuthenticationException {	Class<? extends Authentication> toTest = authentication.getClass();	AuthenticationException lastException = null;	Authentication result = null;	boolean debug = logger.isDebugEnabled();
    	for (AuthenticationProvider provider : getProviders()) {		if (!provider.supports(toTest)) {			continue;		}
    		if (debug) {			logger.debug("Authentication attempt using "					+ provider.getClass().getName());		}
    		try {			result = provider.authenticate(authentication);
    			if (result != null) {				copyDetails(authentication, result);				break;			}		}		catch (AccountStatusException e) {			prepareException(e, authentication);			// SEC-546: Avoid polling additional providers if auth failure is due to			// invalid account status			throw e;		}		catch (InternalAuthenticationServiceException e) {			prepareException(e, authentication);			throw e;		}		catch (AuthenticationException e) {			lastException = e;		}	}
    	if (result == null && parent != null) {		// Allow the parent to try.		try {			result = parent.authenticate(authentication);		}		catch (ProviderNotFoundException e) {			// ignore as we will throw below if no other exception occurred prior to			// calling parent and the parent			// may throw ProviderNotFound even though a provider in the child already			// handled the request		}		catch (AuthenticationException e) {			lastException = e;		}	}
    	if (result != null) {		if (eraseCredentialsAfterAuthentication				&& (result instanceof CredentialsContainer)) {			// Authentication is complete. Remove credentials and other secret data			// from authentication			((CredentialsContainer) result).eraseCredentials();		}
    		eventPublisher.publishAuthenticationSuccess(result);		return result;	}
    	// Parent was null, or didn't authenticate (or throw an exception).
    	if (lastException == null) {		lastException = new ProviderNotFoundException(messages.getMessage(				"ProviderManager.providerNotFound",				new Object[] { toTest.getName() },				"No AuthenticationProvider found for {0}"));	}
    	prepareException(lastException, authentication);
    	throw lastException;}


从上面的代码可以看出， `ProviderManager`有一个可选parent，如果parent不为空，则调用`parent.authenticate(authentication)`

### AuthenticationProvider

`AuthenticationProvider`有多种实现，大家最关注的通常是`DaoAuthenticationProvider`，继承于`AbstractUserDetailsAuthenticationProvider`，核心是通过`UserDetails`来实现认证,`DaoAuthenticationProvider`默认会自动加载，不用手动配。

先来看`AbstractUserDetailsAuthenticationProvide`r，看最核心的`authenticate`：


    public Authentication authenticate(Authentication authentication)		throws AuthenticationException {	// 必须是UsernamePasswordAuthenticationToken	Assert.isInstanceOf(UsernamePasswordAuthenticationToken.class, authentication,			messages.getMessage(					"AbstractUserDetailsAuthenticationProvider.onlySupports",					"Only UsernamePasswordAuthenticationToken is supported"));
    	//  获取用户名	String username = (authentication.getPrincipal() == null) ? "NONE_PROVIDED"			: authentication.getName();
    	boolean cacheWasUsed = true;	// 从缓存获取	UserDetails user = this.userCache.getUserFromCache(username);
    	if (user == null) {		cacheWasUsed = false;
    		try {		   // retrieveUser 抽象方法，获取用户			user = retrieveUser(username,					(UsernamePasswordAuthenticationToken) authentication);		}		catch (UsernameNotFoundException notFound) {			logger.debug("User '" + username + "' not found");
    			if (hideUserNotFoundExceptions) {				throw new BadCredentialsException(messages.getMessage(						"AbstractUserDetailsAuthenticationProvider.badCredentials",						"Bad credentials"));			}			else {				throw notFound;			}		}
      		Assert.notNull(user,				"retrieveUser returned null - a violation of the interface contract");	}
    	try {	    // 预先检查，DefaultPreAuthenticationChecks，检查用户是否被lock或者账号是否可用		preAuthenticationChecks.check(user);				// 抽象方法，自定义检验		additionalAuthenticationChecks(user,				(UsernamePasswordAuthenticationToken) authentication);	}	catch (AuthenticationException exception) {		if (cacheWasUsed) {			// There was a problem, so try again after checking			// we're using latest data (i.e. not from the cache)			cacheWasUsed = false;			user = retrieveUser(username,					(UsernamePasswordAuthenticationToken) authentication);			preAuthenticationChecks.check(user);			additionalAuthenticationChecks(user,					(UsernamePasswordAuthenticationToken) authentication);		}		else {			throw exception;		}	}
              // 后置检查 DefaultPostAuthenticationChecks，检查isCredentialsNonExpired	postAuthenticationChecks.check(user);
    	if (!cacheWasUsed) {		this.userCache.putUserInCache(user);	}
    	Object principalToReturn = user;
    	if (forcePrincipalAsString) {		principalToReturn = user.getUsername();	}
       	return createSuccessAuthentication(principalToReturn, authentication, user);}


上面的检验主要基于UserDetails实现，其中获取用户和检验逻辑由具体的类去实现，默认实现是DaoAuthenticationProvider，这个类的核心是让开发者提供UserDetailsService来获取UserDetails以及 PasswordEncoder来检验密码是否有效：


    private UserDetailsService userDetailsService;
    private PasswordEncoder passwordEncoder;


看具体的实现，`retrieveUser`,直接调用userDetailsService获取用户：


    protected final UserDetails retrieveUser(String username,		UsernamePasswordAuthenticationToken authentication)		throws AuthenticationException {	UserDetails loadedUser;
    	try {		loadedUser = this.getUserDetailsService().loadUserByUsername(username);	}	catch (UsernameNotFoundException notFound) {		if (authentication.getCredentials() != null) {			String presentedPassword = authentication.getCredentials().toString();			passwordEncoder.isPasswordValid(userNotFoundEncodedPassword,					presentedPassword, null);		}		throw notFound;	}	catch (Exception repositoryProblem) {		throw new InternalAuthenticationServiceException(				repositoryProblem.getMessage(), repositoryProblem);	}
    	if (loadedUser == null) {		throw new InternalAuthenticationServiceException(				"UserDetailsService returned null, which is an interface contract violation");	}	return loadedUser;}


再来看验证：


    protected void additionalAuthenticationChecks(UserDetails userDetails,		UsernamePasswordAuthenticationToken authentication)		throws AuthenticationException {	Object salt = null;
    	if (this.saltSource != null) {		salt = this.saltSource.getSalt(userDetails);	}
    	if (authentication.getCredentials() == null) {		logger.debug("Authentication failed: no credentials provided");
    		throw new BadCredentialsException(messages.getMessage(				"AbstractUserDetailsAuthenticationProvider.badCredentials",				"Bad credentials"));	}
            // 获取用户密码	String presentedPassword = authentication.getCredentials().toString();
            // 比较passwordEncoder后的密码是否和userdetails的密码一致	if (!passwordEncoder.isPasswordValid(userDetails.getPassword(),			presentedPassword, salt)) {		logger.debug("Authentication failed: password does not match stored value");
    		throw new BadCredentialsException(messages.getMessage(				"AbstractUserDetailsAuthenticationProvider.badCredentials",				"Bad credentials"));	}}


小结：要自定义认证，使用DaoAuthenticationProvider，只需要为其提供PasswordEncoder和UserDetailsService就可以了。

### 定制 Authentication Managers

Spring Security提供了一个Builder类`AuthenticationManagerBuilder`，借助它可以快速实现自定义认证。

看官方源码说明：


> SecurityBuilder used to create an AuthenticationManager . Allows for easily building in memory authentication, LDAP authentication, JDBC based authentication, adding UserDetailsService , and adding AuthenticationProvider's.


AuthenticationManagerBuilder可以用来Build一个AuthenticationManager，可以创建基于内存的认证、LDAP认证、 JDBC认证，以及添加UserDetailsService和AuthenticationProvider。

简单使用：


    @Configuration
    @EnableWebSecurity
    @EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true)
    public class ApplicationSecurity extends WebSecurityConfigurerAdapter {
    
    
      public SecurityConfiguration(AuthenticationManagerBuilder authenticationManagerBuilder, UserDetailsService userDetailsService,TokenProvider tokenProvider,CorsFilter corsFilter, SecurityProblemSupport problemSupport) {
            this.authenticationManagerBuilder = authenticationManagerBuilder;
            this.userDetailsService = userDetailsService;
            this.tokenProvider = tokenProvider;
            this.corsFilter = corsFilter;
            this.problemSupport = problemSupport;
        }
    
        @PostConstruct
        public void init() {
            try {
                authenticationManagerBuilder
                    .userDetailsService(userDetailsService)
                    .passwordEncoder(passwordEncoder());
            } catch (Exception e) {
                throw new BeanInitializationException("Security configuration failed", e);
            }
        }
    
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
                .antMatchers("/api/register").permitAll()
                .antMatchers("/api/activate").permitAll()
                .antMatchers("/api/authenticate").permitAll()
                .antMatchers("/api/account/reset-password/init").permitAll()
                .antMatchers("/api/account/reset-password/finish").permitAll()
                .antMatchers("/api/profile-info").permitAll()
                .antMatchers("/api/**").authenticated()
                .antMatchers("/management/health").permitAll()
                .antMatchers("/management/**").hasAuthority(AuthoritiesConstants.ADMIN)
                .antMatchers("/v2/api-docs/**").permitAll()
                .antMatchers("/swagger-resources/configuration/ui").permitAll()
                .antMatchers("/swagger-ui/index.html").hasAuthority(AuthoritiesConstants.ADMIN)
            .and()
                .apply(securityConfigurerAdapter());
    
        }
    }


## 授权与访问控制

一旦认证成功，我们可以继续进行授权，授权是通过`AccessDecisionManager`来实现的。框架有三种实现，默认是AffirmativeBased，通过`AccessDecisionVoter`决策，有点像`ProviderManager`委托给`AuthenticationProviders`来认证。


    public void decide(Authentication authentication, Object object,		Collection<ConfigAttribute> configAttributes) throws AccessDeniedException {	int deny = 0;
            // 遍历DecisionVoter 	for (AccessDecisionVoter voter : getDecisionVoters()) {	    // 投票		int result = voter.vote(authentication, object, configAttributes);
    		if (logger.isDebugEnabled()) {			logger.debug("Voter: " + voter + ", returned: " + result);		}
    		switch (result) {		case AccessDecisionVoter.ACCESS_GRANTED:			return;
    		case AccessDecisionVoter.ACCESS_DENIED:			deny++;
    			break;
    		default:			break;		}	}
               // 一票否决	if (deny > 0) {		throw new AccessDeniedException(messages.getMessage(				"AbstractAccessDecisionManager.accessDenied", "Access is denied"));	}
    	// To get this far, every AccessDecisionVoter abstained	checkAllowIfAllAbstainDecisions();}


来看AccessDecisionVoter：


    boolean supports(ConfigAttribute attribute);
    
    boolean supports(Class<?> clazz);
    
    int vote(Authentication authentication, S object,
            Collection<ConfigAttribute> attributes);


object是用户要访问的资源，ConfigAttribute则是访问object要满足的条件，通常payload是字符串，比如ROLE\_ADMIN 。所以我们来看下RoleVoter的实现，其核心就是从authentication提取出GrantedAuthority，然后和ConfigAttribute比较是否满足条件。


    
    public boolean supports(ConfigAttribute attribute) {	if ((attribute.getAttribute() != null)			&& attribute.getAttribute().startsWith(getRolePrefix())) {		return true;	}	else {		return false;	}}
    public boolean supports(Class<?> clazz) {	return true;}
    
    
    public int vote(Authentication authentication, Object object,		Collection<ConfigAttribute> attributes) {	if(authentication == null) {		return ACCESS_DENIED;	}	int result = ACCESS_ABSTAIN;		// 获取GrantedAuthority信息	Collection<? extends GrantedAuthority> authorities = extractAuthorities(authentication);
    	for (ConfigAttribute attribute : attributes) {		if (this.supports(attribute)) {		    // 默认拒绝访问			result = ACCESS_DENIED;
    			// Attempt to find a matching granted authority			for (GrantedAuthority authority : authorities) {			     // 判断是否有匹配的 authority				if (attribute.getAttribute().equals(authority.getAuthority())) {				    // 可访问					return ACCESS_GRANTED;				}			}		}	}
    	return result;}


这里要疑问，ConfigAttribute哪来的？其实就是上面ApplicationSecurity的configure里的。

### web security 如何实现

Web层中的Spring Security（用于UI和HTTP后端）基于Servlet `Filters`，下图显示了单个HTTP请求的处理程序的典型分层。

![过滤链委托给一个Servlet](https://github.com/spring-guides/top-spring-security-architecture/raw/master/images/filters.png)

Spring Security通过`FilterChainProxy`作为单一的Filter注册到web层，Proxy内部的Filter。

![Spring安全筛选器](https://github.com/spring-guides/top-spring-security-architecture/raw/master/images/security-filters.png)

FilterChainProxy相当于一个filter的容器，通过VirtualFilterChain来依次调用各个内部filter


    
    
    public void doFilter(ServletRequest request, ServletResponse response,		FilterChain chain) throws IOException, ServletException {	boolean clearContext = request.getAttribute(FILTER_APPLIED) == null;	if (clearContext) {		try {			request.setAttribute(FILTER_APPLIED, Boolean.TRUE);			doFilterInternal(request, response, chain);		}		finally {			SecurityContextHolder.clearContext();			request.removeAttribute(FILTER_APPLIED);		}	}	else {		doFilterInternal(request, response, chain);	}}
    private void doFilterInternal(ServletRequest request, ServletResponse response,		FilterChain chain) throws IOException, ServletException {
    	FirewalledRequest fwRequest = firewall			.getFirewalledRequest((HttpServletRequest) request);	HttpServletResponse fwResponse = firewall			.getFirewalledResponse((HttpServletResponse) response);
    	List<Filter> filters = getFilters(fwRequest);
    	if (filters == null || filters.size() == 0) {		if (logger.isDebugEnabled()) {			logger.debug(UrlUtils.buildRequestUrl(fwRequest)					+ (filters == null ? " has no matching filters"							: " has an empty filter list"));		}
    		fwRequest.reset();
    		chain.doFilter(fwRequest, fwResponse);
    		return;	}
    	VirtualFilterChain vfc = new VirtualFilterChain(fwRequest, chain, filters);	vfc.doFilter(fwRequest, fwResponse);}private static class VirtualFilterChain implements FilterChain {	private final FilterChain originalChain;	private final List<Filter> additionalFilters;	private final FirewalledRequest firewalledRequest;	private final int size;	private int currentPosition = 0;
    	private VirtualFilterChain(FirewalledRequest firewalledRequest,			FilterChain chain, List<Filter> additionalFilters) {		this.originalChain = chain;		this.additionalFilters = additionalFilters;		this.size = additionalFilters.size();		this.firewalledRequest = firewalledRequest;	}
    	public void doFilter(ServletRequest request, ServletResponse response)			throws IOException, ServletException {		if (currentPosition == size) {			if (logger.isDebugEnabled()) {				logger.debug(UrlUtils.buildRequestUrl(firewalledRequest)						+ " reached end of additional filter chain; proceeding with original chain");			}
    			// Deactivate path stripping as we exit the security filter chain			this.firewalledRequest.reset();
    			originalChain.doFilter(request, response);		}		else {			currentPosition++;
    			Filter nextFilter = additionalFilters.get(currentPosition - 1);
    			if (logger.isDebugEnabled()) {				logger.debug(UrlUtils.buildRequestUrl(firewalledRequest)						+ " at position " + currentPosition + " of " + size						+ " in additional filter chain; firing Filter: '"						+ nextFilter.getClass().getSimpleName() + "'");			}
    			nextFilter.doFilter(request, response, this);		}	}}


## 参考

- [https://spring.io/guides/topicals/spring-security-architecture/](https://spring.io/guides/topicals/spring-security-architecture/)
- [https://docs.spring.io/spring-security/site/docs/5.0.5.RELEASE/reference/htmlsingle/#overall-architecture](https://docs.spring.io/spring-security/site/docs/5.0.5.RELEASE/reference/htmlsingle/#overall-architecture)


* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


