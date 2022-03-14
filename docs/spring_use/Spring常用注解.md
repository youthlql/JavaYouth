---
title: Spring常用注解
tags:
  - Spring
  - 注解
categories:
  - Spring
  - 用法
keywords: Spriong，框架
description: 一些常用的注解
cover: 'https://gitee.com/youthlql/randombg/raw/master/logo/spring.png'
abbrlink: 1a003b7b
date: 2021-10-06 00:21:58
---





> 参考了尚硅谷注解版，注解版后面的源码没看，雷神讲的太散了

## Spring注解

### AnnotationConfigApplicationContext

### 组件添加

#### @Configuration+@Bean

##### XML文件方式

###### Person

```java
public class Person { 
   
   private String name;
    
   private Integer age;
   
   private String nickName;
   
   public String getNickName() {
      return nickName;
   }
   public void setNickName(String nickName) {
      this.nickName = nickName;
   }
   public String getName() {
      return name;
   }
   public void setName(String name) {
      this.name = name;
   }
   public Integer getAge() {
      return age;
   }
   public void setAge(Integer age) {
      this.age = age;
   }
   
   public Person(String name, Integer age) {
      super();
      this.name = name;
      this.age = age;
   }
   public Person() {
      super();
      // TODO Auto-generated constructor stub
   }
   @Override
   public String toString() {
      return "Person [name=" + name + ", age=" + age + ", nickName=" + nickName + "]";
   }
}
```

###### beans.xml-配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xmlns:context="http://www.springframework.org/schema/context"
   xmlns:aop="http://www.springframework.org/schema/aop"
   xmlns:tx="http://www.springframework.org/schema/tx"
   xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
      http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.3.xsd
      http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-4.3.xsd
      http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-4.3.xsd">

   <context:component-scan base-package="com.atguigu" use-default-filters="false"></context:component-scan>
   <bean id="person" class="com.atguigu.bean.Person">
      <property name="age" value="18"></property>
      <property name="name" value="zhangsan"></property>
   </bean>
</beans>
```

###### MainTest

```java
public class MainTest {
   
   @SuppressWarnings("resource")
	public static void main(String[] args) {
		ApplicationContext applicationContext = new ClassPathXmlApplicationContext("beans.xml");
		Person bean = (Person) applicationContext.getBean("person");
		System.out.println(bean);
	}
}
```



输出

```
Person [name=zhangsan, age=18, nickName=null]
```

##### 注解方式

```java
//配置类==配置文件
@Configuration  //告诉Spring这是一个配置类
public class MainConfig {
   
   //给容器中注册一个Bean;类型为返回值的类型，id默认是用方法名作为id(就是bean的名字)，在这里就是person01
   @Bean
   public Person person01(){
      return new Person("lisi", 20);
   }

}
```

或者以下面的这种方式

```java
//配置类==配置文件
@Configuration  //告诉Spring这是一个配置类
public class MainConfig {
   
   //这里bean的name就是person
    @Bean("person")
	public Person person01(){
		return new Person("lisi", 20);
	}

}
```



#### @ComponentScans

```java
//配置类==配置文件
@Configuration  //告诉Spring这是一个配置类

@ComponentScans(
      value = {
            @ComponentScan(value="com.atguigu",includeFilters = {
/*                @Filter(type=FilterType.ANNOTATION,classes={Controller.class}),
                  @Filter(type=FilterType.ASSIGNABLE_TYPE,classes={BookService.class}),*/
                  @Filter(type=FilterType.CUSTOM,classes={MyTypeFilter.class})
            },useDefaultFilters = false)   
      }
      )
//@ComponentScan  value:指定要扫描的包
//excludeFilters = Filter[] ：指定扫描的时候按照什么规则排除那些组件
//includeFilters = Filter[] ：指定扫描的时候只需要包含哪些组件
//FilterType.ANNOTATION：按照注解
//FilterType.ASSIGNABLE_TYPE：按照给定的类型；
//FilterType.ASPECTJ：使用ASPECTJ表达式
//FilterType.REGEX：使用正则指定
//FilterType.CUSTOM：使用自定义规则
public class MainConfig {
   
   //给容器中注册一个Bean;类型为返回值的类型，id默认是用方法名作为id
   @Bean("person")
   public Person person01(){
      return new Person("lisi", 20);
   }

}
```



##### 自定义TypeFilter指定包扫描规则

```java
public class MyTypeFilter implements TypeFilter {

