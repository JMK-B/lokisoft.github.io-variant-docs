# State

## Overview

Whenever the service recieves an API call or an event occurs e.g. item added to a queue or timer event, a message is created. This message is sent through the specific connectors pipeline with the relevent data attached to the headers or payloads of the message. This section describes how the message is passed through the pipeline and the common inbuilt pipes that can manipulate the message.

## The ILokiMessage interface

ILokiMessage is the interface of the main message that is ubiquitous throughout unite server applications. Everything that is to do with a single message is stored in a LokiMessage class as well as other properties for syncing and transactional  support across cloned messages. The interface is structured as below:

![](Pasted%20image%2020231113111549.png)

The three main areas of the messages are:

- Headers: These contain an enumeration of key-value pairs and enable routing, message splitting and contain any additional properties of the message.

- Payload: This is where the actual data of the message is stored. LokiMessage can actually handle multiple data items - called message parts - and Payload is the first one in the list.

- CreateSpawnedMessage() & IsSpawnedMessage: When a LokiMessage is created it may create other LokiMessages - PullMessages is a good example of this - and may need to enumerate them. Each additional message should be tied to its creator, for correlation purposes, and its this method and property which allows it.


## The ModifyMessageStrategy

### .Net core implementation

The core .NET strategy that allows LokiMessage manipulation is the ModifyMessageStrategy class. Its core structure can be seen below:

![](Pasted%20image%2020231113112657.png)

The class adds 5 additional injected properties that are used in the manipulation of the LokiMessage. These are:

* Namespace: This determines which header the value is stored. If this is null then the value will be stored ad the payload. If this is not set in the YAML then it will use the default value of Response.
* Value: This is the value that the namespace is set to. This can either be a string, a substitution value  or JSON / YAML value which itself can contains substitutions at any level.
* TempValues: This is a temporary storage area that can be used to pre-process certain values to save having multiple MessageModifyStrategyPipes. To access any values found here you can use the following Substiitution header ${TampValues.yourproperty}. Temp values are removed after the strategy has processed.
* AppendValue: If this is true then the object added to the Namespace is wrapped in an array. If the Namespace is already an array then the object will be added to it. 
* ForceDeepCopyOfJToken: This is only used if the Value property is a JToken and stops the pollution of the  original value. If the Value is an array which contains substitutions then this value will automatically be set to true.  

> [!NOTE] The Response header is used primary by the Http connectors to return data from any API call that requires a return value.  The default behaviour of the connector (found in the Startup YAML file) is normally to convert any value found in this header to a JSON object using the 'Variant.Json.ConvertObjectToJsonStrategyPipe'    


### YAML implementations

The Unite runtime contains several different specialisations of the ModifyMessageStrategy pipe implementations. These implementations can be found below: 
#### Default implementation

``` yaml
  - key: Variant.Core.ModifyMessageStrategyPipe
    value:
      pipe: Variant.Core.ModifyMessagePipe
      replacements:
        # Pipe replacements
        CAN_EXECUTE_EXPRESSION: null
        CAN_EXECUTE_STRATEGY: null
        CONTINUATION_POLICY: Default

		# Strategy replacements
        VALUE: 
        TEMP_VALUES: 
        NAMESPACE: Response
        APPEND_VALUE: False
        FORCE_DEEP_COPY_OF_J_TOKEN: False

      defaults:
        LOG_ERRORS: True
        ACTIVITY_NAME: 
        EXECUTION_FLOW_STRATEGY: null
        MODIFY_MESSAGE_STRATEGY:
          strategy: Variant.Core.ModifyMessageStrategy
```


#### Specialisations

The Unite runtime contains several different specialisations of the ModifyMessageStrategy pipe implementations. These can be split into 2 groups: Default specialisations and API helper specialisations. These pipe implementations are :
* General specialisations 
	* Variant.Core.ModifyMessageArrayStrategyPipe
	* Variant.Core.SetHeader
	* Variant.Core.AppendHeader
* API helper specialisations
	* Variant.Core.SetResponse
	* Variant.Core.SetResponseHeaders
	* Variant.Core.SetResponseStatusCodeAndHeaders

```yaml
  #
  # Modifiy Message
  #
  - key: Variant.Core.ModifyMessageArrayStrategyPipe
    value:
        pipe: Variant.Core.ModifyMessageStrategyPipe
        replacements:
            CAN_EXECUTE_EXPRESSION: null
            CONTINUATION_POLICY: Default
            VALUE: null
            TEMP_VALUES: null
            NAMESPACE: Response
        defaults: 
            APPEND_VALUE: true

  - key: Variant.Core.AppendHeader
    value:
        pipe: Variant.Core.ModifyMessageStrategyPipe
        replacements:
            CAN_EXECUTE_EXPRESSION: null
            CONTINUATION_POLICY: Default
            VALUE: null
            TEMP_VALUES: null
            NAMESPACE: Response
        defaults: 
            APPEND_VALUE: true

  

  - key: Variant.Core.SetHeader
    value:
      pipe: Variant.Core.ModifyMessageStrategyPipe
      replacements:
        NAMESPACE: Response
        VALUE: null
        CAN_EXECUTE_EXPRESSION: null
        CONTINUATION_POLICY: Default

  - key: Variant.Core.SetResponse
    value: 
      pipe: Variant.Core.ModifyMessageStrategyPipe
      replacements:
        RESPONSE: 
      defaults:
        NAMESPACE: Response
        VALUE: RESPONSE

  - key: Variant.Core.SetResponseHeaders
    value: 
      pipe: Variant.Core.ModifyMessageStrategyPipe
      replacements:
        HEADERS: 
      defaults:
        NAMESPACE!: Response.Headers
        VALUE: HEADERS


   ### Description: This can only set status which are < 300. Use ThrowErrorBreakPipe instead if any errors need to be returned  

  - key: Variant.Core.SetResponseStatusCodeAndHeaders
    value: 
      pipe: Variant.Core.DefaultScopedPipe
      replacements:
        HEADERS: null
        STATUS_CODE: null
        RESPONSE: null

      defaults:
        SCOPED_PIPES: 
          - pipe: Variant.Core.ModifyMessageStrategyPipe
            NAMESPACE!: Response.Headers
            VALUE!: HEADERS

          - pipe: Variant.Core.ModifyMessageStrategyPipe
            NAMESPACE!: Response.StatusCode
            VALUE!: STATUS_CODE

          - pipe: Variant.Core.ModifyMessageStrategyPipe
            NAMESPACE!: Response
            VALUE!: RESPONSE
```