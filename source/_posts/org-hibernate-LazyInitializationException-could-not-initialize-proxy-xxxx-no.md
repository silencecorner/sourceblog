---
title: >-
  org.hibernate.LazyInitializationException: could not initialize proxy xxxx no
  Session
tags:
  - issue
categories: java
date: 2019-08-19 17:07:45
---

# 问题

使用`reflectasm` MethodAccess调用`get`方法出错，报错`org.hibernate.LazyInitializationException: could not initialize proxy [com.bd.post.model.Post#2] - no Session`
# 查错过程
使用的orm框架是`spring data jpa`，`LazyInitializationException`第一时间想到`hibernate`和`spring data jpa`的懒加载机制。我理解的懒加载的概念是`在真正使用数据的时候才去执行sql语句(配置外键关联)，查询对外建关联对象`，但是我的model配置如下：
```
/**
 * @author <a href="mailto:hilin2333@gmail.com">created by silencecorner 2019/7/10 3:28 PM</a>
 */
@NoArgsConstructor
@Entity
@Data
@EntityListeners(AuditingEntityListener.class)
@ProtoClass(PostProto.Post.class)
public class Post {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @ProtoField
    private Integer id;
    @ProtoField
    private String title;
    @ProtoField
    private String body;
    @ProtoField
    private Integer authorId;

    @CreatedDate
    @ProtoField(nullValue = ProtobufNullValueInspectorImpl.class,converter = LocalDateTimeConverterImpl.class)
    private LocalDateTime createdAt;
    @LastModifiedDate
    private LocalDateTime updatedAt;

    public Post(String title, String body) {
        this.title = title;
        this.body = body;
    }

    public Post(String title, String body, Integer authorId) {
        this.title = title;
        this.body = body;
        this.authorId = authorId;
    }
}
```
这里我并没有配置外键的关联对象呀！<br>
具体的错误信息里面又有`no session`，在orm框架里面都有session的概念对应数据库的session，在mybatis中是`sqlsession`,spring data jpa和hibernate中就叫session。检查代码发现使用了`JpaRepository的getOne`方法获取数据库
```
  /**
  * Returns a reference to the entity with the given identifier. Depending on how the JPA persistence provider is
  * implemented this is very likely to always return an instance and throw an
  * {@link javax.persistence.EntityNotFoundException} on first access. Some of them will reject invalid identifiers
  * immediately.
  *
  * @param id must not be {@literal null}.
  * @return a reference to the entity with the given identifier.
  * @see EntityManager#getReference(Class, Object) for details on when an exception is thrown.
  */
  T getOne(ID id);
```
大概的意思是，只返回一个引用具体，信息看异常信息，what?找到调用的方法说明
```
  /**
    * Get an instance, whose state may be lazily fetched.
    * If the requested instance does not exist in the database,
    * the <code>EntityNotFoundException</code> is thrown when the instance 
    * state is first accessed. (The persistence provider runtime is 
    * permitted to throw the <code>EntityNotFoundException</code> when 
    * <code>getReference</code> is called.)
    * The application should not expect that the instance state will
    * be available upon detachment, unless it was accessed by the
    * application while the entity manager was open.
    * @param entityClass  entity class
    * @param primaryKey  primary key
    * @return the found entity instance
    * @throws IllegalArgumentException if the first argument does
    *         not denote an entity type or the second argument is
    *         not a valid type for that entity's primary key or
    *         is null
    * @throws EntityNotFoundException if the entity state 
    *         cannot be accessed
    */
  public <T> T getReference(Class<T> entityClass, Object primaryKey);
```
这里终于说啦是懒加载，然后就是不希望这个对象编程游离态，除非entity manager打开。好吧！这里鼠标回忆一下hibernate和jpa的对象状态，这些状态都是从[martinfowler的工作单元/Unit of Work](https://martinfowler.com/eaaCatalog/unitOfWork.html)思想得来的。这些概念都是隶属于`persistence coentext`，翻译过来就是持久化上下文，既然隶属一个上下文那肯定是有关系的。其中
- 瞬时态/transient 
  > 新new的一个就是表对象，这个对象就是顺势态
- 持久态/persistent 
  > 执行一下save方法，这个就是游离态
- 游离态/detachment 
  > 这个保存过的对象修改属性之后就变成了游离态（ps:持久态修改属性)好了，上面有一句最关键的话就是`unless it was accessed by the application while the entity manager was open`。去找一下我们的`EntityManager`接口，它是实现了`Session`接口的。很自然的就想到了事务，持久化没有事务（我当时写的时候还真没有加，哈哈！）

# 解决
懒加载需要将相应的东西保存到session，我们能控制就是加一个事务注解在方法上`@Transactional`声明这个方法没有执行完之前session不关闭。因为使用的spring boot，加一个配置：
```
spring.jpa.properties.hibernate.enable_lazy_load_no_trans=true
```
搞定
# 总结
jdbc的中sesion和数据库的sesion的是一个东西，sesion中保存当前会话变量事务声明！在hibernate和jpa中还有一个`persistence coentext`的概念：游离态、瞬时态、持久态，这些其实都是跟session密切相关的。

---
多看看学学还是有必要的，have a nice day ^_^!