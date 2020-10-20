---
title: "Running Spring Batch on Multiple Databases in Parallel"
date: 2017-06-21
categories: ["Java"]
tags: ["Java", "Spring"]
---

## The Question
I've created a Spring batch application using Spring boot, and I have a `Job` with 9 steps. These steps are using a `DataSource` which I created its bean in a configuration file as follows:

    @Configuration
    public class DatabaseConfig {
        @ConfigurationProperties(prefix = "spring.datasource")
        @Bean
        @Primary
        public DataSource dataSource(){
            return DataSourceBuilder.create().build();
        }
    }

This `DataSource` is using properties declared in the `application.yml` file:

    spring:
      datasource:
        url: jdbc:mysql://localhost:3306/db_01?zeroDateTimeBehavior=convertToNull
        username: xxxx
        password: ****

So far, all works as expected.

What I want to do, is that I have 4 databases parameterized in a 5th database (db_settings), which I select using an SQL query. This query will return the 4 databases with their usernames and passwords as follows:
```
+--------+-----------------------------------+-----------------+-----------------+
| id     | url                               | username_db     | password_db     |
+--------+-----------------------------------+-----------------+-----------------+
|    243 | jdbc:mysql://localhost:3306/db_01 | xxxx            | ****            |
|    244 | jdbc:mysql://localhost:3306/db_02 | xxxx            | ****            |
|    245 | jdbc:mysql://localhost:3306/db_03 | xxxx            | ****            |
|    247 | jdbc:mysql://localhost:3306/db_04 | xxxx            | ****            |
+--------+-----------------------------------+-----------------+-----------------+
```

So instead of running the steps using the database declared in 'application.yml', I want to run them on all the 4 databases. And considering the volume processed, it is necessary to be able to launch the batch processing on these databases in parallel.

Does anyone know how to implement this?

----

## The Answer
Thanks KeatsPeeks, `AbstractRoutingDataSource` is a good starter for the solution, and here is a [good tutorial](http://howtodoinjava.com/spring/spring-orm/spring-3-2-5-abstractroutingdatasource-example/) on this part.

Mainly the important parts are:

1.  define the lookup code
```
public class MyRoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        String language = LocaleContextHolder.getLocale().getLanguage();
        System.out.println("Language obtained: "+ language);
        return language;
    }
}
```

2.  register the multiple datasources
```
<bean id="abstractDataSource" class="org.apache.commons.dbcp.BasicDataSource"
    destroy-method="close"
    p:driverClassName="${jdbc.driverClassName}"
    p:username="${jdbc.username}"
    p:password="${jdbc.password}" />

<bean id="concreteDataSourceOne"
    parent="abstractDataSource"
    p:url="${jdbc.databaseurlOne}"/>

<bean id="concreteDataSourceTwo"
    parent="abstractDataSource"
    p:url="${jdbc.databaseurlTwo}"/>
```
So after that, the problem is become to:

1.  How to load datasource config properties when spring startup and config the corresponding `dataSource` using the config properties in database.

2.  How to use multiple `dataSource` in spring batch

    Actually when I try to google it, seems this is a most common case, google give the suggestion search words - "spring batch multiple data sources", there are a lots articles, so I choose the answer in

3.  How to define the lookup code based on the spring batch jobs(steps)

    Typically this should be a business point, You need define the lookup strategy and can be injected to the `com.example.demo.datasource.CustomRoutingDataSource#determineCurrentLookupKey` to routing to the dedicated data source.

## Limitation
The really interesting is actually it is supports the multiple `dataSource`, but the db settings cannot store in the DB indeed. The reason is it will get the cycle dependencies issue:

```
The dependencies of some of the beans in the application context form a cycle:

    batchConfiguration (field private org.springframework.batch.core.configuration.annotation.JobBuilderFactory com.example.demo.batch.BatchConfiguration.jobs)
        ↓
    org.springframework.batch.core.configuration.annotation.SimpleBatchConfiguration (field private java.util.Collection org.springframework.batch.core.configuration.annotation.AbstractBatchConfiguration.dataSources)
┌─────┐
|  routingDataSource defined in class path resource [com/example/demo/datasource/DataSourceConfiguration.class]
↑     ↓
|  targetDataSources defined in class path resource [com/example/demo/datasource/DataSourceConfiguration.class]
↑     ↓
|  myBatchConfigurer (field private java.util.Collection org.springframework.batch.core.configuration.annotation.AbstractBatchConfiguration.dataSources)
└─────┘
```

So obviously the solution is break the dependency between `dataSource` and `routingDataSource`, and the workaround should be:
* Save the DB setting in properties
* Or involve other approach but not in the primary `dataSource`

## See Also
- [spring-data-multiple-databases](https://scattercode.co.uk/2013/11/18/spring-data-multiple-databases/) 
- [hello-world-with-spring-batch-3-0-x-with-pure-annotations](https://numberformat.wordpress.com/2013/12/27/hello-world-with-spring-batch-3-0-x-with-pure-annotations/)
- [batch-processing](http://spring.io/guides/gs/batch-processing/)
- [How to java-configure separate datasources for spring batch data and business data? Should I even do it?](https://stackoverflow.com/questions/25256487/how-to-java-configure-separate-datasources-for-spring-batch-data-and-business-da)