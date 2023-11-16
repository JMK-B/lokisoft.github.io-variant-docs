# Creating UI based extension packages

UI based extension packages contain only YAML files and as such can be created using Unite's development platform UI. This document will go through how to build subscription scoped packages. 


# Step 1:  Creating the extension


In order to create an extension package we need to got to the 'Extensions' page and set the dropdown to Local and click on the 'Create new extension button'. After this you should now see the following: 

![](Pasted%20image%2020231116142949.png)

## Step 2: Importing the package
Add an extension name and description and click 'Create' and you should now see the blank extension package in the drop down. Highlight the package and  click install:

![](Pasted%20image%2020231116143013.png)

After you have imported the extension you should see it available in the dependencies section in the left hand side menu. If we click on the lokidemo.myfirstextension_1.0.0 file you should see the following page displayed:

![](Pasted%20image%2020231116143122.png)




> [!Note] As you can see above, there is an asterix next to the dependency name which denotes that the package is in edit mode. This allows the package to be updated, files added or deleted in the UI itself. It is recommended that an application is not released whilst any packages it uses are in edit mode. 

## Step 3: Adding functionality

If we add the following code to the extension file:

```yaml
pipes: 
  - key: LokiDemo.MyFirstExtension.DefaultStorageQueuePushPipe
    value:
      pipe: Variant.AzureStorage.Queue.PushPipe
      replacements:
        CAN_EXECUTE_EXPRESSION: null
        CONTINUATION_POLICY: Default
        DATA_HEADER: null
        BODY: null
      defaults: 
        QUEUE_NAME: defaultqueue
        ACCOUNT_KEY: ${+DefaultStorageKey}
        ACCOUNT_NAME: ${+DefaultStorageKey}
        USE_DYNAMIC_ACCOUNT: False
```

When we  goto our application  we can  use that  pipe as if it was in the application itself. An example of this use can be seen below:

![](Pasted%20image%2020231116144724.png)

![](Pasted%20image%2020231116145101.png)

> [!Note] After saving, adding or deleting an extension package file you may need to refresh the page to see the new functionality in the word completion as above. 


### Step 4 : Publishing the extension

![](Pasted%20image%2020231116145613.png)


![](Pasted%20image%2020231116145635.png)


![](Pasted%20image%2020231116145747.png)

![](Pasted%20image%2020231116145809.png)


## Step 5: Creating a new version

Remove the extension from application or goto a new application where the extension has not been added. 


![](Pasted%20image%2020231116145911.png)

Click on the Clone button: 

![](Pasted%20image%2020231116150056.png)


Update the version and click publish. 

If you now install the package you should have 2 versions available:

![](Pasted%20image%2020231116150207.png)

The new version is now added and has the Asterix next to it denoting the new version is editable. 

![](Pasted%20image%2020231116150244.png)

After updating the package you can now go back to step 4 to publish the new version. Rinse and repeat for future versions.
