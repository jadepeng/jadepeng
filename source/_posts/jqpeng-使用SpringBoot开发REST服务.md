---
title: 使用SpringBoot开发REST服务
tags: ["java","REST服务","Spring Boot","jqpeng"]
categories: ["博客","jqpeng"]
date: 2017-12-08 16:35
---
文章作者:jqpeng
原文链接: [使用SpringBoot开发REST服务](https://www.cnblogs.com/xiaoqi/p/SpringBootRest.html)

本文介绍如何基于Spring Boot搭建一个简易的REST服务框架，以及如何通过自定义注解实现Rest服务鉴权

# 搭建框架

## pom.xml

首先，引入相关依赖，数据库使用mongodb，同时使用redis做缓存


    注意，这里没有使用tomcat，而是使用undertow



    	<dependency>		<groupId>org.springframework.boot</groupId>		<artifactId>spring-boot-starter</artifactId>	</dependency>
    	<dependency>		<groupId>org.springframework.boot</groupId>		<artifactId>spring-boot-starter-test</artifactId>		<scope>test</scope>	</dependency>
    	<dependency>		<groupId>org.springframework.boot</groupId>		<artifactId>spring-boot-starter-web</artifactId>		<exclusions>			<exclusion>				<groupId>org.springframework.boot</groupId>				<artifactId>spring-boot-starter-tomcat</artifactId>			</exclusion>		</exclusions>	</dependency>	<dependency>		<groupId>org.springframework.boot</groupId>		<artifactId>spring-boot-starter-undertow</artifactId>	</dependency>
    	<!--redis支持-->	<dependency>		<groupId>org.springframework.boot</groupId>		<artifactId>spring-boot-starter-data-redis</artifactId>	</dependency>
    	<!--mongodb支持-->	<dependency>		<groupId>org.springframework.boot</groupId>		<artifactId>spring-boot-starter-data-mongodb</artifactId>	</dependency>


- 引入spring-boot-starter-web支持web服务
- 引入spring-boot-starter-data-redis 和spring-boot-starter-data-mongodb就可以方便的使用mongodb和redis了


## 配置文件

### profiles功能

为了方便 区分开发环境和线上环境，可以使用profiles功能，在application.properties里增加  
 spring.profiles.active=dev

然后增加application-dev.properties作为dev配置文件。

### mondb配置

配置数据库地址即可


    spring.data.mongodb.uri=mongodb://ip:port/database?readPreference=primaryPreferred


### redis配置


    spring.redis.database=0  
    # Redis服务器地址
    spring.redis.host=ip
    # Redis服务器连接端口
    spring.redis.port=6379  
    # Redis服务器连接密码（默认为空）
    spring.redis.password=
    # 连接池最大连接数（使用负值表示没有限制）
    spring.redis.pool.max-active=8  
    # 连接池最大阻塞等待时间（使用负值表示没有限制）
    spring.redis.pool.max-wait=-1  
    # 连接池中的最大空闲连接
    spring.redis.pool.max-idle=8  
    # 连接池中的最小空闲连接
    spring.redis.pool.min-idle=0  
    # 连接超时时间（毫秒）
    spring.redis.timeout=0  


## 数据访问

### mongdb

mongdb访问很简单，直接定义接口extends MongoRepository即可，另外可以支持JPA语法，例如：


    @Component
    public interface UserRepository extends MongoRepository<User, Integer> {
    public User findByUserName(String userName);
    }


使用时，加上@Autowired注解即可。


    @Component
    public class AuthService extends BaseService {
    @AutowiredUserRepository userRepository;}


### Redis访问

使用StringRedisTemplate即可直接访问Redis


    @Component
    public class BaseService {@Autowiredprotected MongoTemplate mongoTemplate;
    @Autowiredprotected StringRedisTemplate stringRedisTemplate;
    }


储存数据：


    .stringRedisTemplate.opsForValue().set(token_key, user.getId()+"",token_max_age, TimeUnit.SECONDS);


删除数据：


    stringRedisTemplate.delete(getFormatToken(accessToken,platform));


## Web服务

定义一个Controller类，加上RestController即可，使用RequestMapping用来设置url route


    @RestController
    public class AuthController extends BaseController {
    @RequestMapping(value = {"/"}, produces = "application/json;charset=utf-8", method = {RequestMethod.GET, RequestMethod.POST})@ResponseBodypublic String main() {	return "hello world！";}
    
    }


现在启动，应该就能看到hello world！了

# 服务鉴权

## 简易accessToken机制

提供登录接口，认证成功后，生成一个accessToken，以后访问接口时，带上accessToken，服务端通过accessToken来判断是否是合法用户。

为了方便，可以将accessToken存入redis，设定有效期。


    		String token = EncryptionUtils.sha256Hex(String.format("%s%s", user.getUserName(), System.currentTimeMillis()));	String token_key = getFormatToken(token, platform);	this.stringRedisTemplate.opsForValue().set(token_key, user.getId()+"",token_max_age, TimeUnit.SECONDS);


## 拦截器身份认证

为了方便做统一的身份认证，可以基于Spring的拦截器机制，创建一个拦截器来做统一认证。


    public class AuthCheckInterceptor implements HandlerInterceptor {
    }


要使拦截器生效，还需要一步，增加配置：


    @Configuration
    public class SessionConfiguration extends WebMvcConfigurerAdapter {
    @AutowiredAuthCheckInterceptor authCheckInterceptor;
    @Overridepublic void addInterceptors(InterceptorRegistry registry) {	super.addInterceptors(registry);	// 添加拦截器	registry.addInterceptor(authCheckInterceptor).addPathPatterns("/**");}
    }


## 自定义认证注解

为了精细化权限认证，比如有的接口只能具有特定权限的人才能访问，可以通过自定义注解轻松解决。在自定义的注解里，加上roles即可。


    /**
     *  权限检验注解
     */
    @Target(ElementType.METHOD)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    public @interface AuthCheck {
    /** *  角色列表 * @return */String[] roles() default {};
    }


检验逻辑：

- 只要接口加上了AuthCheck注解，就必须是登陆用户
- 如果指定了roles，则除了登录外，用户还应该具备相应的角色。



        String[] ignoreUrls = new String[]{
                "/user/.*",
                "/cat/.*",
                "/app/.*",
                "/error"
        };
     public boolean preHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object handler) throws Exception {
    
            // 0 检验公共参数
            if(!checkParams("platform",httpServletRequest,httpServletResponse)){
                return  false;
            }
    
            // 1、忽略验证的URL
            String url = httpServletRequest.getRequestURI().toString();
            for(String ignoreUrl :ignoreUrls){
                if(url.matches(ignoreUrl)){
                    return true;
                }
            }
    
            // 2、查询验证注解
            HandlerMethod handlerMethod = (HandlerMethod) handler;
            Method method = handlerMethod.getMethod();
            // 查询注解
            AuthCheck authCheck = method.getAnnotation(AuthCheck.class);
            if (authCheck == null) {
                // 无注解，不需要
                return true;
            }
    
            // 3、有注解，先检查accessToken
            if(!checkParams("accessToken",httpServletRequest,httpServletResponse)){
                return  false;
            }
            // 检验token是否过期
            Integer userId = authService.getUserIdFromToken(httpServletRequest.getParameter("accessToken"),
                    httpServletRequest.getParameter("platform"));
            if(userId==null){
                logger.debug("accessToken timeout");
                output(ResponseResult.Builder.error("accessToken已过期").build(),httpServletResponse);
                return false;
            }
    
            // 4、再检验是否包含必要的角色
            if(authCheck.roles()!=null&&authCheck.roles().length>0){
                User user = authService.getUser(userId);
                boolean isMatch = false;
                for(String role : authCheck.roles()){
                    if(user.getRole().getName().equals(role)){
                        isMatch =  true;
                        break;
                    }
                }
                // 角色未匹配，验证失败
                if(!isMatch){
                    return false;
                }
            }
    
            return true;
        }


# 服务响应结果封装

增加一个Builder，方便生成最终结果


    public class ResponseResult {
    
        public static class Builder{
            ResponseResult responseResult;
    
            Map<String,Object> dataMap = Maps.newHashMap();
    
            public Builder(){
                this.responseResult = new ResponseResult();
            }
    
            public Builder(String state){
                this.responseResult = new ResponseResult(state);
            }
    
    
            public static Builder newBuilder(){
               return new Builder();
            }
    
            public static Builder success(){
                return new Builder("success");
            }
    
            public static Builder error(String message){
                Builder builder =  new Builder("error");
                builder.responseResult.setError(message);
                return builder;
            }
    
            public  Builder append(String key,Object data){
                this.dataMap.put(key,data);
                return this;
            }
    
            /**
             *  设置列表数据
             * @param datas 数据
             * @return
             */
            public  Builder setListData(List<?> datas){
                this.dataMap.put("result",datas);
                this.dataMap.put("total",datas.size());
                return this;
            }
    
            public  Builder setData(Object data){
                this.dataMap.clear();
                this.responseResult.setData(data);
                return this;
            }
    
            boolean wrapData = false;
    
            /**
             * 将数据包裹在data中
             * @param wrapData
             * @return
             */
            public  Builder wrap(boolean wrapData){
                this.wrapData = wrapData;
                return this;
            }
    
            public String build(){
    
                JSONObject jsonObject = new JSONObject();
                jsonObject.put("state",this.responseResult.getState());
                if(this.responseResult.getState().equals("error")){
                    jsonObject.put("error",this.responseResult.getError());
                }
                if(this.responseResult.getData()!=null){
                    jsonObject.put("data", JSON.toJSON(this.responseResult.getData()));
                }else  if(dataMap.size()>0){
                    if(wrapData) {
                        JSONObject data = new JSONObject();
                        dataMap.forEach((key, value) -> {
                            data.put(key, value);
                        });
                        jsonObject.put("data", data);
                    }else{
                        dataMap.forEach((key, value) -> {
                            jsonObject.put(key, value);
                        });
                    }
                }
                return jsonObject.toJSONString();
            }
    
        }
    
        private String state;
        private Object data;
        private String error;
    
    
        public String getError() {
            return error;
        }
    
        public void setError(String error) {
            this.error = error;
        }
    
        public ResponseResult(){}
    
        public ResponseResult(String rc){
            this.state = rc;
        }
    
        /**
         * 成功时返回
         * @param rc
         * @param result
         */
        public ResponseResult(String rc, Object result){
            this.state = rc;
            this.data = result;
        }
    
        public String getState() {
            return state;
        }
    
        public void setState(String state) {
            this.state = state;
        }
    
        public Object getData() {
            return data;
        }
    
        public void setData(Object data) {
            this.data = data;
        }
    
    }
    


调用时可以优雅一点


    
        @RequestMapping(value = {"/user/login","/pc/user/login"}, produces = "application/json;charset=utf-8", method = {RequestMethod.GET, RequestMethod.POST})
        @ResponseBody
        public String login(String userName,String password,Integer platform) {
            User user = this.authService.login(userName,password);
            if(user!=null){
                //  登陆
                String token = authService.updateToken(user,platform);
                return ResponseResult.Builder		         .success()
                        .append("accessToken",token)
                        .append("userId",user.getId())
                        .build();
            }
            return ResponseResult.Builder.error("用户不存在或密码错误").build();
        }
        protected String error(String message){
            return  ResponseResult.Builder.error(message).build();
        }
    
        protected String success(){
            return  ResponseResult.Builder
                    .success()
                    .build();
        }
    
        protected String successDataList(List<?> data){
            return ResponseResult.Builder
                    .success()
                    .wrap(true) // data包裹
                    .setListData(data)
                    .build();
        }


* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


