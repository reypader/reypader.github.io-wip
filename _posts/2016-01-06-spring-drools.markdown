---
layout: post
title:  "How to: Integrate Drools and Spring Framework"
date:   2016-01-06 18:06:00 +0800
description: "Spring is probably the most widely used framework for Java. Even for many enterprise-level applications, Spring is almost always used at some point - maybe even solely. Another buzz word(s) these days for enterprise applications is \"Rule Engine\" and Drools is one of them. Unfortunately, there seems to be a shortage of tutorials on how to actually integrate the two. This article aims to guide the weary developer into using both technologies into one application."
tags: java spring drools
---

There are a few - very few - tutorials that show us how to integrate Spring and Drools. [This article in Scattercode](http://scattercode.co.uk/2015/02/06/a-minimal-spring-boot-drools-web-service/) actually does a 
great job and I highly suggest you have a look at that. [Technorage also has a pretty good tutorial](http://mytechnorage.blogspot.com/2014/10/drools-6-integration-with-spring-mvc-4.html) but I found it
a bit hard to follow. I'm not sure, maybe it's just the layout that makes it a bit intimidating - and the colors? IDK IDK. Anyway, I'll be building upon the tutorial from Scattercode and see where
we can get from there.

This article focuses on how to configure your `KieContainer` which serves as the root of all interactions with Drools. We'll be looking at how to configure a classpath-based rule and a remote-based rule.
The latter would be what you would be looking for if you wish to dynamically change rules without restarting the server, or redeploying.

We'll be creating a "Course Suggestion" application that would return a list of courses that would fit based on a user's ratings of subjects. See the table below:

<div class="row">
    <div class="col-md-8 col-md-offset-2">
        <table class="table">
          <thead>
          <tr>
            <th><em>Course<em></th>
            <th>math</th>
            <th>software</th>
            <th>electronics</th>
            <th>physics</th>
            <th>arts</th>
            <th>social_studies</th>
            <th>culinary</th>
          </tr>
          </thead>
          <tbody>
          <tr>
            <th scope="row">Computer Science</th>
            <td>6</td>
            <td>9</td>
            <td>3</td>
            <td></td>
            <td></td>
            <td></td>
            <td></td>
          </tr>
          <tr>
            <th scope="row">Theoretical Physics</th>
            <td>9</td>
            <td></td>
            <td></td>
            <td>9</td>
            <td></td>
            <td></td>
            <td></td>
          </tr>
          <tr>
            <th scope="row">Theatrical Arts</th>
            <td></td>
            <td></td>
            <td></td>
            <td></td>
            <td>9</td>
            <td>6</td>
            <td></td>
          </tr>
          </tbody>
        </table>
    </div>
</div>

Based on this table, if a user provides a rating of 9 in math, 9 in software, 5 in electronics, and 3 in physics, only the course Computer Science will be suggested. However, if the rating for physics is increased to 9, then
there will be two suggested courses - Theoretical Physics and Computer Science.

For simplicity, the only aspect that will change for each section is how the `KieContainer` is configured. The assumption is that you have an idea on how to use
the `KieContainer` to fire rules. You can find the source code [here](https://github.com/reypader/blogstuff/tree/master/spring-drools).
You will also find a guide there on how to setup [Drools Workbench](https://docs.jboss.org/drools/release/latest/drools-docs/html/pt05.html) and [Wildfly](http://wildfly.org/) (an alternative to Tomcat which is more compatible with Drools Workbench) on your machine.

---------------

## Declaring Rules within the Classpath

If you've seen [the article in Scattercode](http://scattercode.co.uk/2015/02/06/a-minimal-spring-boot-drools-web-service/), you'll find that this section presents exactly the same configuration.
The first step is to create the DRL file corresponding to the table above. If you need a reference, the [official documentation](http://docs.jboss.org/drools/release/latest/drools-docs/html_single/index.html#DroolsLanguageReferenceChapter) is still your best bet.
The DRL file needs to be in the `src/main/resources/` directory. Ideally you should place it in a folder whose name is similar to a package structure (e.g. `com.rmpader.springdrools`).

```js
package com.rmpader.springdrools

import com.rmpader.springdrools.fact.SubjectRating;

global com.rmpader.springdrools.fact.Suggestions suggestions;

rule "Suggest Computer Science"
when
    SubjectRating( subject=="math", rating >= 6)
    SubjectRating( subject=="software", rating >= 9)
    SubjectRating( subject=="electronics", rating >= 3)
then
    suggestions.addSuggestedCourseCode("COMP-SCI");
end

rule "Suggest Theoretical Physics"
when
    SubjectRating( subject=="math", rating >= 9)
    SubjectRating( subject=="physics", rating >= 9)
then
    suggestions.addSuggestedCourseCode("THEOR-PHYS");
end

rule "Suggest Theatrical Arts"
when
    SubjectRating( subject=="arts", rating >= 9)
    SubjectRating( subject=="social_studies", rating >= 6)
then
    suggestions.addSuggestedCourseCode("THEAT-ART");
end
```

Notice the classes being used in this DRL file. `SubjectRating` and `Suggestions` are classes that exists in the classpath (i.e. they are either declared in the same project or in a dependency).
These are *facts*. *Facts* are objects that are passed into the rule engine as input. You'll notice that the concept of imports are the same in Drools as it is in Java. This should help you get an idea on how facts should be declared and used.
In the [sample code](https://github.com/reypader/blogstuff/tree/master/spring-drools), those classes are [defined](https://github.com/reypader/blogstuff/tree/master/spring-drools/drools-demo-facts/src/main/java/com/rmpader/springdrools/fact) in the `drools-demo-facts` directory.
When writing your fact objects, I highly suggest that you place them in a separate maven artifact/project from your application. That artifact should only contain the fact objects and nothing else. It doesn't make sense to do it that way when you're doing the classpath rules configuration but you'll understand why once you use [remote artifacts](#fetching-rules-from-a-remote-location).

Now that we have rules in place, we need to be able do declare them in the rule engine. The rule engine identifies rules that are declared in a `kmodule.xml` file in the `src/main/resources/META-INF` directory.
To declare the rules, the `kmodule.xml` file needs to have the following content:

```xml
<kmodule xmlns="http://jboss.org/kie/6.0.0/kmodule">
    <kbase name="base" default="true" packages="com.rmpader.springdrools">
        <ksession name="session" default="true" type="stateless"/>
    </kbase>
</kmodule>
```

The `packages` attribute of the `<kbase/>` tag needs to be the same as the name of the directory where you've placed the DRL file. In this case, the directory name is `com.rmpader.springdrools`.
I wouldn't go into detail about the other tags and attributes. I implore you to read the [official documentation](http://docs.jboss.org/drools/release/latest/drools-docs/html_single/index.html) to learn more about Drools itself.

Now that we have the rules and the proper configuration for them to be recognized by the rules engine. We need only to tell the rules engine where to look for the configuration file `kmodule.xml`.
To do that, we need to tell the `KieContainer` to look in the classpath.

```java
@Bean
public KieContainer kieContainer() throws IOException {
    KieServices ks = KieServices.Factory.get();
    return ks.getKieClasspathContainer();
}
```

With this, each time you fire rules, the DRL file declared in your classpath `kmodule.xml` will be used. You can see the complete source [here](https://github.com/reypader/blogstuff/tree/master/spring-drools/drools-demo-app-classpath).

This is a very simple use-case. Actually, you'll rarely want this particular configuration. Why? The strength of rule engines is that they are decoupled from
your application. Which means that they could change independently from the application itself without disrupting service. With a classpath rule, evertime you need
to change the rules, you'll need to rebuild and re-deploy the application which needs a certain amount of downtime. So, how do you change the rules without
stopping the server? You should be...

---------------

## Fetching Rules from a Remote Location

Fetching rules remotely is actually the most common use-case for Drools in my opinion. Strangely enough, I've yet to find a tutorial on how to do this. Hopefully, this section
does a decent enough job to show you how.

### Drools Workbench

<img class="img-rounded img-responsive center-block" src="https://docs.jboss.org/drools/release/6.2.0.Final/drools-docs/html_single/images/Workbench/ReleaseNotes/kie-drools-wb.png">
<div style="margin:auto; text-align:center"><em>Taken from the Drools Documentation</em></div>

Before diving into the configuration, let me introduce you first to the [Drools Workbench](https://docs.jboss.org/drools/release/latest/drools-docs/html/pt05.html). This application
is where we will be authoring and serving rules. To those who use [Maven](https://maven.apache.org/), it's very similar to [Nexus](http://books.sonatype.com/nexus-book/reference/intro-sect-intro.html). In fact, it also serves rules as JAR artifacts. Basically
the rules themselves are also Maven Artifacts. Aside from serving as a repository, the Drools Workbench can also serve as your IDE for authoring projects/rules.

The README of the sample source code contains guides on how to [author rules](https://github.com/reypader/blogstuff/tree/master/spring-drools#setup-a-drools-project) as well as setting up your [Drools Workbench and Wildfly](https://github.com/reypader/blogstuff/tree/master/spring-drools#setup-wildfly).
Wildfly was used over Tomcat due to compatibility issues.


### The Facts Artifact

Earlier, I've mentioned that you should place your facts in a separate maven artifact. Also, if you've gone through the guide on how to author rules, you'll also see that the first thing needed
is to have a facts artifact ready to be imported. The reason for this is that since your application and the rules project are two different projects that use the same fact objects, there needs to be a uniform
definition of the classes being used. By now you've probably realized that the same is true with API artifacts of certain libraries (e.g. [log4j-api](http://mvnrepository.com/artifact/org.apache.logging.log4j/log4j-api), [JPA](http://mvnrepository.com/artifact/javax.persistence/persistence-api)).
Hence, separating the fact objects artifact becomes mandatory when doing remote rules.


### Retrieving a rule from Drools Workbench

Alright! Now that we've cleared up some things needed to be able to fetch rules remotely, we can jump right in to how the configuration of the `KieContainer` looks like.

The idea of fetching the rules remotely is that your application will download the JAR file of the rules from the Drools Workbench and include its contents into the classpath.
To be able to do this, you need to add the artifact as a `KieModule` into your application's `KieRepository`. The "source" of the `KieModule` will come from the data contained in the
downloaded rule artifact which is declared as a `org.drools.core.io.impl.UrlResource` (not to be confused with Spring's own `UrlResource`).
Feeling lost? Hopefully this snippet could clear things up a bit.

```java
@Bean
public KieContainer kieContainer() throws IOException {
    KieServices ks = KieServices.Factory.get();
    
    KieResources resources = ks.getResources();
    String url = "http://localhost:8080/kie-drools-wb-6.3.0.Final-wildfly8/maven2/com/rmpader/course-suggestion/1.0/course-suggestion-1.0.jar";
    UrlResource urlResource = (UrlResource) resources.newUrlResource(url);
    urlResource.setUsername("drools-user");
    urlResource.setPassword("password");
    urlResource.setBasicAuthentication("enabled");
    InputStream stream = urlResource.getInputStream();
    
    KieRepository repo = ks.getRepository();
    KieModule k = repo.addKieModule(resources.newInputStreamResource(stream));
    return ks.newKieContainer(k.getReleaseId());
}
```

As you can see here, the authentication parameters need to always be present because Drools Workbench requires authentication details for any form of
interaction. Of course, it would be best if you could externalize the values into a properties file. You can see the complete source [here](https://github.com/reypader/blogstuff/tree/master/spring-drools/drools-demo-app-remote).

Great! So, now we are fetching rules remotely from the workbench. What does this mean, exactly? For one, you no longer need to declare a `kmodule.xml` in your
application's `src/main/resources` folder as well as declare DRL files. That's one less element that new developers in a project need to know about. However,
there is one piece left missing. How do we...

---------------

## Dynamically change rule versions

At this point in the article, the goal to show you how to fetch rules remotely is already met. This section focuses on how to change the version which is basically
executing the same code in the configuration bean for `KieContainer` with a different artifact URL and executing `kieContainer.updateToVersion(...)`.

```java
KieServices ks = KieServices.Factory.get();
KieResources resources = ks.getResources();
String url = "http://localhost:8080/kie-drools-wb-6.3.0.Final-wildfly8/maven2/"
        + createPath(ruleArtifact.getGroupId(),
                     ruleArtifact.getArtifactId(),
                     ruleArtifact.getVersion());
UrlResource urlResource = (UrlResource) resources.newUrlResource(url);
urlResource.setUsername("drools-user");
urlResource.setPassword("password");
urlResource.setBasicAuthentication("enabled");
InputStream is = urlResource.getInputStream();

KieRepository repo = ks.getRepository();
KieModule k = repo.addKieModule(resources.newInputStreamResource(is));
kieContainer.updateToVersion(k.getReleaseId());
```

* `createPath()` transforms the group ID, artifact ID, and the version into its equivalent URL path. For Example:
    `com.rmpader.groupid:artifactid:version` becomes `com/rmpader/groupid/artifactid/version/artifactid-version.jar`
    
* `ruleArtifact` is an entity retrieved from the database.

As with the configuration, you still need to declare the artifact as a `UrlResource` and add it as a `KieModule` in the `KieRepository`. The only new step for that
matter is to update the version that the `KieContainer` itself is using. Of course, there's still the matter of saving the activated version details somewhere (e.g. database).
I'll leave that part up to you but if you wish to see a sample application, you can find one [here](https://github.com/reypader/blogstuff/tree/master/spring-drools/drools-demo-app-versioned).
The sample application saves the group ID, artifact ID, and the version in the database. It also saves which one among all the saved versions is currently active so that
when the application starts up, the application first fetches the version details from the database and constructs the correct artifact URL from it.

**Note: The sample application does not take into account multiple instances. In the case that multiple instances are running, activating a version in one instance does not update the version used by the others. That part could be made into a separate article itself.**

---------------

There you have it, a tutorial on how to fetch rules remotely from Drools Workbench.

*Cheers!*<br>
Rey