---
title: 基于spring security 实现前后端分离项目权限控制
tags: ["Spring Boot","java","Spring Security","jqpeng"]
categories: ["博客","jqpeng"]
date: 2018-06-14 09:04
---
文章作者:jqpeng
原文链接: [基于spring security 实现前后端分离项目权限控制](https://www.cnblogs.com/xiaoqi/p/spring-security-usage.html)

前后端分离的项目，前端有菜单（menu），后端有API（backendApi），一个menu对应的页面有N个API接口来支持，本文介绍如何基于spring security实现前后端的同步权限控制。

## 实现思路

还是基于Role来实现，具体的思路是，一个Role拥有多个Menu，一个menu有多个backendApi，其中Role和menu，以及menu和backendApi都是ManyToMany关系。

验证授权也很简单，用户登陆系统时，获取Role关联的Menu，页面访问后端API时，再验证下用户是否有访问API的权限。

### domain定义

我们用JPA来实现，先来定义Role


    public class Role implements Serializable {
    
    
        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;
    
        /**
         * 名称
         */
        @NotNull
        @ApiModelProperty(value = "名称", required = true)
        @Column(name = "name", nullable = false)
        private String name;
    
        /**
         * 备注
         */
        @ApiModelProperty(value = "备注")
        @Column(name = "remark")
        private String remark;
    
        @JsonIgnore
        @ManyToMany
        @JoinTable(
            name = "role_menus",
            joinColumns = {@JoinColumn(name = "role_id", referencedColumnName = "id")},
            inverseJoinColumns = {@JoinColumn(name = "menu_id", referencedColumnName = "id")})
        @Cache(usage = CacheConcurrencyStrategy.NONSTRICT_READ_WRITE)
        @BatchSize(size = 100)
        private Set<Menu> menus = new HashSet<>();}


以及Menu：


    public class Menu implements Serializable {
    
    
        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;
    
        @Column(name = "parent_id")
        private Integer parentId;
    
        /**
         * 文本
         */
        @ApiModelProperty(value = "文本")
        @Column(name = "text")
        private String text;@ApiModelProperty(value = "angular路由")
        @Column(name = "link")
        private String link;
        @ManyToMany
        @JsonIgnore
        @JoinTable(name = "backend_api_menus",
            joinColumns = @JoinColumn(name="menus_id", referencedColumnName="id"),
            inverseJoinColumns = @JoinColumn(name="backend_apis_id", referencedColumnName="id"))
        @Cache(usage = CacheConcurrencyStrategy.NONSTRICT_READ_WRITE)
        private Set<BackendApi> backendApis = new HashSet<>();
    
        @ManyToMany(mappedBy = "menus")
        @JsonIgnore
        private Set<Role> roles = new HashSet<>();}


最后是BackendApi，区分method（HTTP请求方法）、tag（哪一个Controller）和path（API请求路径）：


    public class BackendApi implements Serializable {
    
        private static final long serialVersionUID = 1L;
    
        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;
    
        @Column(name = "tag")
        private String tag;
    
        @Column(name = "path")
        private String path;
    
        @Column(name = "method")
        private String method;
    
        @Column(name = "summary")
        private String summary;
    
        @Column(name = "operation_id")
        private String operationId;
    
        @ManyToMany(mappedBy = "backendApis")
        @Cache(usage = CacheConcurrencyStrategy.NONSTRICT_READ_WRITE)
        private Set<Menu> menus = new HashSet<>();}


## 管理页面实现

Menu菜单是业务需求确定的，因此提供CRUD编辑即可。  
 BackendAPI，可以通过swagger来获取。  
 前端选择ng-algin，参见[Angular 中后台前端解决方案 - Ng Alain 介绍](https://www.cnblogs.com/xiaoqi/p/angular-ng-alain.html)

### 通过swagger获取BackendAPI

获取swagger api有多种方法，最简单的就是访问http接口获取json，然后解析，这很简单，这里不赘述，还有一种就是直接调用相关API获取Swagger对象。

查看官方的web代码，可以看到获取数据大概是这样的：


            String groupName = Optional.fromNullable(swaggerGroup).or(Docket.DEFAULT_GROUP_NAME);
            Documentation documentation = documentationCache.documentationByGroup(groupName);
            if (documentation == null) {
                return new ResponseEntity<Json>(HttpStatus.NOT_FOUND);
            }
            Swagger swagger = mapper.mapDocumentation(documentation);
            UriComponents uriComponents = componentsFrom(servletRequest, swagger.getBasePath());
            swagger.basePath(Strings.isNullOrEmpty(uriComponents.getPath()) ? "/" : uriComponents.getPath());
            if (isNullOrEmpty(swagger.getHost())) {
                swagger.host(hostName(uriComponents));
            }
            return new ResponseEntity<Json>(jsonSerializer.toJson(swagger), HttpStatus.OK);


其中的documentationCache、environment、mapper等可以直接Autowired获得：


    @Autowired
        public SwaggerResource(
            Environment environment,
            DocumentationCache documentationCache,
            ServiceModelToSwagger2Mapper mapper,
            BackendApiRepository backendApiRepository,
            JsonSerializer jsonSerializer) {
    
            this.hostNameOverride = environment.getProperty("springfox.documentation.swagger.v2.host", "DEFAULT");
            this.documentationCache = documentationCache;
            this.mapper = mapper;
            this.jsonSerializer = jsonSerializer;
    
            this.backendApiRepository = backendApiRepository;
    
        }


然后我们自动加载就简单了，写一个updateApi接口，读取swagger对象，然后解析成BackendAPI，存储到数据库：


    @RequestMapping(
            value = "/api/updateApi",
            method = RequestMethod.GET,
            produces = { APPLICATION_JSON_VALUE, HAL_MEDIA_TYPE })
        @PropertySourcedMapping(
            value = "${springfox.documentation.swagger.v2.path}",
            propertyKey = "springfox.documentation.swagger.v2.path")
        @ResponseBody
        public ResponseEntity<Json> updateApi(
            @RequestParam(value = "group", required = false) String swaggerGroup) {
    
            // 加载已有的api
            Map<String,Boolean> apiMap = Maps.newHashMap();
            List<BackendApi> apis = backendApiRepository.findAll();
            apis.stream().forEach(api->apiMap.put(api.getPath()+api.getMethod(),true));
    
            // 获取swagger
            String groupName = Optional.fromNullable(swaggerGroup).or(Docket.DEFAULT_GROUP_NAME);
            Documentation documentation = documentationCache.documentationByGroup(groupName);
            if (documentation == null) {
                return new ResponseEntity<Json>(HttpStatus.NOT_FOUND);
            }
            Swagger swagger = mapper.mapDocumentation(documentation);
    
            // 加载到数据库
            for(Map.Entry<String, Path> item : swagger.getPaths().entrySet()){
                String path = item.getKey();
                Path pathInfo = item.getValue();
                createApiIfNeeded(apiMap, path,  pathInfo.getGet(), HttpMethod.GET.name());
                createApiIfNeeded(apiMap, path,  pathInfo.getPost(), HttpMethod.POST.name());
                createApiIfNeeded(apiMap, path,  pathInfo.getDelete(), HttpMethod.DELETE.name());
                createApiIfNeeded(apiMap, path,  pathInfo.getPut(), HttpMethod.PUT.name());
            }
            return new ResponseEntity<Json>(HttpStatus.OK);
        }


其中createApiIfNeeded，先判断下是否存在，不存在的则新增：


     private void createApiIfNeeded(Map<String, Boolean> apiMap, String path, Operation operation, String method) {
            if(operation==null) {
                return;
            }
            if(!apiMap.containsKey(path+ method)){
                apiMap.put(path+ method,true);
    
                BackendApi api = new BackendApi();
                api.setMethod( method);
                api.setOperationId(operation.getOperationId());
                api.setPath(path);
                api.setTag(operation.getTags().get(0));
                api.setSummary(operation.getSummary());
    
                // 保存
                this.backendApiRepository.save(api);
            }
        }


最后，做一个简单页面展示即可：

![enter description here](https://gitee.com/jadepeng/blogpic/raw/master/pic/2018/1528886480360.jpg "1528886480360")

### 菜单管理

新增和修改页面，可以选择上级菜单，后台API做成按tag分组，可多选即可：

![enter description here](https://gitee.com/jadepeng/blogpic/raw/master/pic/2018/1528886647893.jpg "1528886647893")

列表页面

![enter description here](https://gitee.com/jadepeng/blogpic/raw/master/pic/2018/1528886586348.jpg "1528886586348")

### 角色管理

普通的CRUD，最主要的增加一个菜单授权页面，菜单按层级显示即可：

![enter description here](https://gitee.com/jadepeng/blogpic/raw/master/pic/2018/1528886692091.jpg "1528886692091")

## 认证实现

管理页面可以做成千奇百样，最核心的还是如何实现认证。

在上一篇文章[spring security实现动态配置url权限的两种方法](https://www.cnblogs.com/xiaoqi/p/spring-security-rabc.html)里我们说了，可以自定义`FilterInvocationSecurityMetadataSource`来实现。

实现`FilterInvocationSecurityMetadataSource`接口即可，核心是根据FilterInvocation的Request的method和path，获取对应的Role，然后交给RoleVoter去判断是否有权限。

### 自定义FilterInvocationSecurityMetadataSource

我们新建一个DaoSecurityMetadataSource实现FilterInvocationSecurityMetadataSource接口，主要看getAttributes方法：


         @Override
        public Collection<ConfigAttribute> getAttributes(Object object) throws IllegalArgumentException {
            FilterInvocation fi = (FilterInvocation) object;
    
            List<Role> neededRoles = this.getRequestNeededRoles(fi.getRequest().getMethod(), fi.getRequestUrl());
    
            if (neededRoles != null) {
                return SecurityConfig.createList(neededRoles.stream().map(role -> role.getName()).collect(Collectors.toList()).toArray(new String[]{}));
            }
    
            //  返回默认配置
            return superMetadataSource.getAttributes(object);
        }


核心是getRequestNeededRoles怎么实现，获取到干净的RequestUrl（去掉参数）,然后看是否有对应的backendAPI，如果没有，则有可能该API有path参数，我们可以去掉最后的path，去库里模糊匹配，直到找到。


     public List<Role> getRequestNeededRoles(String method, String path) {
            String rawPath = path;
            //  remove parameters
            if(path.indexOf("?")>-1){
                path = path.substring(0,path.indexOf("?"));
            }
            // /menus/{id}
            BackendApi api = backendApiRepository.findByPathAndMethod(path, method);
            if (api == null){
                // try fetch by remove last path
                api = loadFromSimilarApi(method, path, rawPath);
            }
    
            if (api != null && api.getMenus().size() > 0) {
                return api.getMenus()
                    .stream()
                    .flatMap(menu -> menuRepository.findOneWithRolesById(menu.getId()).getRoles().stream())
                    .collect(Collectors.toList());
            }
            return null;
        }
    
        private BackendApi loadFromSimilarApi(String method, String path, String rawPath) {
            if(path.lastIndexOf("/")>-1){
                path = path.substring(0,path.lastIndexOf("/"));
                List<BackendApi> apis = backendApiRepository.findByPathStartsWithAndMethod(path, method);
    
                // 如果为空，再去掉一层path
                while(apis==null){
                    if(path.lastIndexOf("/")>-1) {
                        path = path.substring(0, path.lastIndexOf("/"));
                        apis = backendApiRepository.findByPathStartsWithAndMethod(path, method);
                    }else{
                        break;
                    }
                }
    
                if(apis!=null){
                    for(BackendApi backendApi : apis){
                        if (antPathMatcher.match(backendApi.getPath(), rawPath)) {
                            return backendApi;
                        }
                    }
                }
            }
            return null;
        }


其中，BackendApiRepository：


        @EntityGraph(attributePaths = "menus")
        BackendApi findByPathAndMethod(String path,String method);
    
        @EntityGraph(attributePaths = "menus")
        List<BackendApi> findByPathStartsWithAndMethod(String path,String method);


以及MenuRepository


        @EntityGraph(attributePaths = "roles")
        Menu findOneWithRolesById(long id);


### 使用DaoSecurityMetadataSource

需要注意的是，在DaoSecurityMetadataSource里，不能直接注入Repository，我们可以给DaoSecurityMetadataSource添加一个方法，方便传入：


       public void init(MenuRepository menuRepository, BackendApiRepository backendApiRepository) {
            this.menuRepository = menuRepository;
            this.backendApiRepository = backendApiRepository;
        }


然后建立一个容器，存储实例化的DaoSecurityMetadataSource，我们可以建立如下的ApplicationContext来作为对象容器，存取对象：


    public class ApplicationContext {
        static Map<Class<?>,Object> beanMap = Maps.newConcurrentMap();
    
        public static <T> T getBean(Class<T> requireType){
            return (T) beanMap.get(requireType);
        }
    
        public static void registerBean(Object item){
            beanMap.put(item.getClass(),item);
        }
    }
    


在SecurityConfiguration配置中使用`DaoSecurityMetadataSource`，并通过` ApplicationContext.registerBean`将`DaoSecurityMetadataSource`注册：


     @Override
        protected void configure(HttpSecurity http) throws Exception {
            http
                .addFilterBefore(corsFilter, UsernamePasswordAuthenticationFilter.class)
                .exceptionHandling()
                .authenticationEntryPoint(problemSupport)
                .accessDeniedHandler(problemSupport)		....
               // .withObjectPostProcessor()
                // 自定义accessDecisionManager
                .accessDecisionManager(accessDecisionManager())
                // 自定义FilterInvocationSecurityMetadataSource
                .withObjectPostProcessor(new ObjectPostProcessor<FilterSecurityInterceptor>() {
                    @Override
                    public <O extends FilterSecurityInterceptor> O postProcess(
                        O fsi) {
                        fsi.setSecurityMetadataSource(daoSecurityMetadataSource(fsi.getSecurityMetadataSource()));
                        return fsi;
                    }
                })
            .and()
                .apply(securityConfigurerAdapter());
    
        }
    
        @Bean
        public DaoSecurityMetadataSource daoSecurityMetadataSource(FilterInvocationSecurityMetadataSource filterInvocationSecurityMetadataSource) {
            DaoSecurityMetadataSource securityMetadataSource = new DaoSecurityMetadataSource(filterInvocationSecurityMetadataSource);
            ApplicationContext.registerBean(securityMetadataSource);
            return securityMetadataSource;
        }


最后，在程序启动后，通过`ApplicationContext.getBean`获取到daoSecurityMetadataSource，然后调用init注入Repository


     public static void postInit(){
            ApplicationContext
                .getBean(DaoSecurityMetadataSource.class)
     .init(applicationContext.getBean(MenuRepository.class),applicationContext.getBean(BackendApiRepository.class));
        }
    
        static ConfigurableApplicationContext applicationContext;
    
        public static void main(String[] args) throws UnknownHostException {
            SpringApplication app = new SpringApplication(UserCenterApp.class);
            DefaultProfileUtil.addDefaultProfile(app);
            applicationContext = app.run(args);
    
            // 后初始化
            postInit();
    }


大功告成！

## 延伸阅读

- [spring security实现动态配置url权限的两种方法](https://www.cnblogs.com/xiaoqi/p/spring-security-rabc.html)
- [Spring Security 架构与源码分析](https://www.cnblogs.com/xiaoqi/p/spring-security.html)


* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