   /**
    * metadataReader：读取到的当前正在扫描的类的信息
    * metadataReaderFactory:可以获取到其他任何类信息的
    */
   @Override
   public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory)
         throws IOException {
      // TODO Auto-generated method stub
      //获取当前类注解的信息
      AnnotationMetadata annotationMetadata = metadataReader.getAnnotationMetadata();
      //获取当前正在扫描的类的类信息
      ClassMetadata classMetadata = metadataReader.getClassMetadata();
      //获取当前类资源（类的路径）
      Resource resource = metadataReader.getResource();
      
      String className = classMetadata.getClassName();
      System.out.println("--->"+className);
      if(className.contains("er")){
         return true;
      }
      return false;
   }

}
```



#### @Scope

```java
@Configuration
public class MainConfig2 {
   
	/**
	 * @see ConfigurableBeanFactory#SCOPE_PROTOTYPE   任何环境都可以使用
	 * @see ConfigurableBeanFactory#SCOPE_SINGLETON   任何环境都可以使用
	 * @see org.springframework.web.context.WebApplicationContext#SCOPE_REQUEST  request    只能在web容器里用
	 * @see org.springframework.web.context.WebApplicationContext#SCOPE_SESSION	 sesssion   只能在web容器里用
	 * 
	 * @Scope:调整作用域
	 * prototype：多实例的：ioc容器启动并不会去调用方法创建对象放在容器中。
	 * 					每次获取的时候才会调用方法创建对象；
	 * singleton：单实例的（默认值）：ioc容器启动会调用方法创建对象放到ioc容器中。
	 * 			以后每次获取就是直接从容器（map.get()）中拿，
	 * request：同一次请求创建一个实例
	 * session：同一个session创建一个实例
	 *
	 * 默认是单实例的
	 * 
	 */
	@Scope("prototype")
	@Lazy
	@Bean("person")
	public Person person(){
		System.out.println("给容器中添加Person....");
		return new Person("张三", 25);
	}

}
```



#### @Lazy

```java
@Configuration
public class MainConfig2 {
   
   /**
    * 
    * 懒加载：
    *        单实例bean：默认在容器启动的时候创建对象；
    *        懒加载：容器启动不创建对象。第一次使用(获取)Bean创建对象，并初始化；
    * 
    */
   @Lazy
   @Bean("person")
   public Person person(){
      System.out.println("给容器中添加Person....");
      return new Person("张三", 25);
   }
```



#### @Conditional

##### MainConfig2

```java
//类中组件统一设置。满足当前条件，这个类中配置的所有bean注册才能生效；
@Conditional({WindowsCondition.class})
@Configuration
public class MainConfig2 {
   
   
   /**
    * @Conditional({Condition}) ： 按照一定的条件进行判断，满足条件给容器中注册bean
    * 
    * 如果系统是windows，给容器中注册("bill")
    * 如果是linux系统，给容器中注册("linus")
    */
   
   @Bean("bill")
   public Person person01(){
      return new Person("Bill Gates",62);
   }
   
   @Conditional(LinuxCondition.class)
   @Bean("linus")
   public Person person02(){
      return new Person("linus", 48);
   }
}
```

##### LinuxCondition

```java
//判断是否linux系统
public class LinuxCondition implements Condition {

   /**
    * ConditionContext：判断条件能使用的上下文（环境）
    * AnnotatedTypeMetadata：注释信息
    */
   @Override
   public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
      // TODO是否linux系统
      //1、能获取到ioc使用的beanfactory
      ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
      //2、获取类加载器
      ClassLoader classLoader = context.getClassLoader();
      //3、获取当前环境信息
      Environment environment = context.getEnvironment();
      //4、获取到bean定义的注册类
      BeanDefinitionRegistry registry = context.getRegistry();
      
      String property = environment.getProperty("os.name");
      
      //可以判断容器中的bean注册情况，也可以给容器中注册bean
      boolean definition = registry.containsBeanDefinition("person");
      if(property.contains("linux")){
         return true;
      }
      
      return false;
   }
```

##### WindowsCondition

```java
//判断是否windows系统
public class WindowsCondition implements Condition {

   @Override
   public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
      Environment environment = context.getEnvironment();
      String property = environment.getProperty("os.name");
      if(property.contains("Windows")){
         return true;
      }
      return false;
   }

}
```



#### @Import

##### MainConfig2

```java
@Configuration
@Import({Color.class,Red.class,MyImportSelector.class,MyImportBeanDefinitionRegistrar.class})
//@Import导入组件，id默认是组件的全类名
public class MainConfig2 {
   
