Spring REST Exception handler
=============================
[![Build Status](https://travis-ci.org/jirutka/spring-rest-exception-handler.svg)](https://travis-ci.org/jirutka/spring-rest-exception-handler)
[![Coverage Status](https://img.shields.io/coveralls/jirutka/spring-rest-exception-handler.svg)](https://coveralls.io/r/jirutka/spring-rest-exception-handler)
[![Maven Central](https://maven-badges.herokuapp.com/maven-central/cz.jirutka.spring/spring-rest-exception-handler/badge.svg)](https://maven-badges.herokuapp.com/maven-central/cz.jirutka.spring/spring-rest-exception-handler)

TODO


Error message
-------------

Error messages generated by `ErrorMessageRestExceptionHandler` meets the [Problem Details for HTTP APIs][http-problem]
specification.

For an example, the following error message that describes validation exception.

**In JSON format:**

```json
{
    "type": "http://example.org/errors/validation-failed",
    "title": "Validation Failed",
    "status": 422,
    "detail": "The content you've send contains 2 validation errors.",
    "errors": [
        {
            "field": "title",
            "message": "must not be empty"
        },
        {
            "field": "quantity",
            "rejected": -5,
            "message": "must be greater than zero"
        }
    ]
}
```

**… or in XML:**

```xml
<problem>
    <type>http://example.org/errors/validation-failed</type>
    <title>Validation Failed</title>
    <status>422</status>
    <detail>The content you've send contains 2 validation errors.</detail>
    <errors>
        <error>
            <field>title</field>
            <message>must not be empty</message>
        </error>
        <error>
            <field>quantity</field>
            <rejected>-5</rejected>
            <message>must be greater than zero</message>
        </error>
    </errors>
</problem>

```


### Localizable attributes

Message values are read from a _properties_ file through the provided [MessageSource], so it can be simply customized
and localized. Library contains a default [messages.properties] file that is implicitly set as a parent (i.e. fallback)
of the provided message source. This can be disabled by setting `withDefaultMessageSource` to false (on a builder or
factory bean).

The key name is prefixed with a fully qualified class name of the Java exception, or `default` for the default value;
this is used when no value for a particular exception class exists (even in the parent message source).

Value is a message templates that may contain [SpEL] expressions delimited by `#{` and `}`. Inside an expression, you
can access the exception being handled and the current request (instance of [HttpServletRequest]) under the `ex`, resp.
`req` variables.

**For an example:**

```properties
org.springframework.web.HttpMediaTypeNotAcceptableException.type=http://httpstatus.es/406
org.springframework.web.HttpMediaTypeNotAcceptableException.title=Not Acceptable
org.springframework.web.HttpMediaTypeNotAcceptableException.detail=\
    This resource provides only #{ex.supportedMediaTypes}, but you've sent Accept #{req.getHeaderValues('Accept')}.
```


Configuration
-------------

### Java-based configuration

```java
@EnableWebMvc
@Configuration
public class RestContextConfig extends WebMvcConfigurerAdapter {

    @Override
    public void configureHandlerExceptionResolvers(List<HandlerExceptionResolver> resolvers) {
        resolvers.add( exceptionHandlerExceptionResolver() ); // resolves @ExceptionHandler
        resolvers.add( restExceptionResolver() );
    }

    @Bean
    public RestHandlerExceptionResolver restExceptionResolver() {
        return RestHandlerExceptionResolver.builder()
                .messageSource( httpErrorMessageSource() )
                .defaultContentType(MediaType.APPLICATION_JSON)
                .addErrorMessageHandler(EmptyResultDataAccessException.class, HttpStatus.NOT_FOUND)
                .addHandler(MyException.class, new MyExceptionHandler())
                .build();
    }

    @Bean
    public MessageSource httpErrorMessageSource() {
        ReloadableResourceBundleMessageSource m = new ReloadableResourceBundleMessageSource();
        m.setBasename("classpath:/org/example/messages");
        m.setDefaultEncoding("UTF-8");
        return m;
    }

    @Bean
    public ExceptionHandlerExceptionResolver exceptionHandlerExceptionResolver() {
        ExceptionHandlerExceptionResolver resolver = new ExceptionHandlerExceptionResolver();
        resolver.setMessageConverters(HttpMessageConverterUtils.getDefaultHttpMessageConverters());
        return resolver;
    }
}
```

### XML-based configuration

```xml
<bean id="compositeExceptionResolver"
      class="org.springframework.web.servlet.handler.HandlerExceptionResolverComposite">
    <property name="order" value="0" />
    <property name="exceptionResolvers">
        <list>
            <ref bean="exceptionHandlerExceptionResolver" />
            <ref bean="restExceptionResolver" />
        </list>
    </property>
</bean>

<bean id="restExceptionResolver"
      class="cz.jirutka.spring.web.servlet.exhandler.RestHandlerExceptionResolverFactoryBean">
    <property name="messageSource" ref="httpErrorMessageSource" />
    <property name="defaultContentType" value="application/json" />
    <property name="exceptionHandlers">
        <map>
            <entry key="org.springframework.dao.EmptyResultDataAccessException" value="404" />
            <entry key="org.example.MyException">
                <bean class="org.example.MyExceptionHandler" />
            </entry>
        </map>
    </property>
</bean>

<bean id="exceptionHandlerExceptionResolver"
      class="org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver" />

<bean id="httpErrorMessageSource"
      class="org.springframework.context.support.ReloadableResourceBundleMessageSource"
      p:basename="classpath:/org/example/errorMessages"
      p:defaultEncoding="UTF-8" />
```

### Notes

The ExceptionHandlerExceptionResolver is used to resolve exceptions through [@ExceptionHandler] methods. It must be
registered _before_ the RestHandlerExceptionResolver. If you don’t have any @ExceptionHandler methods, then you can
omit the `exceptionHandlerExceptionResolver` bean declaration.

Builder and FactoryBean registers set of the default handlers by default. This can be disabled by setting
`withDefaultHandlers` to false.


### Why is 404 bypassing exception handler?

When the [DispatcherServlet] is unable to determine a corresponding handler for an incoming HTTP request, it sends 404
directly without bothering to call an exception handler (see
[on StackOverflow](http://stackoverflow.com/a/22751886/2217862)). This behaviour can be changed, since
Spring version 4.0.0, using `throwExceptionIfNoHandlerFound` init parameter. You should set this to true for a
consistent error responses.


**When using WebApplicationInitializer:**

```java
public class AppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    protected void customizeRegistration(ServletRegistration.Dynamic reg) {
        reg.setInitParameter("throwExceptionIfNoHandlerFound", "true");
    }
    ...
}
```

**…or classic web.xml:**

```xml
<servlet>
    <servlet-name>rest-dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>throwExceptionIfNoHandlerFound</param-name>
        <param-value>true</param-value>
    </init-param>
    ...
</servlet>
```


Maven
-----

Released versions are available in The Central Repository. Just add this artifact to your project:

```xml
<dependency>
    <groupId>cz.jirutka.spring</groupId>
    <artifactId>spring-rest-exception-handler</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>
```

However if you want to use the last snapshot version, you have to add the Sonatype OSS repository:

```xml
<repository>
    <id>sonatype-snapshots</id>
    <name>Sonatype repository for deploying snapshots</name>
    <url>https://oss.sonatype.org/content/repositories/snapshots</url>
    <snapshots>
        <enabled>true</enabled>
    </snapshots>
</repository>
```


License
-------

This project is licensed under [Apache License 2.0](http://www.apache.org/licenses/LICENSE-2.0.html).


[http-problem]: http://tools.ietf.org/html/draft-nottingham-http-problem-06
[@ExceptionHandler]: http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/bind/annotation/ExceptionHandler.html
[DispatcherServlet]: http://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/DispatcherServlet.html
[MessageSource]: http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/context/MessageSource.html
[SpEL]: http://docs.spring.io/spring/docs/current/spring-framework-reference/html/expressions.html
[HttpServletRequest]: http://docs.oracle.com/javaee/7/api/javax/servlet/http/HttpServletRequest.html

[messages.properties]: src/main/resources/cz/jirutka/spring/web/servlet/exhandler/messages.properties
