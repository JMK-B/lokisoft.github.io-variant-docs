### Selection

All pipes, bar the debugging / diagnostic pipes, have the ability to determine whether they should be run or not. This functionality is added when they are derived from the VariantConditionalPipe or VariantConditionalScopedPipe (if they are a scoped pipe). Conditional pipes have 4 additional settings that can be used to control the execution flow and any return values. These settings are:

- CAN_EXECUTE_EXPRESSION
- CAN_EXECUTE_STRATEGY
- EXECUTION_FLOW_STRATEGY
- CONTINUATION_POLICY

The rest of this section gives a description of the settings and a representation on how the code would run if written in a more traditional language such as c#.

> NOTE: in the code examples below exceptions are seen to be caught and thrown. These are there to make the code easier to understand. Under the covers Unite uses a PipeResult structure to determine flow as exceptions can be detrimental to performance and throughput..

##### The CAN_EXECUTE_EXPRESSION setting

This setting is a string that is commonly used in conjunction with substitution, substitution methods and app settings can can be seen to be the contents of an 'if' statement. E.g.:

```yaml
# Simple substitutions & substitution methods
CAN_EXECUTE_EXPRESSION: true
CAN_EXECUTE_EXPRESSION: ${?ExistingItem} == null
CAN_EXECUTE_EXPRESSION: ${?MyCounter} > 0
CAN_EXECUTE_EXPRESSION: ${?MyCounter} <= 69
CAN_EXECUTE_EXPRESSION: ${?ExistingItem} == null ||  ${?ExistingItem.ChangeType} == "edit"
CAN_EXECUTE_EXPRESSION: ${Inputs.yamlServicesPath.FileExists()}
CAN_EXECUTE_EXPRESSION: ${Inputs.areaName} == ${Inputs.pageName}"
CAN_EXECUTE_EXPRESSION: ${Inputs.pageType.Contains("yaml")}
CAN_EXECUTE_EXPRESSION: ${Request.Property.AbsolutePath.Contains("api/static")} == false

# Using AppSettings
CAN_EXECUTE_EXPRESSION: $[MyAppSetting] == "dev"
CAN_EXECUTE_EXPRESSION: $[IsFeatureEnabled] == true
CAN_EXECUTE_EXPRESSION: ${IsFeatureEnabled} == true
CAN_EXECUTE_EXPRESSION: ${?MyAppSetting} != null

# Nested expressions
CAN_EXECUTE_EXPRESSION: ($[MyAppSetting] == "dev" && ${IsDebug} == true) || ${FeatureEnabled} == true
```

This can be translated into the following:

```csharp
// CAN_EXECUTE_EXPRESSION
if(CAN_EXECUTE_EXPRESSION == null || CAN_EXECUTE_EXPRESSION == true){
    RunPipe();
}

// Run next sibling pipe
```

##### Example

This example sets the Response header only if the MySetting header is not null

```yaml
- pipe: Variant.Core.ModifyMessageStrategyPipe
  CAN_EXECUTE_EXPRESSION: ${MySetting} != null
  NAMESPACE: Response
  VALUE: The Response is set to this message 'MySetting' is not null
```

##### The CAN_EXECUTE_STRATEGY setting

These are strategies that are derived from the ICanExecuteStrategy interface and allows for a more business specific run conditions. Whenever the CAN_EXECUTION_EPRESSION is used Unite wraps this in the DefaultCanExecuteStrategy.

Other other examples of these include:

- InMemoryIdempotentCanExecuteStrategy
- ThrottledCanExecuteStrategy

> NOTE: This setting and the CAN_EXECUTE_EXPRESSION are mutually exclusive and if it is set then the CAN_EXECUTE_EXPRESSION will be ignored.

##### Example

This example restricts the same error from being sent to an error notification endpoint within a minimum of a 5 minute period.

```yaml
- pipe: Variant.Core.Http.PushPipe
  NAMESPACE: Response
  URL: http://mydomain.com/errornotifications
  METHOD: POST
  CAN_EXECUTE_STRATEGY:
    strategy: Variant.Core.InMemoryIdempotentCanExecuteStrategy
    KEY: ${ErrorId}
    HEADER_LIFETIME: 00:05:00
    HEADER_LIFETIME_CHECKER: 00:05:00
  BODY:
    Error: ${ErrorId}
```

##### The EXECUTION_FLOW_STRATEGY setting

Provides a granular approach for setting return values when the continuation has been blocked. These are:

- BlockedWithOutcomeId
- BlockedWithErrorMessage
- BlockWithOutcome
- ContinuationPolicy

