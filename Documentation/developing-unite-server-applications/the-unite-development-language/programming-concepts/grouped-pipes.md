# Code blocks / grouped pipes


Connections and endpoints possess a pipeline property containing a collection of pipes through which data flows in the form of an IVariantMessage. There are three distinct types of pipe sequences available:

* Scoped pipes
* grouped pipes
* Injected pipes

#### Scoped pipes
Scoped pipes are fundamental to the development platform as they offer a singular pipe capable of running a collection of pipes. Typically, scoped pipes are equipped with an implementation that includes specific pipe implementations capable of encompassing multiple child pipes executed sequentially. These pipes can range from simple placeholders to those containing implementations designed to perform specific actions before or after the scoped pipes are executed. Examples of such pipes include:

* **Variant.Core.DefaultScopedPipe**: Simple container with no added functionality
* **Variant.Core.CacheScopedPipe**: Pipe that allows a series of pipes to be executed and then cache the value determined by those pipes 
* **FireAndForgetScopedPipe**: 
* **TryCatchFinallyScopedPipe**: 
* **WhilePipe**: A series of pipes that are run in a loop
* **ForEachPpipe**: A series of pipes that are run in a loop

##### Examples

An example of the DefaultScopedPipe which is  purely a wrapper of additional pipes. When used in conjuction with the CAN_EXECUTE_EXPRESSION you can choose to run only its scoped pipes if it needs to. 

```yaml
- pipe: Variant.Core.DefaultScopedPipe
  CAN_EXECUTE_EXPRESSION: ${ErrorOccurred} == true
  SCOPED_PIPES: 
      # Create Error message
    - pipe: Variant.Core.ModifyMessageStrategyPipe
      NAMESPACE: ErrorMessage
      VALUE:  Error occured. ErrorId: ${ErrorId}, ErrorText: ${ErrorMessage}

      # Send error
    - pipe: Variant.Core.Http.PushPipe
      URL: http://mydomain.com/errornotifications
      NAMESPACE: Response
      METHOD: POST
      BODY: ${ErrorMessage}
```

An example of a scoped pipe that has additional functionality can be seen by the Variant.Core.CacheScopedPipe. The example below demonstrates how this can be used to cache a region specific, ';' separated string.

```yaml
- pipe: Variant.Core.InMemoryCacheScopedPipe
  CACHE_NAMESPACE: Response
  CACHE_FOR_TIME_SPAN: 00:30:00
  MEMOIZATION_KEY: ${Request.Path.region}
  SCOPED_PIPES: 
    - pipe: Variant.Core.Http.PushPipe
      NAMESPACE: Data
      URL: https://www.gov.uk/bank-holidays.json
      HEADERS: { Accept: application/json }
      JSON_PARSE_PATH: {Request.Path.region}.events
      METHOD: GET

    - pipe: Variant.Core.SetResponseWithHeaders
      RESPONSE: ${Response.SelectTokens("$..date").Join(";")}
      HEADERS: { Content-Type: text/plain }
```

#### Grouped pipes
Grouped pipes are purely an amalgamation of a series and pipes and / or scoped pipes. They offer the ability to group a set of pipes into a single pipe which exposes a subset of replacements making it perfect for reuse - basically these represent functions with the replacements see as the arguments. A grouped pipe is normally defined as a default scoped where its functionality is part of its scoped pipes. We can update the previous cached pipe example to create a callable scoped pipe where you can change the type of separator used. Grouped pipes are found under the 'pipes:' section of the YAML file and are named. 

##### Example

```yaml
      # Use a ';' to separate
    - pipe: SetResponseWithGetBankHolidaysUsingSeparator
      SEPARATOR: ;

      # Use a '|' to separate
    - pipe: SetResponseWithGetBankHolidaysUsingSeparator
      SEPARATOR: |

pipes:
  - key: SetResponseWithGetBankHolidaysUsingSeparator
    value:
      pipe: Variant.Core.InMemoryCacheScopedPipe
      replacements:
        SEPARATOR: 
      defaults:
        CACHE_NAMESPACE: Response
        CACHE_FOR_TIME_SPAN: 00:30:00
        MEMOIZATION_KEY: ${Request.Path.region}_SEPARATOR
        SCOPED_PIPES: 
          - pipe: Variant.Core.Http.PushPipe
            NAMESPACE: Data
            URL: https://www.gov.uk/bank-holidays.json
            HEADERS: { Accept: application/json }
            JSON_PARSE_PATH: {Request.Path.region}.events
            METHOD: GET

          - pipe: Variant.Core.SetResponseWithHeaders
            RESPONSE: ${Response.SelectTokens("$..date").Join("SEPARATOR")}
            HEADERS: { Content-Type: text/plain }
```

#### Injected pipes
Injected pipes are grouped pipes that allow you wrap a specific pipe or scoped pipes into some common functionality. Basically injected your pipe into already existing pipeline. As Unite is built with reuse of all components at its heart this is another mechanism to promote reuse and simplify pipes.

##### Example: Updating a file using pipe injection
There are at times you way want to update the contents of a json file in a blob in Azure Storage. There maybe different  file locations or containers that these are found in. Without injecting the update pipe(s) you would have 2 do this using encompass this updates with a fetch and write pipes. This would look like the following:

