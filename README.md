# How to Upgrade to Spring 5.3.18 with Grails 4

Because of [Spring Framework RCE](https://spring.io/blog/2022/03/31/spring-framework-rce-early-announcement), many Grails and Spring apps are impacted. Recently [Grails 5.1.6 was released and update to Spring 5.3.18](https://grails.org/blog/2022-03-31-grails-spring-rce.html), but unfortunately, Grails 4 since last release 4.0.13 still use Spring Boot 2.1.x and Spring Framework 5.1.x, which are all [End of Support](https://spring.io/projects/spring-boot#support), [This demo](https://github.com/rainboyan/grails4-upgrade-spring-demo) show you how to Upgrade Spring to 5.3.18, I hope this demo will help you, but you should **Notice** that, Spring Framework 5.3.x changed lot and may not work with old Grails 4 plugins, please give a hard test and make sure it works.

When upgraded Spring 5.3.18, there is a error came from `groovyPagesTemplateEngine` of `grails-gsp`, because of [Spring 5.3.18 issue#28261, Restrict access to property paths on Class references](https://github.com/spring-projects/spring-framework/issues/28261), 

```
Caused by: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'groovyPagesTemplateEngine': Error setting property values; nested exception is org.springframework.beans.NotWritablePropertyException: Invalid property 'classLoader' of bean class [org.grails.gsp.GroovyPagesTemplateEngine]: Bean property 'classLoader' is not writable or has an invalid setter method. Does the parameter type of the setter match the return type of the getter?
```

Like the workaround for Grails 5.1.6, it also works for Grails 4, we could add the `GrailsIssue12460PostProcessor` to remove `classLoader` from the propertyValues of bean `groovyPagesTemplateEngine` at the runtime,

`grails/conf/spring/resources.groovy`:
```groovy
beans = {
    grailsIssue12460PostProcessor(GrailsIssue12460PostProcessor)
}
```

`sr/main/groovy/org/grails/demo/GrailsIssue12460PostProcessor.groovy`:
```groovy
class GrailsIssue12460PostProcessor implements MergedBeanDefinitionPostProcessor {
    @Override
    void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
        if (beanName == 'groovyPagesTemplateEngine') {
            // workaround for Grails-gsp with Spring 5.3.18
            beanDefinition.getPropertyValues().removePropertyValue("classLoader")
        }
    }
}
```

## Upgrade Tomcat to 9.0.62

Since Apache Tomcat 9.0.62 was released yet, and it's also fixed the class loader issue, so we should update it.  

```
Effectively disable the WebappClassLoaderBase.getResources() method as it is not used and if something accidently exposes the class loader this method can be used to gain access to Tomcat internals.
```

In `gradle.properties`, add `tomcat.version`:

```
tomcat.version=9.0.62
```

Then check the tomcat version, `./gradlew dependencies --configuration runtimeClasspath | grep org.apache.tomcat`

```
|    +--- org.apache.tomcat.embed:tomcat-embed-core:9.0.39 -> 9.0.62
|    |    \--- org.apache.tomcat:tomcat-annotations-api:9.0.62
|    +--- org.apache.tomcat.embed:tomcat-embed-el:9.0.39 -> 9.0.62
|    \--- org.apache.tomcat.embed:tomcat-embed-websocket:9.0.39 -> 9.0.62
|         \--- org.apache.tomcat.embed:tomcat-embed-core:9.0.62 (*)
|    |    +--- org.apache.tomcat.embed:tomcat-embed-logging-log4j:8.5.2
+--- org.apache.tomcat:tomcat-jdbc -> 9.0.62
|    \--- org.apache.tomcat:tomcat-juli:9.0.62
```

When `grails run-app`, you will see that `Starting Servlet engine: [Apache Tomcat/9.0.62]`, it works!

```
2022-04-03 14:45:11.001  INFO --- [  restartedMain] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2022-04-03 14:45:11.013  INFO --- [  restartedMain] o.a.coyote.http11.Http11NioProtocol      : Initializing ProtocolHandler ["http-nio-8080"]
2022-04-03 14:45:11.019  INFO --- [  restartedMain] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2022-04-03 14:45:11.019  INFO --- [  restartedMain] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.62]
2022-04-03 14:45:11.064  INFO --- [  restartedMain] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2022-04-03 14:45:11.064  INFO --- [  restartedMain] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 1572 ms
2022-04-03 14:45:11.165  INFO --- [  restartedMain] o.s.b.d.a.OptionalLiveReloadServer       : LiveReload server is running on port 35729
2022-04-03 14:45:11.341  INFO --- [  restartedMain] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 0 endpoint(s) beneath base path '/actuator'
====================================================================================================
Hacking with Grails : https://github.com/grails/grails-core/issues/12460
====================================================================================================
2022-04-03 14:45:11.740  INFO --- [  restartedMain] o.a.coyote.http11.Http11NioProtocol      : Starting ProtocolHandler ["http-nio-8080"]
2022-04-03 14:45:11.753  INFO --- [  restartedMain] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring GrailsDispatcherServlet 'grailsDispatcherServlet'
2022-04-03 14:45:11.753  INFO --- [  restartedMain] o.g.w.s.mvc.GrailsDispatcherServlet      : Initializing Servlet 'grailsDispatcherServlet'
2022-04-03 14:45:11.754  INFO --- [  restartedMain] o.g.w.s.mvc.GrailsDispatcherServlet      : Completed initialization in 1 ms
2022-04-03 14:45:11.755  INFO --- [  restartedMain] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2022-04-03 14:45:11.757  INFO --- [  restartedMain] org.grails.demo.Application              : Started Application in 2.655524708 seconds (JVM running for 3.261)
```

## What about Grails 5?

Please check this [Grails 5.1.6 demo](https://github.com/rainboyan/grails-issue-12460-demo)

## Links
- [Spring Framework RCE](https://spring.io/blog/2022/03/31/spring-framework-rce-early-announcement)
- [Spring Framework RCE, Mitigation Alternative](https://spring.io/blog/2022/04/01/spring-framework-rce-mitigation-alternative)
- [Spring 5.3.18 issue#28261, Restrict access to property paths on Class references](https://github.com/spring-projects/spring-framework/issues/28261)
- [Apache Tomcat 9.0.62 released](https://tomcat.apache.org/tomcat-9.0-doc/changelog.html#Tomcat_9.0.62_(remm))
- [Grails 5.1.6: integration tests, bootRun, bootWar are broken](https://github.com/grails/grails-core/issues/12460)
- [Grails GSP - Fixed Grails issue #12460](https://github.com/grails/grails-gsp/pull/257)
- [Grails 4 upgrade Spring 5.3.18 demo](https://github.com/rainboyan/grails4-upgrade-spring-demo)
- [Will Dormann twitter status](https://twitter.com/wdormann/status/1509372145394200579)