BreakPipes are a primary way the platform manages execution flow and returning error messages and error codes. Examples can be seen in the next section [BreakPipes](#breakpipes)

##### The CONTINUATION_POLICY setting

This property determines is the core property that determines what the flow is dependent on the result of the pipe. There are 4 values that the property can be and these can be seen from the following enum:

```csharp
public enum ContinuationPolicy
{
    // Allows continuation when not executed
    Default,

    // Returns immediately if the pipe is executed
    ReturnIfExecuted,

    // Allows continuation when pipe execution returns Outcome.Error
    ContinueOnError,

    // Ends execution on any other pipes in the scoped pipe if call is successful
    ReturnOnSuccess,
}
```

###### CONTINUATION_POLICY: Default

```csharp
//CONTINUATION_POLICY: Default
if(CAN_EXECUTE_EXPRESSION == null || CAN_EXECUTE_EXPRESSION == true || (CAN_EXECUTE_STRATEGY != null &&  CAN_EXECUTE_STRATEGY.CanExecute())){

    try { RunPipe();  }
    catch (Exception ex) { throw ; }
}

// Run next sibling pipe
```

###### CONTINUATION_POLICY: ReturnIfExecuted

```csharp
//CONTINUATION_POLICY: ReturnIfExecuted
if(CAN_EXECUTE_EXPRESSION == null || CAN_EXECUTE_EXPRESSION == true || (CAN_EXECUTE_STRATEGY != null &&  CAN_EXECUTE_STRATEGY.CanExecute()){
    try {
        RunPipe();
        return ;
    }
    catch () { throw ; }
}

// Run next sibling pipe
```

> NOTE: When used in conjunction with a DefaultScopedPipe you can get the same functionality as a switch statement.

###### CONTINUATION_POLICY: ContinueOnError

```csharp
//CONTINUATION_POLICY: ContinueOnError
if(CAN_EXECUTE_EXPRESSION == null || CAN_EXECUTE_EXPRESSION == true || (CAN_EXECUTE_STRATEGY != null &&  CAN_EXECUTE_STRATEGY.CanExecute()){
    try { RunPipe(); }
    catch { }
}

// Run next sibling pipe
```

###### CONTINUATION_POLICY: ReturnOnSuccess

```csharp
//CONTINUATION_POLICY: ReturnOnSuccess
if(CAN_EXECUTE_EXPRESSION == null || CAN_EXECUTE_EXPRESSION == true || CAN_EXECUTE_STRATEGY.CanExecute()){
    try {
        RunPipe();
        return ;
    }
    catch (Exception) {  }
}

// Run next sibling pipe
```

#### Break Pipes

The Variant.Core.BreakPipe is a derivative of the VariantConditionPipe and is another way to stop the execution of any further sibling pipes. It is often used in conjunction with the CAN_EXECUTE_EXPRESSION as can be seen below:

```yaml
pipe: BreakPipe
CAN_EXECUTE_EXPRESSION: ${SendEmail} == false
```

This equates to :

```csharp
if(CAN_EXECUTE_EXPRESSION== null || CAN_EXECUTE_EXPRESSION == true || CAN_EXECUTE_STRATEGY.CanExecute()){
   return;
}

// Run next sibling pipe
```

> NOTE: When this pipe is used within a foreach or while loop then this will end the execution of that pipe and either return or continue onto the next pipe dependent on the value of the CONTINUATION_POLICY

As this pipe is a derivative of the VariantConditionPipe we can set the EXECUTION_FLOW_STRATEGY and create specialised break pipes that return specific errors such as:

- Variant.Core.ValidationBreakPipe
- Variant.Core.NotImplementedBreakPipe
- Variant.Core.ThrowErrorBreakPipe

These pipes are YAML only specialisations and only differ by the error message and the outcome id. They do however offer fixed pipe implementations whose intentions are clearly defined.

```yaml
#
# Break pipes
#
- key: Variant.Core.ValidationBreakPipe
  value:
    pipe: Variant.Core.BreakPipe
    replacements:
      VALIDATION_EXPRESSION:
      BLOCKED_WITH_ERROR_MESSAGE: Something's not right
      BLOCKED_WITH_OUTCOME_ID: 400
    defaults:
      CAN_EXECUTE_EXPRESSION: (VALIDATION_EXPRESSION) == false
      EXECUTION_FLOW_STRATEGY:
        strategy: Variant.Core.DefaultExecutionFlowStrategy

- key: Variant.Core.NotImplementedBreakPipe
  value:
    pipe: Variant.Core.BreakPipe
    replacements:
      BLOCKED_WITH_ERROR_MESSAGE: Feature not yet implemented
      BLOCKED_WITH_OUTCOME_ID: 400
    defaults:
      CAN_EXECUTE_EXPRESSION: true
      EXECUTION_FLOW_STRATEGY:
        strategy: Variant.Core.DefaultExecutionFlowStrategy

- key: Variant.Core.ThrowErrorBreakPipe
  value:
    pipe: Variant.Core.BreakPipe
    replacements:
      BLOCKED_WITH_ERROR_MESSAGE: An error occurred
      BLOCKED_WITH_OUTCOME_ID: 400
    defaults:
      CAN_EXECUTE_EXPRESSION: true
      EXECUTION_FLOW_STRATEGY:
        strategy: Variant.Core.DefaultExecutionFlowStrategy
```

These are equivalent to the following code:

```csharp
if(CAN_EXECUTE_EXPRESSION== null || CAN_EXECUTE_EXPRESSION == true || CAN_EXECUTE_STRATEGY.CanExecute()){
   throw Exception("An error occured", 400);
}

// Run next sibling pipe
```