```yaml
- pipe: Variant.AzureStorage.BlockBlob.PullFilePipe
  NAMESPACE: MyFile
  CONTAINER_NAME: MyContainer
  READ_AS: String
  BLOB_NAME: outputs/myfile.txt
  ACCOUNT_KEY: $[AccountName]
  ACCOUNT_NAME: $[AccountName]

- pipe: SetIsEnabledProperty
        
- pipe: Variant.Json.ModifyJsonPathsStrategyPipe
  HEADER: MyFile
  PATHS: |
    isEnabled: ${IsEnabled}
    modifiedOn: ${DateTimeOffset.Now.ToString("yyyy-MM-dd")}

- pipe: Variant.AzureStorage.BlockBlob.PushPipe
  HEADER: MyFile
  CONTAINER_NAME: MyContainer
  BLOB_NAME: outputs/myfile.txt
  PUSH_AS_STREAM: False
  PUSH_TYPE: Upsert
  ACCOUNT_KEY: $[AccountName]
  ACCOUNT_NAME: $[AccountName]
```
As you can see there is a lot of settings that are duplicate and if you have several different files in different locations that need to be updated then you will get code bloat. Pipe injection can help with this and can be rewritten as the following grouped pipe:


```yaml
pipes:

- key: UpdateBlobFile
   value:
     - pipe: Variant.Core.DefaultScopedPipe
       replacements:
         CONTAINER_NAME: 
         BLOB_PATH: 
         STORED_AT_LOCATION: 
         INJECTED_PIPE: 
       defaults:
           ACCOUNT_KEY: $[AccountName]
           ACCOUNT_NAME: $[AccountName]
           SCOPED_PIPES: 
             - pipe: Variant.AzureStorage.BlockBlob.PullFile
               NAMESPACE: STORED_AT_LOCATION
               CONTAINER_NAME: CONTAINER_NAME
               READ_AS: String
               BLOB_NAME: BLOB_PATH
               ACCOUNT_KEY: ACCOUNT_KEY
               ACCOUNT_NAME: ACCOUNT_NAME

             - pipe: INJECTED_PIPE

             - pipe: Variant.AzureStorage.BlockBlob.PushPipe
               HEADER: STORED_AT_LOCATION
               CONTAINER_NAME: CONTAINER_NAME
               BLOB_NAME: BLOB_PATH
               PUSH_AS_STREAM: False
               PUSH_TYPE: Upsert
               ACCOUNT_KEY: ACCOUNT_KEY
               ACCOUNT_NAME: ACCOUNT_NAME
```
This can then be reused for updating all pipes across all containers. Using the following snippet:

```yaml
- pipe: UpdateBlobFile
  CONTAINER_NAME: 
  BLOB_PATH: outputs/myfile.txt
  STORED_AT_LOCATION: MyFile
  INJECTED_PIPE: 
    pipe: Variant.Core.DefaultScopedPipe
    SCOPED_PIPES:         
      - pipe: SetIsEnabledProperty
      - pipe: Variant.Json.ModifyJsonPathsStrategyPipe
        HEADER: MyFile
        PATHS: |
          isEnabled: ${IsEnabled}
          modifiedOn: ${DateTimeOffset.Now.ToString("yyyy-MM-dd")}
```

In the example we have injected multiple pipes using the DefaultScopedPipe. If there is only a single pipe the code would look like this:

```yaml
- pipe: SetIsEnabledProperty
- pipe: UpdateBlobFile
  CONTAINER_NAME: MyContainer
  BLOB_PATH:  outputs/myfile.txt
  STORED_AT_LOCATION: MyFile
  INJECTED_PIPE: 
    pipe: Variant.Json.ModifyJsonPathsStrategyPipe
    HEADER: MyFile
    PATHS: |
      isEnabled: ${IsEnabled}
      modifiedOn: ${DateTimeOffset.Now.ToString("yyyy-MM-dd")}
```
In the above example if you were only injecting the ModifyJsonPathsStrategyPipe then it would be possible to derive a specialisation of the grouped pipe that already includes the modify pipe, further reducing duplication of code and settings. This specialisation would look something like:

```yaml
- key: UpdateJsonBlobFile
  value:
   pipe: UpdateBlobFile
   replacements:
     CONTAINER_NAME: 
     BLOB_PATH: 
     STORED_AT_LOCATION: 
     PATHS: 
   defaults: 
     INJECTED_PIPE: 
       pipe: Variant.Json.ModifyJsonPathsStrategyPipe
       HEADER: STORED_AT_LOCATION
       PATHS: PATHS
```
It could be then called by:

```yaml
- pipe: UpdateJsonBlobFile
  CONTAINER_NAME: MyContainer
  BLOB_PATH:  outputs/myfile.txt
  STORED_AT_LOCATION: MyFile
  PATHS: |
    isEnabled: ${IsEnabled}
    modifiedOn: ${DateTimeOffset.Now.ToString("yyyy-MM-dd")}  
```


> NOTE: If this, and any grouped pipe, is a common pattern used in different projects then it is recommended you create your own local extension package and share the code using that
