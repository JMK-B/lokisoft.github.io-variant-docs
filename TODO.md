
- [ ] #bug${Variant.ENvironment.Name} doesn;t work in commission  statement health call
- [ ]  ${?AccessKeys.value.ConvertStringToJToken().${Request.type.ToLower()}} fails as type.tolower() isn;t substituted before it does the selecttoken in the JTOken
- [ ] #bugClick in tests results row and then selected row is changed and no other menu item works
- [ ] #features 2 types of services : Test and normal where a test service cannot be deployed. (really?)
- [ ] #bug FUNCTION_120000 FUNCTIONTIMEOUT. Add another replacements,defaults,deletions - where deletion would remove the TIMEOU replacement value
- [ ] If we set the Readas.Stream in the service bus that overwrites all ReadAAs properties eg httpPush message. In Fact not default props from connectors should be passed down

Create environment
BUG: sage":"The vault name 'operations562-bluepear-kv' is invalid. A vault's name must be between 3-24 alphanumeric 


Add flag to the endpoint so you can set the endpoints to buiild the pipeline every call. This stops the need to keep going back to the file where the endpoint is and saving the file
* Create queueSasBuilder*
* F12 on a pipe goes to the actual pipe declaration

* Environment groups - Allows a 'global type file to override the environments file settings where you can publish create and delete environments all at once'

bugs:
{
    "applicant":
       {
        "name": "JK2e",
        "nameq": "JK2qqe",
    },
    }
    * Works: ${?Request.applicant.name ?? ?Request.applicant[0].nameq ?? "NoFound"}
    * Fails: ${?Request.applicant[0].nameq ?? ?Request.applicant.name  ?? "NoFound"}
    
          test1: ${?Request.applicant.name ?? ?Request.applicant[0].nameq ?? "NoFound"}

          test2: ${?Request.applicant[0].nameq ?? ?Request.applicant.name  ?? "NoFound"}

* Create API to allow for things like PAT to be updated
* Convert Packagemanager importer to Queue based system
* Notifications: Gui and backend storage
* TODO: Personal access tokens
	* Make user dependent and move out of general vault
* Add _rot folder to allow runtimes to be added_
* Allow setting up of different types of app services for dev environement.  This should include containers
* Different environment de[ployments]
* Increase scope of git to allow local packages to be included
* Deleting a yaml file doesn't remove items from search

------------------------
      - pipe: Variant.Core.TryCatchScopedPipe
        EXCEPTION_HANDLER_BEHAVIOR: Default
        EXCEPTION_PIPES:
	uses default        
      - pipe: Variant.Core.TryCatchScopedPipe
         EXCEPTION_PIPES:
    BUG!!  sets the default behaviour to Replace
    ---------------------------------------------------------------------------------
 ${?Request.orderBy.Replace("'","''") ?? "lastUpdated"} returns null. Coelesse not called if ther is a method

when using cosmos passing in a COMOS_RESPONSE object it takes it as an expendoObject string. 

appServices.SubstitutionService.AddSubstitutionMethod("XmlEncode", (obj, uniteMessage, elementName) =>  If this method already exists then it shuts down the service

THINGS FOIR V1.1
1. All rest dlls converted to yaml
2. symbol libraries
3. Change Exception.PipeResult.ErrorText to Exception.ErrorText and add a type so we have something like Type: Http.NotFound, Or Substitution.NotFound

Guardian: create service that shuts down  app services and boots them up again

1. update Sas query on deployments ko be blob specific


idea
1. Create service form should be  centered and have 3 tabs or wizard
	1. Blank / vanilla projects
	2. Fully built projects 
		1. devops
		2. admin service 
		3. etc
	3. Local projects