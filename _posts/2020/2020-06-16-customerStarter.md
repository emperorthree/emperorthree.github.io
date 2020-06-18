---
layout: post
title: 自定义springboot starter
category: it
tags: [it]
excerpt: 通过自定义starter学习其原理
---

# 自定义springboot starter中发现的问题

下面以空白功能starter为例

## 引入必要的依赖

```xml

<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
  </dependency>
  <!-- 用于引入starter后，编写配置文件有提示-->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
  </dependency>
  <!-- 本质上lombok也可以不用-->
  <dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
  </dependency>
</dependencies>

```

## @SpringBootApplication详解

```Java
/*
* Indicates a {@link Configuration configuration} class that declares one or more
* {@link Bean @Bean} methods and also triggers {@link EnableAutoConfiguration
* auto-configuration} and {@link ComponentScan component scanning}. This is a convenience
* annotation that is equivalent to declaring {@code @Configuration},
* {@code @EnableAutoConfiguration} and {@code @ComponentScan}.
*/
```

翻一下：表示可以声明一个或多个bean的方法、出发EnableAutoConfiguration自动配置和包扫描的配置类。@Configuration+@EnableAutoConfiguration+@ComponentScan的简便版注解。

**ps:坑，**如果@SpringBootApplication和@ComponentScan注解共存，那么@SpringBootApplication注解的扫描的作用将会失效，也就是说不能够扫描启动类所在包以及子包了。因此，我们必须在@ComponentScan注解配置本工程需要扫描的包范围**。

## @AutoConfigurationPackage作用：注册注解类所在包下的bean

## @EnableAutoConfiguration应用

开启自动配置，配合以下注解在注入bean时功能更强大

| @ConditionalOnClass             | classpath中存在该类时起效    |
| ------------------------------- | ---------------------------- |
| @ConditionalOnMissingClass      | classpath中不存在该类时起效  |
| @ConditionalOnBean              | 容器中存在该类型Bean时起效   |
| @ConditionalOnMissingBean       | 容器中不存在该类型Bean时起效 |
| @ConditionalOnExpression        | SpEL表达式结果为true时       |
| @ConditionalOnProperty          | 参数设置或者值一致时起效     |
| @ConditionalOnResource          | 指定的文件存在时起效         |
| @ConditionalOnJava              | 指定的Java版本存在时起效     |
| @ConditionalOnWebApplication    | Web应用环境下起效            |
| @ConditionalOnNotWebApplication | 非Web应用环境下起效          |

## 为了解决引入starter包名不同于主项目包名，直接在入口注解处添加@ComponentScan去扫描starter包下的类，直接可以不使用spring.factories文件内添加配置加载类？starter中spring.factories的实际意义？

答：

1. 理论上是可行的；
2. spring.factories文件中添加自定义autoconfiguration全路径类名，在主项目中引入starter时springboot启动会先检查jar包中是否有META-INF/spring.factories文件，会先加载文件内配置的autoconfiguration类作为starter的入口。

* **@EnableSuchedulingh+@Configuration需要通过注解扫描交个spring管理，定时任务才会生效，new创建再注入则无效**
* **通过spring.factories进到autoconfig的加载bean优先级高于通过注解扫描的方式**



## Unable to read meta-data for class 自定义starter时出现的报错

原因：META-INF/spring.factories文件中有空格，格式问题

http://www.hyhblog.cn/2018/08/29/spring-boot-unable-to-read-meta-data-for-class/

```factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.zxf.starter.autoconfiguration.DemoAutoConfiguration
```

## 方式一：直接在autoconfigutation中new出需要的对象

```Java
@Configuration
@EnableConfigurationProperties({DemoProperties.class})
public class DemoAutoConfiguration {

    private DemoProperties demoProperties;

    public DemoAutoConfiguration(DemoProperties demoProperties){
        this.demoProperties = demoProperties;
    }

    @Bean
    @ConditionalOnMissingBean
    DemoService demoService(){
        return new DemoService(demoProperties);
    }
}
```

**即使这些类在不同的包下主项目也不再添加扫描指定starter包，应为直接new出来交给spring的;starter包下的类之间通过属性+构造函数来组合**



## 方式二：使用全注解的方式加入bean

```Java
@Configuration
@ComponentScan("com.zxf")
@EnableConfigurationProperties({DemoProperties.class})
public class DemoAutoConfiguration {

}
```

**autoConfiguration类中加入componentScan注解去指定扫描starter类下的包，则需要被spring管理的类添加注解；**



参考文章：

1. https://blog.csdn.net/zxc123e/article/details/80222967