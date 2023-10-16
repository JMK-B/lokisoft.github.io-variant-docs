# Substitutions & the substitution service

## Overview

In its most basic form substitutions are simply string replacements with either another string or an object.
Substitutions are a string interpolation feature in Unite that allows strings in YAML code to be converted to either a different string, or an object such as a stream or JToken. There are 2 different types of operators which correspond to when the string substitution is made. The '\$[]' operator denotes a 'compile' time substitution, or a '${}' which denotes a runtime substitution.

The '\$[]' operator is a simple inline string replacement that is done prior to the YAMl code being converted in a JSON structure. This replacement does not need, or use, Unite's substitution service.

The '${}' operator is a far more powerful operator that uses the substitution service's capabilities to provide access to not only the current VariantMessage and app settings, but addition key value repositories such as Azure Vault, Azure app configuration, Redis, database key value implementations etc.

As well as accessing data, it can also be used to to call methods on that data, return default values such as DateTime, perform null coalescing and indexing '[n]'. These are called service extension methods

The following API endpoint demonstrates substitutions in action:

```yaml
- routeTemplate: api/test
  routeMethod: GET
  pipeline:

      # Create  object with settings demonstrated
    - pipe: Variant.Core.SetHeader
      NAMESPACE: Settings
      # Allow creation of temp values that only have the scope of this modifier
      TEMP_VALUES:
        Name: Bobby Davro
        DoesNotExist: null
        children:
          - name: Peppa
            age: 5
          - name: George
            age: 3
      VALUE:
        # App Settings
        projectName: $[Project.Id]
        projectName1: ${Project.Id}
        projectDescription: $[Project.Description]

        # Settings found in message
        name: ${TempValues.Name}
        questionMarkAllowsNulls: ${?DoesNotExist}  #  Returns null

        # Inbuilt
        currentDir: ${System.CurrentDirectory}
        currentDir: ${System.SiteConfig}
        dateTimeNow: ${DateTimeOffset.Now}
        newGuid: ${Guid.NewGuid}

        # Extension methods:
        uppercaseName: ${TempValues.Name.ToUpper()}
        questionMarkAllowsNullWithoutErrors: ${?DoesNotExist.ConvertStringToBase64()}  #  Returns null
        nameAsBase64: ${TempValues.Name.ConvertStringToBase64()}  #  Returns null
        nextMonth: ${DateTimeOffset.Now.AddDays(28).ToString("yyyy-MM-dd")}
        nullCoalescingWithString: ${?DoesNotExist ?? "ItDoesNottExist"} # uses a string value
        nullCoalescingWithHeader: ${?DoesNotExist ?? TempValues.Name} # uses a header value

        # Arrays
        usingIndex1: ${TempValues.children[0].name}
        usingSelectTokens: ${TempValues.children.SelectTokens("$..name")}  l
        usingSelectTokensWithJoin: ${TempValues.children.SelectTokens("$..name").Join(",")}

        # SecureSettings
        secureSetting: ${+CloudStorage-Name}
        secureSettingUppercase: ${+CloudStorage-Name.ToUpper()}

      # We van use substitution to replace objects as well as strings
    - pipe: Variant.Core.SetHeader
      NAMESPACE: Response
      VALUE:
        status: success
        settings: ${Settings}

```

This creates the following output:

```json
{
  "status": "success",
  "settings": {
    "projectName": "licensegenerator",
    "projectDescription": null,
    "projectName1": "licensegenerator",
    "name": "Bobby Davro",
    "questionMarkAllowsNulls": null,
    "currentDir": "C:\\home\\site\\wwwroot",
    "dateTimeNow": "2023-07-19T13:38:42.3728453+00:00",
    "newGuid": "7e9e0957-0f51-4ba5-acf5-5b9c039bb6c0",
    "uppercaseName": "BOBBY DAVRO",
    "questionMarkAllowsNullWithoutErrors": null,
    "nameAsBase64": "Qm9iYnkgRGF2cm8=",
    "nextMonth": "2023-08-16",
    "nullCoalescingWithString": "ItDoesNottExist",
    "nullCoalescingWithHeader": "Bobby Davro",
    "usingIndex1": "Peppa",
    "usingSelectTokens": ["Peppa", "George"],
    "usingSelectTokensWithJoin": "Peppa,George",
    "secureSetting": "comdemosa",
    "secureSettingUppercase": "COMDEMOSA"
  }
}
```

- Substitutions
  - AppSettings
    - Uses notation $[]
    - Compile Time
  - SecureSettings
    - uses ${+}
  - OtherSettings
    - Inbuilt settings
    - VariantMessage
      - Headers
      - Payload
