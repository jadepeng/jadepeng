---
title: spring boot实现超轻量级网关（反向代理、转发）
tags: ["java","jqpeng"]
categories: ["博客","jqpeng"]
date: 2020-11-18 10:11
---
文章作者:jqpeng
原文链接: [spring boot实现超轻量级网关（反向代理、转发）](https://www.cnblogs.com/xiaoqi/p/spring-boot-route.html)

在我们的rest服务中，需要暴露一个中间件的接口给用户，但是需要经过rest服务的认证，这是典型的网关使用场景。可以引入网关组件来搞定，但是引入zuul等中间件会增加系统复杂性，这里实现一个超轻量级的网关，只实现请求转发，认证等由rest服务的spring security来搞定。

如何进行请求转发呢？ 熟悉网络请求的同学应该很清楚，请求无非就是请求方式、HTTP header，以及请求body，我们将这些信息取出来，透传给转发的url即可。

举例：

/graphdb/\*\*   转发到 Graph\_Server/\*\*

## 获取转发目的地址：


    private String createRedictUrl(HttpServletRequest request, String routeUrl, String prefix) {
            String queryString = request.getQueryString();
            return routeUrl + request.getRequestURI().replace(prefix, "") +
                    (queryString != null ? "?" + queryString : "");
        }


## 解析请求头和内容

然后从request中提取出header、body等内容，构造一个`RequestEntity`，后续可以用`RestTemplate`来请求。


    private RequestEntity createRequestEntity(HttpServletRequest request, String url) throws URISyntaxException, IOException {
            String method = request.getMethod();
            HttpMethod httpMethod = HttpMethod.resolve(method);
            MultiValueMap<String, String> headers = parseRequestHeader(request);
            byte[] body = parseRequestBody(request);
            return new RequestEntity<>(body, headers, httpMethod, new URI(url));
        }
    
    
        private byte[] parseRequestBody(HttpServletRequest request) throws IOException {
            InputStream inputStream = request.getInputStream();
            return StreamUtils.copyToByteArray(inputStream);
        }
    
        private MultiValueMap<String, String> parseRequestHeader(HttpServletRequest request) {
            HttpHeaders headers = new HttpHeaders();
            List<String> headerNames = Collections.list(request.getHeaderNames());
            for (String headerName : headerNames) {
                List<String> headerValues = Collections.list(request.getHeaders(headerName));
                for (String headerValue : headerValues) {
                    headers.add(headerName, headerValue);
                }
            }
            return headers;
        }


## 透明转发

最后用`RestTemplate`来实现请求：


     private ResponseEntity<String> route(RequestEntity requestEntity) {
            RestTemplate restTemplate = new RestTemplate();
            return restTemplate.exchange(requestEntity, String.class);
        }


## 全部代码

以下是轻量级转发全部代码：


    import org.springframework.http.*;
    import org.springframework.stereotype.Service;
    import org.springframework.util.MultiValueMap;
    import org.springframework.util.StreamUtils;
    import org.springframework.web.client.RestTemplate;
    
    import javax.servlet.http.HttpServletRequest;
    import javax.servlet.http.HttpServletResponse;
    import java.io.IOException;
    import java.io.InputStream;
    import java.net.URI;
    import java.net.URISyntaxException;
    import java.util.Collections;
    import java.util.List;
    
    @Service
    public class RoutingDelegate {
    
    
        public ResponseEntity<String> redirect(HttpServletRequest request, HttpServletResponse response,String routeUrl, String prefix) {
            try {
                // build up the redirect URL
                String redirectUrl = createRedictUrl(request,routeUrl, prefix);
                RequestEntity requestEntity = createRequestEntity(request, redirectUrl);
                return route(requestEntity);
            } catch (Exception e) {
                return new ResponseEntity("REDIRECT ERROR", HttpStatus.INTERNAL_SERVER_ERROR);
            }
        }
    
        private String createRedictUrl(HttpServletRequest request, String routeUrl, String prefix) {
            String queryString = request.getQueryString();
            return routeUrl + request.getRequestURI().replace(prefix, "") +
                    (queryString != null ? "?" + queryString : "");
        }
    
    
        private RequestEntity createRequestEntity(HttpServletRequest request, String url) throws URISyntaxException, IOException {
            String method = request.getMethod();
            HttpMethod httpMethod = HttpMethod.resolve(method);
            MultiValueMap<String, String> headers = parseRequestHeader(request);
            byte[] body = parseRequestBody(request);
            return new RequestEntity<>(body, headers, httpMethod, new URI(url));
        }
        private ResponseEntity<String> route(RequestEntity requestEntity) {
            RestTemplate restTemplate = new RestTemplate();
            return restTemplate.exchange(requestEntity, String.class);
        }
    
    
        private byte[] parseRequestBody(HttpServletRequest request) throws IOException {
            InputStream inputStream = request.getInputStream();
            return StreamUtils.copyToByteArray(inputStream);
        }
    
        private MultiValueMap<String, String> parseRequestHeader(HttpServletRequest request) {
            HttpHeaders headers = new HttpHeaders();
            List<String> headerNames = Collections.list(request.getHeaderNames());
            for (String headerName : headerNames) {
                List<String> headerValues = Collections.list(request.getHeaders(headerName));
                for (String headerValue : headerValues) {
                    headers.add(headerName, headerValue);
                }
            }
            return headers;
        }
    }


## Spring 集成

Spring Controller,RequestMapping里把GET \ POST\PUT\DELETE 支持的请求带上，就能实现转发了。


    @RestController
    @RequestMapping(GraphDBController.DELEGATE_PREFIX)
    @Api(value = "GraphDB", tags = {
            "graphdb-Api"
    })
    public class GraphDBController {
    
        @Autowired
        GraphProperties graphProperties;
    
        public final static String DELEGATE_PREFIX = "/graphdb";
    
        @Autowired
        private RoutingDelegate routingDelegate;
    
        @RequestMapping(value = "/**", method = {RequestMethod.GET, RequestMethod.POST, RequestMethod.PUT, RequestMethod.DELETE}, produces = MediaType.TEXT_PLAIN_VALUE)
        public ResponseEntity catchAll(HttpServletRequest request, HttpServletResponse response) {
            return routingDelegate.redirect(request, response, graphProperties.getGraphServer(), DELEGATE_PREFIX);
        }
    }


* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