   /**
    * 给容器中注册组件；
    * 1）、包扫描+组件标注注解（@Controller/@Service/@Repository/@Component）[只能注册自己写的类]
    * 2）、@Bean[导入的第三方包里面的组件]
    * 3）、@Import[快速给容器中导入一个组件]
    *        1）、@Import(要导入到容器中的组件)；容器中就会自动注册这个组件，id默认是全类名
    *        2）、ImportSelector:返回需要导入的组件的全类名数组；
    *        3）、ImportBeanDefinitionRegistrar:手动注册bean到容器中
    */
   @Bean
   public ColorFactoryBean colorFactoryBean(){
      return new ColorFactoryBean();
   }
}
```

##### MyImportSelector

```java
//自定义逻辑返回需要导入的组件
public class MyImportSelector implements ImportSelector {

   //返回值，就是到导入到容器中的组件全类名
   //AnnotationMetadata:@Import引入MyImportSelector的类的所有注解信息
   @Override
   public String[] selectImports(AnnotationMetadata importingClassMetadata) {
      //importingClassMetadata.get
      //方法不要返回null值，不然会报错
      return new String[]{"com.atguigu.bean.Blue","com.atguigu.bean.Yellow"};
   }

}
```

##### MyImportBeanDefinitionRegistrar

```java
public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {

   /**
    * AnnotationMetadata：当前类的注解信息
    * BeanDefinitionRegistry:BeanDefinition注册类；
    *        把所有需要添加到容器中的bean；调用
    *        BeanDefinitionRegistry.registerBeanDefinition手工注册进来
    */
   @Override
   public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
      
      boolean definition = registry.containsBeanDefinition("com.atguigu.bean.Red");
      boolean definition2 = registry.containsBeanDefinition("com.atguigu.bean.Blue");
      if(definition && definition2){
         //指定Bean定义信息；（Bean的类型，Bean。。。）
         RootBeanDefinition beanDefinition = new RootBeanDefinition(RainBow.class);
         //注册一个Bean，指定bean名
         registry.registerBeanDefinition("rainBow", beanDefinition);
      }
   }

}
```



```java
public class Color {
   
   private Car car;

   public Car getCar() {
      return car;
   }

   public void setCar(Car car) {
      this.car = car;
   }

   @Override
   public String toString() {
      return "Color [car=" + car + "]";
   }
   
}
```



```java
public class Blue {
   
   public Blue(){
      System.out.println("blue...constructor");
   }
   
   public void init(){
      System.out.println("blue...init...");
   }
   
   public void detory(){
      System.out.println("blue...detory...");
   }
   
}
```



#### FactoryBean

##### MainConfig2

```java
@Configuration
public class MainConfig2 {
  
   
   /**
    * 给容器中注册组件；
    * 1）、包扫描+组件标注注解（@Controller/@Service/@Repository/@Component）[自己写的类]
    * 2）、@Bean[导入的第三方包里面的组件]
    * 3）、@Import[快速给容器中导入一个组件]
    *        1）、@Import(要导入到容器中的组件)；容器中就会自动注册这个组件，id默认是全类名
    *        2）、ImportSelector:返回需要导入的组件的全类名数组；
    *        3）、ImportBeanDefinitionRegistrar:手动注册bean到容器中
    * 4）、使用Spring提供的 FactoryBean（工厂Bean）;
    *        1）、默认获取到的是工厂bean调用getObject创建的对象
    *        2）、要获取工厂Bean本身，我们需要给id前面加一个&
    *           &colorFactoryBean
    *
    * 虽然这里装配的是ColorFactoryBean，但实际上beand的类型是Color
    */
   @Bean
   public ColorFactoryBean colorFactoryBean(){
      return new ColorFactoryBean();
   }

}
```

##### ColorFactoryBean

```java
//创建一个Spring定义的FactoryBean
public class ColorFactoryBean implements FactoryBean<Color> {

   //返回一个Color对象，这个对象会添加到容器中
   @Override
   public Color getObject() throws Exception {
      // TODO Auto-generated method stub
      System.out.println("ColorFactoryBean...getObject...");
      return new Color();
   }

   @Override
   public Class<?> getObjectType() {
      // TODO Auto-generated method stub
      return Color.class;
   }

