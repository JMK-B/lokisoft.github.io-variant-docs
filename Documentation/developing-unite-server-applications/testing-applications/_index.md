#unfinished 

# Testing Unite applications

# Overview

Testing applications is a must in modern software development. As all applications created by Unite have a standard format this enables Unite to standardise testing across all applications. This document explains the Unite testing framework and how the different types of tests are written in the platform 

## Unite Server Testing platform framework

There are 3 constituent parts of the testing framework and these are: 
* **service UI  test page:** This is a page found on all applications and is used to start the tests and display the results
* **Service test endpoints:** These are 2 endpoints on the application itself implements and the development platform calls from the above test page.
* **The 'Variant.Strategies.Community.Test' extension package:** this is a default test extension package which allows the application to create 'Arrange, Act and Assert' type tests

### UI Test results page

The Service UI test page is part of each applications service menu. Its job is to start tests and display the results of those tests. After you have created an application this page will llok like this:

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


> [!Note] Although the tests above use the community tests extension package, this is not a requirement as any code can be used as long as they store the tests results in the Azure Storage test table found in the organisations dev storage account. 


### The Variant.Strategies.Community.Test extension package

This extension package provides several pipes for manging testing that can be used to create 'Arrange, Act and Assert'  type tests and for 

o enable the test results to be displayed

When an application is created from the default template,  it includes with it a couple of example / hello world type endpoints and 2 test endpoints.








Before we start though we need to understand what types of tests we can perform and  how the fact that the application runs in the hosting environment as it would in production , affects that. 



* Approval
* Unit -  Test indivudal pipes or a group of pipes . As the application being tested is already in the cloud then you can test inline items such as blobs and tables. Can in theory spin up  Azure storage environments 
* EndToEnd: Call an API endpoint
* Integrations

	Tests just an endpoint api/integrationtests


Each application 



```yaml

  - routeTemplate: api/integrationtests
    routeMethod: GET
    pipeline:
      - pipe: Variant.Community.Test.Authorize
      - pipe: Variant.Community.Test.Get

```


```yaml
```


```yaml
```


```yaml
```