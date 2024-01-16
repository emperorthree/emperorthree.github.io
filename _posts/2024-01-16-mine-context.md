---
layout: post
title: 实现mine-context模块
date: 2024-01-16
category: spring
---


# mine-context

容器上下文的管理，用来维护管理所有的Bean的生命周期；

## IOC / DI

谁控制谁？IOC容器控制所以对象的生命周期

**Ioc—Inversion of Control，即“控制反转”，不是什么技术，而是一种设计思想。谁控制谁？当然是IoC 容器控制了对象；控制什么？那就是主要控制了外部资源获取（不只是对象包括比如文件等）**  在Java开发中，Ioc意味着将你设计好的对象交给容器控制，而不是传统的在你的对象内部直接控制。

**IoC对编程带来的最大改变不是从代码上，而是从思想上，发生了“主从换位”的变化。应用程序原本是老大，要获取什么资源都是主动出击，但是在IoC/DI思想中，应用程序就变成被动的了，被动的等待IoC容器来创建并注入它所需要的资源了。**

**IoC很好的体现了面向对象设计法则之一—— 好莱坞法则：“别找我们，我们找你”；即由IoC容器帮对象找相应的依赖对象并注入，而不是由对象主动去找。**

**因为由容器帮我们查找及注入依赖对象，对象只是被动的接受依赖对象，所以是反转；哪些方面反转了？依赖对象的获取被反转了。**



**IoC的一个重点是在系统运行中，动态的向某个对象提供它所需要的其他对象。这一点是通过DI（Dependency Injection，依赖注入）来实现的**。



**DI—Dependency Injection，即“依赖注入”**：**组件之间依赖关系**由容器在运行期决定，形象的说，即**由容器动态的将某个依赖关系注入到组件之中**。**依赖注入的目的并非为软件系统带来更多功能，而是为了提升组件重用的频率，并为系统搭建一个灵活、可扩展的平台。**

[https://www.cnblogs.com/xdp-gacl/p/4249939.html](https://www.cnblogs.com/xdp-gacl/p/4249939.html)



## IOC的实现

1. 资源获取：有哪些配置需要读取？yml和properties文件；
2. 扫描class文件：从启动类上面的注解（@CompentScan、@Import）开始扫描包，获取class；
3. 创建BeanDefinition：BeanDefinition是对所有的对象的一个抽象，便利上一步扫描到的class，获取合适构造方法（默认是无参），初始化方法和销毁方法以及相关的注解信息（@order、@Primary、@PostConstruct、@PreDestroy）为每个class创建BeanDefinition对象；通过@Configuration和@Bean注解的信息以工厂模式创建BeanDefinition ，@Bean注解的方法为工厂方法，@Configuration的类为工厂类，@Bean方法返回的类型为BeanDefinition的Class；
4. 先实例化强依赖的Bean：先通过对@Configuration注解的BeanDefinition先实例化；强依赖：工厂方法和构造方法注入，依赖的注入和Bean的实例化是一体的不能分开进行需要优先处理，强依赖出现循环依赖那就只能报错；
5. 再实例化BeanPostProcessor的实现类，先于被代理bean创建：用于进行bean的代理增强，在bean的实例化之后、属性设置之前做一些定制化方法的处理；
6. 后实例化剩下的BeanDef
7. 弱依赖注入和属性注入：扫描@Value和@Autowired的Setter方法注入
8. 调用初始化方法，bean准备就绪：扫描@PostConstruct注解，寻找Definition设置的初始化方法



### 创建BeanDefinition

BeanDefinition的定义

```java
public class BeanDefinition implements Comparable<BeanDefinition> {

    // unique bean name:
    private final String name;
    // bean class:声明类型 [声明类型和实际类型可能不是同一个]
    private final Class<?> beanClass;
    // bean instance:
    private Object instance = null;
    // constructor or null:
    private final Constructor<?> constructor;
    // factory name or null:
    private final String factoryName;
    // factory method or null:
    private final Method factoryMethod;
    // bean order used by ApplicationContext.getBeans(type):
    private final int order;
    // has @Primary?
    private final boolean primary;

    // init/destroy方法名称:
    String initMethodName;
    String destroyMethodName;

    // init/destroy方法:
    Method initMethod;
    Method destroyMethod;

    private boolean isConfigurationDefinition;
    
    @Override
    public int compareTo(BeanDefinition o) {
        int cmp = Integer.compare(this.order, o.order);
        if (cmp != 0) {
            return cmp;
        }
        return this.name.compareTo(o.name);
    }
}
```

对于自己定义的带`@Component`注解的Bean，我们需要获取Class类型，获取**构造方法来创建Bean**，然后收集`@PostConstruct`和`@PreDestroy`标注的初始化与销毁的方法，以及其他信息，如`@Order`定义Bean的内部排序顺序，`@Primary`定义存在多个相同类型时返回哪个“主要”Bean。一个典型的定义如下：

```java
@Component
public class Hello {
    @PostConstruct
    void init() {}

    @PreDestroy
    void destroy() {}
}
```

对于`@Configuration`定义的`@Bean`方法，我们把它看作**Bean的工厂方法**，我们需要获取方法返回值作为Class类型，方法本身作为创建Bean的`factoryMethod`，然后收集`@Bean`定义的`initMethod`和`destroyMethod`标识的初始化于销毁的方法名，以及其他`@Order`、`@Primary`等信息。一个典型的定义如下：

```java
@Configuration
public class AppConfig {
    @Bean(initMethod="init", destroyMethod="close")
    DataSource createDataSource() {
        return new HikariDataSource(...);// 实际的返回类型和方法定义的类型可能不一致，以BeanDefinition中instance.getClass()为准
    }
}
```

### Bean的实例化

对于IoC容器来说，创建Bean的过程分两步：

1. 创建Bean的实例，此时必须注入强依赖；
2. 对Bean实例进行Setter方法注入和字段注入；

核心需要处理依赖问题，Spring支持的4种依赖注入模式

**强依赖无法处理，只能报错，如构造方法注入和工厂方法注入**，Bean的创建与注入是一体的，我们无法把它们分成两个阶段，因为无法中断方法内部代码的执行

**Setter方法注入、字段注入后两种注入方式形成的依赖则是弱依赖，这种循环依赖则很容易解决，因为我们可以分两步，先分别实例化Bean，再注入依赖**；

```java
@Component
public class Hello {
    JdbcTemplate jdbcTemplate;
    // 构造方法的注入
    public Hello(@Autowired JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }
}

@Configuration
public class AppConfig {
    
    // 工厂方法的注入
    @Bean
    Hello hello(@Autowired JdbcTemplate jdbcTemplate) {
        return new Hello(jdbcTemplate);
    }
}


@Component
public class Hello {
    JdbcTemplate jdbcTemplate;
	
    // set方法的注入
    @Autowired
    void setJdbcTemplate(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }
}

@Component
public class Hello {
    // 字段的注入
    @Autowired
    JdbcTemplate jdbcTemplate;
}
```

### BeanPostProcessor的实现

定义BeanPostProcessor接口，切入bean的生命周期，通过特定注解绑定指定的InvocationHandler；在原始bean实例化之后进行InvocationHandler的处理生成代理类（可以是完全替换原始Bean或者是对原始Bean的增强）



一个Bean如果被Proxy替换，则依赖它的Bean应注入Proxy；

一个Bean如果被Proxy替换，如果要注入依赖，则应该注入到原始对象；