   //是单例？
   //true：这个bean是单实例，在容器中保存一份
   //false：多实例，每次获取都会创建一个新的bean；
   @Override
   public boolean isSingleton() {
      // TODO Auto-generated method stub
      return false;
   }

}
```

##### IOCTest

```java
public class IOCTest {
   AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(MainConfig2.class);
   
   
   @Test
   public void testImport(){
      printBeans(applicationContext);
      Blue bean = applicationContext.getBean(Blue.class);
      System.out.println(bean);
      
      //工厂Bean获取的是调用getObject创建的对象
      Object bean2 = applicationContext.getBean("colorFactoryBean");   
      System.out.println("bean的类型："+bean2.getClass()); //pos_1   输出：bean的类型：class com.atguigu.bean.Color
      
      Object bean4 = applicationContext.getBean("&colorFactoryBean");
      System.out.println(bean4.getClass()); //pos_2  输出：class com.atguigu.bean.ColorFactoryBean
   }
   
   private void printBeans(AnnotationConfigApplicationContext applicationContext){
      String[] definitionNames = applicationContext.getBeanDefinitionNames();
      for (String name : definitionNames) {
         System.out.println(name);
      }
   }
}
```

输出:

```java
//前面无关的输出省略

colorFactoryBean
ColorFactoryBean...getObject...
bean的类型：class com.atguigu.bean.Color
class com.atguigu.bean.ColorFactoryBean
```



### 生命周期

#### @Bean指定初始化和销毁方法

##### IOCTest_LifeCycle

> 后面的几个用的都是这个测试类

```java
public class IOCTest_LifeCycle {
   
   @Test
   public void test01(){
      //1、创建ioc容器
      AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(MainConfigOfLifeCycle.class);
      System.out.println("容器创建完成...");
      
      //applicationContext.getBean("car");
      //关闭容器
      applicationContext.close();
   }

}
```

##### MainConfigOfLifeCycle

```java
package com.atguigu.config;

import org.springframework.context.ApplicationListener;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Scope;

import com.atguigu.bean.Car;

/**
 * bean的生命周期：
 *        bean创建---初始化----销毁的过程
 * 容器管理bean的生命周期；
 * 我们可以自定义初始化和销毁方法；容器在bean进行到当前生命周期的时候来调用我们自定义的初始化和销毁方法
 * 
 * 构造（对象创建）
 *        单实例：在容器启动的时候创建对象
 *        多实例：在每次获取的时候创建对象\
 * 初始化：
 * 		对象创建完成，并赋值好，调用初始化方法。。。
 * BeanPostProcessor.postProcessAfterInitialization
 * 销毁：
 * 		单实例：容器关闭的时候
 * 		多实例：容器不会管理这个bean；容器不会调用销毁方法；
 * 
 * 1）、指定初始化和销毁方法；
 *        通过@Bean指定init-method和destroy-method；
 * @author lfy
 *
 */
@ComponentScan("com.atguigu.bean")
@Configuration
public class MainConfigOfLifeCycle {
   
   //@Scope("prototype")
   @Bean(initMethod="init",destroyMethod="detory")
   public Car car(){
      return new Car();
   }

}
```



```java
@Component
public class Car {
   
   public Car(){
      System.out.println("car constructor...");
   }
   
   public void init(){
      System.out.println("car ... init...");
   }
   
   public void detory(){
      System.out.println("car ... detory...");
   }

}
```

##### 输出

```java
car constructor...
car ... init...
容器创建完成
car ... detory...
```



#### InitializingBean和DisposableBean

##### MainConfigOfLifeCycle

```java
/**
 * bean的生命周期：
 *        bean创建---初始化----销毁的过程
 * 容器管理bean的生命周期；
 * 我们可以自定义初始化和销毁方法；容器在bean进行到当前生命周期的时候来调用我们自定义的初始化和销毁方法
 * 
 * 构造（对象创建）
 *        单实例：在容器启动的时候创建对象
 *        多实例：在每次获取的时候创建对象\
 * 
 * BeanPostProcessor.postProcessBeforeInitialization
 * 初始化：
 *        对象创建完成，并赋值好，调用初始化方法。。。
 * BeanPostProcessor.postProcessAfterInitialization
 * 销毁：
 *        单实例：容器关闭的时候
 *        多实例：容器不会管理这个bean；容器不会调用销毁方法；
 * 
 * 
 * 遍历得到容器中所有的BeanPostProcessor；挨个执行beforeInitialization，
 * 一但返回null，跳出for循环，不会执行后面的BeanPostProcessor.postProcessorsBeforeInitialization
 * 
 * BeanPostProcessor原理
 * populateBean(beanName, mbd, instanceWrapper);给bean进行属性赋值
 * initializeBean
 * {
 * applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
 * invokeInitMethods(beanName, wrappedBean, mbd);执行自定义初始化
 * applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
 *}
 * 
 * 
 * 
 * 1）、指定初始化和销毁方法；
 *        通过@Bean指定init-method和destroy-method；
 * 2）、通过让Bean实现InitializingBean（定义初始化逻辑），
 *              DisposableBean（定义销毁逻辑）;
 * 
 * @author lfy
 *
 */
