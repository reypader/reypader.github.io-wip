---
layout: post
title:  "How to: Declare more complex request mappings in Spring"
date:   2015-12-21 18:53:00 +0800
edited: 2015-12-26 14:09:00 +0800
description: "Learn various ways of declaring request handling in Spring. You'll be able to handle request based on path variables, request parameters, content types, etc. You'll also learn how to handle exceptions to create custom error responses."
tags: java spring spring-rest
---
There's a lot of how-to's out there that could teach you how to create a REST API server with [Spring Boot](http://projects.spring.io/spring-boot/). Actually, spring already has a pretty good [example](https://spring.io/guides/gs/rest-service/). However, most of those found out there are pretty simplistic - the usual "Hello World" application. Of course, if that already showed you what you wanted, you wouldn't still be looking and stumble upon my humble blog. So, I would show you instead how to make a REST API with *the common stuff and use cases* that I think you're probably looking for.

<br>
**NOTE: All of the methods shown here assume that they are declared within the following controller:**

```java
@RestController
@RequestMapping(value = "/sample")
public class SampleController {
    ...
}
```
You can find the source code [here](https://github.com/reypader/blogstuff/tree/master/spring-request-mapping).

<br>
Here we go...

---------------

## The Basics
Before I show you how to do the more complex request mappings, I'd like to refresh your memory a bit by showing you the most basic way of declaring request handlers in your controller.

```java
@RequestMapping(method = RequestMethod.GET)
public String sampleGet() {
    return "This is the '/sample' handler for GET requests";
}

@RequestMapping(method = RequestMethod.POST)
public String samplePost() {
    return "This is the '/sample' handler for POST requests";
}
```

Notice that these mappings do not declare their own `value`, which means that they will match the same URL as the containing class. In this case, it would be "/sample".

---------------

## Handling Query Params
One of the most usual case for handling requests are consuming query parameters. For those of you that might call it differently, query parameters are the ones that come after the question mark (`?`) in the URL. For example, in the URL `http://localhost:8080?foo=bar&cat=meow`, the query parameters are `foo` and `cat` with values `bar` and `meow`, respectively.

In the snippet below, the query parameters are automatically parsed into a `Map` object. From what I know, older versions of Spring does not have this transformation.

```java
@RequestMapping(value = "/get-with-params",
                method = RequestMethod.GET)
public String sampleQueryParam(@RequestParam Map<String, String> params) {
    return "You've hit the '/sample/get-with-params' request handler with the following query params:\n" + params;
}
```

There are times when you simply want to do different logic based on certain parameters. For example, if the user passes a query param `foo=bar` you will create a new database record, otherwise you will do some other logic.

You can do this by placing an `if-else` block in your handler or in your service but that could lead to some cluttering. Spring has an option that could help you separate your business logic better by allowing you to declare a separate handler when `foo` is `bar`. See the snippet below.

```java
@RequestMapping(value = "/get",
                method = RequestMethod.GET,
                params = "foo=bar")
public String sampleForFooBar() {
    return "You've hit the '/sample/get' request handler where foo MUST be bar";
}
```

With this in place, whenever the query parameters contain the key `foo` and its value is `baz`, the `sampleForFooBar()` will be the one to handle the request.

Be careful, though. Spring executes request handlers that match the same request based on the "best" match which *may* be the same as the "more strict" match. Consider the two request handlers below.

```java
@RequestMapping(value = "/get",
                method = RequestMethod.GET)
public String unreachable() {
    return "You will never hit this request handler. Why? Because sampleForFooNotBar() has a more strict 'criteria'. See RequestMappingInfo.compareTo()";
}

@RequestMapping(value = "/get",
                method = RequestMethod.GET,
                params = "foo!=bar")
public String sampleForFooNotBar() {
    return "You've hit the '/sample/get' request handler where foo should not be bar";
}
```

The second method declares that it would handle any `GET` request to `/get` where `foo` is not assigned to `bar`. The first method on the other hand simply matches any `GET` request to `/get`. The problem will come to light once you try doing a `GET` to `/get` without any query parameters. You may expect that the method `unreachable()` will be executed but you'll be surprised that it does not. The method `sampleForFooNotBar()` was matched simply because `foo` is technically `null` which is, logically, not `bar`.

---------------

## Handling Path Variables
There are times when you'd prefer to have some values embedded into the URL itself and not use query parameters. These embedded values are called "path variables". Including path variables into your request handler is pretty straight forward.

```java
@RequestMapping(value = "/get/{var}",
                method = RequestMethod.GET)
public String samplePathVariable(@PathVariable("var") String var) {
    return "You've hit the '/sample/get/{var}' request handler with var=" + var;
}
```

---------------

## Receiving the Request Body / Payload
Have you ever been in the situation where you've declared an object parameter in the request handler and expected Spring to deserialize the payload correctly - but it always ends up `null`?

You've probably forgotten to add the ` @RequestBody ` annotation.

```java
@RequestMapping(value = "/post-with-body",
                method = RequestMethod.POST)
public String samplePostBody(@RequestBody PostRequest request) throws JsonProcessingException {
    return "You've hit the '/sample/post-with-body' request handler with the following body:\n"
                + objectMapper.writeValueAsString(request);
}
```

Without going into too much details, ` @RequestBody ` tells Spring that the body of the request payload/body should be mapped to the annotated parameter. This is important because you can mix and match various parameter types that have been shown in previous sections.

---------------

## Responding with a (JSON) Object
So far, we have only been responding with Strings. Of course, this is not the common use-case. Usually, you'd want to respond with a complex JSON or XML object. Strictly speaking, you have to annotate the method or class with ` @ResponseBody ` to declare an object response. However, since this is such a common annotation, you can make do without it. See the snippet below.

```java
@RequestMapping(value = "/post-with-body",
                method = RequestMethod.POST,
                params = "json=true")
public SampleResponse sampleJsonResponse(@RequestBody PostRequest request) throws JsonProcessingException {
    SampleResponse response = new SampleResponse();
    response.setMessage(samplePostBody(request));
    return response;
}
```

---------------

## File Uploads
File uploads in Spring are pretty straightforward. You simply declare a parameter of the type ` MultipartFile ` and annotate it with ` @RequestParam ` so that Spring knows where to read the data from.

```java
@RequestMapping(value = "/upload",
                method = RequestMethod.POST)
public String handleFileUpload(@RequestParam("file") MultipartFile file) throws IOException {
    byte[] bytes = feile.getBytes();
    return "Contents: \n\n" + new String(bytes);
}
```

One important thing to note here is that this creates a temporary file in the system and the reads from that. This is done for large file uploads where keeping them in memory is not a cost-effective option. However, there are times when you're not allowed to write to the file system such as when you're deploying to [Google App Engine](https://cloud.google.com/appengine/docs/java/). 

To work around this, you must simply create a bean of the type `org.springframework.web.multipart.commons.CommonsMultipartResolver` and set the `maxInMemorySize` property to an appropriate value.
 
---------------

## Handling Different Content Types
We've covered a lot of variations in handling requests. I'd dare say that these are the most common types that you will encounter in your development. However, there are rare cases when you want to handle request differently simply based on the `content-type` of the request. Doing so is as simple as declaring the ` consumes ` parameter in the ` @RequestMapping ` annotation.

```java
@RequestMapping(value = "/post-strict",
                method = RequestMethod.POST,
                consumes = MediaType.APPLICATION_JSON_VALUE)
public String sampleStrictJson(@RequestBody PostRequest request) throws JsonProcessingException {
    return "You've hit the '/sample/post-strict' request handler for 'application/json' with the following body:\n"
                + objectMapper.writeValueAsString(request);
}

@RequestMapping(value = "/post-strict",
                method = RequestMethod.POST,
                consumes = MediaType.APPLICATION_XML_VALUE)
public String sampleStrictXml(@RequestBody PostXMLRequest request) throws JsonProcessingException {
    return "You've hit the '/sample/post-strict' request handler for 'application/xml' with the following body (in JSON):\n"
                + objectMapper.writeValueAsString(request);
}
```

---------------

## Adding Translatable Exceptions
You wouldn't want the raw exception stacktrace being returned to the clients. In previous versions of Spring you can present exceptions in a more presentable manner by catching all exceptions and constructing a ` ResponseEntity ` return type with its status and body set to appropriate values. Since Spring 3, we can annotate exceptions with ` @ResponseStatus ` to give Spring an idea on how to present this to the client. You can then throw this annotated exception as-is.

```java
@ResponseStatus(code = HttpStatus.EXPECTATION_FAILED,
                reason = "A sample annotated exception.")
public class SampleTranslatedException extends RuntimeException {

}
```

The downside to this is that the description is static. If you'd want a dynamic error description, you should try...

---------------

## Writing an Exception Handler
Setting up your own exception handler is ideal for those cases when you want to assign different descriptions depending on certain parameters. Another benefit to this is that you can do additional tasks such as audit logging, error dumping, or calling other services when an exception is raised.

```java
@ExceptionHandler({JsonProcessingException.class, IOException.class})
public ResponseEntity<ErrorResponse> handleAllExceptions(Exception e) {
    ErrorResponse errorResponse = new ErrorResponse();
    errorResponse.setMessage(e.getMessage());
    
    return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                         .body(errorResponse);
}
```

---------------

## Other notes
If you're not using Spring Boot (which packages a lot of dependencies for you), you might have problems trying to receive or respond JSON objects. If that's the case, you might be missing the following dependency:

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>
```
---------------

That concludes this article. Hopefully, you've found the information I've presented here useful to your REST API development. If you have questions, feel free to [comment/file an issue](https://github.com/reypader/reypader.github.io/issues/new?title={{page.title}}) or if you feel like reading, I suggest you read the [Core Spring Framework documentation](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/) or the [Spring Boot Documentation](http://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/).

*Cheers!*<br>
Rey