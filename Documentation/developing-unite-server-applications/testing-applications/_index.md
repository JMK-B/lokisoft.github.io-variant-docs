
# Testing Unite applications
- [[#Overview|Overview]]
- [[#Unite Server Testing platform framework|Unite Server Testing platform framework]]
	- [[#Unite Server Testing platform framework#UI Test results page|UI Test results page]]
	- [[#Unite Server Testing platform framework#Application test endpoints|Application test endpoints]]
	- [[#Unite Server Testing platform framework#The Variant.Strategies.Community.Test V1.0.0 extension package|The Variant.Strategies.Community.Test V1.0.0 extension package]]
- [[#Writing application tests|Writing application tests]]
	- [[#Writing application tests#Test types|Test types]]
	- [[#Writing application tests#The POST: api/integrationtests method|The POST: api/integrationtests method]]
	- [[#Writing application tests#The extension package test pipes|The extension package test pipes]]
	- [[#Writing application tests#Fire and forget|Fire and forget]]
	- [[#Writing application tests#Creating an API end to end test|Creating an API end to end test]]



# Overview

Testing applications is a must in modern software development and Unite Server provides a framework in which to do that quickly and easily. This document  explains the Unite testing framework and how the different types of tests are written in the platform. 

## Unite Server Testing platform framework

There are 3 constituent parts of the testing framework and these are: 
* **service UI  test results page:** This is a page found on all applications and is used to start the tests and display the results.
* **Service test endpoints:** These are 2 endpoints on each application created that the service page calls.
* **The 'Variant.Strategies.Community.Test' extension package:** which is in the  above endpoint calls to create the tests themselves. used this is a default test extension package which allows the application to create 'Arrange, Act and Assert' type tests

### UI Test results page

The Service UI test page is part of each application's service menu. Its job is to start tests and display the results of those tests. After you have created an application this page will look like this:

![](Pasted%20image%2020231114144056.png)

After deploying the application and clicking the 'Run tests' button  you should see something like this. 

![](Pasted%20image%2020231114144521.png)

### Application test endpoints
Each application that is created contains 2 test endpoints that the platform calls. These are normally found in the "Services -> Tests" YAML file and are defined as:

* **GET:  api/integrationtests: ** This retrieves the test results
* **POST:  api/integrationtests:** This is called to run the tests

The endpoints configuration can be seen below:

```yaml
endPoints:
    #
    # Called by system UI to retrieve the test results
    #
  - routeTemplate: api/integrationtests
    routeMethod: GET
    pipeline:
      - pipe: Variant.Community.Test.Authorize
      - pipe: Variant.Community.Test.Get
  
    #
    # Called by system UI to run tests
    #
  - routeTemplate: api/integrationtests
    routeMethod: POST
    pipeline:
      - pipe: Variant.Community.Test.Authorize
      - pipe: Variant.Community.Test.DeleteServiceRows

	    # Tests are added here
```

> [!Note] Although the tests above use the community tests extension package, this is not a requirement as any code can be used as long as they store the tests results in the Azure Storage test table found in the subscription's Azure Storage account. 

### The Variant.Strategies.Community.Test V1.0.0 extension package

This extension package provides several pipes for manging and running tests using the 'Arrange, Act and Assert'  type of test. In the packages YAML code it provides the following pipes to use in your applications test endpoints. The public pipes that are used can be seen below :

![](Pasted%20image%2020231115101319.png)
In the next section we will describe each pipe in more detail

## Writing application tests

### Test types 

The are many different types of tests that can be created by the platform: End to end test, integration tests, approval test etc.    However, as the application is always hosted in the cloud , Unit tests per se are not required. You can test individual pipes or groups of pipes as a unit of work but the traditional injection of interfaces into a pipe is not recommended. You can certainly call dummy endpoints in your tests but that is done through Unite's ability to override both the application and pipe settings. Unite is certainly more opinionated towards data in and data out tests rather than any behavioural testing.

As mention the default extension package uses an 'Arrange, Act, Assert' approach and does not include any Behavioural Driven Development( BDD) type tests. The 'Given, When, Then' approach provide will be available in the next version of the tests extension package. 

### The POST: api/integrationtests method

Before we explain how to create tests shall have a recap on where we actually create the tests. 

```yaml
 - routeTemplate: api/integrationtests
   routeMethod: POST
   pipeline:
     - pipe: Variant.Community.Test.Authorize
     - pipe: Variant.Community.Test.DeleteServiceRows

      # Add tests here 
```

As we can see there are 2 pipes that need to be added before we can start writing our test. The 'Authorize' pipe uses an Access key found in the subscription vault to secure the endpoint. The second 'DeleteServiceRows' removes any previous test results from the service.

> [!NOTE] If any additional security is added to the application, such as OAuth2, you will need to exclude the api/integrationtest path from that pipe by setting  the  CAN_EXECUTE_EXPRESSION with an expression like: ${Request.Property.LocalPath} != '/api/integrationtests'


### The extension package test pipes
There are 2 main pipes that are used when creating  tests and those are :
- pipe: Variant.Community.Test.RunTest
- pipe: Variant.Community.Test.Assert

These are defined as follows:

```yaml
 - key: Variant.Community.Test.RunTest
    value:
      pipe: Variant.Core.FireAndForgetScopedPipe
      replacements:
        TEST_NAME:
        ARRANGEMENT_PIPES: []
        ACT_PIPES: []
        ASSERTION_PIPES: []
        ON_ERROR_PIPES: 
        ON_COMPLETED_PIPES: 
      defaults:
        COPY_NAMESPACE: Defaults
        COPY_HEADERS: ${?Defaults}
        SCOPED_PIPES!:
          - pipe: _Variant.Community.Test.CreateTestTableRecord
            TEST_NAME:

          - pipe: Variant.Core.TryCatchFinallyScopedPipe
            EXCEPTION_PIPES: ON_ERROR_PIPES
            FINALLY_PIPES: ON_COMPLETED_PIPES
            SCOPED_PIPES: 
            - pipe: _Variant.Community.Test.RunArrangements
              TEST_NAME:
              ARRANGEMENT_PIPES: []

            - pipe: _Variant.Community.Test.RunActs
              TEST_NAME:
              ACT_PIPES: []

            - pipe: _Variant.Community.Test.RunAssertions
              TEST_NAME:
              ASSERTION_PIPES: []


  - key: Variant.Community.Test.Assert
    value:
      pipe: Variant.Core.BreakPipe
      replacements:
        ASSERT_EXPRESSION:
        ERROR_MSG: "Assert expression failed: ASSERT_EXPRESSION"
      defaults:
        CAN_EXECUTE_EXPRESSION: (ASSERT_EXPRESSION) == false
        EXECUTION_FLOW_STRATEGY:
          strategy: Variant.Core.DefaultExecutionFlowStrategy
          BLOCKED_WITH_OUTCOME_ID: 418
          BLOCKED_WITH_ERROR_MESSAGE: ERROR_MSG
```

There is also a 'Variant.Community.Test.RunTestWithExceptions' pipe that can be used for the testing of errors. The only difference between this and the Variant.Community.Test.RunTest' pipe is that when it runs the Act pipes it continues onto the asserts and doesn't end the pipes execution.


###  Fire and forget 
As can be seen in the above 'RunTest' pipe it is executed as a fire and forget pipe. This means that it will not have access to the original message and its value. To enable data to be passed into each test it you must set a 'Default's header in the main path and this will be copied to each test for that test to access it. To set the defaults value we use the 'ModifyMessageStrategyPipe' as seen below:

```yaml
        # Default value will allows variables to be passed into FireAndForget call
      - pipe: Variant.Core.ModifyMessageStrategyPipe
        NAMESPACE: Defaults
        VALUE: { HostUrl: "https://${Request.Header.Host}" }
```

### Creating an API end to end test

if we have the following endpoint:

```yaml
  - routeTemplate: api/test
    routeMethod: GET
    description: Test api
    pipeline:
      # Set response as string
      - pipe: Variant.Core.SetResponse
        VALUE: Hello :)
```

We can test it using the following 'RunTest' pipe:

```yaml
      - pipe: Variant.Community.Test.RunTest
        TEST_NAME: Test_Greeting
        ACT_PIPES:
          - pipe: Variant.Core.HttpPushMessageStrategyPipe
            NAMESPACE: Response
            METHOD: GET
            URL: ${Defaults.HostUrl}/api/test
            JSON_PARSE_PATH: null
       # For multiple assertions add multiple assertion pipes
        ASSERTION_PIPES:
            # The error message displayed can be overwritten using the ERROR_MSG setting
          - pipe: Variant.Community.Test.Assert
            ASSERT_EXPRESSION: >-
              "${Response}" == "Hello :)"
```
In the above example we do not set ARRANGE_PIPES and these are not needed. However Arrange pipes can be used to set any test specific message data or external data required which can then be reset using the ON_COMPLETED_PIPES setting. 

The ASSERT_EXPRESSION property must  return a Boolean value. It can contain multiple '&&' and '||' expressions and adheres to '( )' notation hierarchy.

> [!Tip]  For more examples of tests see the Tests yaml file in a newly created service. 

The full api/integrationtests endpoint would be: 
```yaml
    #
    # Called by system UI to run tests
    #
  - routeTemplate: api/integrationtests
    routeMethod: POST
    pipeline:
      - pipe: Variant.Community.Test.Authorize
      - pipe: Variant.Community.Test.DeleteServiceRows
  
        # Default value will allows variables to be passed into FireAndForget call
      - pipe: Variant.Core.ModifyMessageStrategyPipe
        NAMESPACE: Defaults
        VALUE: { HostUrl: "https://${Request.Header.Host}" }
  
        #
        # Response test
        #
      - pipe: Variant.Community.Test.RunTest2
        TEST_NAME: Test_Greeting
        ACT_PIPES:
          - pipe: Variant.Core.HttpPushMessageStrategyPipe
            NAMESPACE: Response
            METHOD: GET
            URL: ${Defaults.HostUrl}/api/test
            JSON_PARSE_PATH: null
        ASSERTION_PIPES:
          - pipe: Variant.Community.Test.Assert
            ASSERT_EXPRESSION: >-
              "${Response}" == "Hello :)"
```
