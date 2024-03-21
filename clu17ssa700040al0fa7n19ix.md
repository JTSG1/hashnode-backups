---
title: "Including selective dependencies in maven project"
seoTitle: "using maven profiles to deploy to different environments"
seoDescription: "Discover how to effortlessly transition between local and GCP environments for application testing with our comprehensive guide on utilizing Maven profiles."
datePublished: Thu Mar 21 2024 12:34:42 GMT+0000 (Coordinated Universal Time)
cuid: clu17ssa700040al0fa7n19ix
slug: including-selective-dependencies-in-maven-project
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1711024476780/53c59aa2-cc1a-49e5-bd0c-dbf028c8eba4.webp
tags: java, maven, testing, gcp, profiles

---

I have been involved with writing applications for the public cloud recently, specifically GCP. There are a lot of helper libraries out there that are very useful for getting your application running in the cloud with less effort.

The problems I have encountered arise when I want to test these applications locally, certain dependencies when included change how for instance I connect to a database through the introduction of properties such as 

``` properties
spring.cloud.gcp.sql.database-name
spring.cloud.gcp.sql.instance-connection-name
spring.cloud.gcp.sql.enable-iam-auth
```
Merely including the dependency of:

``` xml
<dependency>
   <dependency>
      <groupId>com.google.cloud</groupId>
      <artifactId>spring-cloud-gcp-starter-sql-postgresql</artifactId>
      <version>5.0.4</version>
   </dependency>
</dependency>
```
Makes the aforementioned properties required. Which is great and not a problem when my target is GCP. What if I want to test my code locally with a local instance of PostgreSQL?
### Using profiles ###
We can make use of maven profiles to selectively include that dependency and also to ensure we choose an appropriate application.properties file for local running.

Initially I was looking at excluding a certain dependency for a certain profiles, but the answer appears to be to do the opposite.

It's fairly straight forward to set up in your pom.xml file

``` xml

...
<!-- remove the dependencies from your higher level <dependencies> -->
<profiles>
    <profile>
        <id>gcp</id>
        <properties>
            <activatedProperties>gcp</activatedProperties>
        </properties>
        <dependencies>
            <dependency>
                <groupId>com.google.cloud</groupId>
                <artifactId>spring-cloud-gcp-starter-sql-postgresql</artifactId>
                <version>5.0.4</version>
            </dependency
        </dependencies>
    </profile>
    <profile>
        <id>local</id>
        <properties>
            <activatedProperties>local</activatedProperties>
        </properties>
    </profile>
</profiles>
...
```

In my src/main/resource folder I have three files defined:
- application.properties
- application-gcp.properties
- application-local.properties

application.properties just contains a single line

```
spring.profiles.active=@activatedProperties@
```
For me the main difference is the properties that are used to connect to the DB

For GCP I am needing to provide the following properties in my application-gcp.properties file:

``` properties
spring.cloud.gcp.sql.databanse-name=${DB_NAME}
spring.cloud.gcp.sql.instance-connection-name=${GCP_CONNECTION_NAME}
spring.cloud.gcp.sql.enable-iam-auth=true
```

For my application-local.properties file I am using more direct properties:

``` properties
spring.datasource.url=${DB_HOST}
```

To select each profile, you can run a command such as 

``` bash
mvn package -P local
```

This was a useful way of leveraging maven profiles for me to be able to test my code locally and in GCP.  

### Conclusion ###
This is a helpful mechanism to use when you are wanting to be able to test your applications locally but they are destined to be deployed in the cloud. Being able to test changes locally is important and can help avoid wasting time later by repeatedly needing to deploy your application code to the remote environment every time an issue is encountered.


