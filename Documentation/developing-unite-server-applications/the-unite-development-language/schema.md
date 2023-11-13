# UNITE YAML DEVELOPMENT LANGUAGE SCHEMA REFERENCE

This document outlines a general structure and explains the default fields of the provided Unite services YAML file and the core default properties found in those types.

- [[#YAMl Schema]]
	- [[#SCHEMA#EndPoints|EndPoints]]
	- [[#SCHEMA#Connections|Connections]]
	- [[#SCHEMA#Connectors|Connectors]]
	- [[#SCHEMA#Pipes|Pipes]]
	- [[#SCHEMA#Strategies|Strategies]]
- [[#Variant Core Pipes|Variant Core Pipes Schemas]]
	- [[#Variant Core Pipes#Variant Pipe|Variant Pipe]]
	- [[#Variant Core Pipes#Variant Conditional Pipe|Variant Conditional Pipe]]
	- [[#Variant Core Pipes#Variant Conditional Scoped Pipe|Variant Conditional Scoped Pipe]]
	- [[#Variant Core Pipes#ContinuationPolicy|ContinuationPolicy]]

## SCHEMA

### EndPoints

This array holds the definitions of all the endpoints the application exposes.

| Key           | Type             | Required | Description                                                                                                                                                                                                        |
| ------------- | ---------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| routeTemplate | string           | Yes      | The URL path for the endpoint. It can include path parameters encapsulated by `{xxx}` (use {\*xxx} to grab the following path in full). These values are accessed by the following notation: '${Request.Path.xxxx} |
| routeMethod   | string           | Yes      | The HTTP method for the endpoint. Allowed values are `GET`, `POST`, `PUT`, `PATCH`, `HEAD`, `DELETE` & `OPTIONS`.                                                                                                  |
| description   | string           | No       | A brief description of the endpoint's purpose.                                                                                                                                                                     |
| response      | OpenApi response | No       | Matches the OpenApi response format                                                                                                                                                                                |
| queries       | object           | No       | Defines the expected query parameters. These values are accessed by the following notation: '${Request.Query.xxxx}                                                                                                 |
| headers       | object           | No       | Defines the expected request headers. These values are accessed by the following notation: '${Request.Header.xxxx}                                                                                                 |
| request       | object           | No       | Defines the expected request body.                                                                                                                                                                                 |
| pipeline      | array            | Yes      | An array of pipes which get executed in sequence. Each item in the array should correspond to a pipe defined in the `pipes` collection.                                                                            |

'Object' type above corresponds to properties written as:

```yaml
headers:
  name: string (Required)
  mobile?: string (Optional)
  age: number (Required)
```

### Connections

This array holds the definitions of all the connectors that runs in the application

| Key          | Type   | Required | Description                                                                                                                             |
| ------------ | ------ | -------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| connector    | string | Yes      | The name of the connector whose key is found in the connectors array.                                                                   |
| replacements | object | No       | The values that should be used to replace placeholders in the connector.                                                                |
| defaults     | object | No       | An object of default values and configurations for the connector.                                                                       |
| pipeline     | array  | Yes      | An array of pipes which get executed in sequence. Each item in the array should correspond to a pipe defined in the `pipes` collection. |

### Connectors

This array holds a list of specialised templated connections that can be used in the connections section.

| Key   | Type   | Required | Description                             |
| ----- | ------ | -------- | --------------------------------------- |
| key   | string | Yes      | The unique identifier of the connector. |
| value | object | Yes      | The details about the connector.        |

The `value` object schema:

| Key          | Type   | Required | Description                                                              |
| ------------ | ------ | -------- | ------------------------------------------------------------------------ |
| connector    | string | Yes      | The name of the connector.                                               |
| replacements | object | No       | The values that should be used to replace placeholders in the connector. |
| defaults     | object | No       | An object of default values and configurations for the connector.        |

### Pipes

This array holds the definitions of all the pipes used in the application. See Variant Core pipes for list of values the can be placed in the replacements or defaults properties.

| Key   | Type   | Required | Description                        |
| ----- | ------ | -------- | ---------------------------------- |
| key   | string | Yes      | The unique identifier of the pipe. |
| value | object | Yes      | The details about the pipe.        |

The `value` object schema:

| Key          | Type   | Required | Description                                                         |
| ------------ | ------ | -------- | ------------------------------------------------------------------- |
| pipe         | string | Yes      | The name of the pipe.                                               |
| replacements | object | No       | The values that should be used to replace placeholders in the pipe. |
| defaults     | object | No       | An object of default values and configurations for the pipe.        |

### Strategies

This array holds the definitions of all the strategies used in the application.

| Key   | Type   | Required | Description                            |
| ----- | ------ | -------- | -------------------------------------- |
| key   | string | Yes      | The unique identifier of the strategy. |
| value | object | Yes      | The details about the strategy.        |

The `value` object schema:

| Key          | Type   | Required | Description                                                             |
| ------------ | ------ | -------- | ----------------------------------------------------------------------- |
| strategy     | string | Yes      | The name of the strategy.                                               |
| replacements | object | No       | The values that should be used to replace placeholders in the strategy. |
| defaults     | object | No       | An object of default values and configurations for the strategy.        |

Please note that the `endPoints` section has a blank pipeline, and no specific endpoint properties (like `response`, `queryParameters`, `headers`, `request`) are defined. These properties need to be defined based on the specific requirements of the endpoints. Similarly, the `value` objects of the `connectors`, `pipes`, and `strategies` sections have placeholders that need to be replaced based on the specific implementations.

## Variant Core Pipes

There are 3 different pipe definitions / types:

- **VariantPipe**: All pipes are themselves derived from this pipe
- **VariantConditionalPipe**: Adds conditional processing to the VariantPipe
- **VariantScopedConditionalPipe**: A pipe that has a collection of child pipes. Itself is derived from VariantConditionalPipe.

All of the keys below can be placed under the replacements or defaults property of its pipe.

> Note: All pipes when executed return either: Success, NotExecuted or Failed.

### Variant Pipe

| Key           | Type   | Required | Description                                                                                                                  |
| ------------- | ------ | -------- | ---------------------------------------------------------------------------------------------------------------------------- |
| INSTANCE_NAME | string | No       | A description that is used when logging or throwing errors. If not present then a the name of the type will be used instead. |
| LOG_ERRORS    | bool   | Yes      | When implemented and true, the pipe should not write any errors to the logging system                                        |

### Variant Conditional Pipe

Derived from VariantPipe and includes includes all the properties that that exposes.

| Key                     | Type               | Required | Description                                                                                                            |
| ----------------------- | ------------------ | -------- | ---------------------------------------------------------------------------------------------------------------------- |
| CAN_EXECUTE_EXPRESSION  | string             | No       | The unique identifier of the strategy. e.g. "${MyValue}" == true                                                        |
| CAN_EXECUTE_STRATEGY    | strategy           | No       | A strategy that is derived from the ICanExecuteStrategy interface. Allows for a more business specific run conditions. |
| EXECUTION_FLOW_STRATEGY | strategy           | No       | Provides a granular approach for return values: outcome, error message & error id dependent on the ContinuationPolicy. |
| CONTINUATION_POLICY     | ContinuationPolicy | No       | Determines execution flow dependent on pipe result                                                                     |

### Variant Conditional Scoped Pipe

Derived from VariantConditionalPipe and includes includes all the properties that that exposes.

| Key          | Type  | Required | Description                 |
| ------------ | ----- | -------- | --------------------------- |
| SCOPED_PIPES | array | Yes      | A list of pipes to execute. |

### ContinuationPolicy

| Key              | Type   | Description                                                                                             |
| ---------------- | ------ | ------------------------------------------------------------------------------------------------------- |
| Default          | string | Execution continues unless there are errors. On an error the pipeline execution stops at that pipe line |
| ReturnIfExecuted | string | If the pipe runs then no other peer pipes are executed in the pipeline                                  |
| ReturnIfNotExecuted | string | If the pip is not executed then no other peer pipes are executed in the pipeline                                  |
| ReturnOnSuccess  | string | Stops execution of any further pipes in the pipeline if the pipe was successful. i.e. Outcome != Failed |
| ContinueOnNotExecuted  | string |Allows continuation when pipe execution returns NotExecuted.                                                |
| ContinueOnError  | string | Continues onto the next pipe even if the pipe errored.                                                  |


