---
layout: post
cover: assets/images/cover.jpg
subclass: post
title: "Spring Data JPA 技术分析" 
navigation: true
logo: assets/images/host.png
date: 2015-12-07 12:34:56
categories: lxd
tags: tech java spring aop spring-data chs
comments: true
---


在另一篇介绍Spring的文章<sup>[[1](/spring-introduction)]</sup>里我介绍了Spring Data JPA，借助Spring Data JPA，对数据库做CRUD需要简单的四步操作：先创建数据库表和Java entity类；声明一个接口`ItemRepository`继承`CrudRepository`；再在接口`ItemRepository`里声明Query Method`findByName`；最后把`ItemRepository`注入到我们需要用到的地方，调用就可以了。

这篇文章再深入一点，解释/分析一下Spring在背后做了什么；为什么作为一个Spring用户，我们声明一个接口，在接口里声明一个方法，我们就可以调用这个方法了操作数据库了？

先从Spring AOP说起。


### Spring AOP Proxy

AOP（面向切面编程，Aspect-Oriented Programming），是Java里常用的一个编程范式。OOP对逻辑和数据做了一定程度的封装，但是当应用运行起来之后，从数据流的角度来看，数据是从一个类实例流到另一个实例，是链式的，我们编写的业务逻辑也是在一条条调用链上做文章。而AOP则是从另一个维度做文章：比如说原来的模块划分，或者调用链是纵向的，那么AOP是从横向切入，把纵向维度中共性的部分，比如logging、auth、cache，抽取出来，并且用IoC<sup>[[2](https://en.wikipedia.org/wiki/Inversion_of_control)]</sup>的思想，由抽取出来的`aspect`承担匹配和执行的责任，原来的业务逻辑不需要添加/改动任何代码。

举例来说，AOP可以做这样的事情：“让一个Web应用中所有Controller的POST方法执行之前打一条log”。抽取出来做这件事情的模块称为`aspect`，描述匹配到“所有Controller的POST方法”的谓词称为`pointcut`，“打log”这样的行为称为`advice`。

AspectJ是这个Java里AOP鼻祖老爷，Spring AOP兼容了一部分AspectJ的annotation，但核心还是Spring自己的，这个核心就是AOP proxy。比如我们有一个接口`Pojo`和它的实现`SimplePojo`，当没有proxy的时候，调用`SimplePojo`的`foo`方法是这样的：

![Plain POJO call without Proxy](/assets/images/aop-proxy-plain-pojo-call.png)

当我们创建一个`SimplePojo`的proxy之后，调用这个proxy的`foo`方法的是这样的：

![AOPproxy](/assets/images/aop-proxy-call.png)
（两张图片都来自Spring官方文档）

即proxy“代理”了对原对象原方法的调用，在这个代理的过程中我们可以执行`advice`，比如Spring AOP中的方法拦截器。下面的代码演示了如何用Spring AOP的底层API创建一个proxy。

{% highlight java %}
public interface I {
}

public class C implements I {
    public String m() { return "m in C"; }
}

public interface I2 extends I {
    String m();
    String m2();
}

void proxyDemo() {
        ProxyFactory result = new ProxyFactory();

        result.setTarget(new C());
        result.setInterfaces(I2.class);
        result.addAdvice(new MethodInterceptor() {
            @Override
            public Object invoke(MethodInvocation invocation) throws Throwable {
                if (invocation.getMethod().getName().equals("m2")) {
                    return "m2 in proxy";
                }
                return invocation.getThis().getClass().getMethod(invocation.getMethod().getName()).invoke(invocation.getThis());
            }
        });
        I2 proxy = (I2) result.getProxy();

        System.out.println(proxy.m());   //Output: `m in C'
        System.out.println(proxy.m2());  //Output: `m2 in proxy'
    }
{% endhighlight %}

首先声明一个接口`I`；类`C`实现接口`I`的同时自己定义了一个方法`m()`；接口`I2`继承`I`并声明了方法`m()`和方法`m2()`。ProxyFactory是Spring AOP的proxy工厂类，实例化之后，设置proxy的target为`C`的实例，要代理的接口为`I2`，然后添加一个方法拦截器。

方法拦截器是一个接口，我们实例化的匿名内部类的`invoke()`方法就是这个拦截器的回调。当调用的方法名字是`m2`的时候，拦截器直接返回一个字符串`m2 in proxy`。当调用的方法名字是`m`的时候，通过反射获取到target（`C`的实例）的`m()`方法，调用执行。
<br />
介绍了Spring AOP Proxy之后，就可以解释一开始那个问题的前半部分了——为什么在我们只声明了一个接口和方法的情况下，这个方法可以被调用到？事实上上面的demo已经基本上了回答了这个问题，接下来我把各部分逻辑对应到Spring Data JPA上。
{% highlight java %}
public interface ItemRepository extends CrudRepository<Item, String> {
    List<Item> findByName(String name);
}
{% endhighlight %}

Spring在创建的名为`itemRepository`的bean，就是一个proxy，这个proxy在`RepositoryFactorySupport`里完成了初始化和配置<sup>[[3](https://github.com/spring-projects/spring-data-commons/blob/01f2c30b1d1c342e168b3b541974332cc429e3e2/src/main/java/org/springframework/data/repository/core/support/RepositoryFactorySupport.java#L177)]</sup>；Proxy的target是`SimpleJpaRepository`<sup>[[4](https://github.com/spring-projects/spring-data-jpa/blob/fda74889de51e586bfa22033aed0affb6f7f4c76/src/main/java/org/springframework/data/jpa/repository/support/SimpleJpaRepository.java)]</sup>，它用`EntityManager`实现了`CrudRepository`里的基本CRUD方法；Proxy的代理接口即`ItemRepository`和`Repository`<sup>[[5](https://github.com/spring-projects/spring-data-commons/blob/01f2c30b1d1c342e168b3b541974332cc429e3e2/src/main/java/org/springframework/data/repository/core/support/RepositoryFactorySupport.java#L190)]</sup>；Proxy添加了多个方法拦截器，处理自定义Query Method的拦截器是<sup>[[6](https://github.com/spring-projects/spring-data-commons/blob/01f2c30b1d1c342e168b3b541974332cc429e3e2/src/main/java/org/springframework/data/repository/core/support/RepositoryFactorySupport.java#L375)]</sup>，这个拦截器在构造函数里遍历`ItemRepository`里的Query Method，创建一个`RepositoryQuery`对象放到一个map中<sup>[[7](https://github.com/spring-projects/spring-data-commons/blob/01f2c30b1d1c342e168b3b541974332cc429e3e2/src/main/java/org/springframework/data/repository/core/support/RepositoryFactorySupport.java#L461)]</sup>。

当我们的代码调用到`itemRepository.findByName()`的时候，即进入到方法拦截器的`invoke`方法<sup>[[8](https://github.com/spring-projects/spring-data-commons/blob/01f2c30b1d1c342e168b3b541974332cc429e3e2/src/main/java/org/springframework/data/repository/core/support/RepositoryFactorySupport.java#L438)]</sup>，拦截器首先判断是不是Query Method<sup>[[9](https://github.com/spring-projects/spring-data-commons/blob/01f2c30b1d1c342e168b3b541974332cc429e3e2/src/main/java/org/springframework/data/repository/core/support/RepositoryFactorySupport.java#L461)]</sup>，如果是的话就取出`RepositoryQuery`对象带参数执行，不是的话调用`SimpleJpaRepository`里的方法<sup>[[10](https://github.com/spring-projects/spring-data-commons/blob/01f2c30b1d1c342e168b3b541974332cc429e3e2/src/main/java/org/springframework/data/repository/core/support/RepositoryFactorySupport.java#L467)]</sup>。



### Mini Parser

下面接着继续分析开题问题的第二部分，为什么我们只声明一个`findByName`的方法，Spring就知道如何去查询数据库了？

事实上细细想来这个并不特别神奇，我们在声明query methods的时候，是需要遵循规范的<sup>[[11](http://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.query-methods.query-creation)]</sup>，既然我们定义方法遵循了规范，Spring基于同样的规范，用一个parser解析这个方法名，就能生成JPA的查询对象。

我们在前面提到Spring会根据`ItemRepository`里的Query Method，创建一个`RepositoryQuery`对象<sup>[[7](https://github.com/spring-projects/spring-data-commons/blob/01f2c30b1d1c342e168b3b541974332cc429e3e2/src/main/java/org/springframework/data/repository/core/support/RepositoryFactorySupport.java#L461)]</sup>，这个对象就是一个迷你的parser解析query method方法名的结果，解析的入口在<sup>[[12](https://github.com/spring-projects/spring-data-jpa/blob/master/src/main/java/org/springframework/data/jpa/repository/query/JpaQueryLookupStrategy.java#L95)]</sup>。

这个mini parser是自顶向下解析的，最顶上的节点类是`PartTree`<sup>[[13](https://github.com/spring-projects/spring-data-commons/blob/01f2c30b1d1c342e168b3b541974332cc429e3e2/src/main/java/org/springframework/data/repository/query/parser/PartTree.java#L76)]</sup>，它包含两个字段/子节点，`subject`和`predicate`。`subject`表示要查询的结果，`predicate`表示查询的条件。比如对于`findByName`，`subject`为空，`predicate`为`Name`；再比如稍微复杂一点的`findDistinctUserByNameOrderByAge`，subject是`DistinctUser`，predicate是`NameOrderByAge`。

`Subject`有3个boolean属性，分别是`distinct`，`count`，`delete`，`maxResults`<sup>[[14](https://github.com/spring-projects/spring-data-commons/blob/01f2c30b1d1c342e168b3b541974332cc429e3e2/src/main/java/org/springframework/data/repository/query/parser/PartTree.java#L275)]</sup>，如果query带了`Distinct``distinct`就是true，count方法`count`为true，delete方法`delete`为true。`maxResults`是limit查询的值，如`findFirst10ByLastname`。

`Predicate`有一个ArrayList`nodes`包含以`Or`split后的节点`OrPart`，还有一个`orderBySource`包含用`OrderBy`split的排序节点<sup>[[15](https://github.com/spring-projects/spring-data-commons/blob/01f2c30b1d1c342e168b3b541974332cc429e3e2/src/main/java/org/springframework/data/repository/query/parser/PartTree.java#L345)]</sup>。

……

`OrPart`节点往后不再赘述推导规则了，最后，定义一个比较复杂的方法看一下AST。对于`findDistinctByStateAndCountryLikeOrMapAllIgnoringCaseOrderByNameDesc`，AST如下图

![AST of a long Query Method](/assets/images/ast-of-a-long-query-method.png)


有了AST之后，Spring Data再调用JPA的接口创建`Predicate`对象、`CriteriaQuery`对象等，交给JPA查询DB<sup>[[16](https://github.com/spring-projects/spring-data-commons/blob/01f2c30b1d1c342e168b3b541974332cc429e3e2/src/main/java/org/springframework/data/repository/query/parser/AbstractQueryCreator.java#L98)]</sup>。

这个规则和parser都很简陋，很容易出现二义的情况，如果遇到这样的情况，就不一定非要苦苦纠结于解析规则了，它目的毕竟还是为了给用户提供方便，如果不能很方便的提供方便，那就给字段改个名儿，或者改用Named Query<sup>[[17](http://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods.named-queries)]</sup>。


<br /><br />







参考资料：

- http://docs.spring.io/spring-data/jpa/docs/current/reference/html/
- http://docs.spring.io/spring/docs/current/spring-framework-reference/html/aop-api.html
- [1]http://post.lxd.me/2015-12-01-spring-introduction/
- [2]（控制反转的思想，不特指Spring DI） https://en.wikipedia.org/wiki/Inversion_of_control
- [3]https://github.com/spring-projects/spring-data-commons/blob/01f2c30b1d1c342e168b3b541974332cc429e3e2/src/main/java/org/springframework/data/repository/core/support/RepositoryFactorySupport.java#L177
- [4]https://github.com/spring-projects/spring-data-jpa/blob/fda74889de51e586bfa22033aed0affb6f7f4c76/src/main/java/org/springframework/data/jpa/repository/support/SimpleJpaRepository.java
- [5]https://github.com/spring-projects/spring-data-commons/blob/01f2c30b1d1c342e168b3b541974332cc429e3e2/src/main/java/org/springframework/data/repository/core/support/RepositoryFactorySupport.java#L190
- [6]https://github.com/spring-projects/spring-data-commons/blob/01f2c30b1d1c342e168b3b541974332cc429e3e2/src/main/java/org/springframework/data/repository/core/support/RepositoryFactorySupport.java#L375
- [7]https://github.com/spring-projects/spring-data-commons/blob/01f2c30b1d1c342e168b3b541974332cc429e3e2/src/main/java/org/springframework/data/repository/core/support/RepositoryFactorySupport.java#L461
- [8]https://github.com/spring-projects/spring-data-commons/blob/01f2c30b1d1c342e168b3b541974332cc429e3e2/src/main/java/org/springframework/data/repository/core/support/RepositoryFactorySupport.java#L438
- [9]https://github.com/spring-projects/spring-data-commons/blob/01f2c30b1d1c342e168b3b541974332cc429e3e2/src/main/java/org/springframework/data/repository/core/support/RepositoryFactorySupport.java#L461
- [10]https://github.com/spring-projects/spring-data-commons/blob/01f2c30b1d1c342e168b3b541974332cc429e3e2/src/main/java/org/springframework/data/repository/core/support/RepositoryFactorySupport.java#L467
- [11]http://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.query-methods.query-creation
- [12]https://github.com/spring-projects/spring-data-jpa/blob/master/src/main/java/org/springframework/data/jpa/repository/query/JpaQueryLookupStrategy.java#L95
- [13]https://github.com/spring-projects/spring-data-commons/blob/01f2c30b1d1c342e168b3b541974332cc429e3e2/src/main/java/org/springframework/data/repository/query/parser/PartTree.java#L76
- [14]https://github.com/spring-projects/spring-data-commons/blob/01f2c30b1d1c342e168b3b541974332cc429e3e2/src/main/java/org/springframework/data/repository/query/parser/PartTree.java#L275
- [15]https://github.com/spring-projects/spring-data-commons/blob/01f2c30b1d1c342e168b3b541974332cc429e3e2/src/main/java/org/springframework/data/repository/query/parser/PartTree.java#L345
- [17]http://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods.named-queries
- [16]https://github.com/spring-projects/spring-data-commons/blob/01f2c30b1d1c342e168b3b541974332cc429e3e2/src/main/java/org/springframework/data/repository/query/parser/AbstractQueryCreator.java#L98


