### Iteration

In the Unite platform there are many ways to loop or iterate but they boil do to a 'ForEach' type loop and the 'While' type loops. In this section we'll go through the all the iteration pipes that are prebaked into the platform.

#### For loops

The Unite development language offers 3 different pipes for iteration. These are:
* ForEachMessagePipe
* ForEachPipe
* ForPipe

In regard to the ForEach type pipes, the main difference between each is where the data they loop from is found.  We'll go through each one and describe them in more details:

##### ForEachMessagePipe

This pipe obtains its data from  strategies that are derived from the IPullMessagesInterface and provides not only batching capabilities, but also the ability to execute items asynchronously. It's core property settings (removing the conditional settings) are:

```yaml
pipe:  Variant.Core.ForEachMessagePipe
replacements:
  PULL_MESSAGES_STRATEGY:                 # This can be strategies such as BlockBlobPullMessages
  BATCH_COMPLETED_PIPES:                  # If set this scoped pipe runs after each batch is run. It is normally used for sequence aggregation or 
                                          # bulk loading. 
  BATCH_SIZE: 10                          # This is passed to the PullMessages strategy. Depending on the strategy used it may not be honoured. 
  CORRELATION_ID:                         # A nullable id added to each spawned message
  MAXIMUM_ERROR_COUNT: 0                  # If set to -1 then a error in one message will be ignore and not stop the further processing of messages
  PRE_LOAD_COUNTER: 0                     # Can be used to preload the next batch of message
  PROCESS_MESSAGES_ASYNCHRONOUSLY: False  # If set to true then all messages in the batch will be run at once.
  SCOPED_PIPES:                           # A collection of pipes to run  
```

The IPullMesages interface denotes that all messages returned must be a spawned message of the original message. This allow for headers such as file name, path, BlocbClient, etc. to be available to the pipes that process the message. The actual contents of the message is normally store in the spawned message payload.

###### Additional message headers

As mentioned above what is passed into the scoped pipes is a spawned transient message with additional headers added. However, the pipe itself adds additional headers which can be accessed. These include:
* **Batch.BatchSize**: The BATCH_SIZE set in the pipes settings.
* **Batch.ErrorCount**: Number of errors that have occured.Only set if MAXIMUM_ERROR_COUNT > 0.
* **Batch.Loop**: The number of batches run. 
* **Batch.SequenceNumber**: The 'index' of the message. eg. 28th of 150. 
* **Batch.CorrelationId**: The CORRELATION_ID set in the pipes settings. 

> NOTE: As the spawned messages are completed they are disposed. That means that any values in its headers are also gone. Therefore, tasks like aggregation must be stored on the parent message. To store any data in the parent message add a '~' in front of the header name. This will then store that data in the parent message. '~~'  stores on the parent's parent etc.


##### ForEachPipe

This pipe uses data from either an IEnumerableStrategy, an array or IEnumerable value found in the headers or a data array set directly on the pipe.  It's core property settings (removing the conditional settings) are:

```yaml
pipe: Variant.Core.ForEachPipe
replacements:
  ENUMERABLE_STRATEGY: # a strategy derived from the IEnumerableStrategy interface. Many IPullMessages implmentation also implment the
                       # IEnumerableStrategy strategy. 
  PATH_OR_ARRAY:       # Can either be data found in the headers or a distinct value
  START_COUNTER_AT:    # If set adds a counter to each row or item processed
  ITEM_NS: Item
  SCOPED_PIPES:        # A set of scoped pipes that runs for each item
    # A header namespace to access each item in the array.
```

Unlike the ForEachMessagePipe no spawned messages are created and each item of the array is passed into the scoped pipes under the ITEM_NS header. If we take the following example:

```yaml
pipe: Variant.Core.ForEachPipe
START_COUNTER_AT: 0
PATH_OR_ARRAY:
  - firstName: Bob
    surname: Scratch
  - firstName: Iva
    surname: Itch
ITEM_NS: Person
SCOPED_PIPES:
   - pipe: Variant.Core.LogMessageLoggerPipe
     LOG_MESSAGE: Index: ${Person.__counter}, FullName: ${Person.firstName} ${Person.surname} 
     LOG_LEVEL: Debug
```

