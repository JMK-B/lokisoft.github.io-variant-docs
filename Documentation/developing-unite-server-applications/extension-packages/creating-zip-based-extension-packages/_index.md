# Creating Zip based extension packages

Zip based extension packages allow organisations to create domain specific libraries that can be added to the platform to enabled 

This example creates an extension package with a simple modify strategy to either uppercase or lowercase 

. Assemblies can contain :
* Substitution methods
* Strategies, pipes, Connectors

## Creating the c# library

1. Create a library project

Edit the csproj file and add the following:
 ```xml
<PropertyGroup>
 <Version>1.0.0</Version>
 <FileVersion>1.0.0</FileVersion>
 <AssemblyVersion>1.0.0</FileVersion>
</PropertyGroup>
```

The Assembly version should match the runtime major version number i.e. 1.0.0 or 1.1.0. . The FileVersion which follows version numbering n.n.n.n. Finally there is the version which relates to the product version relating to the product  rather than an individual assembly. This need not following normal numbering conventions. 

1. Add variant.strategies.core dependency
2. Add file named assemblyinfo.cs add add the following code:
```cs
using Variant.Core.Attributes;
 
[assembly: UniteServerExtensionAssembly()]
```

If you want to add any substitution methods you can update the above and use:

```cs
[assembly: UniteServerExtensionAssembly(InitialiserType = typeof(AssemblyInitializer), InitialiserMethod = nameof(AssemblyInitializer.InitialiseAsync))]

public static class AssemblyInitializer
 {
     public static Task InitialiseAsync(IAppServices appServices)
     {
         //appServices.SubstitutionService.AddSubstitutionMethod("GetClaim", (obj, uniteMessage, item, property) =>
         //{
         //    Check.NotNull(obj, nameof(obj));
         //    Check.NotNull(item, nameof(item));
 
         //    //var item = obj switch
         //    //{
         //    //    string => Path.GetDirectoryName((string)obj),
         //    //    StringBuilder => Path.GetDirectoryName(obj.ToString()),
         //    //    null => null,
         //    //    _ => throw new Exception($"Unknown type in GetDirectoryName: {obj.GetType().FullName}")
         //    //};
 
         //    return item;
         //});
 
         return Task.CompletedTask;
     }
```

Step 4: Create a class file called ChangeCaseStrategy.cs and derive from the class from VariantStrategy and IModifyMessageStrategy.  After adding the interface implmentation it should look like this: 

```cs
public class ChangeCaseStrategy : VariantStrategy<ChangeCaseStrategy>, IModifyMessageStrategy
{

    public Task<IPipeResult> OnModifyAsync(ILokiMessage uniteMessage, CancellationToken cancellationToken = default)
    {
        throw new NotImplementedException();
    }
}
```

Step 5: Add the required enum and settings 


```cs

public enum ChangeCaseTypes
{
    ToUpper,
    ToLower,
}

...


[InjectedSetting("Determines what to change string to", DefaultValue = ChangeCaseTypes.ToUpper)]
public ChangeCaseTypes ChangeCaseType { get; set; }
 
[InjectedSetting("Namespace", DefaultValue = "Response")]
public string Namespace { get; set; } = default!;
```

Step 6: Add the implementation
```cs
var str = await uniteMessage.GetHeaderAsAsync<string>(Namespace);
var res = ChangeCaseType switch
{
    ChangeCaseTypes.ToUpper => str.ToUpper(),
    ChangeCaseTypes.ToLower => str.ToLower()
};
 
uniteMessage.SetHeader(Namespace, res);
return Success();
```

This should give you the following class:


```cs
using Variant.Core;
using Variant.Core.Attributes;
using Variant.Strategies.Core;
 
namespace Lokisoft.Strategies.Example;
 
public enum ChangeCaseTypes
{
    ToUpper,
    ToLower,
}
 
public class ChangeCaseStrategy : VariantStrategy<ChangeCaseStrategy>, IModifyMessageStrategy
{
    [InjectedSetting("Determines what to change string to", DefaultValue = ChangeCaseTypes.ToUpper)]
    public ChangeCaseTypes ChangeCaseType { get; set; }
 
    [InjectedSetting("Namespace", DefaultValue = "Response")]
    public string Namespace { get; set; } = default!;
 
    public override Task InitialiseAsync(IAppServices appServices)
    {
        return base.InitialiseAsync(appServices);
    }
 
    public async Task<IPipeResult> OnModifyAsync(ILokiMessage uniteMessage, CancellationToken cancellationToken = default)
    {
        var str = await uniteMessage.GetHeaderAsAsync<string>(Namespace);
        var res = ChangeCaseType switch
        {
            ChangeCaseTypes.ToUpper => str.ToUpper(),
            ChangeCaseTypes.ToLower => str.ToLower()
        };
 
        uniteMessage.SetHeader(Namespace, res);
        return Success();
    }
}```



## Packaging the files


In order to upload the extension we need to package it as a zip file. The easiest way of doing this is to Create the following build file structure:

![](Pasted%20image%2020231223141440.png)

>  [!Note]  under the builds folder you should have a directory that is your project name lowercased.  The version should match your FileVersion attribute in your .csproj file

Adding package and version information.

The package.json contents should contain :

```json
{
  "name": "Lokisoft.Strategies.Community",
  "description": "Strategies for the community framework"
}
```

version.json contents should contain the version description:

```json
{
  "description": "Initial version"
}
```

Then add the following post build script to your environment whilst updating the folder and version path and  to match your extension name:
```dos
COPY "$(TargetDir)*" "$(ProjectDir)builds\lokisoft.strategies.example\1.0.0\assemblies\*"
DEL "$(ProjectDir)builds\lokisoft.strategies.example\1.0.0\assemblies\Variant*"
```

Now when we build the project we should see the following:  

![](Pasted%20image%2020231223142155.png)


> [!Note]  If you also want to add any specialisations to your strategies you can add the YAMl files unders the services folder. You may want to import your extension first,  create any specialisations in the GUI  and then import it again.

The last thing we need to do is package the files into a zip file. Open file explorer and goto the builds directory above. right click on the package named folder and select compress to zip file. You should then see something like the following:

![](Pasted%20image%2020231222122009.png)


# Importing the extension using the Variant API.

All local packages are imported using the packages service API found at packages.applicationhub.io.

Step 1: Open postman.

Set the following values:
* Url: https://packages.applicationhub.io/api/extensions/{packagename}/{packageversion}. In This example it will be https://packages.applicationhub.io/api/extensions/lokisoft.strategies.example/1.0.0
* Method: POST
* Headers values:
   * Content-Type: application/zip
   * Authorization: Bearer [YourBearerToken]. You can get this from pressing F12 when opening the Variant development environment and viewing any calls to apps.appl;icationhub.io and extracting the bearer token from that call.
* Body: Goto the Body tab and click on binary and add the path to the zip file.

This should now look something like this:

![](Pasted%20image%2020231222123825.png)

With the headers looking like this:

![](Pasted%20image%2020231222123910.png)

