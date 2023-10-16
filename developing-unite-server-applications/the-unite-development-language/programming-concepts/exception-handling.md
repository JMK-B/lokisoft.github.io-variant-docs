#### Exception handling

Exception handling in Unite uses 4 core actors:

- The PipeLine.RunScopedPipes method
- The PipeResult response
- The **TryCatchScopedPipe** & **TryCatchFinallyScopedPipe**

In order to understand how exception handling works we need to understand how pipes are run and how the results of those pipes are passed around. The following section will run through how each of these actors work together. Finally we'll end with some real world examples on how, using error handling we can provide a basic canonical response to all API calls

#### The RunScopedPipes method

All pipes in Unite are part of a collection of pipes whether it is a ScopedPipe property or the pipeline property found in connections and endpoints. The RunScopedPipes implementation is Unite's default pipe collection handler that runs all pipes in the collection synchronously, handles any BreakPipes correctly and return a single PipeResult. Asychronous pipe execution is also allowed but should be implemented in the Pipe itself. A example of this can be found in the pipes which which use the IPullMessagesStrategy.

> Note: For simplicity the Unite platform does not allow aggregated errors to pass across the scoped pipe boundaries. If this functionality is required pipe results can be aggregated to a message header inline using either an AggregationPipe, ModifyHeader pipe or a user defined pipe. The previous pipes message result can be accessed, whether the result is a Success or not, can be accessed via the '**System.PreviousPipeResult**' header.

#### The PipeResult class

Every time a pipe is called , whether it runs or not, it returns a PipeResult. In this PipeResult is an Outcome that is set to one of 3 states as seen below:

```csharp

    public enum Outcome
    {
        Success,      // Everything OK
        NotExecuted,  // The pipe wasn't executed. On none scoped pipes this is set if the condition CanExecute condition returns false.
                      // For scoped pipes this will also be set if none of the pipes in the collection executed.
        Failed,       // An error was returned
    }

```

The pipe result not only contains the Outcome of the pipe but also additional information such as any error codes, text and the instance name of the Pipe that created the result. The core of the PipeResult class can be below:

```csharp

    public class PipeResult : IPipeResult
    {

        public const int DefaultOutcomeId = 200;
        public const int DefaultFailedOutcomeId = 500;

        public bool StopProcessing { get; }         // Used to terminate pipe execution. BreakPipe can set this.
        public int OutcomeId { get; }               // This normally corresponds to an HttpStatusCode although this can be overwritten by
                                                    // setting the Response.StatusCode header.
        public string? ErrorText { get; }           // The error text
        public Exception? Exception { get; }        // Any exception that araose
        public string InstanceName { get; }         // The instance name of the pipe that returns this object.
        public Outcome Outcome { get; }             // See Outcome Enum above

        public bool IsBadMessage => OutcomeId >= 400 && OutcomeId < 500;
        public bool IsSuccessful => OutcomeId >= 200 && OutcomeId < 400;

        .....

        public override string ToString()
        {
            if (Exception == null)
                return $"PipeResult: InstanceName: {InstanceName}, Outcome: {Outcome}, OutcomeId: {OutcomeId}, ErrorText: {ErrorText}, " +
                "Exception: {Exception}";

            return $"PipeResult: InstanceName: {InstanceName}, Outcome: {Outcome}, OutcomeId: {OutcomeId}, ErrorText: {ErrorText}";
        }
     }
```

'In any pipe you can access the previous pipe result using substitutions pointing at the '**System.PreviousPipeResult**' header. To access the Outcome simple use this substitution: '${System.PreviousPipeResult.Outcome}

#### The TryCatchScopedPipe and TryCatchFinallyScopedPipe pipes

There are 2 try catch pipes within the core platform that can be used to handle errors and unexpected results. These are the **TryCatchScopedPipe** & the **TryCatchFinallyScopedPipe**. Both offer the same core functionality however, the TryCatchFinallyScopedPipe offers a scoped pipe array that is guaranteed to run after the pipe has completed there there has been any errors or not. What is returned if there is an error after the EXCEPTION_PIPES have run is determined by EXCEPTION_HANDLER_BEHAVIOR enum. The descriptions of all these can be seen below:

##### TryCatchScopedPipe

The schema for this pipe is as follows:

```yaml
pipe: Variant.Strategies.Core.TryCatchScopedPipe
replacements:
  EXCEPTION_HANDLER_BEHAVIOR: Default
  EXCEPTION_PIPES:
  SCOPED_PIPES:
```

##### TryCatchFinallyScopedPipe

```yaml
pipe: Variant.Strategies.Core.TryCatchFinallyScopedPipe
replacements:
  EXCEPTION_HANDLER_BEHAVIOR: Default
  EXCEPTION_PIPES:
  SCOPED_PIPES:
  FINALLY_PIPES:
```

##### EXCEPTION_HANDLER_BEHAVIOR

```csharp
public enum ExceptionHandlerType
{
    //  Run all exception pipes and return the original PipeResult
    Default,

    //  If the exception pipes don't return an error then set the pipe result to success
    SwallowIfHandled,

    //  Sets the pipe result to that of the EXCEPTION_PIPES
    Replace,
}
```

The following code is a pseudo representation to show what's happening when we use the TryCatchFinallyScopedPipe. The TryCatchScopedPipe is pretty much the same without the call to the Finally pipes.

