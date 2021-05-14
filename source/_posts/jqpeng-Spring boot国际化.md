---
title: Spring boot国际化
tags: ["Spring Boot","jqpeng"]
categories: ["博客","jqpeng"]
date: 2018-08-19 13:27
---
文章作者:jqpeng
原文链接: [Spring boot国际化](https://www.cnblogs.com/xiaoqi/p/spring-boot-i18n.html)

国际化主要是引入了MessageSource，我们简单看下如何使用，以及其原理。

## 1.1 设置资源文件

在 properties新建i18n目录

新建message文件：

messages.properties


    error.title=Your request cannot be processed


messages\_zh\_CN.properties


    error.title=您的请求无法处理


## 1.2 配置

修改properties文件的目录：在application.yml或者application.properties中配置 spring.message.basename


    spring:
        application:
            name: test-worklog
        messages:
            basename: i18n/messages
            encoding: UTF-8


## 1.3 使用

引用自动注解的MessageSource,调用`messageSource.getMessage`即可，注意，需要通过` LocaleContextHolder.getLocale()`获取当前的地区。


    @Autowired
    private MessageSource messageSource;
    /**
     * 国际化
     *
     * @param result
     * @return
     */
    public String getMessage(String result, Object[] params) {
        String message = "";
        try {
            Locale locale = LocaleContextHolder.getLocale();
            message = messageSource.getMessage(result, params, locale);
        } catch (Exception e) {
            LOGGER.error("parse message error! ", e);
        }
        return message;
    }


如何设置个性化的地区呢? `forLanguageTag` 即可


     Locale locale = Locale.forLanguageTag(user.getLangKey());


## 1.4 原理分析

`MessageSourceAutoConfiguration`中，实现了autoconfig


    @Configuration
    @ConditionalOnMissingBean(value = MessageSource.class, search = SearchStrategy.CURRENT)
    @AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
    @Conditional(ResourceBundleCondition.class)
    @EnableConfigurationProperties
    @ConfigurationProperties(prefix = "spring.messages")
    public class MessageSourceAutoConfiguration {


该类一方面读取配置文件，一方面创建了MessageSource的实例:


    @Beanpublic MessageSource messageSource() {	ResourceBundleMessageSource messageSource = new ResourceBundleMessageSource();	if (StringUtils.hasText(this.basename)) {		messageSource.setBasenames(StringUtils.commaDelimitedListToStringArray(				StringUtils.trimAllWhitespace(this.basename)));	}	if (this.encoding != null) {		messageSource.setDefaultEncoding(this.encoding.name());	}	messageSource.setFallbackToSystemLocale(this.fallbackToSystemLocale);	messageSource.setCacheSeconds(this.cacheSeconds);	messageSource.setAlwaysUseMessageFormat(this.alwaysUseMessageFormat);	return messageSource;}


因此，默认是加载的`ResourceBundleMessageSource`，该类派生与于AbstractResourceBasedMessageSource

![enter description here](http://oyqmmpkcm.bkt.clouddn.com/1532590285975.jpg "1532590285975")


    @Overridepublic final String getMessage(String code, Object[] args, String defaultMessage, Locale locale) {	String msg = getMessageInternal(code, args, locale);	if (msg != null) {		return msg;	}	if (defaultMessage == null) {		String fallback = getDefaultMessage(code);		if (fallback != null) {			return fallback;		}	}	return renderDefaultMessage(defaultMessage, args, locale);}


最终是调用resolveCode来获取message，通过ResourceBundle来获取message


    	@Overrideprotected MessageFormat resolveCode(String code, Locale locale) {    // 遍历语言文件路径	Set<String> basenames = getBasenameSet();	for (String basename : basenames) {		ResourceBundle bundle = getResourceBundle(basename, locale);		if (bundle != null) {			MessageFormat messageFormat = getMessageFormat(bundle, code, locale);			if (messageFormat != null) {				return messageFormat;			}		}	}	return null;}
    // 获取ResourceBundle	
    protected ResourceBundle getResourceBundle(String basename, Locale locale) {	if (getCacheMillis() >= 0) {		// Fresh ResourceBundle.getBundle call in order to let ResourceBundle		// do its native caching, at the expense of more extensive lookup steps.		return doGetBundle(basename, locale);	}	else {		// Cache forever: prefer locale cache over repeated getBundle calls.		synchronized (this.cachedResourceBundles) {			Map<Locale, ResourceBundle> localeMap = this.cachedResourceBundles.get(basename);			if (localeMap != null) {				ResourceBundle bundle = localeMap.get(locale);				if (bundle != null) {					return bundle;				}			}			try {				ResourceBundle bundle = doGetBundle(basename, locale);				if (localeMap == null) {					localeMap = new HashMap<Locale, ResourceBundle>();					this.cachedResourceBundles.put(basename, localeMap);				}				localeMap.put(locale, bundle);				return bundle;			}			catch (MissingResourceException ex) {				if (logger.isWarnEnabled()) {					logger.warn("ResourceBundle [" + basename + "] not found for MessageSource: " + ex.getMessage());				}				// Assume bundle not found				// -> do NOT throw the exception to allow for checking parent message source.				return null;			}		}	}}
    
    //  ResourceBundle	
    protected ResourceBundle doGetBundle(String basename, Locale locale) throws MissingResourceException {	return ResourceBundle.getBundle(basename, locale, getBundleClassLoader(), new MessageSourceControl());
    }


最后来看getMessageFormat：


    /** * Return a MessageFormat for the given bundle and code, * fetching already generated MessageFormats from the cache. * @param bundle the ResourceBundle to work on * @param code the message code to retrieve * @param locale the Locale to use to build the MessageFormat * @return the resulting MessageFormat, or {@code null} if no message * defined for the given code * @throws MissingResourceException if thrown by the ResourceBundle */protected MessageFormat getMessageFormat(ResourceBundle bundle, String code, Locale locale)		throws MissingResourceException {
    	synchronized (this.cachedBundleMessageFormats) {	    // 从缓存读取		Map<String, Map<Locale, MessageFormat>> codeMap = this.cachedBundleMessageFormats.get(bundle);		Map<Locale, MessageFormat> localeMap = null;		if (codeMap != null) {			localeMap = codeMap.get(code);			if (localeMap != null) {				MessageFormat result = localeMap.get(locale);				if (result != null) {					return result;				}			}		}        // 缓存miss，从bundle读取		String msg = getStringOrNull(bundle, code);		if (msg != null) {			if (codeMap == null) {				codeMap = new HashMap<String, Map<Locale, MessageFormat>>();				this.cachedBundleMessageFormats.put(bundle, codeMap);			}			if (localeMap == null) {				localeMap = new HashMap<Locale, MessageFormat>();				codeMap.put(code, localeMap);			}			MessageFormat result = createMessageFormat(msg, locale);			localeMap.put(locale, result);			return result;		}
    		return null;	}}


* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


