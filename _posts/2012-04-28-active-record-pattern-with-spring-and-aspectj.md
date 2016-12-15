---
layout: post
title: "Active Record pattern with Spring and AspectJ"
tags: [AspectJ, Java, Persistence, Spring]
comments: true
---
By looking at [Spring Roo](http://www.springsource.org/spring-roo) I was amazed by how they implemented the Active Record pattern on top of JPA. It uses the Spring Framework (of course!) and AspectJ to decorate the domain objects with JPA annotations and add the necessary methods to deal with persistence. Despite this, I don't like the automagically generated code from the command line.

In this short tutorial I am going to explain how to do it without Roo. The complete source code can be found in this Gist: [https://gist.github.com/2500752](https://gist.github.com/2500752). The three commits reflect the development stages.

<!--break-->

Let's start by defining a basic domain model:

```java
public class User {

  private String username;

  public User() {
  }

  public User(final String username) {
    this.username = username;
  }

  public String getUsername() {
    return username;
  }

  public void setUsername(final String username) {
    this.username = username;
  }
}
```

It's nothing but a simple bean. Next we write the first test for our active record implementation:

```java
public class UserTest {

  @Test
  public void noUserIsPresent() {
    assertEquals(0, User.findAll().size());
  }
}
```

It should complain that the method findAll() is undefined for the type User. Let's fix the compilation errors before we do any real implementation. Let's create a UserPersistence aspect that enriches the User class with the static method.

```java
public aspect UserPersistence {

  public static List User.findAll() {
    return new ArrayList();
  }
}
```

The project should compile and the test should pass meaning that AspectJ is working. Now we are going to add some real implementation. What we have to do is to alter the aspect, as opposed to changing the model itself.

```java
public aspect UserPersistence {

  declare @type: User: @Configurable;

  declare @type: User: @Entity;
  declare @field: * User.username : @Id;

  @PersistenceContext
  private transient EntityManager User.entityManager;

  public static List<User> User.findAll() {
    return new User().entityManager
        .createQuery("SELECT o FROM User o", User.class)
        .getResultList();
  }
}
```

Of course the test will fail with a null pointer exception, since the entity manager has not been injected. Here it comes Spring. We need to specify the JPA configuration in the application-context.xml:

```xml
<beans>

  <context:spring-configured/>

  <bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
    <property name="driverClassName" value="org.hsqldb.jdbcDriver" />
    <property name="url" value="jdbc:hsqldb:mem:mydb" />
    <property name="username" value="sa" />
    <property name="password" value="" />
  </bean>

  <bean id="entityManagerFactory"
  class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean">
    <property name="dataSource" ref="dataSource" />
    <property name="packagesToScan" value="net.paoloambrosio.model" />
    <property name="jpaVendorAdapter">
      <bean class="org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter">
        <property name="showSql" value="true" />
        <property name="generateDdl" value="true" />
        <property name="databasePlatform" value="org.hibernate.dialect.HSQLDialect" />
      </bean>
    </property>
  </bean>

</beans>
```

We also need to annotate the test class with the Spring runner to bootstrap the framework:

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations={"classpath:application-context.xml"})
```

Now the test should run as expected and in the logs you should see the HyperSQL database starting and being queried.

To complete this tutorial let's see how to add transaction management and persist the User objects. We annotate the test with Spring's @Transactional annotation and add a few more test cases (see the Gist for the complete test class source). Then we add a couple more methods to the aspect:

```java
public aspect UserPersistence {

  // ...

  @Transactional
  public void User.persist() {
    entityManager.persist(this);
  }

  public static User User.findByUsername(final String username) {
    return new User().entityManager.find(User.class, username);
  }
}
```

Finally we add transaction management to the application-context.xml:

```xml
<beans>

  <!-- ... -->

  <bean id="transactionManager" class="org.springframework.orm.jpa.JpaTransactionManager">
    <property name="entityManagerFactory" ref="entityManagerFactory" />
  </bean>

  <tx:annotation-driven mode="aspectj" />

</beans>
```

All the tests should be passing now, with a beautiful decoupling of the User bean from its persistence methods.