@ComponentScan("com.atguigu.bean")
@Configuration
public class MainConfigOfLifeCycle {
   
   //@Scope("prototype")
   @Bean(initMethod="init",destroyMethod="detory")
   public Car car(){
      return new Car();
   }

}
```

##### Cat

```java
@Component
public class Cat implements InitializingBean,DisposableBean {
   
   public Cat(){
      System.out.println("cat constructor...");
   }

   @Override
   public void destroy() throws Exception {
      // TODO Auto-generated method stub
      System.out.println("cat...destroy...");
   }

   @Override
   public void afterPropertiesSet() throws Exception {
      // TODO Auto-generated method stub
      System.out.println("cat...afterPropertiesSet...");
   }

}
```

##### 输出

```java
cat constructor...
cat...afterPropertiesSet...
car constructor...
car ... init...
容器创建完成
car ... detory...
cat...destroy...
```



#### @PostConstruct和@PreDestroy

##### MainConfigOfLifeCycle

```java
/**
 * bean的生命周期：
 *        bean创建---初始化----销毁的过程
 * 容器管理bean的生命周期；
 * 我们可以自定义初始化和销毁方法；容器在bean进行到当前生命周期的时候来调用我们自定义的初始化和销毁方法
 * 
 * 构造（对象创建）
 *        单实例：在容器启动的时候创建对象
 *        多实例：在每次获取的时候创建对象\
 * 
 * BeanPostProcessor.postProcessBeforeInitialization
 * 初始化：
 *        对象创建完成，并赋值好，调用初始化方法。。。
 * BeanPostProcessor.postProcessAfterInitialization
 * 销毁：
 *        单实例：容器关闭的时候
 *        多实例：容器不会管理这个bean；容器不会调用销毁方法；
 * 
 * 
 * 遍历得到容器中所有的BeanPostProcessor；挨个执行beforeInitialization，
 * 一但返回null，跳出for循环，不会执行后面的BeanPostProcessor.postProcessorsBeforeInitialization
 * 
 * BeanPostProcessor原理
 * populateBean(beanName, mbd, instanceWrapper);给bean进行属性赋值
 * initializeBean
 * {
 * applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
 * invokeInitMethods(beanName, wrappedBean, mbd);执行自定义初始化
 * applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
 *}
 * 
 * 
 * 
 * 1）、指定初始化和销毁方法；
 *        通过@Bean指定init-method和destroy-method；
 * 2）、通过让Bean实现InitializingBean（定义初始化逻辑），
 *              DisposableBean（定义销毁逻辑）;
 * 3）、可以使用JSR250；
 *        @PostConstruct：在bean创建完成并且属性赋值完成；来执行初始化方法
 *        @PreDestroy：在容器销毁bean之前通知我们进行清理工作
 * 
 * @author lfy
 *
 */
@ComponentScan("com.atguigu.bean")
@Configuration
public class MainConfigOfLifeCycle {
   
   //@Scope("prototype")
   @Bean(initMethod="init",destroyMethod="detory")
   public Car car(){
      return new Car();
   }

}
```

##### Dog

```java
@Component
public class Dog implements ApplicationContextAware {
   
   //@Autowired
   private ApplicationContext applicationContext;
   
   public Dog(){
      System.out.println("dog constructor...");
   }
   
   //对象创建并赋值之后调用
   @PostConstruct
   public void init(){
      System.out.println("Dog....@PostConstruct...");
   }
   
   //容器移除对象之前
   @PreDestroy
   public void detory(){
      System.out.println("Dog....@PreDestroy...");
   }

   @Override
   public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
      // TODO Auto-generated method stub
      this.applicationContext = applicationContext;
   }
}
```



##### 输出

```java
cat constructor...
cat...afterPropertiesSet...
dog constructor...
Dog....@PostConstruct...
car constructor...
car ... init...
容器创建完成
car ... detory...
Dog....@PreDestroy...
cat...destroy...
```



#### BeanPostProcessor

```java

