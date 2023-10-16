# The Unite YAML Development Language

This article provides guidance on the Unite development language and how it is built around the reuse through object specialization. After reading this article you should have an understanding around the coding principals of Unite and the ability to build complex applications easily.

This article will be broken into the following areas:

Reuse, Reuse, reuse

## Overview

- **appsettings.json**: This corresponds to a .Net application settings file. This is where all application settings are found and is .Net app settings file with the same schema.
- **local.settings.json**: This is a development only settings file that overrides any settings found in the appsettings.json file. This file is not included in any release.
- **site-config.json**: This contains information regarding the service such as version, name, et.c It also contains any dependencies i.e. extensions.
- **service.yaml**: This is an initialisation file that adds any additional servicesthat to the project. This includes , but not limited to, additional app setting repositories, any instrumentation initialisational, etc.
- **application yaml files**. This is a group of files which contain the logic / code that a service runs. This includes any endpoints, connector, connections, pipes & strategies.

In this article we will concentrate on applications files only.

## Application files

Application files are structured YAML files that contain the service code (The schema documentation can be found [here](./schema.md)). They can contain the following property types:

- **Connectors**: An array of listener implementations
- **End points**: An API list of http endpoints
- **Pipes**: an implementation of behaviours that run in a pipeline or scoped pipes
- **Connections**: An array of key value pairs of available connectors
- **Strategies**: Where pipes have intentions i.e. Push message, pull message, etc. These are the varied implementations that pipe intention. Examples include queue push, service bus push, etyc.

### Connectors & end points

- http endpoint
- Background info

#### Connectors

#### Endpoints

Dues to there specific nature endpoints cannot be specialised although the pipes in the pipeline can be.

### Pipes, Connections and strategies

#### Pipes

#### Connections

#### Strategies

## Reuse through specialisations

### Connectors and Connections

Pipes and strategies

pipes can be considered as functions or methods

typesof pipes:
BasicPipe
ConditionalPipe:

allows different execturions paths such as if, else, return
ScopedConditionalPipes

And each of these can either be strategised or none strategised.

- BaseLokiPipe\<T>
  - LoggerPipe : BaseLokiPipe\<LoggerPipe>
- BaseScopedPipe<T>
  Base

- **Data**: This includes both configuration and transient (per call) information.

### Data

#### Configuraion data

NAMESPACE common property

## Programming constructs

Like all programming languages the Unite development language is built from 4 common building blocks:

- **Sequence**: Sequence is the order in which instructions occur and are processed.
- **Selection**: Selection determines which path a program takes when it is running.
- **Iteration**: Iteration is the repeated execution of a section of code when a program is running.
- **Exception handling**:

####################################################################################

Messages start in either a connection or endpoint and placed in a UniteMessage payload, with additional data placed in the headers, and then sent down a pipeline. Each pipe takes this message and performs an action , mostly dependent on what type of pipe it is, push pipe, pull messages pipe, modify message pipe.
Connectors and endpoints have a pipeline property whiuch is just a lists of pipes to run. These pipes are executd in a top to bottom approach with any scoped pipes i.e. grouping of pipes run inline.

pipes can be constructured in yaml code and amalgamate other series of pipes and replacements values may be set to produt a reuseable pipe with preset values applied to certqin pipes.

Methods are rarely run all

- Grouped pipes
  Amalgamation
- Injected pipes

A brief

#### CAN_EXECUTE_EXPRESSION

this property can use the substitution service to determine whether this

> Note: On every single pipe class there is the IsEnabled (IS_ENABLED) property. This can be considered a compilation type switch that takes a conditional expression and determines whether that pipe is to included when instantiating the pipeline. this is normally, but not exclusively used with the setting "Variant.Environment.Type": "dev".

- Variables
- Conditional
  - If statements
  - switch
- Loops
- Exception Handling

#### Conditionmal

## Under the hood

- BaseLokiPipe
- BaseLokiStrategy
- ConditionalBaseLokiPipe

```csharp
[InjectedSetting("Allows for more complex filtering", IsMandatory = false)]
public ICanExecuteStrategy CanExecuteStrategy { get; set; }

[InjectedSetting("Basic filtering")]
public string CanExecuteExpression { get; set; }

[InjectedSetting("Shortcut policy to control flow only and not return values", DefaultValue = Core.ContinuationPolicy.Default)]
public ContinuationPolicy ContinuationPolicy { get; set; }

[InjectedSetting("Strategy for deciding execution flow from this pipe", IsMandatory = false)]
public IExecutionFlowStrategy ExecutionFlowStrategy { get; set; }
```

```csharp
public class BreakPipe : VariantConditionalPipe<BreakPipe>
```

#### Scoped pipes

```csharp
public abstract class VariantConditionalPipe<T> : VariantPipe<T> where T : IVariantPipe
{
    [InjectedSetting("Allows for more complex filtering", IsMandatory = false)]
    public ICanExecuteStrategy CanExecuteStrategy { get; set; }

    [InjectedSetting("Basic filtering")]
    public string CanExecuteExpression { get; set; }

    [InjectedSetting("Shortcut policy to control flow only and not return values", DefaultValue = Core.ContinuationPolicy.Default)]
    public ContinuationPolicy ContinuationPolicy { get; set; }

    [InjectedSetting("Strategy for deciding execution flow from this pipe", IsMandatory = false)]
    public IExecutionFlowStrategy ExecutionFlowStrategy { get; set; }
```

###

```csharp
public interface ILokiPipe : IDebuggerDescriptor
{
    IAppServices AppServices { get; }
    string IsEnabled { get; }
    Task InitialiseAsync(IAppServices appServices);
    Task<IPipeResult> ExecuteAsync(IVariantMessage variantMessage, CancellationToken cancellationToken);
}
```
