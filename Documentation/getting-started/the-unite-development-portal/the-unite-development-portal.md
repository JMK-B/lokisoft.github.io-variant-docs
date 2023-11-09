# The Unite Server Development Portal

Unite Server user portal is a web based , multi-tenanted development platform for building and deploying both API and workflow services. Its simple interface and common approach to tasks make it easy to grasp and develop new applications.

This section will first give an overview of all the pages that can be access through the menu options and then will do a deep dive of the service pages where each application is configured, developed and deployed.

- [Menu items](#menu-items) This section will give an overview of the of the landing page and it's menus.
- [Service Page](#the-services'-page) This section gives an overview of all the constituent part that makes up a service.

## Menu items

On the home page you will see there are 6 menu items in the left hand menu.

- [Home page](#home-page)
- [All tests](#all-tests)
- [Services](#services)
- [Source control](#source-control)
- [Vault settings](#vault-settings)
- [Global settings](#global-settings)

### Home page

This page currently is a placeholder that displays the current version of the Unite Server Development Studio

![](Main.Home.png)

### All tests

This page display the status of each of the services tests. The search box will search will filter the results taking into account each column in the list. To display failed tests type 'fail' into the search box.

![](Home.Tests.png)

### Services

This page is for building deleting and configuring each service. The default page of this service displays a list of all the services. This page will be explored in further details later in this document.

![](Main.Service.png)

For details on a specific applications service page click [here](#the-services-page).

### Source control

This page handles manages source control of your entire subscription assets: config files, services files, etc using Git. Its displays the currently added , modified & deleted files and push directly in the git repository specified upon the subscriptions creation.

![](Main.SourceControl.png)

### Vault settings

This page shows the vaults settings for this development platform. If any secure environment settings are required you can create them in environment specific Azure Vaults and add them to the service.yaml found in your application. 

![](vaultsettings.png)

### Global Settings

This page allows appsettings that are shared across multiple services to be configured in a single place. Any settings can be overriden by any service by simply adding the same key value in a services app settings file or through the appservices configuration settings.

![](Main.GlobalSettings.png)

> [!Note] The Variant.Environment.Type setting denotes what type of environment the service is running in. This then allows the developer to remove certain implementations, endpoints and connectors from the runtime when they are running in environments such as test and prod. 

## The Services Page

When create a new service and click on it you will see the following page:

![](Services.Overview.NoneDeployed.png)

There are 3 main sections in the service left hand side menu and these are

- **Service configuration**. This where the service is deployed, app settings configured, tests are viewed and extensions added.
- **Service Definitions**. This area contains the yaml files that are used to configure and implement the API's and worklflows that the service uses.
- **Dependencies**. This display all the extensions that the service uses. Extensions provide additional functionality to the service. Examples of this are but not limited to:
  - **Variant.Runtime.FunctionApp**. Provides the connectors and libraries that allow the service to be run as an Azure isolated function app. This can be changed to a Kubernetes console to enable the service to be hosted in a container instead.
  - **Variant.Strategies.Xml**. Provides additional pipe and strategy definitions as well as addition substitution methods for dealing with Xml formats.
  - **Variant.Strategies.Azure.Management**. Provides pipe definitions for reading and managing Azure resources such as app services, virtual machines, resource groups etc.
  - **Variant.Strategies.MicrosoftGraph**. Provides functionality that deals with Users in Microsoft's Active Directory (default and B2C)

### The overview page

This page comes in 2 flavours: the first can be seen in the image sown in the previous section, which is displayed when you first create and service; and the seconds can be seen below where the actual service has been deployed. To deploy an application simply click on the deploy button found on this page.

![](Services.overview.deployed.png)

When an application is deployed you can see the default host name, this is the Azure endpoint url. This endpoint is made up of the parts:

- The subscriptionId
- The service Id
- A randomly generated 8 character string. This can be changed at anytime by simply going to configuration=>site-config.json tab and updating the suffix value.

Additional buttons allow you to stop or restart the service. Or if required to delete the app service deployment.

### Test results

This is a simple page where you can start the applications tests and view the results. As the tests are run asynchronously you will need to refresh this page to get the results

![](Services.overview.tests.png)

### Configuration

This section display 4 configuration files that the service uses:

- appsettings.json
- local.settings.json
- site-config.json
- service.yaml

#### Appsetting.json

As with any .Net Core application these are the general settings for the application. However, As can bee seen in the following image there is an additional appsettings property found under the Lokisoft property. It is recommeded that any settings that do not require to be changed are placed in the root directry and any that will be changed, i.e. those spoecific to an environment are placed under the Lokisoft appsettings property. This allows developers to easily see what needs to be overriden ion an environment and the platform itself to provide an API endpoint which returns these values to allow devops to ensure that all these appsettings are actually found in any environment settings repository.

![](ServicePage.AppSettings.png)

#### Local.settings.json

This is local settings file that is only ever deployed to the development environment. Project settings such as logging level can be updated here to provide additional debug information for the development environment.

![](ServicePage.LocalAppSettings.png)

#### Site-config.json

This is essentially a project file that contains information details of the application itself. Core items like the service id, description, deployment state and extensions installed are found in this file.

![](ServicePage.SiteConfig.png)

#### service.yaml

This file contains information that deals with the instantiation of the application and global services that the application uses. Services such as any access to vaults and external app settings such as Azure App Configuration.

![](ServicePage.Service.yaml.png)

### Extensions

This is where additional functionality can be added in the form of pipes, connectors & strategies. extensions can be downloaded from a public repository or from the subscribers private repository allowing domain specific implementations to be written once and shared amongst different applications. Things can can be shared can be anything such JWt token creation, Storage queue connections etc.

![](ServicePage.extensions.png)

### Service definitions

Service definitions are YAML coding files that describe what the service does. This is basically where the application is coded. When creating a new service it adds 3 files which are :

- The 'App' file
- The 'Startup' file
- The 'Tests' file

Both the App file and Tests file can be deleted or renamed. However, the StartUp file cannot. This is because the Startup file should contains the App Service general endpoint and pipeline. In saying this though it does not mean the file cannot be updated and the pipeline modified. If for example you wanted to return the data in a specific format inside a data tag with a result property on it this is where you would do that.

> Note: Once the service is deployed any changes to any service files are updated in the service within a maximum of 5 seconds. This enables developers to update these files, add new endpoints and connectors and immediately see those changes running in the cloud and a development form of continuous integration

#### The App file

When creating a default application the App file should look like the following:

![](ServicePage.AppFile.png)

This file contains 2 simple endpoints to get the developer started: The first returns a simple hello response and the second returns a greeting with the passed in name uppercased and additional return headers.

#### The 'Startup file'

This file contains 2 connectors and 2 endpoints as can be seen in the following image:

![](ServicePage.Startup.png)

- **connector: Variant.AzureFunctions.AzureFunctionIsolatedConnector**. This connector is a wrapper around the Azure function pipeline and all calls to this service will be routed through these 2 pipes. When productionising a service additional pipes will normally be placed at the start to authenticate and authorise users via methods such as Bearer tokens or access keys. It must be noted though if any authentication is added then the line 'CAN_EXECUTE_EXPRESSION: ${Request.Property.LocalPath} != "/api/integrationtests"' should be added to allow any tests to run as tests use their own test authentication access key.
- **connector: Variant.AzureFunctions.KeepAliveConnector.** This is an Azure Function App specific connector that is used to keep alive consumption app plans indefinitely rather than their default time of 5 minutes. The Url used here does not have to exist for this connector to work.
- **routeTemplate: api/health**. This is a placeholder endpoint for health and can either be extended or removed.
- **routeTemplate: api/openapi.json**. This endpoint provides an openapi / swagger endpoint that returns the services openapi definition. Developers can go to https://validator.swagger.io/ and add the services swagger endpoint to call the service.

#### The 'Tests' file

This page is where the default tests are placed. All pipes and endpoints use the pipes found in the 'Variant.Strategies.Community.Tests extension file.

![](ServicePage.tests.yaml.png)

The endpoint paths should not be changed though as the development platform uses this naming convention to call the tests and get the results.

> [!Note] This file can be deleted and any tests can be placed in any services files.

### Dependencies

The Dependencies section displays what YAML files are found in the extensions added. There are 2 types of YAML files found in these extensions:

- Auto generated files
- Hand crafted files

#### Auto-generated files

These are created by the platform when a new extension with a strategies dll is imported into the system. The generated code allows the developer to call the .net written code using Yaml. Every pipe, connector and strategy is a specialisation of these files which, in turn , calls extension assemblies directly.

![](ServicePage.autogenerated.png)

#### Hand crafted files

The benefits of these handcrafted file is that its allows pipes and connectors to be created that are an amalgamation of other pipes and connectors and provide reusable components that can include properties that can be left as default or overridden.

The example below is the connector that keeps the app service alive. This is a timer connector that calls an endpoint every n minutes:

![](ServicePage.handcrafted.yaml.png)
