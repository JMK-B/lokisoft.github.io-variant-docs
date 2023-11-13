# Specialisations and Derivatives

- [[#Pipe Specialisation|Pipe Specialisation]]
- [[#Process Specialisations|Process Specialisations]]
- [[#Process Specialisations With Implementation Injection|Process Specialisations With Implementation Injection]]
- [[#Code stack, replacements, defaults and pitfalls|Code stack, replacements, defaults and pitfalls]]
- [[## The HttpPushMessageStrategyPipe Pipe|The Variant.Core.HttpPushMessageStrategyPipe pipe]]

### Overview 

The ability to specialise not only pipelines but connectors (using connections) is Unite Servers biggest advantage over any other development platforms, especially when these are placed in an extension package and shared across projects. Specifications are basically a reuse tool that allows developers to wrap up standardised functionality into a unified pipe where none specific settings can be overridden as and when required. Example implementations include the creation of standardised pipes for creating JWT tokens or validating these token or wrapping up functionality that requires multiple pipes. Each of these can be then wrapped in a simple pipe whose use is pretty much self explanatory.

Specialisation come in 3 forms:

- Pipe specialisations
- Process specialisations
- Process specialisations with implementation injection

We shall go through each of these in detail then go further into into the code stack, replacements, defaults and describe some of the pitfalls that may arise.

> [!Note] Versioned local extension packages are a great location to store any specialisations that can be considered a cross cutting functionality. For example, this can range from common endpoint clients / pipes that  simplify sending  a message to an API or direct call to the a  queue or file system. 

### Pipe Specialisation

In each YAML file there is a property called pipes and it is here we create the specialisations. If we take the above BlockBlobPushMessagePipe pipe we can create a derivative that can automatically set default values for us so, say we have many storage accounts each with their own account keys and names we can create a new pipe and set those values as the default values and name it accordingly as seen below:

![](Images/speciailiedPipe.png)

Then we could simple use this new pipe using either of the following:

![](Pasted%20image%2020231111124317.png)

> [!Note] The differences between replacements are explains  [[#Code stack, replacements, defaults and pitfalls#Replacements & defaults|here.]] 
### Process Specialisations

Process specialisations are when multiple pipes are combined into a single callable pipe. If we take for an example we have to send an email with an attachment and all email requests have to be placed on a queue, we can't add the attachment to the queue due to the possibility it might be 2 big for the queue to fit. So what is needed is that we write the attachment to blob storage and put a reference link into the queues message. If we did this without any process or pipe specialisation the code would be something like:

![](Images/processspecialisation.png)

Would then be a single pipe:

![](Images/speciailiedProcessPipe.png)

This pipe could then be added to a extension package so in future anyone that needs to send an email simple adds the extension to the project and then simply uses this:

![](Images/processpipeSend.png)

### Process Specialisations With Implementation Injection

As we can see from above we can inject specific values such as email and paths to a pipe. However, we can also inject both specification pipes and process specification pipes. Within a process specification. An example of this would be if we wanted to read a blob from azure storage, update it and then write the contents back to blob storage. There can be various types of documents that can be updated but the general process of fetching and storing the data remains the same. The example below shows an implementation of this and how its called:

![](Images/speciailiedProcessPipeWIth%20Implmentation.png)

To use the above see below:

![](Images/speciailiedProcessPipeWIth%20Implmentation2.png)

In practise though you would be able to create another specification based on 'Variant.AzureStorage.BlockBlob.GetAndUpdateFileAsString' pipe above. This would allow the developer to sespecificic updates to specific storage accounts.

![](Images/processspecicwithinspec.png)

This then enables the user to call the pipe with the following pipe:

![](Images/speciailiedProcessPipeWIth%20Implmentation3.png)

> [!Note[If the implementation pipe you are wanting to insert contains multiple pipes you can simply set your implementation pipe as a Variant.Core.DefaultScopedPipe and then add your implementations to its SCOPED_PIPES setting.

## Code stack, replacements, defaults and pitfalls

### Code stack

Before we discuss what replacements and defaults are it helps if we explain how pipes are transformed into the actual code that is run. All pipes and strategies can be resolved to .NET code as seen below:

![](Images/LoggerCode.png)

The assemblies that contain this code are then uploaded into the platform which takes these classes and produces the following YAML file:

![](Images/LoggerYanl.png)

As seen previously we can then create several specialisations of this YAML to give us:

![](Images/loggerspecialisations.png)

These specialisation, using the replacement and default values, is then used to instantiate the C# code with all the relevant values.

### Replacements & defaults

When create specialised pipes the settings can be done as replacements or defaults. The only difference between these values are that the replacements is that when the editor provides word completion options its only the replacement values that are outputted. If you then delete the replacement line the code will automatically set that properties default value to the default replacement value. Just because a setting is under the defaults property it doesn't mean that you cannot overwrite it. Many settings that cannot be code completed are still settable. These include, but are not limited to the shared pipe settings: LOG_ERRORS, IS_ENABLED, EXECUTION_FLOW_STRATEGY, INSTANCE NAME etc.

### Pitfalls

As replacement values can override less specialised values in a pipe - a good example of this is the NAMESPACE property -care must be taken with common names across the different pipes used. This my result in unintended results. You can however ensure that a value will not be overridden by adding an exclamation mark at the end of the setting name. An example of this seen below:

![](Images/exclamationmark.png)

By adding these to the end of the ACCOUNT_NAME & ACCOUNT_KEY we can be sure that even if a pipe 'above' this pipe uses theses properties that the properties here will not override as the per the default intended use.

## The HttpPushMessageStrategyPipe Pipe
#unfinished

The HttpPushMessageStrategyPipe is basically a wrapper around the .NET HttpClient class and provides a way  to use  the functionality around that class via YAML. Specialisations are especially useful when creating derivatives of this class  as these specialisations can create entire libraries for dealing with HTTP API endpoints such as Azure Graph, Office 365, Azure Management          API, etc. 
