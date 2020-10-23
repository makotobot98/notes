# Spring Framework

## IOC
- **dependency injection**(or **Inversion of control**) is the idea of creating the dependencies for an object externally(inject), the dependencies will be passed to constructor(or setter) instead of manually initializing the dependency within the constructor. Spring container is used to handle those dependencies while user only need to specify it in xml(or annotation)
    - [What is  Inversion of Control](https://stackoverflow.com/questions/3058/what-is-inversion-of-control)
    - [Spring dependency Injection](https://www.tutorialspoint.com/spring/spring_dependency_injection.htm)
## Spring Bean
- The objects that form the backbone of your application and that are managed by the Spring IoC container are called **beans**
- A bean is an object that is instantiated, assembled, and otherwise managed by a **Spring IoC container**(usually we use **`ApplicationContext` Container**)

### **Bean Definition** 

- definition of bean can be provided by xml or annotation
- can specify attributes and properties for the initialization of the bean
- can specify **callback methods** during the **life cycle** of the **Bean**(`init()`, `destroy()`)
    - [**BeanPostProcessor**](https://www.tutorialspoint.com/spring/spring_bean_post_processors.htm)**:** similar to the callback methods, we can define a sequence of PostProcessor methods that will be invoked after the instantiating, configuration, and initialization a bean
        - can specify the order of post processors being invoked
        - An **ApplicationContext** automatically detects any beans that are defined with the implementation of the **BeanPostProcessor** interface and registers these beans as postprocessors, to be then called appropriately by the container upon bean creation.
- A **child bean definition** inherits configuration data from a parent definition. The child definition can override some values, or add others, as needed.
    - Spring Bean definition inheritance has nothing to do with Java class inheritance but the inheritance concept is same. You can define a parent bean definition as a template and other child beans can inherit the required configuration from the parent bean.

`registerShutdownHook()` method is defined by `ApplicationContext Container` , the method register a shutdown hook, which will ensure a graceful shutdown and call the relevant destroy methods.

- **Bean Scope:** singleton, prototype(create a new bean for each time invoked from container), session

### **Inner Bean**

- **inner beans** are beans that are defined within the scope of another bean.
- Inner beans will be constructed before constructing the outer beans

### **Injecting Collections as bean attributes**

- Spring allows injecting **List**, **Set**, **Map**, and **props**(a collection of name-value pairs where the name and value are both Strings)
- You will come across two situations:
    1. Passing direct values of the collection
    2. Passing a **reference of a bean** as one of the collection elements.

### [**Auto Wiring**](https://www.tutorialspoint.com/spring/spring_beans_autowiring.htm)
[](https://www.tutorialspoint.com/spring/spring_beans_autowiring.htm)

- recall we can declare and inject beans in XML config files, The Spring container can **autowire** relationships **between collaborating beans without using <constructor-arg> and <property> elements,** which **helps cut down on the amount of XML configuration you write for a big Spring-based application**.

  > autowiring can be thought as a mean to let the spring framework search for dependent beans during compile time so the programmer does not need to nest the bean definitions inside one and another

- **Auto Wiring has nothing to do with annotations**. We can, however, use annotations to specify autowiring relationship using `@AutoWired` Annotation

- **types of auto wiring**: `ByName`, `ByType`, `ByConstructor`. 

- **limitation**: once a type of auto wiring is enabled for a bean, all dependent beans of that bean must be injected **in the same way!**

### [**Annotation Based Configuration**](https://www.tutorialspoint.com/spring/spring_annotation_based_configuration.htm) 

- instead of using xml to describe a bean wiring, **we can move the bean configuration into the component class itself by using annotations** on the relevant class, method, or field declaration.
- need to enable `<context:annotation-config/>` in the xml file in order to use Annotation Based Configuration
1. [`@Required`](https://www.tutorialspoint.com/spring/spring_required_annotation.htm)
2. [`@AutoWired`](https://www.tutorialspoint.com/spring/spring_autowired_annotation.htm): [example link](https://examples.javacodegeeks.com/enterprise-java/spring/beans-spring/spring-autowire-example/)
    1. on setters - same as `ByName`
    2. on property - same as `ByType`
    3. on constructor - same as `ByConstructor`
3. `@Qualifier`: There may be a situation when you create **more than one bean of the same type(different bean id but with the same class)** and want to wire only one of them with a property

### [**Java-based configuration**](https://www.tutorialspoint.com/spring/spring_java_based_configuration.htm)
Java-based configuration option enables you to **write most of your Spring configuration without XML** but with the help of few Java-based annotations explained in this chapter.

- Annotating a class with the `@Configuration` indicates that the class can be used by the Spring IoC container as a source of bean definitions

- The `@Bean` annotation tells Spring that a method annotated with `@Bean` will return an object that should be registered as a bean in the Spring application context. 

    ```java
    package com.tutorialspoint;
    import org.springframework.context.annotation.*;
    
    @Configuration
    public class HelloWorldConfig {
       @Bean 
       public HelloWorld helloWorld(){
          return new HelloWorld();
       }
    }
    ```

    > This is equivalent to using xml as following:

    ```xml
    <beans>
       <bean id = "helloWorld" class = "com.tutorialspoint.HelloWorld" />
    </beans>
    ```

- **Injecting dependency using java-based configuration**

    ```java
    package com.tutorialspoint;
    import org.springframework.context.annotation.*;
    
    @Configuration
    public class AppConfig {
       @Bean
       public Foo foo() {	//Foo contains the dependency to Bar object
          return new Foo(bar());
       }
       @Bean
       public Bar bar() {
          return new Bar();
       }
    }
    ```

    

- The **`@Import`** annotation allows for loading @Bean definitions from another configuration class

- Lifecycle Callbacks: The @Bean annotation supports specifying arbitrary initialization and destruction callback methods
  
    ```java
    public class Foo {
       public void init() {
          // initialization logic
       }
       public void cleanup() {
          // destruction logic
       }
    }
    @Configuration
    public class AppConfig {
       @Bean(initMethod = "init", destroyMethod = "cleanup" ) //methods inside Foo.class
       public Foo foo() {
          return new Foo();
       }
    }
    ```
    
- `@Scope("prototype")` can be used to specify the life scope of a Bean 


## [Event Handling in Spring](https://www.tutorialspoint.com/spring/event_handling_in_spring.htm)
- The `ApplicationContext` publishes certain types of events when loading the beans. For example, a `ContextStartedEvent` is published when the context is started and `ContextStoppedEvent` is published when the context is stopped.
- Event handling in the `ApplicationContext` is provided through the `ApplicationEvent` class and `ApplicationListener` interface. Hence, if a bean implements the `ApplicationListener`, then every time an `ApplicationEvent` gets published to the `ApplicationContext`, that bean is notified.
- Spring's event handling is **single-threaded** so if an event is published, until and **unless all the receivers get the message, the processes are blocked and the flow will not continue.** Hence, care should be taken when designing your application if the event handling is to be used.

### Custom Events in Spring

- by defining custom event class (extends `ApplicationEvent`), event handler (implements `ApplicationListener<MyCustomEvent>`, event publisher (implements `ApplicationEventPublisherAware`), we can implement our own flow of event handling in spring

- need to declare publisher and handler as bean in the xml file to let the spring aware of the custom event

## [AOP with Spring Framework](https://www.tutorialspoint.com/spring/aop_with_spring.htm)

- One of the key components of Spring Framework is the **Aspect oriented programming (AOP)** framework. Aspect-Oriented Programming entails breaking down program logic into distinct parts called so-called concerns.

- The functions that span multiple points of an application are called **cross-cutting concerns** and these cross-cutting concerns are conceptually separate from the application's business logic.

  > There are various common good examples of aspects like **logging**, auditing, declarative transactions, security, **caching**, etc

### AOP Terminologies

**Aspect**

- a module which has a set of APIs providing cross-cutting requirements.

**Join point**

- a point in your application where you can plug-in the AOP aspect

**Advice**

- This is an actual piece of code that is invoked during the program execution by Spring AOP framework.
- set of advice includes: `before`, `after`, `after-returning`, `after-throwing`, `around`

**Pointcut**

- This is a set of **one or more join points**(e.g., methods) where an advice should be executed. You can specify pointcuts using expressions or patterns as we will see in our AOP examples.

### AOP Usage

- can be [xml based](https://www.tutorialspoint.com/spring/schema_based_aop_appoach.htm) or [annotation based](https://www.tutorialspoint.com/spring/aspectj_based_aop_appoach.htm)

## [Spring JDBC Framework](https://www.tutorialspoint.com/spring/spring_jdbc_framework.htm)

- Spring JDBC Framework takes care of all the low-level details starting from opening the connection, prepare and execute the SQL statement, process exceptions, handle transactions and finally close the connection.
- [Spring JDBC example](https://www.tutorialspoint.com/spring/spring_jdbc_example.htm)

### DAO

- DAO stands for **Data Access Object**, which is commonly used for database interaction. DAOs exist to provide a means to read and write data to the database and they should expose this functionality through an interface by which the rest of the application will access them.

- The DAO support in Spring makes it easy to work with data access technologies like JDBC, Hibernate, JPA, or JDO in a consistent way.

### JDBC Template Class

- The JDBC Template class executes SQL queries, updates statements, stores procedure calls, performs iteration over `ResultSets`, and extracts returned parameter values.
- It also catches JDBC exceptions and translates them to the generic, more informative, exception hierarchy defined in the `org.springframework.dao` package.

- A common practice when using the `JDBC Template` class is to configure a `DataSource` in your Spring configuration file, and then dependency-inject that shared `DataSource` bean into your `DAO` classes, and the `JdbcTemplate` is created in the setter for the `DataSource`.

  > In the JDBC API, databases are accessed via `DataSource` objects. A `DataSource` **has a set of properties that identify and describe the real world data source that it represents**. These properties include information such as the location of the database server, the name of the database, the network protocol to use to communicate with the server, and so on. In the Application Server, a data source is called a JDBC resource.
  >
  > 
  >
  > Applications access a data source using a connection, and **a `DataSource` object can be thought of as a factory for connections to the particular data source that the `DataSource` instance represents**. In a basic `DataSource` implementation, a call to the `getConnection` method returns a connection object that is a physical connection to the data source. **说人话：DataSource其实就是DB的connection Pool**

- [JDBC Student table usage example](https://www.tutorialspoint.com/spring/spring_jdbc_example.htm), 流程：

  1. 创建`StudentDAO` Interface, 这个接口是由后面的`StudentJDBCTemplate`来实现，define了Student table我们需要用到什么样的operation；`DAO`不是必须的，但是提供这样一个接口提高代码可读性和可改性

     ```java
     package com.tutorialspoint;
     
     import java.util.List;
     import javax.sql.DataSource;
     
     public interface StudentDAO {
        /** 
           * This is the method to be used to initialize
           * database resources ie. connection.
        */
        public void setDataSource(DataSource ds);
        
        /** 
           * This is the method to be used to create
           * a record in the Student table.
        */
        public void create(String name, Integer age);
        
        /** 
           * This is the method to be used to list down
           * a record from the Student table corresponding
           * to a passed student id.
        */
        public Student getStudent(Integer id);
        
        /** 
           * This is the method to be used to list down
           * all the records from the Student table.
        */
        public List<Student> listStudents();
        
        /** 
           * This is the method to be used to delete
           * a record from the Student table corresponding
           * to a passed student id.
        */
        public void delete(Integer id);
        
        /** 
           * This is the method to be used to update
           * a record into the Student table.
        */
        public void update(Integer id, Integer age);
     }
     ```

     

  2. 创建 `Student` class，这个class define了每个student record的column

     ```java
     package com.tutorialspoint;
     
     public class Student {
        private Integer age;
        private String name;
        private Integer id;
     
        public void setAge(Integer age) {
           this.age = age;
        }
        public Integer getAge() {
           return age;
        }
        //...similarly getters and setters for name, id
     }
     ```

  3. 创建`StudentMapper` class, 这个class需要实现`org.springframework.jdbc.core.RowMapper<MyClass>`, `StudentMapper`相当于告诉了spring我们从student table取出row的时候帮我们把student row转换成student object. 我们在之后使用`JDBCTemplate` query的时候需要传入一个`StudentMapper`的instance来接收Student Object:

     ```java
     String SQL = "select * from Student";
     List <Student> students = jdbcTemplateObject.query(SQL, new StudentMapper());
     ```

     >  StudentMapper的实现：

     ```java
     package com.tutorialspoint;
     
     import java.sql.ResultSet;
     import java.sql.SQLException;
     import org.springframework.jdbc.core.RowMapper;
     
     public class StudentMapper implements RowMapper<Student> {
        public Student mapRow(ResultSet rs, int rowNum) throws SQLException {
           Student student = new Student();
           student.setId(rs.getInt("id"));
           student.setName(rs.getString("name"));
           student.setAge(rs.getInt("age"));
           
           return student;
        }
     }
     ```

     

  4. 这一步和第一步一样是可选的，我们实现一个自己的`StudentJDBCTemplate.java` , 相当于对Spring已提供的`JDBCTemplate`进行二次开发，把我们会对Student Table的query api都encapsulate到`StudentJDBCTemplate`类中, `StudentJDBCTemplate` 实现了`StudentDao`定义的`api`, 由此可见我们几个class在一起一环套一环地把逻辑从上到下的实现了出来，提高了代码的可读性和可改性

     ```java
     package com.tutorialspoint;
     
     import java.util.List;
     import javax.sql.DataSource;
     import org.springframework.jdbc.core.JdbcTemplate;
     
     public class StudentJDBCTemplate implements StudentDAO {
        private DataSource dataSource;
        private JdbcTemplate jdbcTemplateObject;
        
        public void setDataSource(DataSource dataSource) {
           this.dataSource = dataSource;
           this.jdbcTemplateObject = new JdbcTemplate(dataSource);
        }
        public void create(String name, Integer age) {
           String SQL = "insert into Student (name, age) values (?, ?)";
           jdbcTemplateObject.update( SQL, name, age);
           System.out.println("Created Record Name = " + name + " Age = " + age);
           return;
        }
        public Student getStudent(Integer id) {
           String SQL = "select * from Student where id = ?";
           Student student = jdbcTemplateObject.queryForObject(SQL, 
              new Object[]{id}, new StudentMapper());
           
           return student;
        }
        public List<Student> listStudents() {
           String SQL = "select * from Student";
           List <Student> students = jdbcTemplateObject.query(SQL, new StudentMapper());
           return students;
        }
        public void delete(Integer id) {
           String SQL = "delete from Student where id = ?";
           jdbcTemplateObject.update(SQL, id);
           System.out.println("Deleted Record with ID = " + id );
           return;
        }
        public void update(Integer id, Integer age){
           String SQL = "update Student set age = ? where id = ?";
           jdbcTemplateObject.update(SQL, age, id);
           System.out.println("Updated Record with ID = " + id );
           return;
        }
     }
     ```

  5. 定义xml文件，我们可以把`StudentJDBCTemplate.class`当作一个Bean，因为这个class依赖于`DataSource`, 可以让Spring来管理`DataSource`并将其传给我们的`StudentJDBCTemplate`, **注意`private JdbcTemplate jdbcTemplateObject`也是`StudentJDBCTemplate`的依赖类，但由于创建这个依赖不需要任何多余的配置信息(创建`DataSource`需要database url, port, username ... etc), 所以我们不交给spring来管理**

     ```xml
     <?xml version = "1.0" encoding = "UTF-8"?>
     <beans xmlns = "http://www.springframework.org/schema/beans"
        xmlns:xsi = "http://www.w3.org/2001/XMLSchema-instance" 
        xsi:schemaLocation = "http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-3.0.xsd ">
     
        <!-- Initialization for data source -->
        <bean id="dataSource" 
           class = "org.springframework.jdbc.datasource.DriverManagerDataSource">
           <property name = "driverClassName" value = "com.mysql.jdbc.Driver"/>
           <property name = "url" value = "jdbc:mysql://localhost:3306/TEST"/>
           <property name = "username" value = "root"/>
           <property name = "password" value = "password"/>
        </bean>
     
        <!-- Definition for studentJDBCTemplate bean -->
        <bean id = "studentJDBCTemplate" 
           class = "com.tutorialspoint.StudentJDBCTemplate">
           <property name = "dataSource" ref = "dataSource" />    
        </bean>
           
     </beans>
     ```

     

  6. 使用案例

     ```java
     package com.tutorialspoint;
     
     import java.util.List;
     
     import org.springframework.context.ApplicationContext;
     import org.springframework.context.support.ClassPathXmlApplicationContext;
     import com.tutorialspoint.StudentJDBCTemplate;
     
     public class MainApp {
        public static void main(String[] args) {
           ApplicationContext context = new ClassPathXmlApplicationContext("Beans.xml");
     
           StudentJDBCTemplate studentJDBCTemplate = 
              (StudentJDBCTemplate)context.getBean("studentJDBCTemplate");
           
           System.out.println("------Records Creation--------" );
           studentJDBCTemplate.create("Zara", 11);
           studentJDBCTemplate.create("Nuha", 2);
           studentJDBCTemplate.create("Ayan", 15);
     
           System.out.println("------Listing Multiple Records--------" );
           List<Student> students = studentJDBCTemplate.listStudents();
           
           for (Student record : students) {
              System.out.print("ID : " + record.getId() );
              System.out.print(", Name : " + record.getName() );
              System.out.println(", Age : " + record.getAge());
           }
     
           System.out.println("----Updating Record with ID = 2 -----" );
           studentJDBCTemplate.update(2, 20);
     
           System.out.println("----Listing Record with ID = 2 -----" );
           Student student = studentJDBCTemplate.getStudent(2);
           System.out.print("ID : " + student.getId() );
           System.out.print(", Name : " + student.getName() );
           System.out.println(", Age : " + student.getAge());
        }
     }
     ```

     

## [Spring MVC](https://www.tutorialspoint.com/spring/spring_web_mvc_framework.htm)





# [Java Web Application](https://docs.oracle.com/javaee/5/tutorial/doc/geysj.html)

- **web components** provide the dynamic extension capabilities for a web server. Web components are either Java servlets, JSP pages, or web service endpoints.

  ![image-20201022225311486](C:\Users\13354\projects\notes\rsrc\spring_java_web_comp)

- Java Web Application dev cycle

  1. Develop the web component code.

  2. Develop the web application **deployment descriptor**. (`web.xml`)

  3. Compile the web application components and helper classes referenced by the components.

  4. Optionally package the application into a deployable unit.

     > A web module can be deployed as an unpacked file structure or can be packaged in a JAR file known as a web archive (**WAR**) file

  5. Deploy the application into a web container.

     > e.g., Tomcat, GlassFish, Jetty

  6. Access a URL that references the web application



## Servlet

- Servlets are the **Java programs** that runs on the Java-enabled web server or application server. They are used to handle the request obtained from the web server, process the request, produce the response, then send response back to the web server.

Execution of Servlets involves six basic steps:

1. The clients send the request to the web server.
2. The web server receives the request.
3. The web server passes the request to the corresponding servlet.
4. The servlet processes the request and generates the response in the form of output.
5. The servlet sends the response back to the web server.
6. The web server sends the response back to the client and the client browser displays it on the screen.

### JSP(Java Server Pages)

- **JSP pages** are text-based documents that execute as servlets but allow a more natural approach to creating static content. (**think it as dynamic html**)

### [Servlet lifecycle](https://docs.oracle.com/javaee/5/tutorial/doc/bnafi.html)

The life cycle of a servlet is controlled by the container in which the servlet has been deployed. When a request is mapped to a servlet, the container performs the following steps.

1. If an instance of the servlet does not exist, the web container
   1. Loads the servlet class.
   2. Creates an instance of the servlet class.
   3. Initializes the servlet instance by calling the `init` method. Initialization is covered in [Initializing a Servlet](https://docs.oracle.com/javaee/5/tutorial/doc/bnafu.html).
2. Invokes the `service` method, passing request and response objects. Service methods are discussed in [Writing Service Methods](https://docs.oracle.com/javaee/5/tutorial/doc/bnafv.html).

If the container needs to remove the servlet, it finalizes the servlet by calling the servlet’s `destroy` method. Finalization is discussed in [Finalizing a Servlet](https://docs.oracle.com/javaee/5/tutorial/doc/bnags.html).

#### Handling Servlet Life-Cycle Events(event Listener)

You can monitor and react to events in a servlet’s life cycle by defining **listener objects whose methods get invoked when life-cycle events occur**. To use these listener objects you must define and specify the listener class.

- You define a listener class as an implementation of a listener interface.
- When a listener method is invoked, it is passed an event that contains information appropriate to the event.  For example, the methods in the `HttpSessionListener` interface are passed an `HttpSessionEvent`, which contains an `HttpSession`.

#### Specifying Event Listener Classes

- You specify an event listener class using the `listener` element of the deployment descriptor.

#### Invoke Other Web Resouces/Component

To invoke a resource available on the server(e.g., other web components) that is running a web component, you must first obtain a `RequestDispatcher` object using the `getRequestDispatcher("URL")` method.

You can get a `RequestDispatcher` object from either a request or the web context; however, the two methods have slightly different behavior. The method takes the path to the requested resource as an argument. A request can take a relative path (that is, one that does not begin with a `/`), but the web context requires an absolute path. If the resource is not available or if the server has not implemented a `RequestDispatcher` object for that type of resource, `getRequestDispatcher` will return null. Your servlet should be prepared to deal with this condition.

#### Session

- Many applications require that a series of requests from a client be associated with one another.
- Web-based applications are responsible for maintaining such state, called a **session**, because HTTP is stateless. To support applications that need to maintain state, Java Servlet technology provides an API for managing sessions and allows several mechanisms for implementing sessions.
- Usually a session between client and server is identified by identifier(cookie, or appended identifier in url)
- a session has timeout, and when session ends we can persist the session state info into db 

##### Accessing a Session

Sessions are represented by an `HttpSession` object. You access a session by calling the `getSession` method of a request object. This method returns the current session associated with this request, or, **if the request does not have a session, it creates one**(Usually we can persist the information of a session into database, so next time client comes in we create a new session but using the persisted state in database).

##### Notifying Objects That Are Associated with a Session

Recall that your application can notify web context and session listener objects of servlet life-cycle events ([Handling Servlet Life-Cycle Events](https://docs.oracle.com/javaee/5/tutorial/doc/bnafi.html#bnafj)). You can also notify objects of certain events related to their association with a session such as the following:

- When the object is added to or removed from a session. To receive this notification, your object must implement the `javax.servlet.http.HttpSessionBindingListener` interface.
- When the session to which the object is attached will be passivated or activated. A session will be passivated or activated when it is moved between virtual machines or saved to and restored from persistent storage. To receive this notification, your object must implement the `javax.servlet.http.HttpSessionActivationListener` interface.

## Servlet Container

- **Servlet container**, also known as **Servlet engine** is an integrated set of objects that provide run time environment for Java Servlet components. In simple words, it is a system that manages Java Servlet components on top of the Web server to handle the Web client requests.

  > GlassFish, Tomcat, Jetty

  