The above example uses an inline array but having the array in a header would produce the same results. 

###### Additional message headers
 The only addition header added is the ${ITEM_NS.__counter} propery and this is only added when the START_COUNTER_AT is set.


#### ForPipe

The ForPipe is an iterator used when you know exactly how many times you want to loop by. It's core property settings (removing the conditional settings) are:

```yaml
pipe: Variant.Core.ForPipe
replacements:
  START_AT: 0                      # Starts the counter at this value
  INCREMENT_BY: 1                  # Increments the counter by. Can be a negative number
  END_AT:                          # Conditional check. If INCREMENT_BY is positive then this will be the < value will be used. If INCREMENT_BY
                                   # is negative then the check will be >=.  This value is substitutable. 
  COUNTER_NAMESPACE: ForCounter    # The header the counter is stored at
  SCOPED_PIPES:                    # A collection of pipes to run
```

An example of this pipe in action using a subsitutable END_AT value can be found in the following example:

```yaml
- pipe: Variant.Core.ModifyMessageStrategyPipe
  NAMESPACE: Items
  VALUE: 
    - firstName: Ian
      surname: Hill
    - firstName: Mike
      surname: Douglas

- pipe: Variant.Core.ForPipe
  # START_AT: 1
  # INCREMENT_BY: 1
  END_AT: ${Items.Count()}
  COUNTER_NAMESPACE: ForCounter
  SCOPED_PIPES: 
    - pipe: Variant.Core.LogMessagePipe
      LOG_MESSAGE: Name ${Items.GetValueAt(${ForCounter}).firstName}
```
Which equates to:
```csharp
for (var n = 0; n < ${EndAt}; n++){
    Console.WriteLine("Name " + Items[n].furstName");
}
```

 
#### WhilePipe and DoWhilePipe loops
 
 If you want to perform an iteration a certain  amount of times then you can use either a  WhilePipe or a DoWhilePipe. Both these pipes have the same core property settings:

 ```yaml
pipe:  Variant.Core.WhilePipe
replacements:
  BREAK_EXPRESSION:                      # Optional. Break expression
  BREAK_EXECUTION_STRATEGY:              # Optional. Strategy derive from ICanExecuteStrategy
  MAX_NUMBER_OF_LOOPS: 10                # Optional. Can be used to simulate a for loop
  START_COUNTER_AT:                      # Optional. Determines what number is initially set in the COUNTER_NAMESPACE.                     
  COUNTER_NAMESPACE: WhilePipe.Counter   # The header name for the loop counter
  SCOPED_PIPES:                          # A collection of pipes to run
```

The only difference between these pipes is the DoWhilePipe scoped pipes is executed at least once. All futures example will therefor use the WhilePipe.

##### DoWhilePipe examples

##### While(n < 10){

This is similar approach

```yaml
pipe: Variant.Core.WhilePipe
BREAK_EXPRESSION: ${MyCounter} < 10
MAX_NUMBER_OF_LOOPS: 10
COUNTER_NAMESPACE!: MyCounter
SCOPED_PIPES: 
    - pipe: DoSomething Pipe    
}
```

Equates to:

```csharp
var counter = 0;
while(counter < 10){

    // Do Stuff
    
    counter++;
}
```

##### While(${SomeValue} == true)
```yaml
- pipe: Variant.Core.WhilePipe
  BREAK_EXPRESSION: ${SomeValue} == true
  MAX_NUMBER_OF_LOOPS: -1
  SCOPED_PIPES: 
    - pipe: DoSomething Pipe
```

Equates to:

```csharp
while(${SomeValue} == true){

    // Do Stuff
}
```

##### While(true)

```yaml
- pipe: Variant.Core.WhilePipe
  MAX_NUMBER_OF_LOOPS: -1
  SCOPED_PIPES: 
    - pipe: DoSomething Pipe
    - pipe: Variant.Core.BreakPipe
      CAN_EXECUTE_EXPRESSION: ${SomeValue} == true
```
Equates to:
```csharp
while(true){

    // Do Stuff
    
    if(${SomeValue} == true)
      break ;
}
```











