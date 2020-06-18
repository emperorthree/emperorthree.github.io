---
layout: post
title: 读取配置文件问题
category: it
tags: [it]
excerpt: @value、@ConfigurationProperties
---
# 读取配置文件问题

## @value读取properties文件属性

* value注解可以：

  * 注入普通字符串
  * 注入操作系统属性
  * 注入表达式结果
  * 注入其他Bean属性：注入beanInject对象的属性another
  * 注入文件资源
  * 注入URL资源
  * **@values("${name:zxf}")设置默认值**

```java
@Value("classpath:com/hry/spring/configinject/config.txt")
private Resource resourceFile; // 注入文件资源

@Value("#{systemProperties['os.name']}")
private String systemPropertiesName; // 注入操作系统属性

@Value("normal")
private String normal; // 注入普通字符串

@Value("${spider.schedule.cron}")
private String cron;  //读取配置文件中的属性properties
```

## @ConfigurationProperties

**@ConfigurationProperties注解的作用是把yml或者properties配置文件转化为bean。@EnableConfigurationProperties注解的作用是使@ConfigurationProperties注解生效。如果只配置@ConfigurationProperties注解，在spring容器中是获取不到yml或者properties配置文件转化的bean的。**

配置文件如下：

``` spider.schedule.cron= 0/3 * * * * ?``` 

```java
@ConfigurationProperties("spider.schedule")
@EnableScheduling
public class ScheduleTask implements SchedulingConfigurer {
    private String cron;
```

* ## 表达式中$和#的区别

  "${}"常用来读取配置文件中的属性；"#{}"常用来获取某个bean中的属性，或调用bean的某个方法

* ## 实践

  实现对多个环境使用不同配置文件的方法

  resource/

  application.properties

  ``` spring.profiles.active=test``` 

  application-**prod**.properties

