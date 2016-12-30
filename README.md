# starwizard

Supporting classes for Dropwizard REST APIs

Dropwizard is awesome, and I've found it incredibly useful for building simple, scalable REST APIs. But while Dropwizard is opinionated, there are some things that it (appropriately) doesn't include in the box, which I've found I tend to re-use over and over when building Dropwizard APIs.

* I usually run Dropwizard APIs on AWS behind an Elastic Load Balancer. ELB doesn't have an automatic way to force HTTPS, so I always need an easy way to redirect HTTP requests to HTTPS.
* I use [Swagger](http://swagger.io/) to document my APIs, using the excellent [dropwizard-swagger](https://github.com/smoketurner/dropwizard-swagger) bundle for swagger-core. However, swagger-core doesn't provide an easy way to exclude Dropwizard @Auth parameters from the Swagger definitions.
* I want all my error responses to be JSON. These days Dropwizard defaults to JSON errors for most exceptions, which is great! But 401s when using auth still returns a text/plain response, which is not so great. A simple `JsonUnauthorizedHandler` class forces a JSON response.

Here are all these reusables in one place. Hope someone else finds it useful!

Several of these classes are not of my invention:
  * `AuthParamFilter` is from this  excellent blog post by [Pablo Meier](https://github.com/pablo-meier): https://www.reonomy.com/augmenting-dropwizard-with-swagger/, based on this StackOverflow answer by [Özkan Can](http://stackoverflow.com/users/2494590/%C3%96zkan-can) http://stackoverflow.com/questions/21911166/how-can-i-set-swagger-to-ignore-suspended-asyncresponse-in-asynchronous-jax-rs
  * `JsonUnauthorizedHandler` is from this equally excellent blog post by [Nick Babcock](https://github.com/nickbabcock): https://nbsoftsolutions.com/blog/writing-a-dropwizard-json-app

# Installation

To use Starwizard, add the following to your project's POM file:

```
<dependency>
    <groupId>io.stardog</groupId>
    <artifactId>starwizard</artifactId>
    <version>0.1.0</version>
</dependency>
```

# Features and Usage

## HTTP to HTTPS redirect for Amazon ELB

If you are running Dropwizard behind ELB and want to redirect all HTTP requests to HTTPS:

```java
env.getApplicationContext().addFilter(new FilterHolder(new LbHttpsRedirectFilter()), "/*",
    EnumSet.of(DispatcherType.REQUEST));
```

I often add an `forceHttps` setting to the yml file to conditionally determine whether to do this, so I have some flexibility for non-HTTPS dev deployments.

## swagger 'internal' designation

If you are using Swagger, the following will allow you to designate parameters as "internal", meaning they won't be included in the Swagger definitions. This is primarily useful for @Auth parameters.

```java
FilterFactory.setFilter(new AuthParamFilter());
```

Example of how to define a parameter as internal:
```java
@ApiParam(access = "internal") @Auth User user,
```

## JSON 401 responses

To force JSON responses when Unauthorized, call `setUnauthorizedHandler(new JsonUnauthorizedHandler())` on your `AuthFilterBuilder`. Here is an example for Oauth:

```java
env.jersey().register(new AuthDynamicFeature(
        new OAuthCredentialAuthFilter.Builder<UserAuth>()
                .setAuthenticator(authenticator)
                .setPrefix("Bearer")
                .setUnauthorizedHandler(new JsonUnauthorizedHandler())
                .buildAuthFilter()));
```


# Other stuff included

## TooManyRequestsException

HTTP status code 429 (Too Many Requests) is a useful error for rate-limiting schemes. This provides a JAX-RS exception for this code.

## @Lowercase Hibernate validation annotation

Sometimes (especially for email addresses) it's important to enforce lowercase. This provides a simple annotation to do that:

```java
@Lowercase
public abstract String getEmail();
```

## @Required Hibernate validation group annotation

On models, mark fields as required with a `@NotNull(groups = Required.class)` annotation.

Now you can force the use of required fields with the following annotation on the method signature in your resource:

``@Valid @Validated(Required.class) MyModel model``

# Other setup recommendations

None of the below involves starwizard -- these are all part of standard Dropwizard. But I usually do the below every time, so here they are in one place:

## Allow logging of full request/response

When you're debugging a nasty problem, there's nothing quite like examining full requests and responses (including headers). The below configures Jersey's `LoggingFeature` to log in DEBUG mode:

```java
env.jersey().register(new LoggingFeature(Logger.getLogger(LoggingFeature.DEFAULT_LOGGER_NAME),
                                         Level.FINE, LoggingFeature.Verbosity.PAYLOAD_ANY, 100000));
```

If you're using default INFO log levels, requests/responses will not log. To temporarily turn on logging for your service, use the builtin Dropwizard `log-level` task:

```shell
curl -X POST "http://localhost:8081/tasks/log-level?logger=org.glassfish.jersey.logging.LoggingFeature&level=DEBUG"
```

Set it back after you're finished debugging:

```shell
curl -X POST "http://localhost:8081/tasks/log-level?logger=org.glassfish.jersey.logging.LoggingFeature&level=INFO"
```

## Configure the ObjectMapper

I always use Java 8's time library so I want JavaTimeModule support. I like my Instants to render as ISO strings. I also prefer to be generous about ignoring unknown properties and including non-nulls (this is a matter of taste):

```java
ObjectMapper objectMapper = env.getObjectMapper();
objectMapper.registerModule(new JavaTimeModule());
objectMapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
objectMapper.disable(SerializationFeature.WRITE_DATE_TIMESTAMPS_AS_NANOSECONDS);
objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
objectMapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
```

## Set up CORS to allow cross domain requests

I nearly always host the API on a different domain than my UI, so the below is handy:

```java
final FilterRegistration.Dynamic cors = env.servlets().addFilter("CORS", CrossOriginFilter.class);
cors.setInitParameter("allowedOrigins", "*");
cors.setInitParameter("allowedHeaders", "X-Requested-With,Content-Type,Accept,Origin,Authorization");
cors.setInitParameter("allowedMethods", "OPTIONS,GET,PUT,POST,DELETE,PATCH,HEAD");
cors.addMappingForUrlPatterns(EnumSet.allOf(DispatcherType.class), true, "/*");
```

## Turn on multipart-form support (must include dropwizard-forms)

Only if you are dealing with file uploads. I almost always wind up needing file uploads in my APIs.

```java
env.jersey().register(MultiPartFeature.class);
```
