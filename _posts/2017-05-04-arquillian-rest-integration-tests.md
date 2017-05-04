---
layout: post
title:  Arquillian - Rest Integration Tests
tags: javaee testing
---

How to create fast and lean integration tests has been a concern for me for some time.

I found that using **Arquillian** with remote containers is the fastest way for development phase, but for continuous integration purposes sometimes we have to use an embedded container. That does not mean it cannot be fast regardless.

Using an embedded _Tomee_ container is very simple and with these simple guidelines you will be up and running in no time.

***

### Maven Dependencies

For Arquillian tests with Tomee container you need the following _Maven_ dependencies:

```xml
    
    <!-- project coordinates omitted-->
    
    <!-- standard properties -->
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
        <failOnMissingWebXml>false</failOnMissingWebXml>
    </properties>

    <dependencies>

        <!-- javaee -->
        <dependency>
            <groupId>javax</groupId>
            <artifactId>javaee-api</artifactId>
            <version>7.0</version>
            <scope>provided</scope>
        </dependency>

        <!-- Unit testing -->
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <version>2.5.0</version>
            <scope>test</scope>
        </dependency>

        <!-- Arquillian -->
        <dependency>
            <groupId>org.apache.tomee</groupId>
            <artifactId>arquillian-tomee-embedded</artifactId>
            <version>7.0.1</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.tomee</groupId>
            <artifactId>apache-tomee</artifactId>
            <version>7.0.1</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.glassfish.web</groupId>
            <artifactId>el-impl</artifactId>
            <version>2.2</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
```

Next we create an arquillian configuration file for tomee container to avoid port conficts.

***

### Arquillian Configuration

```xml

    <arquillian xmlns="http://jboss.org/schema/arquillian"
                xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                xsi:schemaLocation="
            http://jboss.org/schema/arquillian
            http://jboss.org/schema/arquillian/arquillian_1_0.xsd">
        <container qualifier="tomee" default="true">
            <configuration>
                <property name="httpPort">-1</property>
                <property name="stopPort">-1</property>
            </configuration>
        </container>
    </arquillian>

```

Save this file as **arquillian.xml** in **src/test/resources** directory.

... and finally we create the test class.

****

### Test Class

```java

@RunWith(Arquillian.class)
public class HelloResourceTest {
    
    @Deployment(testable = false)
    public static WebArchive createDeployment() {
        return ShrinkWrap.create(WebArchive.class)
                .addClasses(RestConfiguration.class, HelloResource.class)
                .addAsWebInfResource(EmptyAsset.INSTANCE, "beans.xml");
    }

    @Test
    public void shouldSayHello(@ArquillianResource URL url) throws URISyntaxException {

        WebClient webClient = WebClient.create(url.toExternalForm()).path("rest/hello");
        JsonObject message = webClient.get(JsonObject.class);

        assertThat(message).containsKey("message");
    }
}

``` 

Very straightforward. We create the deployment package. use `testable=false` to emulate sandbox testing and verify the result.
The HelloResource returns a simple JSON message:

```json
{
   "message": "Hello World"
}
```

That's it. Powerful Integration testing with minimum overhead.

****



