## 从一个问题说起

最近一位同事在开发中遇到一个Spring事务嵌套的问题，导致程序没有按照他的想法执行，所以记下来，顺便了解一下Spring事务嵌套和传播机制的相关问题。

具体的问题是这样的，他写了这样一段代码，具体是在一个外层事务方法中调用了另一个类的事务方法对数据库进行更新，并且外层使用了try-catch捕获，目的是不管内层事务方法是否抛出异常，都不要影响外层事务方法的执行。

```java
@Service
public class BaseService{  
      @Autowired
    	private UserService userService;
  
  		@Transactional(rollbackFor = Exception.class)
      public void add(UserInputData userInputData) {
          //对数据库进行更新
          //XXXInsert()
          try{
              //使用try-catch进行了捕获，目的是为了不影响方法的执行。
              userService.add();
          }catch(Exception e){
            	e.printStackTrace();
          }
          //进行其他的操作
      }
}

@Service
public class UserService{  
  		@Transactional(rollbackFor = Exception.class)
      public void add() {
          //对数据库进行更新，在不满足某些校验条件时，抛出异常
  				if(XXXXX){
            throw new RuntimeException("校验未通过");
          }
      }
}
```

但是，很不幸，代码未通过内层方法的校验，抛出了异常，导致外层事务一并回滚，并且在日志中只能看到这么一句话  `Transaction rolled back because it has been marked as rollback-only` 。

## 事务嵌套和传播属性

先来说说什么是事务嵌套，事务嵌套是说在一个事务方法（以下称为外层事务）中调用另一个事务方法（以下称为内层事务）引起的事务嵌套问题，在外层事务开启的情况下，内层事务中的操作如何执行呢？是加入到外层事务中执行还是单独开启一个事务去执行，不同的操作会有不同的结果，所以Spring定义了传播属性来解决这种事务嵌套的问题。

需要说明的是，传播属性是Spring定义的用来解决事务嵌套的方案，和数据库没有关系，数据库中没有事务嵌套这个说法，只有事务并发执行的概念，所以数据库有事务隔离性的概念，关于数据库事务的特性这里就不再展开说了。 

Spring中定义了7种传播属性，根据对外层事务的不同操作大致可以分为以下几类:

**支持外层事务的情况：**

- TransactionDefinition.PROPAGATION_REQUIRED： 如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。默认的传播级别
- TransactionDefinition.PROPAGATION_SUPPORTS： 如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
- TransactionDefinition.PROPAGATION_MANDATORY： 如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。（mandatory：强制性）

**不支持外层事务的情况：**

- TransactionDefinition.PROPAGATION_REQUIRES_NEW：创建一个新的事务，如果当前存在事务，则把当前事务挂起。
- TransactionDefinition.PROPAGATION_NOT_SUPPORTED： 以非事务方式运行，如果当前存在事务，则把当前事务挂起。
- TransactionDefinition.PROPAGATION_NEVER： 以非事务方式运行，如果当前存在事务，则抛出异常。

**其他情况：**

- TransactionDefinition.PROPAGATION_NESTED： 如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于TransactionDefinition.PROPAGATION_REQUIRED。

我们先来解决一下上面同事遇到的问题，从上面可以看出，传播属性默认的级别是 `TransactionDefinition.PROPAGATION_REQUIRED` ，即把内层事务加入到外层事务中一起执行，实则就是两个方法是在同一个事务中运行的，所以在内层方法抛了异常之后，整个事务都回滚了，这个是没有问题的，那么`Transaction rolled back because it has been marked as rollback-only` 是怎么来的呢？

## 传播属性的具体使用

前面已经说过，Spring默认的传播属性是 `TransactionDefinition.PROPAGATION_REQUIRED` ，即将内层事务方法中的操作加入到外层事务中执行，所以，一旦内层方法抛了异常，则会将全局回滚标识置为true，但是因为外层事务进行了try-catch，并且没有在catch中手动抛出异常，所以外层事务会认为是执行成功了的，尝试commit提交事务，但是提交事务过程中发现全局回滚标识为true，所以抛出了`Transaction rolled back because it has been marked as rollback-only` ，并回滚了事务。

针对上面遇到的问题，我们只需要在内层事务中的注解中改为 `@Transactional(rollbackFor = Exception.class,propagation = Propagation.REQUIRES_NEW)` 即可，代表内层事务不能和外层事务一起执行，要单独开启一个事务去执行。

```java
@Service
public class BaseService{  
      @Autowired
    	private UserService userService;
  
  		@Transactional(rollbackFor = Exception.class)
      public void add(UserInputData userInputData) {
          //对数据库进行更新
          //XXXInsert()
          try{
              //使用try-catch进行了捕获，目的是为了不影响方法的执行。
              userService.add();
          }catch(Exception e){
            	e.printStackTrace();
          }
          //进行其他的操作
      }
}

@Service
public class UserService{
      //开启一个新事务去执行
  		@Transactional(rollbackFor = Exception.class,propagation = Propagation.REQUIRES_NEW)
      public void add() {
          //对数据库进行更新，在不满足某些校验条件时，抛出异常
  				if(XXXXX){
            throw new RuntimeException("校验未通过");
          }
      }
}
```

## Spring中事务的其他值得注意的地方

在使用Spring的声明式事务处理时，需要注意的是方法的自调用问题，来看一个例子。

```java
@Service
public class BaseService{  
      public void add(UserInputData userInputData) {
              insert();
      }
  
  		@Transactional(rollbackFor = Exception.class,propagation = Propagation.REQUIRES_NEW)
      public void insert() {
          //对数据库进行更新，在不满足某些校验条件时，抛出异常
  				if(XXXXX){
            throw new RuntimeException("校验未通过");
          }
      }
}
```

在上面的方法中，我们通过非事务方法add()去直接调用insert()，是不会开启事务的，原因在于，Spring的事务是通过动态的方式去实现的，即会对标记了@transaction注解的类生成代理类对象，在代理类对象中会开启事务，并在执行完后进行提交或者回滚操作。

但通过上面这种方式实际上是通过this去调用的事务方法，并没有通过代理对象去调用，所以也就不会开启事务了。解决办法也很简单，通过Spring注入BaseService的代理对象，通过代理对象再去调用insert。

```java
@Service
public class BaseServiceImpl implements BaseService, ApplicationContextAware{  
  		private ApplicationContext applicationContext;
    	@Override
    	public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        	this.applicationContext = applicationContext;
    	}
      public void add(UserInputData userInputData) {
        			BaseService bean = applicationContext.getBean(BaseService.class);
              bean.insert();
      }
  
  		@Transactional(rollbackFor = Exception.class,propagation = Propagation.REQUIRES_NEW)
      public void insert() {
          //对数据库进行更新，在不满足某些校验条件时，抛出异常
  				if(XXXXX){
            throw new RuntimeException("校验未通过");
          }
      }
}
```