- Adding Azure vaults
  - Substitution methods

## Compile time / app setting substitutions

These types of substitutions are simple string replacements that are that allow any app setting values to be replaced directly into the YAML before it is parsed. If we have the following appSettings we can access their values using the $[AppSettingName] notation.

```json
{
    "Project.Name": "MyProject",
    "Project.Description": "MyProject description",
    "Varaint.Environment.Type": "dev",
    .....
}
```

If we look at the StartUp YAML file of an AppFunction project we can use these types of substitutions to allow a common pipe / pipeline for all applications with project specific information.

```yaml
#
# OPEN API ENDPOINT
#
- routeTemplate: api/openapi.json
  routeMethod: GET
  pipeline:
    - pipe: Variant.Core.WebApi.GenerateSwaggerEndpointsStrategyPipe
      OPERATIONS_NAMESPACE: Operations
      IGNORE_IF_NO_DESCRIPTION: true
    - pipe: Variant.Core.ModifyMessageStrategyPipe
      VALUE:
        openapi: 3.0.0
        paths: ${Operations}
        info:
          version: 1.0.0
          title: $[Project.Name]
          description: $[Project.Description]
```

Another use of app setting substitutions is in the IS_ENABLED property of all endPoints, connections and pipes. This IS_ENABLED property is used at at initialisation to determine whether the endPoint, connection or pipe is included in the application when it is compiled. We can use app setting substitutions so the above openApi endpoint is only available in the dev environment.

```yaml
#
# OPEN API ENDPOINT
#
- routeTemplate: api/openapi.json
  routeMethod: GET
  IS_ENABLED: $[Varaint.Environment.Type] == "dev"
  pipeline:
    - pipe: Variant.Core.WebApi.GenerateSwaggerEndpointsStrategyPipe
     .....
```

> NOTE: App settings do not necessarily have to be in a config file. Any settings found under the configuration section of an Azure AppService can also be accessed this way. Adding the configuration setting '"Variant.Environment.Type": "test" or "Varaint.Environment.Type": "prod" would exclude the openapi.json endpoint above from being available.

## Runtime substitutions and the substitutions service

In this section we will look at the subs service:

- Syntax:
- Inbuilt replacement locations
- Extension methods
- Extending its replacement search locations to include external repositories

### Replacement syntax

There are 3 different types of substitutions that the service recognises as replacements:

- ${MyHeader}
- ${?MyHeader}
- ${+MyHeader}

#### ${MyHeader}

This is the most common form of replacement and simply returns the value. If the value is not found, or a extension method uses is used on that method fails, eg. null value, due to type mismatch, etc, then the service will throw an exception.

#### ${?MyHeader}

This notation tells the service that if the value is not found or any exceptions are raised when getting the value then the service should return a null value.

#### ${+MyHeader}

This notation tells the service that the value is a secure setting and it should get it from any secure setting repositories. We'll go further in secure settings later in the document.

### Inbuilt replacement locations

Excluding any addition locations added in the service file, the subs service uses 3 different sources to get replacement values:

- AppSettings
- VariantMessage
- Default replacements

#### App settings

Not only can app settings be used at compile time but they can also be accessed at runtime using the '${[?]AppSettingName}' notation. This allows initialisers with in the code to access these values and also allows extension methods to be called of the value. Something that cannot be done at runtime.

#### VaraintMessage

This is where most subs will be accessed from. Both the message headers, payloads and any of their sub properties are are accessible using the '${' notation.

#### Default replacements

Although extension methods can be used to get or create any value they can come at a cost regarding memory or clock cycles. Unite defines a set of fixed subs that can be accessed at anytime and these are:

##### General

- **${Guid.NewGuid}**: Creates a new Guid

##### File system

- **${Temp.Dir}**: Path to the temp directory
- **${System.CurrentDirectory}**: Path to the current directory. Can be used to access any file resources added to the project

##### DateTime properties

The following substitutions can all be formatted using the ToString(""). method. e.g. ${DateTimeOffset.Now.ToString("yyyy-MM-dd")}

- **${DateTime.Now}**
- **${DateTime.UtcNow}**
- **${DateTimeOffset.Now}**

##### Message properties

- **${Message.SpawnId}**: returns the spawned message id if available.
- **${Message.Id}**: The current message Id.

##### Payload conversion properties

The following are used to easily grab any payload and convert it inline. If the current payload is an IDisposable or IAsyncDisposable derived class the payload will be automatically disposed on conversion. How each of these are converted depends on the type the object payload is.