/**
 * bean的生命周期：
 *        bean创建---初始化----销毁的过程
 * 容器管理bean的生命周期；
 * 我们可以自定义初始化和销毁方法；容器在bean进行到当前生命周期的时候来调用我们自定义的初始化和销毁方法
 * 
 * 构造（对象创建）
 *        单实例：在容器启动的时候创建对象
 *        多实例：在每次获取的时候创建对象\
 * 
 * BeanPostProcessor.postProcessBeforeInitialization
 * 初始化：
 *        对象创建完成，并赋值好，调用初始化方法。。。
 * BeanPostProcessor.postProcessAfterInitialization
 * 销毁：
 *        单实例：容器关闭的时候
 *        多实例：容器不会管理这个bean；容器不会调用销毁方法；
 * 
 * 
 * 遍历得到容器中所有的BeanPostProcessor；挨个执行beforeInitialization，
 * 一但返回null，跳出for循环，不会执行后面的BeanPostProcessor.postProcessorsBeforeInitialization
 * 
 * BeanPostProcessor原理
 * populateBean(beanName, mbd, instanceWrapper);给bean进行属性赋值
 * initializeBean
 * {
 * applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
 * invokeInitMethods(beanName, wrappedBean, mbd);执行自定义初始化
 * applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
 *}
 * 
 * 
 * 
 * 1）、指定初始化和销毁方法；
 *        通过@Bean指定init-method和destroy-method；
 * 2）、通过让Bean实现InitializingBean（定义初始化逻辑），
 *              DisposableBean（定义销毁逻辑）;
 * 3）、可以使用JSR250；
 *        @PostConstruct：在bean创建完成并且属性赋值完成；来执行初始化方法
 *        @PreDestroy：在容器销毁bean之前通知我们进行清理工作
 * 4）、BeanPostProcessor【interface】：bean的后置处理器；
 *        在bean初始化前后进行一些处理工作；
 *        postProcessBeforeInitialization:在初始化之前工作
 *        postProcessAfterInitialization:在初始化之后工作
 * 
 * Spring底层对 BeanPostProcessor 的使用；
 *        bean赋值，注入其他组件，@Autowired，生命周期注解功能，@Async,xxx BeanPostProcessor;
 * 
 * @author lfy
 *
 */
@ComponentScan("com.atguigu.bean")
@Configuration
public class MainConfigOfLifeCycle {
   
   //@Scope("prototype")
   @Bean(initMethod="init",destroyMethod="detory")
   public Car car(){
      return new Car();
   }

}
```

##### MyBeanPostProcessor

```java
/**
 * 后置处理器：初始化前后进行处理工作
 * 将后置处理器加入到容器中
 * @author lfy
 */
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {

   @Override
   public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
      // TODO Auto-generated method stub
      System.out.println("postProcessBeforeInitialization..."+beanName+"=>"+bean);
      return bean;
   }

   @Override
   public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
      // TODO Auto-generated method stub
      System.out.println("postProcessAfterInitialization..."+beanName+"=>"+bean);
      return bean;
   }

}
```

输出

> 自己写的组件输出内容

```java
car constructor...
postProcessBeforeInitialization...car=>com.atguigu.bean.Car@5ef60048
car ... init...
postProcessAfterInitialization...car=>com.atguigu.bean.Car@5ef60048
cat constructor...
postProcessBeforeInitialization...cat=>com.atguigu.bean.Cat@780cb77
cat...afterPropertiesSet...
postProcessAfterInitialization...cat=>com.atguigu.bean.Cat@780cb77
dog constructor...
postProcessBeforeInitialization...dog=>com.atguigu.bean.Dog@4034c28c
Dog....@PostConstruct...
postProcessAfterInitialization...dog=>com.atguigu.bean.Dog@4034c28c
容器创建完成...
Dog....@PreDestroy...
cat...destroy...
car ... detory...
```



> 1. BeanPostProcessor在Spring源码里大量被使用到，仅凭这里雷丰阳老师讲的一点点原理，是无法体会的，建议自己去看看Spring源码。所以这里的原理部分我也就直接省略了，在本视频中讲的太浅了。

### 属性赋值

#### @Value和@PropertySource

##### Person

```java
public class Person {
   
   //使用@Value赋值；
   //1、基本数值
   //2、可以写SpEL； #{}
   //3、可以写${}；取出配置文件【properties】中的值（在运行环境变量里面的值）
   
   @Value("张三")
   private String name;
   @Value("#{20-2}")
   private Integer age;
   
   @Value("${person.nickName}")
   private String nickName;
   
   
   
   public String getNickName() {
      return nickName;
   }
   public void setNickName(String nickName) {
      this.nickName = nickName;
   }
   public String getName() {
      return name;
   }
   public void setName(String name) {
      this.name = name;
   }
   public Integer getAge() {
      return age;
   }
   public void setAge(Integer age) {
      this.age = age;
   }
   
