# Overview
Spring Data JPA, part of the larger Spring Data family, makes it easy to easily implement JPA based repositories. This module deals with enhanced support for JPA based data access layers. It makes it easier to build Spring-powered applications that use data access technologies.

Implementing a data access layer of an application has been cumbersome for quite a while. Too much boilerplate code has to be written to execute simple queries as well as perform pagination, and auditing. Spring Data JPA aims to significantly improve the implementation of data access layers by reducing the effort to the amount that’s actually needed. As a developer you write your repository interfaces, including custom finder methods, and Spring will provide the implementation automatically.

## Features
* Sophisticated support to build repositories based on Spring and JPA
* Support for Querydsl predicates and thus type-safe JPA queries
* Transparent auditing of domain class
* Pagination support, dynamic query execution, ability to integrate custom data access code
* Validation of @Query annotated queries at bootstrap time
* Support for XML based entity mapping
* JavaConfig based repository configuration by introducing @EnableJpaRepositories.

引自《[Spring Data JPA](https://spring.io/projects/spring-data-jpa)》
# 主要对象
## 7 个大 Repository 接口：
```
Repository(org.springframework.data.repository)，没有暴露任何方法；

CrudRepository(org.springframework.data.repository)，简单的 Curd 方法；

PagingAndSortingRepository(org.springframework.data.repository)，带分页和排序的方法；

QueryByExampleExecutor(org.springframework.data.repository.query)，简单 Example 查询；

JpaRepository(org.springframework.data.jpa.repository)，JPA 的扩展方法；

JpaSpecificationExecutor(org.springframework.data.jpa.repository)，JpaSpecification 扩展查询；

QueryDslPredicateExecutor(org.springframework.data.querydsl)，QueryDsl 的封装。
```
## 两大 Repository 实现类：
```
//JPA 所有接口的默认实现类；
SimpleJpaRepository(org.springframework.data.jpa.repository.support)
//QueryDsl 的实现类
QueryDslJpaRepository(org.springframework.data.jpa.repository.support)
```
![jpa.png](https://www.hounk.world/upload/2021/01/jpa-7950c8c16dec49ba9caf5c9e61289701.png)

# 自定义方法
## 命名关键字
![image.png](https://www.hounk.world/upload/2021/01/image-447b40c3d52a4b7394128ff40264b1fb.png)
## 没有实现类是如何调用的
今天被其他人为了一个问题，自定义方法里面IsIn和In有什么区别吗？
有什么区别，打印一下执行的sql就可以看出来了呗。
```java
    @Autowired
    private ActivityRepository activityRepository;
    @Test
    public void testJpa() {
        activityRepository.findAllByIdIn(Lists.newArrayList(1, 2, 3));
        System.out.println("___________________________");
        activityRepository.findAllByIdIn(Lists.newArrayList(1, 2, 3));
    }
```
执行结果
```
 Hibernate: select activity0_.id as id1_0_, activity0_.activity_info as activity2_0_ from activity activity0_ where activity0_.id in (? , ? , ?)
___________________________
Hibernate: select activity0_.id as id1_0_, activity0_.activity_info as activity2_0_ from activity activity0_ where activity0_.id in (? , ? , ?)
```
所以我就告诉他，这两个一样。但是只通过执行的sql来说服好像有点牵强啊。那就跟着代码看一下吧。

首先，我们断点到IsIn方法上，
![image.png](https://www.hounk.world/upload/2021/01/image-67c82b4fc1fa47b3bb8ce291d4d597d4.png)
然后step into这个方法
![image.png](https://www.hounk.world/upload/2021/01/image-3714de02f13f4c429f200898e29131d3.png)
可以看到，我们自定义的接口是通过动态代理来实现的，代理的类就是SimpleJpaRepository，一步步往下走，每个if都没有进去，继续执行
![image.png](https://www.hounk.world/upload/2021/01/image-4660c2aadfdb48018d09e128775efea9.png)
可以看到这里有八个拦截器，每个拦截器这里就不展开介绍了，根据拦截器的命名或者我们点开这几个拦截器，在第六个拦截器中可以看到有这样一个Map，这不正是自定义的方法吗！
![image.png](https://www.hounk.world/upload/2021/01/image-d816040193a04c43b620e8ff4fec9eee.png)
我们打开这个拦截器，看下这个拦截器初始化的时候都做了什么
![image.png](https://www.hounk.world/upload/2021/01/image-448202de514d49a0a0e181112e65b58b.png)
最后可以看到调用了lookupQuery方法，在这个方法中有一个调用，是搜索查询策略，我们进去这个方法
```java
org.springframework.data.repository.query.QueryLookupStrategy#resolveQuery
```
![image.png](https://www.hounk.world/upload/2021/01/image-33ac7d3395cd41f09df0d3739a590373.png)
![image.png](https://www.hounk.world/upload/2021/01/image-245722db73fa4d8898561c84b3f38812.png)
执行lookupStrategy.resolveQuery方法，抛出IllegalStateException，开始执行createStrategy.resolveQuery，这个方法中构造了一个PartTreeJpaQuery
![image.png](https://www.hounk.world/upload/2021/01/image-2312f77a45d044a7ab362c57ba27caec.png)
进入构造方法，方法中构造了一个PartTree，看下这个PartTree是什么东西
![image.png](https://www.hounk.world/upload/2021/01/image-227ad6ea0813460d9bb8469a7f2b8e43.png)
在这个对象中终于看到了令人兴奋的东西，这不就是我们自定义方法里面用到的一些关键字吗？而且PartTree的构造中可以看到初始化了Subject和Predicate，现在代码跟一下这个两个参数，以Predicate为例
![image.png](https://www.hounk.world/upload/2021/01/image-be8ef46aeace4d7c9b5926e3492fdaab.png)
![image.png](https://www.hounk.world/upload/2021/01/image-4e4583d8fef84cf19ad6de33fc79368f.png)
![image.png](https://www.hounk.world/upload/2021/01/image-533453873055474ca20ca6fa3136fd26.png)
一步步跟到Part中，在Part的构造方法中有一步调用
```java
this.type = Type.fromProperty(partToUse);
```
![image.png](https://www.hounk.world/upload/2021/01/image-5bb223b6609a4ebca29b9ad89db9d96b.png)
看下这个type是什么东西
![image.png](https://www.hounk.world/upload/2021/01/image-22c6673cf3a54d0ab1669beeff23f17f.png)
眼前一亮吧，自定义用到的其他一些词也都在这里，所有的问题都可以解开了，Is和不带Is没有啥区别，个人猜测可能是开发人员为了照顾不同人的习惯吧。

综上，我们在使用Spring Data Jpa时，虽然只写了接口，但框架内部通过动态代理，来生成对应接口的方法，根据方法名称解析出sql，然后执行。

# 对异步的支持
Repository 对 Feature/CompletableFuture 异步返回结果的支持：
我们可以使用 Spring 的异步方法执行Repository查询，这意味着方法将在调用时立即返回，并且实际的查询执行将发生在已提交给 Spring TaskExecutor 的任务中，比较适合定时任务的实际场景。异步使用起来比较简单，直接加@Async 注解即可，如下所示：
```java
 @Async
 Future<MStockBom> findFirstByCreateName(String create);

 @Async
 CompletableFuture<MStockBom> findFirstByCreate(String create);

 @Async
 ListenableFuture<MStockBom> findFirstByCreateNum(String create);
```
# 部分使用eg.
## Sort
``` java
//所有排序字段都为正序或倒叙
List<User> descAll = userRomRepository.findAll(Sort.by(Sort.Direction.DESC, "createAt", "createBy"));
//排序字段倒叙正序都有
List<MStockBom> createAt_desc = mStockBomRepository.findAll(Sort.by(Sort.Order.desc("createAt"), Sort.Order.asc("createBy")));
```

## Specification动态参数查询
``` java
    @Test
    public void testSpecFind() {
        Specification<MStockBom> spec1 = new Specification<MStockBom>() {
            @Override
            public Predicate toPredicate(Root<MStockBom> root, CriteriaQuery<?> query, CriteriaBuilder criteriaBuilder) {
                Predicate predicate = criteriaBuilder.equal(root.get("name").as(String.class), "sss");

                return predicate;
            }
        };

        Specification<MStockBom> spec2 = new Specification<MStockBom>() {
            @Override
            public Predicate toPredicate(Root<MStockBom> root, CriteriaQuery<?> query, CriteriaBuilder criteriaBuilder) {
                List<Predicate> pr = Lists.newArrayList();
                pr.add(criteriaBuilder.equal(root.get("name").as(String.class), "sss"));
                pr.add(criteriaBuilder.like(root.get("bomMid").as(String.class), "%000%"));
                pr.add(criteriaBuilder.between(root.get("createAt").as(Date.class), new Date(), new Date()));
                return criteriaBuilder.and(pr.toArray(new Predicate[0]));
            }
        };

        Specification<MStockBom> spec3 = new Specification<MStockBom>() {
            @Override
            public Predicate toPredicate(Root<MStockBom> root, CriteriaQuery<?> query, CriteriaBuilder criteriaBuilder) {
                List<Predicate> pr = Lists.newArrayList();
                pr.add(criteriaBuilder.equal(root.get("name").as(String.class), "sss"));
                pr.add(criteriaBuilder.like(root.get("bomMid").as(String.class), "%000%"));

                List<Predicate> or = Lists.newArrayList();
                or.add(criteriaBuilder.equal(root.get("category").as(Integer.class), 1));
                or.add(criteriaBuilder.equal(root.get("institutionMid").as(String.class),"sss"));

                Predicate prAnd = criteriaBuilder.and(pr.toArray(new Predicate[0]));
                Predicate prOr = criteriaBuilder.or(or.toArray(new Predicate[0]));
                return query.where(prAnd,prOr).getRestriction();
            }
        };

//         1 单条件
        Optional<MStockBom> one = mStockBomRepository.findOne(spec1);
//        // 2 多条件
        Optional<MStockBom> mu = mStockBomRepository.findOne(spec2);
//        // 3 查询多个
        List<MStockBom> all = mStockBomRepository.findAll(spec2);
//        // 4 排序
        List<MStockBom> allSort = mStockBomRepository.findAll(spec2, Sort.by(Sort.Direction.DESC, "id"));
        //5 分页
        Pageable pageable = PageRequest.of(0, 3, Sort.by(Sort.Direction.DESC, "id"));
        Page<MStockBom> page = mStockBomRepository.findAll(spec2, pageable);
        page.get().forEach(System.out::println);
        //6 and or 一起用
        //repository需要集成JpaRepositoryImplementation
        mStockBomRepository.findAll(spec3,pageable);
    }
```
and or一起用的执行结果如图
![image.png](https://www.hounk.world/upload/2021/03/image-9f40bc77b08b4999902c173821bba1b1.png)