- **${Payload.ToString}**: Gets the Payload as a string.
- **${Payload.ToBase64}**: Gets the payload to a base 64 string
- **${Payload.ToStream}**: Gets the payload to a stream. If it is already a stream then is will return a reference to that stream.

### Extension methods

Extension methods allows inline code execution directly on a subs value. The substitution service allows a '.' notation after the data name to run specific methods and return an updated value rather than an existing value. Every 'Strategies' assembly loaded into the runtime has the ability to add extension methods which are both synchronous and asynchronous. An example of the .Net code required can be seen below:

```csharp
using System.Text;
using Variant.Core;
using Variant.Core.Attributes;
using Variant.Core.Utilities;
using YamlDotNet.Serialization;

[assembly: UniteServerExtensionAssembly(InitialiserType = typeof(AssemblyInitializer), InitialiserMethod = nameof(AssemblyInitializer.InitialiseAsync))]

namespace Variant.Core;

internal static partial class AssemblyInitializer
{
    // Called during the application startup process
    public static Task InitialiseAsync(IAppServices appServices)
    {
        appServices.SubstitutionService.AddSubstitutionMethod("First", (enumerable, uniteMessage) =>
        {
            var res = ((IEnumerable<object>)obj!).First();
            return res;
        });

        appServices.SubstitutionService.AddSubstitutionMethod("Skip", (enumerable, uniteMessage, skip) =>
        {
            var res = ((IEnumerable<object>)obj!).Skip(ParseInt(skip));
            return res;
        });

        ...

        // Async extension method.
        appServices.SubstitutionService.AddAsyncSubstitutionMethod("ConvertStreamToJTokenAsync", async (stream, uniteMessage) =>
        {
            using (StreamReader sr = new StreamReader((Stream)stream))
            using (JsonReader reader = new JsonTextReader(sr))
            {
                return await JToken.ReadFromAsync(reader);
            }
        });
   }
}
```

When you add new extension packages that contain assemblies, such as Variant.Strategies.FileSystem, these packages may contain additional substitution extension methods which can be accessed using the the following notations:

- ${DateTime.Now.AddDays(2)}
- ${number.AddNumber(2)}
- ${jToken.SelectTokens("$..users")}

You can also chain multiple substitution methods like:

- ${DateTime.Now.AddDays(2).ToString("yyyy-MM-dd")}

The Variant.Runtime.AppFunctions runtime includes the following extension methods.

- First (IEnumerable, uniteMessage) 72
- Last (IEnumerable, uniteMessage) 78
- Any (IEnumerable, uniteMessage) 84
- Take (IEnumerable, uniteMessage, take)
- Skip (IEnumerable, uniteMessage, skip)
- Contains (string, uniteMessage, item, caseInsensitive)
- Ternary (bool, uniteMessage, trueExp, falseExp)
- CreateHMACSHA256Signature (stringToSignObj, uniteMessage, keyObj)
- Base64ToByteArray (string, uniteMessage)
- UTF8StringToByteArray (string, uniteMessage)
- UriEscapeDataString (string, uniteMessage)
- Equals (object, uniteMessage, object)
- DoesNotEqual (object, uniteMessage, object)
- AddNumber (number, uniteMessage, number)
- SubtractNumber (number, uniteMessage, number)
- AsJson (object, uniteMessage, bool indented)
- AsJson (object, uniteMessage)
- SelectToken (JToken, uniteMessage, string)
- JArrayContains (JArray, uniteMessage, JValue)
- SelectTokens (JToken, uniteMessage, string)
- JTokenGetProperty (JToken, uniteMessage, string propName)
- JTokenDeepEquals (JToken, uniteMessage, jtokenObj)
- JsonOrderBy (JArray, uniteMessage)
- JsonOrderBy (JArray, uniteMessage, orderBy)
- JsonOrderByDesc (JArray, uniteMessage)
- JsonOrderByDesc (JArray, uniteMessage, orderBy)
- ConvertObjectToJToken (object, uniteMessage)
- ConvertStringToJToken (string, uniteMessage)
- ConvertYamlToJToken (string, uniteMessage)
- ParseDateTime (dateObj, uniteMessage)
- AddTimeSpan (DateTime, uniteMessage, timeSpan)
- AddDays (DateTime, uniteMessage, dateCount)
- AddHours (DateTime, uniteMessage, dateCount)
- AddMinutes (DateTime, uniteMessage, dateCount)
- IsNull (object, uniteMessage)
- IsNullOrWhiteSpace (string, uniteMessage)
- IsNullOrEmpty (string, uniteMessage)
- IsNotNull (string, uniteMessage)
- IsNotNullOrWhiteSpace (string, uniteMessage)
- IsNotNullOrEmpty (string, uniteMessage)
- ToTitle (string, uniteMessage)
- PadLeft (string, uniteMessage, count, padding)
- PadRight (string, uniteMessage, count, padding)
- Split (string, uniteMessage, separatorStr)
- SplitAndRemoveEmptyEntries (string, uniteMessage, separatorStr)
- Split (string, uniteMessage, separatorStr, count)
- Join (string, uniteMessage, separator)
- Trim (string, uniteMessage, trim)
- TrimStart (string, uniteMessage, trim)
- TrimEnd (string, uniteMessage, trim)
- StartsWith (string, uniteMessage, startsWithStr)
- StartsWith (string, uniteMessage, startsWithStr, caseInsensitive)
- EndsWith (string, uniteMessage, endsWithStr)
- Random (\_, uniteMessage, to)
- Random (\_, uniteMessage, from to)

