---
layout: post
cover: assets/images/spring-java-ee.png
subclass: post
title: "Spring 介绍" 
navigation: true
logo: assets/images/host.png
date: 2015-12-01 12:34:56
categories: lxd
tags: tech java spring aop ioc spring-DI spring-boot spring-mvc chs
comments: true
---

这篇文章对Spring Framework做一个基础的介绍，先介绍Spring和Spring相关的几个概念，再介绍3个常用的Spring项目/模块：Spring MVC、Spring Data JPA、和Spring Boot。


### Spring 和 Java EE

![Spring and Java EE](/assets/images/spring-java-ee.png)

Java平台有多个“版本”，和Spring相关的有两个：Java SE和Java EE。Java SE（Java Platform, Standard Edition）包含JVM和Java最核心/基本的类库API；而Java EE（Java Platform, Enterprise Edition）是在Java SE之上，定义的一套specifications，用来支持所谓的Enterprise/企业级开发，比如用来接受处理HTTP请求的servlet specification、和关系型数据库交互的API（JPA）等。Java EE定义的每一条规范称为JSR（Java Specification Request），由JCP（Java Community Process）开发维护，就像RFC之于IETF。

有标准就有实现，基于Java EE的标准，业界有好些个实现，这些实现基本都是“大公司”的产品，比如IBM的WebSphere Application Server（WAS），Oracle的WebLogic，RedHat的JBoss/WildFly等等，这些实现都遵循Java EE的标准，而且实现了Servlet spec、JPA spec等大部分重要的specification，称为应用服务器（Application Server），Java EE规范保证了我们编写的应用能够部署/运行在上面；而在开源界更常用的Tomcat/Jetty则称为Web容器（Web Container），只实现了Java EE里的Servlet spec、JSP spec等和Web相关的specification，我们如果需要依赖注入，就需要自己引入Spring，需要用JPA访问关系型数据库，就需要自己引入Hibernate。

而且Spring在严格意义上来说并不是Java EE的实现，因为它一方面遵循了Java EE的Servlet、JPA等核心接口，但是对于更上层的API，Spring用的是自己的东西，比如依赖注入（Spring DI vs CDI），RESTful Web Service（Spring MVC vs JAX-RS）。


### Spring IoC / DI

Spring框架最核心的概念就是它基于控制反转技术（Inversion of Control - IoC）的依赖注入（Dependency Injection - DI），这两个狰狞的名词到底是什么意思？我简单描述一下。

抛开Spring，IoC是一种思想，一种技术，即对于一个主体而言，原来由他主动负责/发起的工作，变成了由它被动接受，这个由主动变被动的过程，称为IoC。比如library和framework就是一个很典型的好例子：当我们使用library，我们编写的代码调用lib，我们是主动方、发起方；而当我们改用framework，我们编写的代码则作为framework的回调、实现、或被托管对象，我们从主动方变成了被动方。通过IoC，我们可以解耦，可以解除主体的责任，可以解放自己。

Spring IoC（a.k.a. Spring DI）就是IoC在Spring里的一个应用，它把本来应该由我们自己写代码主动负责的对象实例化和依赖引用的过程，交由Spring来完成，举个例子：

在不借助DI，我们用Java写一个application的时候，如果Class A依赖于Class B和Class C，我们可以把B的实例由构造函数/setter函数传进去，C的实例由A自己实例化一个出来，就像这样：
{% highlight java %}
public class B {
}

public class C {
}

public class A {
    private B b;
    private C c = new C();

    public A(B b) { this.b = b; }
    public void setB(B b) { this.b = b; }
}
A a = new A(new B());
{% endhighlight %}

借助Spring DI和其他一些技术，可以这样：
{% highlight Java %}
@Component
public class B {
}

@Component
public class C {
}

@Component
public class A {
    private B b;
    @Autowired private C c;

    @Autowired
    public A(B b) { this.b = b; }
}
{% endhighlight %}

这里步子跨的有点大，`@Component`标签告诉Spring这个类需要被实例化为一个bean，bean是由Spring管理的一个Java对象实例，所以在这里Class A B C都会被Spring实例化为bean。`@Autowired`标签告诉Spring去寻找一个类型匹配的bean注入到该字段/方法，形成对B和C的引用，这便是<u>依赖注入</u>。

所以，本来应该由的代码负责的，把B和C的实例交给A的实例引用的过程，变成了在代码中只声明依赖，由Spring完成实例化和实例之间的引用，这就是IoC — Inversion of Control——<u>控制反转</u>。


### Web Container

