# openapi-generator-maven-plugin-template-customization-example

## Motivation

The [OpenAPI
Generator](https://github.com/OpenAPITools/openapi-generator) is a
marvelous piece of technology. It allows users to generate model,
interface, and implementation code for server and client for a variety
of platforms from an OpenAPI spec quickly and easily. At least, for
default configurations.

But sometimes users need to do something outside of the
default. Fortunately, the project offers rich capabilities for
[template](https://openapi-generator.tech/docs/templating) and
[ad-hoc](https://openapi-generator.tech/docs/customization)
customization. Unfortunately, those customization features can be
difficult to set up and use in practice.

This project is a [SSCCE](http://sscce.org/) for performing template
customization using the OpenAPI Generator Maven Plugin. The chosen
customization uses
[extensions](https://openapi-generator.tech/docs/templating/#extensions)
to add per-method support for JAX-RS 2.0 `@Suspended AsyncResponse`
features to the
[`jaxrs-spec`](https://openapi-generator.tech/docs/generators/jaxrs-spec)
generator. If anyone happens to need `AsyncResponse` code generation
for the OpenAPI Generator Maven Plugin, this should work out of the
box.

This project is also a fine starting point for a Reasonable Default of
using the OpenAPI Generator Maven Plugin, for anyone looking for an
example to start from.

## Introduction

Let's dive into the particulars of this example.

### OpenAPI Spec

The OpenAPI spec for this example is very simple:

    openapi: 3.0.2
    info:
      title: OpenAPI Generator Maven Plugin Template Customization Example
      description: An example Maven project that uses the OpenAPI Generator Maven plugin with template customization to generate an API model and server.
      contact:
        email: andy.boothe@gmail.com
      version: 0.0.0
    servers:
      - url: http://localhost:8080/v1
    tags:
      - name: example
        description: Example endpoint
    paths:
      /greet:
        post:
          tags:
            - example
          summary: Greet the user
          description: Generate a greeting for the given name
          operationId: greet
          # We can set determine whether each method is synchronous or asynchronous using this flag.
          # This particular option is an extension, so you won't find documentation for it anywhere.
          # Implementing it is the point of this exercise.
          x-async-enabled: false
          requestBody:
            description: The name to greet
            content:
              application/json:
                schema:
                  $ref: '#/components/schemas/Name'
          responses:
            200:
              description: The greeting was generated successfully
              content:
                application/json:
                  schema:
                    $ref: '#/components/schemas/Greeting'
    components:
      schemas:
        Name:
          type: object
          properties:
            name:
              type: string
    
        Greeting:
          type: object
          properties:
            greeting:
              type: string

There is one endpoint, which generates a greeting given a name, and
one model object each for a Name and a Greeting. We'll be looking at
code generated for this spec a little later on.

### POM

Naturally, the OpenAPI Generator Maven Plugin is configured using the
POM. This project uses a reasonably simple and sane configuration of
the plugin:

    <configuration>
        <!-- This section includes the "global" configuration options, so called because they apply to all generators. Users generally don't need to care about this dxcept to make sure their configuration information lands in the correct place - <configuration> vs <configOptions>. -->
        <!-- https://github.com/OpenAPITools/openapi-generator/tree/master/modules/openapi-generator-maven-plugin -->
    
        <!-- The OpenAPI spec is in a file in this repository called "openapi.yml". It can also be a hyperlink. -->
        <inputSpec>openapi.yml</inputSpec>
    
        <!-- There are many generators available. We are using jaxrs-spec. -->
        <generatorName>jaxrs-spec</generatorName>
    
        <!-- APIs are service interfaces or implementations. -->
        <generateApis>true</generateApis>
    
        <!-- Models are DTOs, generally from the schemas portion of the spec. -->
        <generateModels>true</generateModels>
    
        <!-- In this example, we're not generating the POM, README, etc. Just the code, ma'am. -->
        <generateSupportingFiles>false</generateSupportingFiles>
    
        <!-- We are showing off template customization today. The custom templates are stored here. -->
        <templateDirectory>${project.basedir}/openapi/templates</templateDirectory>
    
        <!-- This is the director where generated code will be stored. The plugin automatically adds this directory to your build. This is the default value. You generally don't need to change this, but it's worth calling out in this example so users know where to look for generated code. -->
        <output>${project.build.directory}/generated-sources/openapi</output>
    
        <configOptions>
            <!-- These are the "local" configuration options that apply only to this generator. -->
            <!-- https://openapi-generator.tech/docs/generators/jaxrs-spec -->
    
            <!-- These are the packages we want our APIs and Models generated in, respectively. -->
            <apiPackage>com.sigpwned.greeting.spi.service</apiPackage>
            <modelPackage>com.sigpwned.greeting.spi.model</modelPackage>
    
            <!-- I personally consider these to be best practices. However, they are up to you! -->
            <dateLibrary>java8</dateLibrary>
            <useSwaggerAnnotations>false</useSwaggerAnnotations>
    
            <!-- These values are largely up to you, and depend on your goals and philosophy. This SSCCE should support all permutations of the below. -->
    
            <!-- I prefer to have interface methods return model objects, not JAX-RS Response objects. I can still generate any response I want by throwing WebApplicationException classes. -->
            <!-- If this is set to true, then all generated service methods will return javax.ws.rs.core.Response objects. -->
            <!-- If this is set to false, then all generated service methods will return their respective model objects. -->
            <returnResponse>false</returnResponse>
    
            <!-- I prefer to generate interfaces, not implementation. This allows me to make sure that my client and server implementations are in sync. -->
            <!-- If this is set to true, then service interfaces will be generated. -->
            <!-- If this is set to false, then service implementations will be generated. -->
            <interfaceOnly>true</interfaceOnly>
    
            <!-- This is a useful feature for web frameworks that support the JAX-RS 2.1 feature of implementing asynchronous processing by returning CompletionStage instances. In all cases, generated service methods return either Response or a model object, depending on the above configuration. This configuration option affects all methods. Users cannot choose to make individual methods synchronous versus asynchronous.  -->
            <!-- If this is set to true, then all generated service method return types will be changed from T to CompletionStage<T>. -->
            <!-- If this is set to false, then all generate service method return types will be unchanged. -->
            <supportAsync>false</supportAsync>
        </configOptions>
    </configuration>

The `<configuration>` (as opposed to `<configOptions>`) portion of
this project is largely boilerplate, and won't change much even if you
switch generators. The `<configOptions>` section is much more tailored
to the task. In the default example, note that we are generating
interfaces that return model objects synchronously.

## Default Behavior

Before we dive into template customization, let's understand the
generator's out-of-the-box functionality first. Some of this is
specific to the `jaxrs-spec` generator, but much of it is universal to
all generators.

### Running the build

Let's run the maven build with this default configuration. This
produces the following output:

    $ mvn clean compile
    [INFO] Scanning for projects...
    [INFO] 
    [INFO] --< com.sigpwned:openapi-generator-maven-plugin-template-customization-example >--
    [INFO] Building openapi-generator-maven-plugin-template-customization-example 0.0.0-SNAPSHOT
    [INFO] --------------------------------[ pom ]---------------------------------
    [INFO] 
    [INFO] --- maven-clean-plugin:2.5:clean (default-clean) @ openapi-generator-maven-plugin-template-customization-example ---
    [INFO] Deleting /Users/aboothe/eclipse-workspace/openapi-generator-maven-plugin-template-customization-example/target
    [INFO] 
    [INFO] --- openapi-generator-maven-plugin:5.4.0:generate (generate) @ openapi-generator-maven-plugin-template-customization-example ---
    [INFO] Generating with dryRun=false
    [INFO] Output directory (/Users/aboothe/eclipse-workspace/openapi-generator-maven-plugin-template-customization-example/target/generated-sources/openapi) does not exist, or is inaccessible. No file (.openapi-generator-ignore) will be evaluated.
    [INFO] OpenAPI Generator: jaxrs-spec (server)
    [INFO] Generator 'jaxrs-spec' is considered stable.
    [INFO] Environment variable JAVA_POST_PROCESS_FILE not defined so the Java code may not be properly formatted. To define it, try 'export JAVA_POST_PROCESS_FILE="/usr/local/bin/clang-format -i"' (Linux/Mac)
    [INFO] NOTE: To enable file post-processing, 'enablePostProcessFile' must be set to `true` (--enable-post-process-file for CLI).
    [INFO] Invoker Package Name, originally not set, is now derived from api package name: com.sigpwned.greeting.spi
    [INFO] Processing operation greet
    [INFO] writing file /Users/aboothe/eclipse-workspace/openapi-generator-maven-plugin-template-customization-example/target/generated-sources/openapi/src/gen/java/com/sigpwned/greeting/spi/model/Greeting.java
    [INFO] writing file /Users/aboothe/eclipse-workspace/openapi-generator-maven-plugin-template-customization-example/target/generated-sources/openapi/src/gen/java/com/sigpwned/greeting/spi/model/Name.java
    [INFO] writing file /Users/aboothe/eclipse-workspace/openapi-generator-maven-plugin-template-customization-example/target/generated-sources/openapi/src/gen/java/com/sigpwned/greeting/spi/service/GreetApi.java
    [INFO] Skipping generation of supporting files.
    ################################################################################
    # Thanks for using OpenAPI Generator.                                          #
    # Please consider donation to help us maintain this project 🙏                 #
    # https://opencollective.com/openapi_generator/donate                          #
    ################################################################################
    [INFO] ------------------------------------------------------------------------
    [INFO] BUILD SUCCESS
    [INFO] ------------------------------------------------------------------------
    [INFO] Total time:  0.679 s
    [INFO] Finished at: 2022-04-07T11:54:52-05:00
    [INFO] ------------------------------------------------------------------------

The OpenAPI Generator Maven Plugin generates some useful diagnostic
and progress output. In particular, it logs the files it generates,
which is a good starting point to confirm you're generating the code
you want.

Of course, you can generate a lot more output with `mvn -X -e
generate-sources`.

### Checking the code

Let's see what code we generated:

    $ find target/generated-sources/openapi -type f                                      
    target/generated-sources/openapi/.openapi-generator/openapi.yml-generate.sha256
    target/generated-sources/openapi/src/gen/java/com/sigpwned/greeting/spi/model/Greeting.java
    target/generated-sources/openapi/src/gen/java/com/sigpwned/greeting/spi/model/Name.java
    target/generated-sources/openapi/src/gen/java/com/sigpwned/greeting/spi/service/GreetApi.java

    $ cat target/generated-sources/openapi/src/gen/java/com/sigpwned/greeting/spi/model/Greeting.java
    package com.sigpwned.greeting.spi.model;
    
    import javax.validation.constraints.*;
    import javax.validation.Valid;
    
    import java.util.Objects;
    import com.fasterxml.jackson.annotation.JsonProperty;
    import com.fasterxml.jackson.annotation.JsonCreator;
    import com.fasterxml.jackson.annotation.JsonValue;
    import com.fasterxml.jackson.annotation.JsonTypeName;
    
    
    
    @JsonTypeName("Greeting")
    @javax.annotation.Generated(value = "org.openapitools.codegen.languages.JavaJAXRSSpecServerCodegen", date = "2022-04-07T11:54:52.045724-05:00[America/Chicago]")public class Greeting   {
      
      private @Valid String greeting;
    
      /**
       **/
      public Greeting greeting(String greeting) {
        this.greeting = greeting;
        return this;
      }
    
      
    
      
      @JsonProperty("greeting")
      public String getGreeting() {
        return greeting;
      }
    
      @JsonProperty("greeting")
      public void setGreeting(String greeting) {
        this.greeting = greeting;
      }
    
    
      @Override
      public boolean equals(Object o) {
        if (this == o) {
          return true;
        }
        if (o == null || getClass() != o.getClass()) {
          return false;
        }
        Greeting greeting = (Greeting) o;
        return Objects.equals(this.greeting, greeting.greeting);
      }
    
      @Override
      public int hashCode() {
        return Objects.hash(greeting);
      }
    
      @Override
      public String toString() {
        StringBuilder sb = new StringBuilder();
        sb.append("class Greeting {\n");
        
        sb.append("    greeting: ").append(toIndentedString(greeting)).append("\n");
        sb.append("}");
        return sb.toString();
      }
    
      /**
       * Convert the given object to string with each line indented by 4 spaces
       * (except the first line).
       */
      private String toIndentedString(Object o) {
        if (o == null) {
          return "null";
        }
        return o.toString().replace("\n", "\n    ");
      }
    
    
    }

    $ cat target/generated-sources/openapi/src/gen/java/com/sigpwned/greeting/spi/model/Name.java
    package com.sigpwned.greeting.spi.model;
    
    import javax.validation.constraints.*;
    import javax.validation.Valid;
    
    import java.util.Objects;
    import com.fasterxml.jackson.annotation.JsonProperty;
    import com.fasterxml.jackson.annotation.JsonCreator;
    import com.fasterxml.jackson.annotation.JsonValue;
    import com.fasterxml.jackson.annotation.JsonTypeName;
    
    
    
    @JsonTypeName("Name")
    @javax.annotation.Generated(value = "org.openapitools.codegen.languages.JavaJAXRSSpecServerCodegen", date = "2022-04-07T11:54:52.045724-05:00[America/Chicago]")public class Name   {
      
      private @Valid String name;
    
      /**
       **/
      public Name name(String name) {
        this.name = name;
        return this;
      }
    
      
    
      
      @JsonProperty("name")
      public String getName() {
        return name;
      }
    
      @JsonProperty("name")
      public void setName(String name) {
        this.name = name;
      }
    
    
      @Override
      public boolean equals(Object o) {
        if (this == o) {
          return true;
        }
        if (o == null || getClass() != o.getClass()) {
          return false;
        }
        Name name = (Name) o;
        return Objects.equals(this.name, name.name);
      }
    
      @Override
      public int hashCode() {
        return Objects.hash(name);
      }
    
      @Override
      public String toString() {
        StringBuilder sb = new StringBuilder();
        sb.append("class Name {\n");
        
        sb.append("    name: ").append(toIndentedString(name)).append("\n");
        sb.append("}");
        return sb.toString();
      }
    
      /**
       * Convert the given object to string with each line indented by 4 spaces
       * (except the first line).
       */
      private String toIndentedString(Object o) {
        if (o == null) {
          return "null";
        }
        return o.toString().replace("\n", "\n    ");
      }
    
    
    }

    $ cat target/generated-sources/openapi/src/gen/java/com/sigpwned/greeting/spi/service/GreetApi.java
    package com.sigpwned.greeting.spi.service;
    
    import com.sigpwned.greeting.spi.model.Greeting;
    import com.sigpwned.greeting.spi.model.Name;
    
    import javax.ws.rs.*;
    import javax.ws.rs.core.Response;
    
    import java.util.concurrent.CompletionStage;
    import java.util.concurrent.CompletableFuture;
    
    import java.io.InputStream;
    import java.util.Map;
    import java.util.List;
    import javax.validation.constraints.*;
    import javax.validation.Valid;
    
    @Path("/greet")
    @javax.annotation.Generated(value = "org.openapitools.codegen.languages.JavaJAXRSSpecServerCodegen", date = "2022-04-07T11:54:52.045724-05:00[America/Chicago]")public interface GreetApi {
    
        @POST
        @Consumes({ "application/json" })
        @Produces({ "application/json" })
        Greeting greet(@Valid Name name);
    
    }

### Generated Model Classes

We have two Model classes -- `Greeting` and `Name` -- which correspond
to our named model objects from the spec. They are totally generic
POJOs. They support a default constructor, private attributes with
accessors, plus sane implementations of `hashCode`, `equals`, and
`toString`.

The generator does exactly what it says on the tin here, and we aren't
customizing the Model objects in this example, so we won't spend any
more time looking at the Model objects here.

### Generated API Classes

The API class is more interesting.

We have one endpoint, `POST /greet`, so we have one method across all
generated API classes. That method is called `greet`, which is its
`operationId` in the spec.

Methods are collected into classes by the first slug on their endpoint
paths, so in this case we have one class called `GreetApi`. If our
spec had two endpoints `POST /greet/one` and `GET /greet/two`, then
there would be one class `GreetApi` with two methods in it. If our
spec had two endpoints `POST /greet` and `POST /farewell`, then there
would be two classes `GreetApi` and `FarewellApi` with one method each.

Because of the configuration values we chose in our POM file:

* `GreetApi` is an interface because `<interfaceOnly>true</interfaceOnly>`
* `greet` returns a `Greeting` (as opposed to a `Response`) because `<returnResponse>false</returnResponse>`
* `greet` returns a `Greeting` (as opposed to a `CompletionStage<Greeting>`) because `<supportAsync>false</supportAsync>`

## Template Customization

Now that we understand the generator's default behavior, let's make
some customizations. For the purposes of this example, we will add
JAX-RS `AsyncResponse` code generation to the generator.

### The Customization Requirements

Out of the box, the OpenAPI Generator supports code generation for
asynchronous processing using the JAX-RS 2.1 `CompletionStage`
paradigm. However, some web frameworks -- e.g.,
[Dropwizard](https://www.dropwizard.io/) -- only support the earlier
`AsyncResponse` paradigm. Let's add `@Suspended AsyncResponse`
asynchronous processing code generation to the generator.

In our example spec, the generator can already generate method
definitions using `CompletionStage` using
`<supportAsync>true</supportAsync>`:

    @POST
    @Consumes({ "application/json" })
    @Produces({ "application/json" })
    CompletionStage<Greeting> greet(@Valid Name name);

We want to be able to generate method definitions like this:

    @POST
    @Consumes({ "application/json" })
    @Produces({ "application/json" })
    void greet(@Valid Name name, @Suspended AsyncResponse response);

### OpenAPI Spec Extensions

The [OpenAPI Spec](https://swagger.io/specification/) explicitly
provides the
[extensions](https://swagger.io/docs/specification/openapi-extensions/)
mechanism to allow users to extend the spec for their own
needs. Essentially, users can add specially-named attributes anywhere
in their specs, and they will be treated as "vendor extensions."

For this example, we will add a new method-level vendor extension,
`x-async-enabled`, to indicate which methods should use async
processing.

This will allow us to be use compatible asynchronous processing with
more web frameworks, and to declare on a per-method basis exactly
which methods should and should not use asynchronous processing.

### The Template Customizations

The OpenAPI Generator provides rich customization hooks. The easiest
to use are
[template](https://github.com/OpenAPITools/openapi-generator/blob/master/docs/templating.md)
overrides.

Each generator is implemented using [mustache
templates](https://en.wikipedia.org/wiki/Mustache_%28template_system%29),
specifically
[jmustache](https://github.com/samskivert/jmustache). Typically,
generators implement multiple templates that call each other. The
generator allows users to
[BYOT](https://openapi-generator.tech/docs/templating/) ("bring your
own templates") to override default templates.

In our POM, we told the generator we were going to BYOT:

    <!-- We are showing off template customization today. The custom templates are stored here. -->
    <templateDirectory>${project.basedir}/openapi/templates</templateDirectory>

Here are the templates we brought:

    $ ls openapi/templates
    apiInterface.mustache
    apiMethod.mustache
    asyncParams.mustache
    returnAsyncTypeInterface.mustache
    returnTypeInterface.mustache

Notice that these templates overlap with [the existing JavaJaxRS/spec
templates](https://github.com/OpenAPITools/openapi-generator/tree/master/modules/openapi-generator/src/main/resources/JavaJaxRS/spec). When
the generator generates code, it will find our templates instead of
the default templates, and use them instead. So in essence, each
template is a customization hook that we can plug into to generate
whatever code we need.

### Understanding the Template Customization

The default `JavaJaxRS/spec` `apiInterface.mustache` template looks like this:

    @{{httpMethod}}{{#subresourceOperation}}
    @Path("{{{path}}}"){{/subresourceOperation}}{{#hasConsumes}}
    @Consumes({ {{#consumes}}"{{{mediaType}}}"{{^-last}}, {{/-last}}{{/consumes}} }){{/hasConsumes}}{{#hasProduces}}
    @Produces({ {{#produces}}"{{{mediaType}}}"{{^-last}}, {{/-last}}{{/produces}} }){{/hasProduces}}{{#useSwaggerAnnotations}}
    @ApiOperation(value = "{{{summary}}}", notes = "{{{notes}}}"{{#hasAuthMethods}}, authorizations = {
        {{#authMethods}}{{#isOAuth}}@Authorization(value = "{{name}}", scopes = {
            {{#scopes}}@AuthorizationScope(scope = "{{scope}}", description = "{{description}}"){{^-last}},
            {{/-last}}{{/scopes}} }){{^-last}},{{/-last}}{{/isOAuth}}
        {{^isOAuth}}@Authorization(value = "{{name}}"){{^-last}},{{/-last}}
        {{/isOAuth}}{{/authMethods}} }{{/hasAuthMethods}}, tags={ {{#vendorExtensions.x-tags}}"{{tag}}"{{^-last}}, {{/-last}}{{/vendorExtensions.x-tags}} })
    {{#implicitHeadersParams.0}}
    @io.swagger.annotations.ApiImplicitParams({
        {{#implicitHeadersParams}}
        @io.swagger.annotations.ApiImplicitParam(name = "{{{baseName}}}", value = "{{{description}}}", {{#required}}required = true,{{/required}} dataType = "{{{dataType}}}", paramType = "header"){{^-last}},{{/-last}}
        {{/implicitHeadersParams}}
    })
    {{/implicitHeadersParams.0}}
    @ApiResponses(value = { {{#responses}}
        @ApiResponse(code = {{{code}}}, message = "{{{message}}}", response = {{{baseType}}}.class{{#returnContainer}}, responseContainer = "{{{.}}}"{{/returnContainer}}){{^-last}},{{/-last}}{{/responses}} }){{/useSwaggerAnnotations}}
    {{#supportAsync}}{{>returnAsyncTypeInterface}}{{/supportAsync}}{{^supportAsync}}{{#returnResponse}}Response{{/returnResponse}}{{^returnResponse}}{{>returnTypeInterface}}{{/returnResponse}}{{/supportAsync}} {{nickname}}({{#allParams}}{{>queryParams}}{{>pathParams}}{{>headerParams}}{{>bodyParams}}{{>formParams}}{{^-last}},{{/-last}}{{/allParams}});

Our customized template looks like this:

    @{{httpMethod}}{{#subresourceOperation}}
    @Path("{{{path}}}"){{/subresourceOperation}}{{#hasConsumes}}
    @Consumes({ {{#consumes}}"{{{mediaType}}}"{{^-last}}, {{/-last}}{{/consumes}} }){{/hasConsumes}}{{#hasProduces}}
    @Produces({ {{#produces}}"{{{mediaType}}}"{{^-last}}, {{/-last}}{{/produces}} }){{/hasProduces}}{{#useSwaggerAnnotations}}
    @ApiOperation(value = "{{{summary}}}", notes = "{{{notes}}}"{{#hasAuthMethods}}, authorizations = {
        {{#authMethods}}{{#isOAuth}}@Authorization(value = "{{name}}", scopes = {
            {{#scopes}}@AuthorizationScope(scope = "{{scope}}", description = "{{description}}"){{^-last}},
            {{/-last}}{{/scopes}} }){{^-last}},{{/-last}}{{/isOAuth}}
        {{^isOAuth}}@Authorization(value = "{{name}}"){{^-last}},{{/-last}}
        {{/isOAuth}}{{/authMethods}} }{{/hasAuthMethods}}, tags={ {{#vendorExtensions.x-tags}}"{{tag}}"{{^-last}}, {{/-last}}{{/vendorExtensions.x-tags}} })
    {{#implicitHeadersParams.0}}
    @io.swagger.annotations.ApiImplicitParams({
        {{#implicitHeadersParams}}
        @io.swagger.annotations.ApiImplicitParam(name = "{{{baseName}}}", value = "{{{description}}}", {{#required}}required = true,{{/required}} dataType = "{{{dataType}}}", paramType = "header"){{^-last}},{{/-last}}
        {{/implicitHeadersParams}}
    })
    {{/implicitHeadersParams.0}}
    @ApiResponses(value = { {{#responses}}
        @ApiResponse(code = {{{code}}}, message = "{{{message}}}", response = {{{baseType}}}.class{{#returnContainer}}, responseContainer = "{{{.}}}"{{/returnContainer}}){{^-last}},{{/-last}}{{/responses}} }){{/useSwaggerAnnotations}}
    {{#supportAsync}}{{>returnAsyncTypeInterface}}{{/supportAsync}}{{^supportAsync}}{{#returnResponse}}Response{{/returnResponse}}{{^returnResponse}}{{>returnTypeInterface}}{{/returnResponse}}{{/supportAsync}} {{nickname}}({{#allParams}}{{>queryParams}}{{>pathParams}}{{>headerParams}}{{>bodyParams}}{{>formParams}}{{^-last}},{{/-last}}{{/allParams}}{{#vendorExtensions.x-async-enabled}}{{#allParams.0}},{{/allParams.0}}{{>asyncParams}}{{/vendorExtensions.x-async-enabled}});

These templates are pretty dense! Let's look at the differences between the two:

    $ diff original.mustache customization.mustache
    20c20
    <     {{#supportAsync}}{{>returnAsyncTypeInterface}}{{/supportAsync}}{{^supportAsync}}{{#returnResponse}}Response{{/returnResponse}}{{^returnResponse}}{{>returnTypeInterface}}{{/returnResponse}}{{/supportAsync}} {{nickname}}({{#allParams}}{{>queryParams}}{{>pathParams}}{{>headerParams}}{{>bodyParams}}{{>formParams}}{{^-last}},{{/-last}}{{/allParams}});
    ---
    >     {{#supportAsync}}{{>returnAsyncTypeInterface}}{{/supportAsync}}{{^supportAsync}}{{#returnResponse}}Response{{/returnResponse}}{{^returnResponse}}{{>returnTypeInterface}}{{/returnResponse}}{{/supportAsync}} {{nickname}}({{#allParams}}{{>queryParams}}{{>pathParams}}{{>headerParams}}{{>bodyParams}}{{>formParams}}{{^-last}},{{/-last}}{{/allParams}}{{#vendorExtensions.x-async-enabled}}{{#allParams.0}},{{/allParams.0}}{{>asyncParams}}{{/vendorExtensions.x-async-enabled}});

There's only one line of difference! Essentially, we just added logic
to generate special async parameters using a new template if the
vendor extension is present and truthy.

So even though these templates are pretty complex, the good news is
that they're also pretty well-factored and easy to work on once you
understand the task you're trying to accomplish.

Of course, this isn't the only change. Readers are encouraged to look
at the other changes, too. But they're all pretty simple, like this
one.

### Confirming Code Generation

If we update our spec to use `x-async-enabled: true` on a method, we
do indeed get our updated code generation:

    @POST
    @Consumes({ "application/json" })
    @Produces({ "application/json" })
    void greet(@Valid Name name,@Suspended final AsyncResponse asyncResponse);

Success!

## Summary

That's it! Once you have a working code generation configuration, you
typically only need to override a couple of template updates to effect
any code generation change you want. Good luck, and happy customizing!

This configuration should work and generate useful code for generating
`AsyncResponse` methods using the OpenAPI Generator for anyone who
needs it.

## References

[This Stack Overflow
answer](https://stackoverflow.com/a/57791107/2103602) was the starting
point for this example. Thank you,
[philonous](https://stackoverflow.com/users/191466/philonous)!

The JAX-RS 2.1 spec is available
[here](https://download.oracle.com/otndocs/jcp/jaxrs-2_1-final-eval-spec/index.html)
for anyone who is curious to read it. The `AsyncResponse` asynchronous
processing paradigm is covered in section 8.2.1. The `CompletionStage` asynchronous
processing paradigm is covered in section 8.2.2.