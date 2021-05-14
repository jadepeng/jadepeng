---
title: Spring中的@Valid 和 @Validated注解你用对了吗
tags: ["Spring Boot","java","jqpeng"]
categories: ["博客","jqpeng"]
date: 2021-01-14 18:24
---
文章作者:jqpeng
原文链接: [Spring中的@Valid 和 @Validated注解你用对了吗](https://www.cnblogs.com/xiaoqi/p/spring-valid.html)

## 1.概述

本文我们将重点介绍Spring中 [@Valid](https://docs.oracle.com/javaee/7/api/javax/validation/Valid.html)和[@Validated](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/validation/annotation/Validated.html)注解的区别 。

验证用户输入是否正确是我们应用程序中的常见功能。Spring提供了`@Valid`和@`Validated`两个注解来实现验证功能，下面我们来详细介绍它们。

## 2. @Valid和@Validate注解

在Spring中，我们使用`@Valid` 注解进行方法级别验证，同时还能用它来标记成员属性以进行验证。

但是，此注释不支持分组验证。`@Validated`则支持分组验证。

## 3.例子

让我们考虑一个使用Spring Boot开发的简单用户注册表单。首先，我们只有`名称`和`密码`属性：


    public class UserAccount {
    
        @NotNull
        @Size(min = 4, max = 15)
        private String password;
    
        @NotBlank
        private String name;
    
        // standard constructors / setters / getters / toString
    
    }
    


接下来，让我们看一下控制器。在这里，我们将使用带有`@Valid`批注的`saveBasicInfo`方法来验证用户输入：


    @RequestMapping(value = "/saveBasicInfo", method = RequestMethod.POST)
    public String saveBasicInfo(
      @Valid @ModelAttribute("useraccount") UserAccount useraccount, 
      BindingResult result, 
      ModelMap model) {
        if (result.hasErrors()) {
            return "error";
        }
        return "success";
    }


现在让我们测试一下这个方法：


    @Test
    public void givenSaveBasicInfo_whenCorrectInput`thenSuccess() throws Exception {
        this.mockMvc.perform(MockMvcRequestBuilders.post("/saveBasicInfo")
          .accept(MediaType.TEXT_HTML)
          .param("name", "test123")
          .param("password", "pass"))
          .andExpect(view().name("success"))
          .andExpect(status().isOk())
          .andDo(print());
    }


在确认测试成功运行之后，现在让我们扩展功能。下一步的逻辑步骤是将其转换为多步骤注册表格，就像大多数向导一样。第一步，`名称`和`密码`保持不变。在第二步中，我们将获取其他信息，例如`age` 和 `phone`。因此，我们将使用以下其他字段更新域对象：


    public class UserAccount {
    
        @NotNull
        @Size(min = 4, max = 15)
        private String password;
    
        @NotBlank
        private String name;
    
        @Min(value = 18, message = "Age should not be less than 18")
        private int age;
    
        @NotBlank
        private String phone;
    
        // standard constructors / setters / getters / toString   
    
    }
    


但是，这一次，我们将注意到先前的测试失败。这是因为我们没有传递`年龄`和`电话`字段。

为了支持此行为，我们引入支持分组验证的`@Validated`批注。

`分组验证`,就是将字段分组，分别验证，比如我们将用户信息分为两组：`BasicInfo`和`AdvanceInfo`

可以建立两个空接口：


    public interface BasicInfo {
    }



    public interface AdvanceInfo {
    }


第一步将具有`BasicInfo`接口，第二步 将具有`AdvanceInfo`  。此外，我们将更新`UserAccount`类以使用这些标记接口，如下所示：


    public class UserAccount {
    
        @NotNull(groups = BasicInfo.class)
        @Size(min = 4, max = 15, groups = BasicInfo.class)
        private String password;
    
        @NotBlank(groups = BasicInfo.class)
        private String name;
    
        @Min(value = 18, message = "Age should not be less than 18", groups = AdvanceInfo.class)
        private int age;
    
        @NotBlank(groups = AdvanceInfo.class)
        private String phone;
    
        // standard constructors / setters / getters / toString   
    
    }
    


另外，我们现在将更新控制器以使用`@Validated`注释而不是`@Valid`：


    @RequestMapping(value = "/saveBasicInfoStep1", method = RequestMethod.POST)
    public String saveBasicInfoStep1(
      @Validated(BasicInfo.class) 
      @ModelAttribute("useraccount") UserAccount useraccount, 
      BindingResult result, ModelMap model) {
        if (result.hasErrors()) {
            return "error";
        }
        return "success";
    }


更新后，再次执行测试，现在可以成功运行。现在，我们还要测试这个新方法：


    @Test
    public void givenSaveBasicInfoStep1`whenCorrectInput`thenSuccess() throws Exception {
        this.mockMvc.perform(MockMvcRequestBuilders.post("/saveBasicInfoStep1")
          .accept(MediaType.TEXT_HTML)
          .param("name", "test123")
          .param("password", "pass"))
          .andExpect(view().name("success"))
          .andExpect(status().isOk())
          .andDo(print());
    }


也成功运行！

接下来，让我们看看`@Valid`对于触发嵌套属性验证是必不可少的。

## 4.使用`@Valid`批注标记嵌套对象

@Valid 可以用于嵌套对象。例如，在我们当前的场景中，让我们创建一个 `UserAddress `对象：


    public class UserAddress {
    
        @NotBlank
        private String countryCode;
    
        // standard constructors / setters / getters / toString
    }


为了确保验证此嵌套对象，我们将使用`@Valid`批注装饰属性：


    public class UserAccount {
    
        //...
    
        @Valid
        @NotNull(groups = AdvanceInfo.class)
        private UserAddress useraddress;
    
        // standard constructors / setters / getters / toString 
    }


## 5. 总结

`@Valid`保证了整个对象的验证, 但是它是对整个对象进行验证，当仅需要部分验证的时候就会出现问题。 这时候，可以使用`@Validated` 进行分组验证。

## 参考

- [https://www.baeldung.com/spring-valid-vs-validated](https://www.baeldung.com/spring-valid-vs-validated)


* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