   public Person(String name, Integer age) {
      super();
      this.name = name;
      this.age = age;
   }
   public Person() {
      super();
      // TODO Auto-generated constructor stub
   }
   @Override
   public String toString() {
      return "Person [name=" + name + ", age=" + age + ", nickName=" + nickName + "]";
   }
}
```

##### person.properties

```java
person.nickName=\u5C0F\u674E\u56DB
```



##### MainConfigOfPropertyValues

```java
//使用@PropertySource读取外部配置文件中的k/v保存到运行的环境变量中;加载完外部的配置文件以后使用${}取出配置文件的值
@PropertySource(value={"classpath:/person.properties"})
@Configuration
public class MainConfigOfPropertyValues {
   
   @Bean
   public Person person(){
      return new Person();
   }

}
```

##### IOCTest_PropertyValue

```java
public class IOCTest_PropertyValue {
   AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(MainConfigOfPropertyValues.class);
   @Test
   public void test01(){
      printBeans(applicationContext);
      System.out.println("=============");
      
      Person person = (Person) applicationContext.getBean("person");
      System.out.println(person);
      
      
      ConfigurableEnvironment environment = applicationContext.getEnvironment();
      String property = environment.getProperty("person.nickName");
      System.out.println(property);
      applicationContext.close();
   }
   
   private void printBeans(AnnotationConfigApplicationContext applicationContext){
      String[] definitionNames = applicationContext.getBeanDefinitionNames();
      for (String name : definitionNames) {
         System.out.println(name);
      }
   }

}
```

##### 输出

```java
mainConfigOfPropertyValues
person
=============
Person [name=张三, age=18, nickName=小李四]
小李四
```



### 自动装配

#### @Autowired-@Qualifier-@Primary-@Resource-@Inject

```java
@Controller
public class BookController {
   
   @Autowired
   private BookService bookService;

}
```



```java
@Service
public class BookService {
   
   //@Qualifier("bookDao")
   //@Autowired(required=false)
   //@Resource(name="bookDao2")
   @Inject
   private BookDao bookDao;
   
   public void print(){
      System.out.println(bookDao);
   }

   @Override
   public String toString() {
      return "BookService [bookDao=" + bookDao + "]";
   }
   

}
```



```java
//名字默认是类名首字母小写
@Repository
public class BookDao {
   
   private String lable = "1";

   public String getLable() {
      return lable;
   }

   public void setLable(String lable) {
      this.lable = lable;
   }

   @Override
   public String toString() {
      return "BookDao [lable=" + lable + "]";
   }
   
   

}
```



##### MainConifgOfAutowired

```java
/**
 * 自动装配;
 *        Spring利用依赖注入（DI），完成对IOC容器中中各个组件的依赖关系赋值；
 * 
 * 1）、@Autowired：自动注入：
 *        1）、默认优先按照类型去容器中找对应的组件:applicationContext.getBean(BookDao.class);找到就赋值
 *        2）、如果找到多个相同类型的组件，再将属性的名称作为组件的id去容器中查找
 *                       applicationContext.getBean("bookDao")
 *        3）、@Qualifier("bookDao")：使用@Qualifier指定需要装配的组件的id，而不是使用属性名
 *        4）、自动装配默认一定要将属性赋值好，没有就会报错；
 *           可以使用@Autowired(required=false);
 *        5）、@Primary：让Spring进行自动装配的时候，默认使用首选的bean；
 *              也可以继续使用@Qualifier指定需要装配的bean的名字
 *        BookService{
 *           @Autowired
 *           BookDao  bookDao;
 *        }
 * 
 * 2）、Spring还支持使用@Resource(JSR250)和@Inject(JSR330)[java规范的注解]
 *        @Resource:
 *           可以和@Autowired一样实现自动装配功能；默认是按照组件名称进行装配的；
 *           没有能支持@Primary功能没有支持@Autowired（reqiured=false）;
 *        @Inject:
 *           需要导入javax.inject的包，和Autowired的功能一样。没有required=false的功能；
 *  @Autowired:Spring定义的； @Resource、@Inject都是java规范
 *     
 * AutowiredAnnotationBeanPostProcessor:解析完成自动装配功能；       
 * 
 * 3）、 @Autowired:构造器，参数，方法，属性；都是从容器中获取参数组件的值
 *        1）、[标注在方法位置]：@Bean+方法参数；参数从容器中获取;默认不写@Autowired效果是一样的；都能自动装配
 *        2）、[标在构造器上]：如果组件只有一个有参构造器，这个有参构造器的@Autowired可以省略，参数位置的组件还是可以自动从容器中获取
 *        3）、放在参数位置：
 			public Boss(@Autowired Car car){
				this.car = car;
				System.out.println("Boss...有参构造器");
			}
 * 
 * 4）、自定义组件想要使用Spring容器底层的一些组件（ApplicationContext，BeanFactory，xxx）；
 *        自定义组件实现xxxAware；在创建对象的时候，会调用接口规定的方法注入相关组件；Aware；
 *        把Spring底层一些组件注入到自定义的Bean中；
 *        xxxAware：功能使用xxxProcessor；
 *           ApplicationContextAware==》ApplicationContextAwareProcessor；
 *     
 *        
 * @author lfy
 *
 */