```csharp
        try {
        //
            IPipeResult pipeResult = RunScopedPipes(ScopedPipes);
            if (pipeResult.IsSuccessful)
                return pipeResult;

            try {

                // The result is added to the message and can be accessed in all ExceptionPipes
                SetVariantMessage("Exception.PipeResult", pipeResult);

                var exceptionPipeResult = RunScopedPipes(ExceptionPipes);
                switch (handlerType)
                {
                    case ExceptionHandlerType.Default:
                        return pipeResult;
                    case ExceptionHandlerType.SwallowIfHandled:
                        return exceptionPipeResult.Outcome != Outcome.Failed
                            ? new PipeResult(pipeResult.InstanceName); // Creates a Success result
                            : pipeResult;
                    case ExceptionHandlerType.Replace:
                        return exceptionPipeResult;
                }
            }
            finally{
                RemoveVariantMessage(Exception.PipeResult);
            }
        }
        finally
        {
             // Finally pipes are  only found in TryCatchFinallyScopedPipe
            _ = RunScopedPipes(FinallyPipes);
        }
```

> Note: In both these pipes no exceptions are actually caught. This is because the RunScopedPipes 'call' catches all .net exceptions to avoid these crossing any boundaries between pipes.

#### Example usage

Exceptions can be caught both at both method level and a global level and at multiple places.
In the following example we can use a specialisation of the TryCatchScopedPipe to provide a clean and common approach for error handling across all projects:

```yaml
- pipe: TryCatchAndLogWrapper
    DATA: ${Request.ToString().MaxSize(n)}
    SCOPED_PIPES:
          # ForceError
        - pipe: Variant.Core.ThrowErrorBreakPipe
          BLOCKED_WITH_ERROR_MESSAGE: An error occurred
          BLOCKED_WITH_OUTCOME_ID: 400

pipes:
    # This probably should be placed in a local extension package so it can be shared across all projects proving a unified approach for
    # error handling.
  - key: TryCatchAndLogWrapper
    value:
      pipe: Variant.Core.TryCatchScopedPipe
      replacements:
        CONTINUATION_POLICY: Default
        CORRELATION_ID: ${Message.Id}
        DATA: ${Request.ToString().MaxSize(n)}
        SCOPED_PIPES:
      defaults:
        EXCEPTION_PIPES:
          - pipe: Variant.Core.LogMessagePipe
            LOG_LEVEL: Error
            LOG_MESSAGE: "MessageId: {MessageId}, InstanceName: {InstanceName}, ErrorId: {ErrorId}, ErrorText: {ErrorText}"
            LOG_MESSAGE_PARAMETERS:
              - CORRELATION_ID
              - ${Exception.PipeResult.InstanceName}
              - ${Exception.PipeResult.OutcomeId}
              - ${?Exception.PipeResult.ErrorText}

            #  Push error to Queue, Jira, table storage, etc.
            #
            # Ensure total length < storage limit
          - pipe: Variant.AzureStorage.Queue.PushPipe
            ACCOUNT_KEY: ${...AccountKey...}
            ACCOUNT_NAME: ${...AccountName...}
            CONTAINER_NAME: mycontainer
            QUEUE_NAME: ErrorQueue
            BODY:
              messageId: CORRELATION_ID
              instanceName: ${Exception.PipeResult.InstanceName}
              errorId: ${Exception.PipeResult.OutcomeId}
              errorText: ${?Exception.PipeResult.ErrorText.MaxSize(n)}
              data: DATA
```

Another common use is include wrapping any response from an API into a common structure. In the example below we updated the entry point for all api applications to return a common structure. We have also added logic that returns different error information based on which environment the code is running. As we have a common entry and exit point we have also injected the above handler so all errors across all types of projects are handled the same.

```yaml
#
# WEB API ENTRY POINT
#
- connector: Variant.AzureFunctions.AzureFunctionIsolatedConnector
  pipeline:
    - pipe: Variant.Core.TryCatchFinallyScopedPipe
      # Global Exception handler
      EXCEPTION_PIPES:
        - pipe: Variant.Core.SetResponse
          CAN_EXECUTE_EXPRESSION: ${Variant.Environment.Type} == "dev"
          CONTINUATION_POLICY: ReturnIfExecuted
          RESPONSE:
            status: failed
            error:
              Outcome: ${System.PreviousPipeResult.Outcome.ToString()}
              ErrorId: ${System.PreviousPipeResult.OutcomeId}
              Error: ${?System.PreviousPipeResult.ErrorText}

        - pipe: Variant.Core.SetResponse
          RESPONSE:
            success: false
            traceId: ${Message.Id}
            error: "An error has occurred . Please call admin quoting reference ${Message.Id}"

      # Happy path here
      SCOPED_PIPES:
        - pipe: TryCatchAndLogWrapper
          DATA: ${?CreatedHeaderContainingRelevantInputInformation}
          SCOPED_PIPES:
            - pipe: Variant.Core.WebApi.WebApiPushMessageStrategyPipe
            - pipe: Variant.Core.SetResponse
              RESPONSE:
                status: success
                result: ${Response}

      # Convert response to a json object
      FINALLY_PIPES:
        - pipe: Variant.Json.ConvertObjectToJsonStrategyPipe
          DATA_HEADER: Response
          NAMESPACE: Response
```
