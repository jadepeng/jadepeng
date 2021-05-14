---
title: JAVA使用SnakeYAML解析与序列化YAML
tags: ["SnakeYAML","jqpeng"]
categories: ["博客","jqpeng"]
date: 2019-12-19 19:09
---
文章作者:jqpeng
原文链接: [JAVA使用SnakeYAML解析与序列化YAML](https://www.cnblogs.com/xiaoqi/p/SnakeYAML.html)

## **1.概述**

本文，我们将学习如何使用[SnakeYAML](https://bitbucket.org/asomov/snakeyaml/overview)库将  
**YAML文档转换为Java对象，以及JAVA对象如何序列化为YAML文档**。

## **2.项目设置**

要在项目中使用SnakeYAML，需要添加Maven依赖项（可在[此处](https://search.maven.org/classic/#search%7Cgav%7C1%7Cg%3A%22org.yaml%22%20AND%20a%3A%22snakeyaml%22)找到最新版本）：


    <dependency>
        <groupId>org.yaml</groupId>
        <artifactId>snakeyaml</artifactId>
        <version>1.25</version>
    </dependency>


## **3.入口点**

该`YAML`类是API的入口点：


    Yaml yaml = new Yaml()


由于实现不是线程安全的，因此不同的线程必须具有自己的`Yaml`实例。

## **4.加载YAML文档**

`SnakeYAML`支持从`String`或`InputStream`加载文档，我们从定义一个简单的YAML文档开始，然后将文件命名为`customer.yaml`：


    firstName: "John"
    lastName: "Doe"
    age: 20


### **4.1。基本用法**

现在，我们将使用`Yaml`类来解析上述YAML文档：


    Yaml yaml = new Yaml();
    InputStream inputStream = this.getClass()
      .getClassLoader()
      .getResourceAsStream("customer.yaml");
    Map<String, Object> obj = yaml.load(inputStream);
    System.out.println(obj);


上面的代码生成以下输出：


    {firstName=John, lastName=Doe, age=20}


默认情况下，`load（）`方法返回一个`Map`对象。查询`Map`对象时，我们需要事先知道属性键的名称，否则容易出错。更好的办法是自定义类型。

### **4.2自定义类型解析**

`SnakeYAML`**提供了一种将文档解析为自定义类型的方法**

让我们定义一个`Customer`类，然后尝试再次加载该文档：


    public class Customer {
     
        private String firstName;
        private String lastName;
        private int age;
     
        // getters and setters
    }


现在我么来加载：


    Yaml yaml = new Yaml();
    InputStream inputStream = this.getClass()
     .getClassLoader()
     .getResourceAsStream("customer.yaml");
    Customer customer = yaml.load(inputStream);


还有一种方法是使用Constructor：


    Yaml yaml = new Yaml(new Constructor(Customer.class));


### **4.3。隐式类型**

**如果没有为给定属性定义类型，则库会自动将值转换为隐式type**。

例如：


    1.0 -> Float
    42 -> Integer
    2009-03-30 -> Date


让我们使用一个TestCase来测试这种隐式类型转换：


    @Test
    public void whenLoadYAML_thenLoadCorrectImplicitTypes() {
       Yaml yaml = new Yaml();
       Map<Object, Object> document = yaml.load("3.0: 2018-07-22");
      
       assertNotNull(document);
       assertEquals(1, document.size());
       assertTrue(document.containsKey(3.0d));   
    }


### **4.4 嵌套对象**

`SnakeYAML` 支持嵌套的复杂类型。

让我们向“ `customer.yaml”`添加“ `联系方式”`  和“ `地址” `详细信息`，`并将新文件另存为`customer_with_contact_details_and_address.yaml. `。

现在，我们将分析新的YAML文档：


    firstName: "John"
    lastName: "Doe"
    age: 31
    contactDetails:
       - type: "mobile"
         number: 123456789
       - type: "landline"
         number: 456786868
    homeAddress:
       line: "Xyz, DEF Street"
       city: "City Y"
       state: "State Y"
       zip: 345657


我们来更新java类：


    public class Customer {
        private String firstName;
        private String lastName;
        private int age;
        private List<Contact> contactDetails;
        private Address homeAddress;    
        // getters and setters
    }
    
    public class Contact {
        private String type;
        private int number;
        // getters and setters
    }
    
    public class Address {
        private String line;
        private String city;
        private String state;
        private Integer zip;
        // getters and setters
    }


现在，我们来测试下`Yaml`＃`load（）`：


    @Test
    public void
      whenLoadYAMLDocumentWithTopLevelClass_thenLoadCorrectJavaObjectWithNestedObjects() {
      
        Yaml yaml = new Yaml(new Constructor(Customer.class));
        InputStream inputStream = this.getClass()
          .getClassLoader()
          .getResourceAsStream("yaml/customer_with_contact_details_and_address.yaml");
        Customer customer = yaml.load(inputStream);
      
        assertNotNull(customer);
        assertEquals("John", customer.getFirstName());
        assertEquals("Doe", customer.getLastName());
        assertEquals(31, customer.getAge());
        assertNotNull(customer.getContactDetails());
        assertEquals(2, customer.getContactDetails().size());
         
        assertEquals("mobile", customer.getContactDetails()
          .get(0)
          .getType());
        assertEquals(123456789, customer.getContactDetails()
          .get(0)
          .getNumber());
        assertEquals("landline", customer.getContactDetails()
          .get(1)
          .getType());
        assertEquals(456786868, customer.getContactDetails()
          .get(1)
          .getNumber());
        assertNotNull(customer.getHomeAddress());
        assertEquals("Xyz, DEF Street", customer.getHomeAddress()
          .getLine());
    }


### **4.5。类型安全的集合**

当给定Java类的一个或多个属性是泛型集合类时，需要通过`TypeDescription`来指定泛型类型，以以便可以正确解析。

让我们假设一个 一个`Customer`拥有多个`Contact`：


    firstName: "John"
    lastName: "Doe"
    age: 31
    contactDetails:
       - { type: "mobile", number: 123456789}
       - { type: "landline", number: 123456789}


为了能正确解析，**我们可以在顶级类上为给定属性指定`TypeDescription `**：


    Constructor constructor = new Constructor(Customer.class);
    TypeDescription customTypeDescription = new TypeDescription(Customer.class);
    customTypeDescription.addPropertyParameters("contactDetails", Contact.class);
    constructor.addTypeDescription(customTypeDescription);
    Yaml yaml = new Yaml(constructor);


### **4.6。载入多个文件**

在某些情况下，单个`文件中`可能有多个YAML文档，而我们想解析所有文档。所述`YAML`类提供了一个`LOADALL（）`方法来完成这种类型的解析。

假设下面的内容在一个文件中：


    ---
    firstName: "John"
    lastName: "Doe"
    age: 20
    ---
    firstName: "Jack"
    lastName: "Jones"
    age: 25


我们可以使用`loadAll（）`方法解析以上内容，如以下代码示例所示：


    @Test
    public void whenLoadMultipleYAMLDocuments_thenLoadCorrectJavaObjects() {
        Yaml yaml = new Yaml(new Constructor(Customer.class));
        InputStream inputStream = this.getClass()
          .getClassLoader()
          .getResourceAsStream("yaml/customers.yaml");
     
        int count = 0;
        for (Object object : yaml.loadAll(inputStream)) {
            count++;
            assertTrue(object instanceof Customer);
        }
        assertEquals(2,count);
    }


## **5.生成YAML文件**

`SnakeYAML` 支持 将java对象序列化为yml。

### **5.1。基本用法**

我们将从一个将`Map <String，Object>`的实例转储到YAML文档（`String`）的简单示例开始：


    @Test
    public void whenDumpMap_thenGenerateCorrectYAML() {
        Map<String, Object> data = new LinkedHashMap<String, Object>();
        data.put("name", "Silenthand Olleander");
        data.put("race", "Human");
        data.put("traits", new String[] { "ONE_HAND", "ONE_EYE" });
        Yaml yaml = new Yaml();
        StringWriter writer = new StringWriter();
        yaml.dump(data, writer);
        String expectedYaml = "name: Silenthand Olleander\nrace: Human\ntraits: [ONE_HAND, ONE_EYE]\n";
     
        assertEquals(expectedYaml, writer.toString());
    }


上面的代码产生以下输出（请注意，使用`LinkedHashMap`的实例将保留输出数据的顺序）：


    name: Silenthand Olleander
    race: Human
    traits: [ONE_HAND, ONE_EYE]


### **5.2。自定义Java对象**

我们还可以选择**将自定义Java类型转储到输出流中**。


    @Test
    public void whenDumpACustomType_thenGenerateCorrectYAML() {
        Customer customer = new Customer();
        customer.setAge(45);
        customer.setFirstName("Greg");
        customer.setLastName("McDowell");
        Yaml yaml = new Yaml();
        StringWriter writer = new StringWriter();
        yaml.dump(customer, writer);        
        String expectedYaml = "!!com.baeldung.snakeyaml.Customer {age: 45, contactDetails: null, firstName: Greg,\n  homeAddress: null, lastName: McDowell}\n";
     
        assertEquals(expectedYaml, writer.toString());
    }


生成内容会包含!!com.baeldung.snakeyaml.Customer，为了避免在输出文件中使用标签名，我们可以使用库提供的  `dumpAs（）`方法。

因此，在上面的代码中，我们可以进行以下调整以删除标记：


    yaml.dumpAs(customer, Tag.MAP, null);


## **六 结语**

本文说明了SnakeYAML库解析和序列化YAML文档。

所有示例都可以在[GitHub项目中](https://github.com/eugenp/tutorials/tree/master/libraries-data-io "Java 8-Lambda表达式比较示例")找到。

## 附录

- 英文原文： [Parsing YAML with SnakeYAML](https://www.baeldung.com/java-snake-yaml)


* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


