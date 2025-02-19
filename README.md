
[![Build Status](https://travis-ci.com/LivePersonInc/faas-client-jdk.svg?branch=develop)](https://travis-ci.com/LivePersonInc/faas-client-jdk)

# Functions-Client (Java)   ![Alt text](logo.png "Logo")

The client can be used for invoking lambdas that have been deployed on LivePerson Functions site.
It offers functionality to retrieve all lambdas and to invoke them via lambda UUID or event IDs.

For more details on LivePerson Functions & its API have a look at:

* [LivePerson Functions Overview](https://developers.liveperson.com/liveperson-functions-overview.html)

## Adding the client as maven dependency

Go to your project's pom.xml file and add as dependency.

```xml
<dependency>
  <groupId>com.liveperson.faas</groupId>
  <artifactId>functions-client</artifactId>
  <version>1.1.7</version>
</dependency>
```

## Initializing the client

Prerequisite is having an account id as every instance of this client is bound to a single account id.

Initialize the client using its builder.

```java
String accountId = "YOUR_ACCOUNT_ID";
FaasClient.Builder builder = new FaasWebClient.Builder(accountId);
```

Furthermore you have to choose a method of authorization.
Either you provide a client secret and client Id, as we use OAuth 2.0 with client credentials by default, 
or you alternatively pass your own implementation of the `AuthSignatureBilder`.

* [More information on Client Credentials](https://developers.liveperson.com/liveperson-functions-external-invocations-client-credentials.html)


```java
String clientId = "clientId";
String clientSecret = "secret";
builder.withClientId(clientId);
builder.withClientSecret(clientSecret);
or
AuthSignatureBuilder authSignatureBuilder = new YourAuthSignatureBuilder();
builder.withAuthSignatureBuilder(authSignatureBuider);
```

This is the bare minimum that has to be provided to build the client.

```java
FaasClient faasClient = builder.build();
```

### Optional fields for builder

Additionally you can pass your own implementations of specific fields to the builder before building the client.
Examples are given below.

<details><summary>RestClient</summary>
<p>

```java
RestClient restClient = new YourRestClient();
builder.withRestClient(restClient);
```

</p>
</details>

<details><summary>CsdsClient</summary>
<p>

There are two ways to set the `CsdsClient` which resolves necessary LP domains for our application.
Either you use a map whose keys represent the service names and whose values represent the domains or you implement the `CsdsClient` interface. 
Using the map reduces the calls to the CSDS endpoint. If no `CsdsClient`is provided, the library provides a default implementation, that caches the domains for two hours.

```java
CsdsClient csdsClient = new YourCsdsClient();
builder.withCsdsClient(CsdsClient csdsClient);
or
Map<String, String> csdsMap = new HashMap<String, String>;
builder.withCsdsMap(csdsMap);
```

</p>
</details>

<details><summary>isImplementedCache</summary>
<p>

You can set the IsImplementedCache for the `isImplemented` method which determines whether there are deployed lambdas that implement a given event.
By instantiating the client yourself you can set a custom caching time. Otherwise we will default back to 60 seconds. 

```java
int cachingTimeInSeconds = 30;
builder.withIsImplementedCache(new IsImplementedCache(cachingTimeInSeconds));
```

</p>
</details>

<details><summary>MetricCollector</summary>
<p>

```java
MetricCollector metricCollector = new YourMetricCollector();
builder.withMetricCollector(metricCollector);
```

</p>
</details>

## Method calls

### Preparing data for RESTful API calls

```java
String lambdaUUID = "UUID";
String externalSystem = "botStudio";
```

Optional params allow you to set a timeout for the function calls and to set a requestId. 
The default value for the timeout is 15 seconds.
If you do not provide any requestId it will be autogenerated.
The maximum for the timeout should be 35 seconds as FaaS functions time out after 30 seconds.

```java
OptionalParams optionalParams = new OptionalParams();
optionalParams.setTimeOutInMs(35000)
optionalParams.setRequestId("requestId");
```


### Fetching lambdas


**You have to use your own authentication method when fetching lambdas as it still relies on OAuth 1.0.**


<details><summary>Fetching lambdas of account</summary>
<p>

```java
try {
    // After setting the builder up, instantiate the client
    FaasClient faasClient = builder.build();

    HashMap<String,String> filterMap = new HashMap<String, String>();
    filterMap.put("state", "Draft") // Filter lambdas by state ("Draft", "Productive", "Modified", "Marked Undeployed")
    filterMap.put("eventId", FaaSEvent.ControllerBotMessagingNewConversation.toString()); // Filter lambdas by event name (also substring)
    filterMap.put("name", "lambda_substring") // Filter lambdas by name substring

    List<LambdaResponse> lambdas = client.getLambdas(userId, filterMap, optionalParams);
    ...

} catch (FaaSException e) {...}
```

</p>
</details>

### Invoking lambda by UUID

<details><summary>Invoking a lambda by UUID with response</summary>
<p>

```java
//Set request payload
User payload = new User();
payload.name = "John Doe";

//Set header
Map<String, String> headers = new HashMap<String, String>() {{
    put("Accept-Language", "en-US");
}};

//Create invocation data => Send via body during invocation
FaaSInvocation<User> invocationData = new FaaSInvocation(headers, payload);

try {
    // After setting the builder up, instantiate the client
    FaasClient faasClient = builder.build()
    User result = client.invokeByUUID(externalSystem, lambdaUUID, invocationData, User.class, optionalParams);
    //Or call it with a requestID:
    String requestId = "requestId";
    User result = client.invokeByUUID(externalSystem, lambdaUUID, invocationData, User.class, requestId, optionalParams);
    ...

} catch (FaaSException e) {...}
```

</p>
</details>

<details><summary>Invoking a lambda by UUID without response</summary>
<p>

```java
//Set request payload
User payload = new User();
payload.name = "John Doe";

//Set header
Map<String, String> headers = new HashMap<String, String>() {{
    put("Accept-Language", "en-US");
}};

//Create invocation data => Send via body during invocation
FaaSInvocation<User> invocationData = new FaaSInvocation(headers, payload);
try {
    // After setting the builder up, instantiate the client
    FaasClient faasClient = builder.build()
    client.invokeByUUID(externalSystem, lambdaUUID, invocationData, optionalParams);

} catch (FaaSException e) {...}
```

</p>
</details>

### Invoking lambda by Event

Calling by event invokes all deployed lambdas that implement the given event.
The result will therefore always be an array of objects the following structure:

```java
public class Response {
    //The invoked lambda UUID
    public String uuid;

    //The invocation timestamp
    public Date timestamp;

    //The result of the specific lambda, i.e. an instance of the User class
    //This has to changed according to your wanted return type
    public User result;
}
```

<details><summary>Invoking a lambda by event with response</summary>
<p>

```java
//Set request payload
User payload = new User();
payload.name = "John Doe";

//Set header
Map<String, String> headers = new HashMap<String, String>() {{
    put("Accept-Language", "en-US");
}};

//Create invocation data => Send via body during invocation
FaaSInvocation<User> invocationData = new FaaSInvocation(headers, payload);

try {
    // After setting the builder up, instantiate the client
    FaasClient faasClient = builder.build()

    //Check if lambdas are implemented for event
    boolean isImplemented = client.isImplemented(externalSystem, FaaSEvent.DenverPostSurveyEmailTranscript);

    if(isImplemented){
        //Invoke lambdas for event
        Response[] result = client.invokeByEvent(externalSystem, FaaSEvent.DenverPostSurveyEmailTranscript, invocationData, Response[].class, optionalParams);

        // cast to list for convenience
        List<Response> = Arrays.asList(result);
    }
    ...

} catch (FaaSException e) {...}
```

</p>
</details>

<details><summary>Invoking a lambda by event without response</summary>
<p>

```java
//Set request payload
User payload = new User();
payload.name = "John Doe";

//Set header
Map<String, String> headers = new HashMap<String, String>() {{
    put("Accept-Language", "en-US");
}};

//Create invocation data => Send via body during invocation
FaaSInvocation<User> invocationData = new FaaSInvocation(headers, payload);
try {
    // After setting the builder up, instantiate the client
    FaasClient faasClient = builder.build()

    //Check if lambdas are implemented for event
    boolean isImplemented = client.isImplemented(externalSystem, FaaSEvent.DenverPostSurveyEmailTranscript);

    if(isImplemented){
        //Invoke lambdas for event
        client.invokeByEvent(externalSystem, FaaSEvent.DenverPostSurveyEmailTranscript, invocationData, optionalParams);
    }
    ...

} catch (FaaSException e) {...}
```

</p>
</details>

<details><summary>Invoking a lambda by eventId using custom event</summary>
<p>

Custom events are events that are not yet fully acknowledged and thus not part of the FaasEvent enum. Instead you have to
provide a string.`

```java
     //Check if lambdas are implemented for event
    boolean isImplemented = client.isImplemented(externalSystem, event);

    //With return type: 
    if(isImplemented) {
        Response[] result = client.invokeByEvent(externalSystem, event, invocationData, Response[].class, optionalParams);
    }
    //Without return type:
    if(isImplemented) {
        client.invokeByEvent(externalSystem, event, invocationData, optionalParams);
    }
```

## Logging

Our Application offers logging with Log4j2. For it to work properly you will have to create a log4j2.xml file or a similar configuration file.
It should be placed in src/main/resources. For instance you could use following configuration to log to the console.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <Console name="STDOUT" target="SYSTEM_OUT">
      <PatternLayout pattern="%m%n"/>
    </Console>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="STDOUT"/>
    </Root>
  </Loggers>
</Configuration>
```

More detailed information about Log4j2 can be found [here](https://logging.apache.org/log4j/2.x/).

</p>
</details>

## Exception handling

LivePerson Functions can raise different kind of exceptions. General exceptions are wrapped in a `FaaSException`.
`FaaSDetailedException`s occur when the request has been rejected by the service. For instance if the UUID of the lambda for invocation request does not exist. In order to get more details we recommend using the `getFaaSError` method. For a list of all error codes see [here](https://developers.liveperson.com/liveperson-functions-external-invocations-error-codes.html).
`FaaSLambdaException` inherits from `FaaSDetailedException`. It is only raised during invocations, if the error is caused by the implementation of the lambda itself. For instance if the lambda returns an error on purpose or the lambda has a timeout. We recommend to monitor all the exceptions by using `e.printStackTrace()`, as this provides way more details then the error message alone. Alerting should only be done for `FaaSException`s or `FaaSDetailedException`.

```java
try {
    ...
    //Invoke lambdas for event
    Response[] result = client.invoke(externalSystem, FaaSEvent.DenverPostSurveyEmailTranscript, invocationData, Response[].class, optionalParams);
    ...

} catch (FaaSLambdaException e){
  /**
   * Lambda exceptions occur when the lambda fails due to the implementation.
   * These exceptions are not relevant for alerting, because there are no issues with the service itself.
   */
  e.printStackTrace();

  //Get Details of exception
  FaaSError faaSError = e.getFaaSError();

  faaSError.getErrorCode();
  faaSError.getErrorMsg();
}
catch (FaaSDetailedException e) {
  /**
   * Detailed Exceptions contain custom error codes that give more information.
   * These errors are relevant for alerting
   */
  e.printStackTrace();

  //Get Details of exception
  FaaSError faaSError = e.getFaaSError();

  faaSError.getErrorCode();
  faaSError.getErrorMsg();
} catch (FaaSException e){
  /**
   * All general errors are wrapped in a FaaSException (i.e. parsing the JSON response, or 401 on authentication).             *
   * These errors are relevant for alerting.
   */
  e.printStackTrace();
}
```

### Thread-Safety

Client is programmed to be thread-safe so that multiple threads can use one client without causing any problems due to shared data access.

### Collecting metrics

All you need to do to collect metrics is to provide an implementation of our interface metric collector and pass the values passed in the associated methods to your metric monitor.
The methods themselves are called in the appropriate place.

### Test setup

For the system tests to run you will have to create a .env with following values and remove the @Ignore in the file.
They are disabled by default as they are just for local testing not for the CI.


```java
ACCOUNT_ID=
SUCCESS_LAMBDA_UUID=
FAILURE_LAMBDA_UUID=
CLIENT_ID=
CLIENT_SECRET=
EVENT=
FAILURE_EVENT=
UNIMPLEMENTED_EVENT=
UNIMPLEMENTED_EVENT_AS_STRING=
EVENT_AS_STRING=
```