Web Container，Web容器，也叫Servlet容器。我们用Spring构建的Web应用是要借助一个容器来运行的，比如Tomcat、Jetty，GlassFish。因为无论是用Spring还是其他Java EE框架编写出来的Web应用，都是是一个个独立的servlet，或者基于servlet的东西，这些独立的servlet不能直接运行，也不能直接接收来自浏览器的HTTP请求。Tomcat/Jetty这些容器一方面是一个Web服务器——即和浏览器交互，接收/响应HTTP请求；另一方面是Servlet容器——加载servlet，负责管理他们的生命周期，根据mapping规则把分发给响应的servlet，如下图

![Servlet](/assets/images/servlet-container.png)

这一整套都是Java EE Servlet规范，Spring和容器都得遵守这些规范，用Spring写的Web应用才可以部署到Web容器里去。


### Spring MVC

Spring框架由诸多模块组成，最核心的有spring-core、spring-beans，和数据库交互的有spring-jdbc、spring-orm等，我们用Spring来构建Web项目比较多，所以介绍一下Spring Web MVC模块。

Servlet是Java EE里处理HTTP请求的技术/规范，Spring MVC也是基于servlet来设计的，基于一个核心的`DispatcherServlet`，HTTP请求进来之后由它分发给其他的Controller处理。下面是在Spring里写一个Controller的demo，另外Spring MVC提供了Model-View-Controller的完整功能，但在这前后端分离的年代，就只demo提供RESTful API、返回JSON的Controller写法：

{% highlight java %}
@RestController
@RequestMapping("/stores/{storeId}/items")
public class ItemController {
    @RequestMapping(path = "/{itemId}", method = RequestMethod.GET, produces = MediaType.APPLICATION_JSON_UTF8_VALUE)
    public Item retrieveItem(@PathVariable String storeId, @PathVariable String itemId) {
        ...
        return item;
    }

    @RequestMapping(method = RequestMethod.POST)
    public ResponseEntity createItem(@RequestBody Item item) {
        ...
        return new ResponseEntity<>(HttpStatus.OK);
    }
}
{% endhighlight %}

