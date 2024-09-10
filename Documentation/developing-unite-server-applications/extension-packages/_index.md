
# Extension Packages

## Overview

Extension packages are code libraries that can imported into your application and provide a set of predefined connectors, pipes & strategies. These artifacts can then be directly called by your application.  There are 2 different scopes of extension packages: public and scoped.

**Public extension packages** are created  by external developers and provide a set of tested functionality around specific areas. Example of these include libraries that handle file system calls, compression algorithms such as zip, Azure Management functionality, etc.  

**Local extension packages**** or private packages are available only to your subscription and can be shared across all your applications. 

To import, or update, different packages into an application you can goto the Extension link in the service, search for the functionality required and then click on Import : 

![](Pasted%20image%2020231120101324.png)

## Local extension package types

Local extensions fall into 2 different types: those which are created using the the UI and contain only YAML files; and those which are imported as a zip file into the UI and can contain both YAML and .NET core assembly files.  The latter enables organisations to take advantage of all the features of the platform whilst enabling domain specific code to be integrated into their applications.

### Creating local extension packages  

* [Creating a UI based extension package](_index%201.md)
* [Create a Zip based extension package](creating-zip-based-extension-packages/_index.md)





**YAML only extension packages**, as the name suggests, contain only YAML files. Because of this they can be created directly in the UI and are great for building reusable, versioned components that can be used across all applications in your subscription. They can contain:  endpoints, connections and specialised  connectors, pipes and strategies. See the  [Specialisations and Derivatives](specialisations-and-derivatives.md) section for help building fully reusable components. 

**.NET Assemblies with YAML extension packages** 


#unfinished 



In Unite,  extension packages provide a way to share code across both organisations and services. 

**Public** extension package are available to all organisations that use Unite
**Local** are organisation

2 type:
* Local
	* These are specific to the subscription and have the scope to it's 
* Public

Extensions packages contain


Extension packages
	* Examples 
		* Runtime
			* Cloud
		* Tests
		* Azure
			* Management
			* Application configuration
			* Azure managment
			* Service Bus
		* Microsoft Specific
			* Microsoft Graph
			* Office 365
		* Compression
		* File System
		* Saleforce
		* Sql Server
		* Twillio
		* Xml

#unfinished 