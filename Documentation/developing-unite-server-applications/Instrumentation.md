# Instrumentation  using OpenTelemetry

## Overview

Instrumentation in software engineering involves embedding additional code within a software system to enable the monitoring and measuring of its behaviour. This process is essential for understanding how a system operates under various conditions, identifying performance bottlenecks, and detecting anomalies. Techniques for instrumentation vary from manual code integration to utilizing profiling tools or middleware for automatic data capture. Instrumentation plays a critical role in several aspects of software management, such as performance monitoring, debugging, and enhancing the reliability and efficiency of applications. By providing insights into system functionality, it enables developers and operators to maintain high system performance and quickly resolve issues.

> [!Note] This document does not provide an indepth look at instrumentation prinipals but instead concentrates on how to output instrumentation using the Unite platform.

]## OpenTelemetry Background

OpenTelemetry is an observability framework designed for cloud-native software. It provides a single set of APIs, libraries, agents, and instrumentation that allow you to collect and export telemetry data (metrics, logs, and traces) from your applications and services.  for Data collection It supports the collection of three primary types of telemetry data:    
    - **Traces**: Information about the execution path through a system and the latency of each component or service involved.
    - Events: Events 
    - **Metrics**: Numerical data that represents the state of the system at a point in time, like memory usage, CPU load, etc.
    - **Log**s: At the present time OpenTelemetry logs is in beta so Unite uses Microsoft's internal logging APIs using Serilog. 

### Azure Application Insights for data collection

Although OpenTelemetry provides the infrastructure for data collection where that data is stored is platform agnostic. However, Unite uses Azure's monitoring services such as Application Insights to store this data and is automatically configured. Application Insights is used as the default storage for instrumentation although, extension packages will soon be available  available for other platforms  that can be configured in the services.yaml file. 

### Adding instrumentation using activities

Activities are core to adding instrumentation to applications and as the storage of logs and instrumentation is already configured in the application,  in Unite its as simple as adding the ACTIVITY_NAME setting to either a connector or pipe.  Doing this provides us with 3 different types of instrumentation categories:

* Basic instrumentation
* Domain specific instrumentation
* Http instrumentation

#### Basic Instrumentation

If we look at the following  we can see that all we have done is  added the activity name to the end point and 2 pipes. 

![](Pasted%20image%2020231114103956.png)

In Application insights the following information is stored:

![](Pasted%20image%2020231114104307.png)

With the  Json.ConvertAndValidateRequest storing the basic information

![](Pasted%20image%2020231114104423.png)

Although this is helpful for flow it can be improved slight by adding additional information

#### Adding Domain specific Information

Adding custom properties is done via the Variant.Core.UpdateActivityTraceStrategyPipe with the properties seen below

```yaml
      - pipe: Variant.Core.UpdateActivityTraceStrategyPipe
        # Allows the storage of an activities status code
        ACTIVITY_STATUS_CODE: null
        # These are the custom properties and are stored as key/value pairs
        TAGS: null
        # This is the baggage and again stored as key/value pairs
        BAGGAGE: null
```


If we take the previous example if might be useful if we stored the request and response data as well.  This can be done by adding the following pipe to the end of the pipeline:
```yaml
  - pipe: Variant.Core.UpdateActivityTraceStrategyPipe
	TAGS: 
	  chat: ${Request.chat}
	  id: ${?Request.id}
	  responseChat: ${Response.response}
	  responseId: ${Response.id}
```

This then gives us the following information in the logs:
![](Pasted%20image%2020231114110133.png)

> [!Note] The above example shows us the benefits of instrumentation. As we can see that the actual call took over 28 seconds to return. If we look at the output we can see it was the call to ChatGpt which took the majority of the time.  

#### Adding Http instrumentation

Although we now have good domain specific instrumentation we can provide extra information on every HTTP call that is made during our request. To do this we need ensure the 'Variant.Instrumentation.AddHttpClient' setting is set to true as below (default is false): 

![](Pasted%20image%2020231114113108.png)

Now when we call the service we get the following trace created:

![](Pasted%20image%2020231114113522.png)

This now shows that are out ChatGPT call performs 2 Http client calls to booth the ChatGPT API endpoint and then to an Azure Storage  blob container to store the conversation. If we add an id to the request we can see that it first makes a call to return the conversation from storage before calling the endpoint :

![](Pasted%20image%2020231114114220.png)