---
layout: post
title:  "How to: Integrate Google Cloud Endpoints into the Spring Container"
date:   2015-12-26 18:53:00 +0800
description: "When using Google App Engine, you have the a new REST framework available at your disposal - Google Cloud Endpoints. Unfortunately, it's not meant for use with Spring Beans. This article shows you how to tweak Google Cloud Endpoints and make them instantiate as Spring Beans."
tags: java spring google-app-engine google-cloud-endpoints
---

Before we go through this article, I just need to make one thing clear. If you have the freedom to choose other REST frameworks such as Jersey or Spring REST, I urge you to use them instead.
Google Cloud endpoints is okay, but it's not as flexible as those that I've mentioned. I'll list down some of the limitations I've encountered while using Google Cloud Endpoints:

* **You're pretty much stuck with its authentication mechanism.** - Cloud Endpoints rely on Google's OAuth to authenticate users. There is a way to [implement your own Authenticator](https://cloud.google.com/appengine/docs/java/endpoints/javadoc/com/google/api/server/spi/config/Authenticator) but its documentation is virtually non-existent. You can follow this [stackoverflow answer](http://stackoverflow.com/a/25390994), though.
* **The default security implementation is not meant for inter-server communication.** - As with the previous item, unless you create your own Authenticator, the default authentication (Google Login) is meant for end-users.
* **Custom exception-to-error-response handling doesn't seem to exist.** - I've yet to see a similar feature to Spring's `@ExceptionHandler / @ResponseStatus` or Jersey's `WebApplicationException / ExceptionMapper`. You can extend Cloud Endpoint's existing exceptions but you're stuck with how it appears in JSON.
* **Does not support FileUpload.** - This is understandable because you really can't write files in GAE. However, there are times when you wish to capture the file in memory and write it somewhere else (Google Cloud Storage). There is a way to do this using servlets but why not do it with a uniform syntax (i.e. the same way you handle other requests).

Anyway, that's just what I was able to gather while using it. If any of the above points are no longer valid, please feel free to message me or write a comment.

On to the article!

---------------

## The SystemServiceServlet

Google Cloud Endpoint instances are created by the `SystemServiceServlet` class from the App Engine SDK. It uses the `newInstance()` method to create instances. This is fine unless you're using IoC frameworks such as Spring.
The problem with how `SystemServiceServlet` creates instances is that those instances are not "tracked" by Spring. The instances themselves need to exist in a [Bean Registry](http://docs.spring.io/spring-framework/docs/2.5.x/api/org/springframework/beans/factory/support/BeanDefinitionRegistry.html).
You can do this by annotating the Endpoints with `@Component`. However, you'll soon find out that if the endpoint has dependencies to other beans, a `NullPointerException` will be thrown. This is because the instance that Cloud Endpoints used was not able to wire its dependencies.
Recall that `SystemServiceServlet` uses `newInstance()` to create the endpoints as shown in the snippet below:

```java
public class SystemServiceServlet extends HttpServlet {
    ...
    protected <T> T createService(Class<T> serviceClass) {
        try {
            return serviceClass.newInstance();
        } catch (InstantiationException var3) {
            throw new RuntimeException(String.format("Cannot instantiate service class: %s", new Object[]{serviceClass.getName()}), var3);
        } catch (IllegalAccessException var4) {
            throw new RuntimeException(String.format("Cannot access service class: %s", new Object[]{serviceClass.getName()}), var4);
        }
    }
    ...
}
```

This means that there will be two instances of an endpoint (Cloud Endpoints call them "service"). The instances created by Spring will never be used because `SystemServiceServlet` does not know about it.
We need to be able to create the instances as beans in Spring and at the same time, make the `SystemServiceServlet` use them.

---------------

## Customizing SystemServiceServlet.createService()
To ensure that only one instance is created that is both recognized by Spring and used by the servlet, we need to override the implementation of the `createService()` method with the code below:

```java
protected <T> T createService(Class<T> serviceClass) {
    ApplicationContext context = WebApplicationContextUtils
            .getRequiredWebApplicationContext(this.getServletContext());
    AutowireCapableBeanFactory beanFactory = context.getAutowireCapableBeanFactory();
    try {
        return beanFactory.getBean(serviceClass);
    } catch (BeansException e) {
        String beanName = StringUtils.uncapitalize(serviceClass.getSimpleName());
        LOGGER.info("No bean of type'" + serviceClass.getName() + "'." +
                            " Creating a new bean named '" + beanName + "'");
        T bean = (T) beanFactory.createBean(serviceClass, AutowireCapableBeanFactory.AUTOWIRE_BY_TYPE, true);
        return (T) beanFactory.initializeBean(bean, beanName);
    }
}
```

Instead of returning the result of `serviceClass.newInstance()`, we now return an instance from Spring's bean registry or create one if it's not yet there.
Afterwards, make sure to update your `web.xml` to use the new class as shown below:

```xml
<servlet>
    <servlet-name>SystemServiceServlet</servlet-name>
    <servlet-class>
        com.rmpader.cloudendpointsspring.compatibility.AutowireSupportSystemServiceServlet
    </servlet-class>
    <init-param>
        <param-name>services</param-name>
        <param-value>
            com.rmpader.cloudendpointsspring.api.endpoint.HelloEndpoint
        </param-value>
    </init-param>
</servlet>
```

Voila! Your Cloud Endpoints are now officially Spring beans.

---------------

If you wish to see a sample application, you can find it [here](https://github.com/reypader/blogstuff/tree/master/cloud-endpoints-spring).

*Cheers!*<br>
Rey