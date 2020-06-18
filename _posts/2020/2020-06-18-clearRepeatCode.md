---
layout: post
title: 抽离重复代码和if分支的两种方法
category: it
tags: [it]
excerpt: 总算是用上了设计模式。。。
---

## 抽离重复代码和if分支的两种方法

* 背景：

  不同会员等级使用不同的折扣计算方法；

  选择对接不同第三方实现同一功能；

## 方法一：工厂模式+模板方法

* 使用方法：

  创建一个抽象类，在抽象类中创建公共方法将公共代码抽离出来；对于需要特定业务逻辑的定义成抽象方法由继承类自己去实现；

  这个抽象类就是一个模板，所有继承了抽象类都需要实现抽象类中的方法；这样的好处还有一点是对于调用这个抽象类的消费者来说不需要去考虑到底是哪个类被调用；符合单一责任原则；

  * 工厂模式的调用

    这里的工厂模式主要是运用在通过spring的上下文得到具体继承类；我们通过注解的方式将继承类加载到spring的上下文中（注解需要设置value，以区分），**在使用的时候通过上下文getBean；**

  * 组合模式+map的调用

    将继承类都注入到调用者中，调用者中创建全局map使用@PostConstruct修饰的初始化方法将继承类放进map

* 代码示例

  ```Java
  // 公共抽象类
  public abstract class AbstractCart {
    	// 抽离的重复代码
      public Cart process(long userId, Map<Long, Integer> items){
  				...
          itemList.forEach(item -> {
              processCouponPrice(userId,item);
              processDeliveryPrice(userId,item);
          });
  				...
          return cart;
      }
    	// 具有不同逻辑的由继承类自己实现
      protected abstract void processCouponPrice(long userId, Item item);
      protected abstract void processDeliveryPrice(long userId, Item item);
  
  }
  ```

  ```Java
  // 继承抽象类实现普通会员的逻辑
  @Service(value = "normalUserCart")
  public class NormalUserCart extends AbstractCart {
      @Override
      protected void processCouponPrice(long userId, Item item) {
          item.setCouponPrice(BigDecimal.ZERO);
      }
      @Override
      protected void processDeliveryPrice(long userId, Item item) {
          item.setDeliveryPrice(item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())).multiply(new BigDecimal("0.1")));
      }
  }
  ```

  ```Java
  @Controller
  public class CartController {
      @Resource
      VipUserCart vipUserCart;
      @Resource
      NormalUserCart normalUserCart;
  
      private static Map<String, AbstractCart> cartMap = new ConcurrentHashMap<>();
  		// 初始化map
      @PostConstruct
      public void initMap(){
          cartMap.put("vipUserCart", vipUserCart);
          cartMap.put("normalUserCart", normalUserCart);
      }
  		// 组合模式+map的调用
      @ResponseBody
      @GetMapping("/cart/{type}")
      public Cart showCart(@PathVariable(name = "type") Integer type) {
          return cartMap.get(Db.getType(type)).process(1L, items);
      }
    	// 工厂模式的调用
      @ResponseBody
      @GetMapping("/getcart/{type}")
      public Cart cart(@PathVariable(name = "type") Integer type){
          AbstractCart cart = (AbstractCart) ApplicationContextUtils.getBean(Db.getType(type));
          return cart.process(1L, items);
      }
  }
  ```

  ## 方法二：自定义注解+反射

  * 使用方法：

    业务要求不同请求参数类型、长度使用不同的处理逻辑时，利用字段注解的方式将请求参数的公共属性抽出来（将这个注解理解成字段上的一个对象）；在处理参数时通过反射获取注解上的值；就可以将处理请求参数的逻辑抽离出来；且之后再有新的接口直接定义一个新的api请求参数对象传入公共方法完成调用；

  * 示例代码

  

  ```Java
  @Retention(RetentionPolicy.RUNTIME)
  @Target(ElementType.FIELD)// 表示作用于字段、属性
  @Documented
  @Inherited
  public @interface APIField {
      int order() default -1;
      int length() default -1;
      String type() default "";
  }
  ```

  ```java
  @CustomerApi(url = "/bank/pay", desc = "支付接口")
  @Data
  public class PayAPI extends AbstractAPI{
      @APIField(order = 1, type = "N", length = 20)
      private long userId;
      @APIField(order = 2, type = "M", length = 10)
      private BigDecimal amount;
  }
  ```

  ```java
  @Slf4j
  public class BankService {
  
      private static String remoteCall(AbstractAPI api) throws IOException {
          //从BankAPI注解获取请求地址
          CustomerApi bankAPI = api.getClass().getAnnotation(CustomerApi.class);
          bankAPI.url();
          StringBuilder stringBuilder = new StringBuilder();
          //获得所有字段
          Arrays.stream(api.getClass().getDeclaredFields())
                  //查找标记了注解的字段
                  .filter(field -> field.isAnnotationPresent(APIField.class))
                  //根据注解中的order对字段排序
                  .sorted(Comparator.comparingInt(a -> a.getAnnotation(APIField.class).order()))
                  //设置可以访问私有字段
                  .peek(field -> field.setAccessible(true))
                  .forEach(field -> {
                      //获得注解
                      APIField APIField = field.getAnnotation(APIField.class);
                      Object value = "";
                      try {
                          //反射获取字段值
                          value = field.get(api);
                      } catch (IllegalAccessException e) {
                          e.printStackTrace();
                      }
                      //根据字段类型以正确的填充方式格式化字符串
                      switch (APIField.type()) {
                          case "S": {
                              stringBuilder.append(String.format("%-" + APIField.length() + "s", value.toString()).replace(' ', '_'));
                              break;
                          }
                          case "N": {
                              stringBuilder.append(String.format("%" + APIField.length() + "s", value.toString()).replace(' ', '0'));
                              break;
                          }
                          case "M": {
                              if (!(value instanceof BigDecimal)) {
                                  throw new RuntimeException(String.format("{} 的 {} 必须是BigDecimal", api, field));
                              }
                              stringBuilder.append(String.format("%0" + APIField.length() + "d", ((BigDecimal) value).setScale(2, RoundingMode.DOWN).multiply(new BigDecimal("100")).longValue()));
                              break;
                          }
                          default:
                              break;
                      }
                  });
          //签名逻辑
  //        stringBuilder.append(DigestUtils.md2Hex(stringBuilder.toString()));
          String param = stringBuilder.toString();
          long begin = System.currentTimeMillis();
  /*        //发请求
          String result = Request.Post("http://localhost:45678/reflection" + bankAPI.url())
                  .bodyString(param, ContentType.APPLICATION_JSON)
                  .execute().returnContent().asString();*/
          log.info("调用银行API {} url:{} 参数:{} 耗时:{}ms", bankAPI.desc(), bankAPI.url(), param, System.currentTimeMillis() - begin);
          return param;
      }
      //创建用户方法
      public static String createUser(String name, String identity, String mobile, int age) throws IOException {
          CreateUserApi createUserAPI = new CreateUserApi();
          createUserAPI.setName(name);
          createUserAPI.setIdentity(identity);
          createUserAPI.setAge(age);
          createUserAPI.setMobile(mobile);
          return remoteCall(createUserAPI);
      }
      //支付方法
      public static String pay(long userId, BigDecimal amount) throws IOException {
          PayAPI payAPI = new PayAPI();
          payAPI.setUserId(userId);
          payAPI.setAmount(amount);
          return remoteCall(payAPI);
      }
  }
  ```

  

## 获取spring上下文的到容器中的bean

实现ApplicationContextAware接口

```Java
@Component
public class ApplicationContextUtils implements ApplicationContextAware {

    private static ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        System.out.println("初始化。。。");
        ApplicationContextUtils.applicationContext = applicationContext;
    }

    public static ApplicationContext getApplicationContext(){
        return applicationContext;
    }

    public static Object getBean(String name){
        return applicationContext.getBean(name);
    }
}
```