@Configuration
@ComponentScan({"com.atguigu.service","com.atguigu.dao",
   "com.atguigu.controller","com.atguigu.bean"})
public class MainConifgOfAutowired {
   
   @Primary
   @Bean("bookDao2")
   public BookDao bookDao(){
      BookDao bookDao = new BookDao();
      bookDao.setLable("2");
      return bookDao;
   }
   
   /**
    * @Bean标注的方法创建对象的时候，方法参数的值默认从容器中获取
    * @param car
    * @return
    */
   @Bean
   public Color color(Car car){
      Color color = new Color();
      color.setCar(car);
      return color;
   }
   

}
```



##### IOCTest_Autowired

```java
public class IOCTest_Autowired {
   
   @Test
   public void test01(){
      AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(MainConifgOfAutowired.class);
      
      BookService bookService = applicationContext.getBean(BookService.class);
      System.out.println(bookService);
      
      //BookDao bean = applicationContext.getBean(BookDao.class);
      //System.out.println(bean);
      
      Boss boss = applicationContext.getBean(Boss.class);
      System.out.println(boss);
      Car car = applicationContext.getBean(Car.class);
      System.out.println(car);
      
      Color color = applicationContext.getBean(Color.class);
      System.out.println(color);
      System.out.println(applicationContext);
      applicationContext.close();
   }

}
```



#### @Profle

##### MainConfigOfProfile

```java
/**
 * Profile：
 *        Spring为我们提供的可以根据当前环境，动态的激活和切换一系列组件的功能；
 * 
 * 开发环境、测试环境、生产环境；
 * 数据源：(/A)(/B)(/C)；
 * 
 * 
 * @Profile：指定组件在哪个环境的情况下才能被注册到容器中，不指定，任何环境下都能注册这个组件
 * 
 * 1）、加了环境标识的bean，只有这个环境被激活的时候才能注册到容器中。默认是default环境
 * 2）、写在配置类上，只有是指定的环境的时候，整个配置类里面的所有配置才能开始生效
 * 3）、没有标注环境标识的bean在，任何环境下都是加载的；
 */

@PropertySource("classpath:/dbconfig.properties")
@Configuration
public class MainConfigOfProfile implements EmbeddedValueResolverAware{
   
   @Value("${db.user}")
   private String user;
   
   private StringValueResolver valueResolver;
   
   private String  driverClass;
   
   
   @Bean
   public Yellow yellow(){
      return new Yellow();
   }
   
   @Profile("test")
   @Bean("testDataSource")
   public DataSource dataSourceTest(@Value("${db.password}")String pwd) throws Exception{
      ComboPooledDataSource dataSource = new ComboPooledDataSource();
      dataSource.setUser(user);
      dataSource.setPassword(pwd);
      dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/test");
      dataSource.setDriverClass(driverClass);
      return dataSource;
   }
   
   
   @Profile("dev")
   @Bean("devDataSource")
   public DataSource dataSourceDev(@Value("${db.password}")String pwd) throws Exception{
      ComboPooledDataSource dataSource = new ComboPooledDataSource();
      dataSource.setUser(user);
      dataSource.setPassword(pwd);
      dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/ssm_crud");
      dataSource.setDriverClass(driverClass);
      return dataSource;
   }
   
   @Profile("prod")
   @Bean("prodDataSource")
   public DataSource dataSourceProd(@Value("${db.password}")String pwd) throws Exception{
      ComboPooledDataSource dataSource = new ComboPooledDataSource();
      dataSource.setUser(user);
      dataSource.setPassword(pwd);
      dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/scw_0515");
      
      dataSource.setDriverClass(driverClass);
      return dataSource;
   }

   @Override
   public void setEmbeddedValueResolver(StringValueResolver resolver) {
      // TODO Auto-generated method stub
      this.valueResolver = resolver;
      driverClass = valueResolver.resolveStringValue("${db.driverClass}");
   }

}
```

##### dbconfig.properties

```properties
db.user=root
db.password=123456
db.driverClass=com.mysql.jdbc.Driver
```