As the subs service parses each string that contains a substitution value into a syntax tree subs replacements can be embedded in another substitution as in the following Yaml example:

```yaml
- routeTemplate: api/substring/{mystring}/{from}
  routeMethod: GET
  pipeline:
    - pipe: Variant.Core.SetResponse
      RESPONSE: ${Request.Path.mystring.Substring(${Request.Path.from})}
```

##### Reflected extension methods

If an extension method signature is not found as a subs service subs method then the subs service will use .Net reflection to see if that method exists on the type. In the above example the SubString method doesn't actually exist as an extension method so its actually calling the Substring method that is found on the .net String type. In fact any public method on a type can be called using this method. Another benefit is that any public property can be access using this method using 1 caveat. The extension method notation must call a property using it's get (**get\_**PropertyName) function rather than directly as seen in the example below:

```yaml
- pipe: Variant.Core.SetResponse
  # RESPONSE: ${DateTimeOffset.Now.Date} This will not work
  RESPONSE: ${DateTimeOffset.Now.get_Date} This will work
```

### Adding addition key value repositories

Additional key value repositories come in 2 forms:

- Those derived from the IAdditionalSecureSettings interface. Uses the ${+YourSecret} notation
- Those derived from the IAdditonalSettings interface. Uses the ${YourSecret} notation

The only difference is that the ${+YourSecret} has some limitations on where it can be used. i.e. the notation cannot be passed in the body of an API call or in the path or query parameters. IAdditionalSettings setting can contain secrets but care must be taken to ensure the values are securely stored and used.

To add any additional key value repositories is we need to add them to the dependendencies section of the Service.yaml configuration file. The Service.yaml below adds 3 addition repositories that can be accessed by the substitution service:

```yaml
dependencies:
  - name:
    interface: Variant.Core.IAdditionalSecureSettings, Variant.Core, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null
    implementation: Variant.Strategies.Azure.Vault.AzureVaultUsingClientSecret, Variant.Strategies.Azure.Vault, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null
    values:
      CacheValues: true
      TenantId: $[Vault_TenantId]
      ClientId: $[Vault_ClientId]
      ClientSecret: $[Vault_ClientSecret]
      VaultUrl: $[Vault_Url]

  - name: additionalVaultSetting
    interface: Variant.Core.IAdditionalSecureSettings, Variant.Core, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null
    implementation: Variant.Strategies.Azure.Vault.AzureVaultUsingClientCertificate, Variant.Strategies.Azure.Vault, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null
    values:
      CacheValues: true
      TenantId: 2e793447-1377-44de-b149-aaaaaaaa
      ClientId: db102636-b357-40a9-9b95-bbbbbbb
      Thumbprint: AAAAAAAA83E66EAB840A9AAAAAAAAAA
      VaultUrl: https://myKeyVault.vault.azure.net/


   - name: AzureAppConfigurationSettings
     interface: Variant.Core.IAdditionalSettings, Variant.Core, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null
     implementation: Variant.Azure.AppConfiguration.GetAzureAppConfigurationSettings, Variant.Strategies.Azure.AppConfiguration, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null
     values:
         ConnectionString: ${AppConfigConnectionString}" # Endpoint=https://myconfig.azconfig.io;Id=hZ6i-lo-s0:ev4eLm/Xn+xxxxxy+l;Secret=exxxx
         LabelFilter: app-underwritingservice
         KeyFilter:  null
```

The first 2 are examples show how to add access to an Azure Vault using a client secret or a client certificate and the final allows the subs service to access key value pairs from Azure App Configuration.

> Note: For example on how to create both extension methods and additional key value repositores see the 'Extending Unite' documentation.