- `@RestController`标记在一个Java类上，告诉Spring MVC这是一个RESTful API Controller，所有的Controller同时也都是Spring管理的bean；
- `@RequestMapping`描述了URL的路由映射。标记在类上的`@RequestMapping`定义了顶层的路径，结合在方法上的`@RequestMapping`的路径和HTTP method，将一个HTTP请求映射到一个Java方法上，由这个方法来处理HTTP请求，返回HTTP响应。
- `@RequestMapping`中定义路径的字符串支持URI Template<sup>[[1](https://tools.ietf.org/html/rfc6570)]</sup>，路径中{变量}所代表的的值，用`@PathVariable`标记在同名的Java方法参数上提取出来。
- `@RequestBody`标记在Java方法参数上，Spring MVC会自动把HTTP请求转成Java对象，绑定在这个参数上；同样的，因为我们标记了这个类为RESTful Controller，Java方法的返回值会直接转成JSON；
- `ResponseEntity`是一个泛型类，包装了Controller返回的HTTP response内容，增加了可以控制HTTP响应头、状态码的方法，代码里返回一个空的HTTP响应，状态码设置为200。

写一个最简单的Item类，用cURL调用RESTful接口可以看到下面的执行结果：

{% highlight java %}
public class Item {
    private String name;
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
}
{% endhighlight %}

{% highlight text %}
$ curl -v http://localhost:8080/stores/16/items/1
...
> 
< HTTP/1.1 200 OK
< Server: Apache-Coyote/1.1
< Content-Type: application/json;charset=UTF-8
...

{"name":"hehe"}%
{% endhighlight %}


{% highlight text %}
curl -v -H "Content-Type: application/json" -i -X POST -d '{"name":"haha"}' http://localhost:8080/stores/16/items
...
> Content-Type: application/json
> Content-Length: 15
> 
< HTTP/1.1 200 OK
< Server: Apache-Coyote/1.1
< Content-Length: 0
...

{% endhighlight %}


### Spring Data JPA

过去我们谈起Spring是一个framework，但是现在的Spring更多的像一个family/collection，基于核心的Spring Framework之上做了好些project<sup>[[2](https://spring.io/projects)]</sup>，Spring Data就是其中之一。Spring Data给数据库的访问提供了一个更优雅/强大的接口，Spring Data JPA是基于JPA、针对关系型数据库的子模块，给我们写应用日常遇到的简单CRUD提供了一个非常干净的接口。

Spring Data / JPA 提供给我们用户的是一系列Java interface，它在背后用基于Dynamic Proxy的AOP帮我们完成了实际的工作。最顶层的`Repository<T, ID extends Serializable>`，提供了最基本CRUD操作的`CrudRepository<T, ID extends Serializable>`，JPA相关操作的`JpaRepository<T, ID extends Serializable>`等。它们都是泛型接口，第一个类型参数是对应到数据库的实体类，第二个参数是实体类里ID的类型。我们声明一个接口继承这些接口，再申明一些我们自己的方法进去，就可以完成大部分简单的数据库操作了。

借助Spring Data JPA，对数据库做简单的CRUD需要四步操作：

第一步，在数据库中创建表，然后创建一个entity类映射它
{% highlight sql %}
CREATE TABLE item (
  id          VARCHAR(36)  NOT NULL,
  name        VARCHAR(128) NOT NULL,
  description VARCHAR(128),
  CONSTRAINT itemPk PRIMARY KEY (id)
);
{% endhighlight %}

{% highlight java %}
@Entity
public class Item {
    @Id
    @GeneratedValue(generator = "uuid")
    @GenericGenerator(name = "uuid", strategy = "uuid2")
    private String id;

    public Item() {
    }

    public Item(String name, String description) {
        this.name = name;
        this.description = description;
    }
    private String name;
    private String description;
}
{% endhighlight %}

第二步，声明一个接口，继承填入类型参数的`CrudRepository`接口
{% highlight java %}
public interface ItemRepository extends CrudRepository<Item, String> {
}
{% endhighlight %}

第三步，`CrudRepository`接口里已经声明了一些最基本的CRUD方法`save(S entity)`，`findOne(ID id)`，`exists(ID id)`，`delete(ID id)`等，我们可以在声明的接口里再进一步声明一些我们业务上需要的方法，但是要根据一定的命名规则<sup>[[3](http://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.query-methods.query-creation)]</sup>。比如根据`name`查找出所有同名的item的方法声明为`findByName`。
{% highlight java %}
public interface ItemRepository extends CrudRepository<Item, String> {
    List<Item> findByName(String name);
}
{% endhighlight %}

第四步，把`ItemRepository`注入到我们需要用到的地方，调用就可以了
{% highlight java %}
public class SomeClient {
  @Autowired private ItemRepository itemRepository;

  public void doSomething() {
    List<Item> items = itemRepository.findByName("teapot");
  }
  public void doSomethingElse() {
    Item newItem = new Item("smug", "smug without mug");
    itemRepository.save(newItem);
  }
}
{% endhighlight %}

自定义查询方法除了简单的findBy...以外还可以进行更丰富的组合：and、or、pagination、limit、asc/desc等<sup>[[3](http://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.query-methods.query-creation)]</sup>。

那么Spring Data是怎么做这些magic的？其实这对Spring核心来说来说都不是很困难。首先用基于Dynamic Proxy的Spring AOP，创建一个proxy作为`itemRepository`的bean实例，然后解析所有的自定义方法如`findByName`，用一个迷你的parser解析方法名，将AST保存在bean上；当应用启动我们的代码调用到`findByName`的时候，Spring AOP拦截了方法调用，取出AST，解析成一个Query对象，再结合参数交给JPA实现做查询，我在另外一篇文章<sup>[[4](http://post.lxd.me/2015-12-07-spring-data-jpa-internals)]</sup>做了更详细的分析。


### Spring Boot

Spring Boot是另一个Spring project，Convention over Configuration风的追随者。它基于Spring做了很多工作，使得编写一个Spring应用变的非常容易。


以往我们初始化一个Java应用需要在maven/gradle配置文件里历数写上它的依赖，非常繁复，Spring Boot非常体贴的给我们提供了`Start POMs`——一站式的jar包依赖解决方案。它从Spring用户的视角，对有关联的、通常会一起使用的第三方依赖做了编组，当我们需要Spring的哪部分功能的时候，在配置文件里写入对应编组的artifactId作为依赖就好了。比如我们需要做Web应用，一般需要Spring MVC的依赖，spring-web模块，Jackson的依赖等，在Spring Boot里我们引入`spring-boot-starter-web`就可以了。类似的还有包含JUnit、Hamcrest和Mockito等测试工具的`spring-boot-starter-test`，Spring Data JPA相关的`spring-boot-starter-data-jpa`等。

传统的Spring Web应用打包之后需要放在Tomcat/Jetty容器下面运行，Spring Boot应用引入`spring-boot-starter-web`依赖之后还会引入一个内嵌的Tomcat，这个内嵌的Tomcat可以通过调用`SpringApplication.run()`再借助Spring Boot的自动配置启动。这样我们编写一个入口main函数，就可以直接启动Web应用了，无论是命令行、IDE、亦或打成一个独立的jar包用`java -jar`运行。


Spring Boot在减少配置方面还做了很多细的调优。比如对于ORM，一个Java entity类对应到数据库的一张表的时候，默认Java entity类的字段名对应到数据库的字段名。如果JPA用的是Hibernate，并且配置了`ImprovedNamingStrategy`，就可以把entity类里的camelCase的字段名映射到数据库表里snake_case的字段名，很有Convention Over Configuration的范儿了，但是对于外键却是不生效的。Spring Boot提供了一个`SpringNamingStrategy`继承`ImprovedNamingStrategy`，添加了对外键的支持。




### 一些细节
> 不要在意那些细节！
> 
> &nbsp; <span style="float:right; margin-right:10%" >—— Coney Wu</span>


Spring 自动装配（autowire）就是自动发现匹配的bean注入到有依赖的bean里面去。所谓自动发现，就是Spring扫描ApplicationContext。所谓匹配，有两种，一种是byName——有依赖的bean的属性名字和被依赖的bean名字相同；另一种是byType——有依赖的bean的属性类型和被依赖的bean类型相同，当有多个同一类型的bean的时候，就会抛exception。@Autowired annotation是byType匹配的，如果有多个同类型的bean，可以结合@Qualifier annotation<sup>[[5](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/beans.html#beans-autowired-annotation-qualifiers)]</sup>指定到特定的bean上。


如果bean之间互相依赖，或者出现环会不会报错？——如果整条环上的bean之间都是<u>构造函数依赖注入</u>的方式互相依赖的话就会报错，前面实例代码里的注入方式即<u>构造函数依赖注入</u>，这种依赖注入方式表明要想实例化class A，必须先注入class B的实例化bean，如果class B也构造函数注入依赖于class A，那么就陷入死循环了。解决方法是把Spring启动先初始化的那个bean的<u>构造函数依赖注入</u>换成基于setter函数的依赖注入，或者用`@Autowired`注入到类的字段。但是通常情况下我们又不想去关心/控制bean的初始化顺序，这时候全部用setter函数注入或者用`@Autowired`注入类字段就是比较省心的方案。



Web容器和Web应用交互的入口点是一个称为deployment descriptor file的文件，即web.xml，再WEB-INF/文件夹下面，它定义了诸如servlet、servlet和URL的mapping、event listeners等


Java对象到JSON的相互转换只是属于Java和HTTP req/res相互转换中的一种，转换方法都继承于`HttpMessageConverter<T>`接口。在Spring里注册有有多个实现类，JSON的转换是由`MappingJackson2HttpMessageConverter`完成的，依赖于Jackson。


Spring Boot的另一个消灭手动配置的重要手法是自动配置，即根据应用现在CLASSPATH里的依赖的jar自动配置Spring，用`@EnableAutoConfiguration`注解开启，对Spring Boot来说几乎也是必须的。

Spring Boot应用有一个入口类标记了`@Configuration`。还有一个静态的main方法作为程序的入口，main方法再调用`SpringApplication`，把当前类作为入口配置传参数进去，`SpringApplication`再去启动内嵌tomcat等后续过程。



### Further Reading

这篇文章对Spring做了一个很基本的介绍，略去了很多的细节，Spring的官方文档提供了更全面和详细的阐述<sup>[[0](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/)]</sup>。Spring的模块/项目/子框架很多，除了我介绍的这三个，还有如切面编程的Spring AOP，Authentication和Authorization的框架Spring Security等等都是易学好用的车轮子。Spring Boot除了我上述的几个特性以外也还有很多值得关注的内容，如Spring Initializer、用于生产环境监控的actuator等等，一切的一切，都在Spring的官方文档里。

<br />

参考资料：

- https://docs.oracle.com/javase/8/docs/technotes/guides/reflection/proxy.html
- http://docs.spring.io/spring/docs/current/spring-framework-reference/html/aop-api.html
- http://docs.spring.io/spring-data/jpa/docs/current/reference/html
- http://sivalabs.in/2015/06/a-developers-perspective-on-spring-vs-javaee.html
- [0]http://docs.spring.io/spring/docs/current/spring-framework-reference/html/
- [1]https://tools.ietf.org/html/rfc6570
- [2]https://spring.io/projects
- [3]http://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.query-methods.query-creation
- [4]http://post.lxd.me/2015-12-07-spring-data-jpa-internals/
- [5]http://docs.spring.io/spring/docs/current/spring-framework-reference/html/beans.html#beans-autowired-annotation-qualifiers


