# How To: N-Central API Automation

## Table of Contents
- [Overview](#overview)
- [Connecting](#connecting)
  * [PS-NCentral](#ps-ncentral)
  * [PowerShell WebserviceProxy](#powershell-webserviceproxy)
- [Performing Queries](#performing-queries)
  * [PS-NCentral](#ps-ncentral-1)
  * [PowerShell WebserviceProxy](#powershell-webserviceproxy-1)
- [Bind to the namespace, using the Webserviceproxy](#bind-to-the-namespace--using-the-webserviceproxy)
- [Updating a Value](#updating-a-value)
  * [PS-NCentral](#ps-ncentral-2)
  * [PowerShell WebserviceProxy](#powershell-webserviceproxy-2)
    + [Registration token injection](#registration-token-injection)
    + [Gather organization property ID](#gather-organization-property-id)
    + [Update customer property](#update-customer-property)
    + [Add new a new Customer](#add-new-a-new-customer)
- [Appendix A – N-Central Web Service members](#appendix-a---n-central-web-service-members)
- [Appendix B PS-NCentral cmdlets](#appendix-b-ps-ncentral-cmdlets)
- [Appendix C – GetAllCustomerProperties.ps1](#appendix-c---getallcustomerpropertiesps1)
- [Appendix D – Customer Property variables](#appendix-d---customer-property-variables)

# Overview

N-Central's API is a flexible, programmatic, object oriented, Java based interface by which developers can achieve integration and automation via native SOAP API calls.

For the purposes of this guide we'll be covering connectivity and basic usage with PowerShell based automation through the PS-NCentral module, as well as native WebserviceProxy cmdlet.

The information covering the PS-NCentral is useful for those with starting some experience with PowerShell or need to quickly put together code where module dependency isn't an issue, while the usage of the WebserviceProxy method is for those more familiar with object oriented coding or need code portatability.

PS-NCentral provides cmdlets for 17 Get cmdlets and 3 Set cmdlets that cover the majority, so should cover the majority of automation. This can be downloaded from: [https://github.com/ToschAutomatisering/PS-NCentral](https://github.com/ToschAutomatisering/PS-NCentral)

Or installed with the cmdlet

```powershell
Install-Module PS-NCentral
```

# Connecting

The first step required before connecting is to create a new automation account with appropriate permissions.

With N-Central 2020 or 12.3 HF4 and later you must disable the MFA requirement for the account so use a long and complex password.

Once the account is created, select the API Authentication tab and click on the ' **Generate JSON Web Token**' button, otherwise referred to JSON Web Token **(JWT)** save this token somewhere secure, if you lose your JWT, you can generate another one at any time.

This token will be used for the authentication to the server.

## PS-NCentral

Connecting to your N-Central service with PS-NCentral only needs to be done once per session. Your first require the following information:

- The fqdn of your N-Central server, ie: n-central.myserver.com
- The account name you created above, ie. ncentral\_automation@myserver.com
- The JWT from above

Then enter the following:
```powershell
#Import the PS-NCentral module
import-module .\PS-NCentral.psm1 -Verbose

#$credential = Get-Credential
$password = ConvertTo-SecureString "YOUR JWT TOKEN" -AsPlainText -Force
$credential = New-Object System.Management.Automation.PSCredential ("ACCOUNT NAME REQUIRED", $password)

#Connect to NC
New-NCentralConnection -ServerFQDN YOUR SERVER FQDN -PSCredential $credential
```

If successful you will get an output similar to the below:

|Property | Value|
|--------|-----|
| Error | |
| ConnectionURL | n-central.myserver.com|
| BindingURL | https://n-central.myserver.com/dms2/services2/ServerEI2?wsdl |
| IsConnected | True |
| NCVersion | |
| tCreds | |
| DefaultCustomerID | 50 |
| CustomerValidation | {zip/postalcode, street1, street2, city...} |

## PowerShell WebserviceProxy

As a preface to the usage of the New-WebserviceProxy cmdlet, we will focus on the v2 rather than v1 legacy API as the v1 maybe endpoint maybe deprecated at some point.

The main differences between the v1 and v2 endpoints are:

- The WSDL endpoint
- Different classes, including the KeyPair constructor class used for adding custom settings for queries and update/change methods
- V2 has all the newer methods available

It will be necessary to review the Javadocs provided on your server for the lastest information on the classes and constructors, you can find them under your own N-Central server under https://\&lt;fqdn\_of\_server\&gt;/dms.

If reviewing other WebserviceProxy powershell code on the internet, you can identify v1/legacy code as it will have the following in the binding URL string: /dms/services/ServerEI?wsdl while v2 has /dms2/services2/ServerEI2?wsdl

For connecting to webservice you will need the same information as with the PS-NCentral which connects in the same way underneath:

- The fqdn of your N-Central server, ie: n-central.myserver.com
- The account name you created above, ie. powershell_automation@myserver.com
- The JWT for the account

With our examples we'll use v2 connections and classes, below is a common method seen in examples:
```powershell
#Example host
$serverHost = "n-central.myserver.com"

# Bind to the namespace, using the Webserviceproxy
$NWSNameSpace = "NAble" + ([guid]::NewGuid()).ToString().Substring(25)
$bindingURL = "https://" + $serverHost + "/dms2/services2/ServerEI2?wsdl"
$nws = New-Webserviceproxy $bindingURL -Namespace ($NWSNameSpace)
```

```$NWSNameSpace``` here can be most anything of your choosing, the point of the GUID generation is to ensure the namespace for the classes to be used inside the webservice are _unique_ to anything else on your system or current context, you could use a static namespace such as MyNCentralConnection or PowerShellIsAwesome.

After you've run this the $nws variable will contain all the available public methods from the endpoint, you can interrogate this by running
```powershell
$nws | Get-Member
```
From this you will see a lot of | Event |s, Methods and Properties ie.

|Name | MemberType |
|----| ----------|
|versionInfoGetCompleted || Event ||
| Abort| Method |
| accessGroupAdd | Method |
| BeginaccessGroupAdd | Method |
| BeginaccessGroupGet | Method |

The above output has been shortened, see Appendix A – N-Central Web Service members for the complete output. In addition you will have a **Definition** column, and you will observe that your ```$NWSNameSpace``` is seen prefixed to the methods/classes noted in them. All classes/methods/constructors available in the Javadocs can be created and called upon through the ```$nws``` variable. Eg. The customerListmethod would be called with

```powershell
$nws.customerList("", $password, $settings)
```
As you will note when connecting with the $nws variable, at no point did you use your username or JWT, as you will observe in the $nws.customerListmethod called above, the $usernameand $password is used in every get or set.

Underneath the PS-NCentral saves these variables and re-uses each time a cmdlet is used.

# Performing Queries

## PS-NCentral

Performing queries with the PS-Central module is quick and easy, and several examples are provided in it's own documentation [here](https://github.com/ToschAutomatisering/PS-NCentral/blob/master/PS-NCentral_Examples.ps1) . The outcomes of the examples are fairly self explanatory. For our example we'll take a common query like the Customer table and join it with the Organisation properties table using the PS-NCentral cmdlets, then in the advanced section we'll give the same example using native cmdlets.
```powershell
Import-Module PS-NCentral.psm1 -Verbose

$username = "ACCOUNT NAME"
$JWT = "JWT TOKEN"
$password = ConvertTo-SecureString $JWT -AsPlainText -Force
$credential = New-Object System.Management.Automation.PSCredential ($username, $password)

#Connect to NC
New-NCentralConnection -ServerFQDN n-central.myserver.com -PSCredential $credential

#Grab the customer list/details
$CustomerList = Get-NCCustomerList

#Get the customer properties
$CustomerPropertyList = Get-NCCustomerPropertyList

#Create array list for table

$CustomerReport = New-Object System.Collections.ArrayList

#Merge
foreach ($Customer in $CustomerList) {
    $Properties = $CustomerPropertyList | ?{$_.CustomerID -eq $Customer.customerid}
    $CustomerHashtable = [Ordered]@{}
    $Customer.psobject.properties | ?{$_.Name -ne 'customerid'} | %{$CustomerHashtable[$_.Name] = $_.Value}
    $PropertiesHashTable = [Ordered]@{}
    $Properties.psobject.properties | %{$PropertiesHashtable[$_.Name] = $_.Value}
    $ReportItem = $CustomerHashtable + $PropertiesHashTable
    $ReportItem = [PSCustomObject]$ReportItem
    $CustomerReport.Add($ReportItem) > $Null
}

#Output the report to the screen/Gridview
$CustomerReport | Out-GridView
```

The important parts of this example are the simple one line calls for the **New-CentralConnection** , **Get-NCCsutomerList** and **Get-NCCustomerPropertyList**. With very little effort we can connect, retrieve the data then process into a single table for review.

## PowerShell WebserviceProxy

In this section we'll perform the same example as above but using the native cmdlets we'll go through an example of a fully functioning cmdlet that uses native cmdlet calls.

```powershell
# Define the command-line parameters to be used by the script
[CmdletBinding()]
Param(
    [Parameter(Mandatory = $true)]$serverHost,
    [Parameter(Mandatory = $true)]$JWT
)
```

Here we establish the mandatory variables we'll be using to connect, in this case the username of the automation account used, the server URI and the Java Web Token

We'll then connect using a static namespace, you can equally use the pseudo-unique namespace above.

```powershell
$NWSNameSpace = "NAbleAPI"
$KeyPairType = $NWSNameSpace.eiKeyValue
```

# Bind to the namespace, using the Webserviceproxy
```powershell
$bindingURL = "https://" + $serverHost + "/dms2/services2/ServerEI2?wsdl"
$nws = New-Webserviceproxy $bindingURL -Namespace ($NWSNameSpace)
```
Using a static namespace can be useful if you prefer to call the class based constructor rather than the New-Object cmdlet . Ie. when using the constructor for the EiKeyValue that is commonly required for queries.
```powershell
$KeyPairType = $NWSNameSpace.eiKeyValue
$KeyPair = New-Object -TypeName $KeyPairType
You can instead call
$KeyPair = [NAbleAPI.eiKeyValue]::new()
```
Both methods achieve the construction of the eiKeyValue instance, but one can be used where there is a syntactic preference for C#/classic programming rather than cmdlets.

Next we wrap the steps of retrieving the Customer List and Organisation properties list with a try/catch block that exits if there is an error with retrieving the data.
```powershell
#Attempt to connect
Try {
    $CustomerList = $nws.customerList("", $JWT, $KeyPair)
    $OrgPropertiesList = $nws.organizationPropertyList("", $JWT, $null, $false)
}

Catch {
    Write-Host Could not connect: $($_.Exception.Message)
    exit
}
```

We then create a hash table of all the customer properties with the Customer ID as the Key and the Properties as the value.

```powershell
$OrgPropertiesHashTable = @{}
foreach ($Org in $OrgPropertiesList) {
    $CustomerOrgProps = @{}
    foreach ($p in $Org.properties) { $CustomerOrgProps[$p.label] = $p.value }
    $OrgPropertiesHashTable[$Org.customerId] = $CustomerOrgProps

}
```

In the next step we take create an ArrayList in preference to a simple array to increase performance on inserts, then enumerate through the customer list match a custom list of tables and join it to the properties Hash table and output it to the screen.

```powershell
#Create customer report ArrayList
$CustomersReport = New-Object System.Collections.ArrayList

ForEach ($Entity in $CustomerList) {
    $CustomerAssetInfo = @{}
    #Custom select the required columns, use Ordered to keep them in the order we list them in

    ForEach ($item in $Entity.items) { $CustomerAssetInfo[$item.key] = $item.Value }
    $o = [Ordered]@{
        ID                = $CustomerAssetInfo["customer.customerid"]
        Name              = $CustomerAssetInfo["customer.customername"]
        parentID          = $CustomerAssetInfo["customer.parentid"]
        RegistrationToken = $CustomerAssetInfo["customer.registrationtoken"]
    }

    #Retrieve the properties for the given customer ID
    $p = $OrgPropertiesHashTable[[int]$CustomerAssetInfo[customer.customerid]]

    #Merge the two hashtables
    $o = $o + $p

    #Cast the hashtable to a PSCustomObject
    $o = [PSCustomObject]$o

    #Add the PSCustomObject to our CustomersReport ArrayList
    $CustomersReport.Add($o) > $null
}

#Output to the screen
$CustomersReport | Out-GridView
```

For the complete script see **Appendix C – GetAllCustomerProperties.ps1**

# Updating a Value

A common case for updating a value would be automating the update/change of Organisation or Device properties. Examples of Organisation properties could be: tokens/keys for MSP applications deployed to devices, the customer name to pass through to a script for output or the N-Central API registration token for installation of the agent. Updating these properties is straightforward with either the web proxy or PS-NCentral cmdlets.

Please note at time of writing the only way for a Registration Token to be generated is through the UI, requiring an administrator to navigate to every _customer_ and every _site_ and click on the **Get Registration Token button** under **Download Agent/Probe** ; this will hopefully change in future. Plan your token expiration period around this.

## PS-NCentral

In the example for PS-NCentral we'll take the Customer name from the Get-NCCustomerList and inject it into _custom_ property called  **Agent – Registration Token**, this is useful if we need to programmatically inject token information into files or registry settings for later use. We'll assume we already have a connection to N-Central:
```powershell
$CustomerList = Get-NCCustomerList
foreach ($Customer in $CustomerList){
    Set-NCCustomerProperty -CustomerIDs $Customer.customerid -PropertyLabel Agent - RegistrationToken -PropertyValue $Customer.registrationtoken
}
```

Or if we wanted to take the customer's name and inject it into a custom property like  **Reporting – Customer Name** to inject it into an AMP's output for easier to identify the device AMPs run across a service organization:
```powershell
$CustomerList = Get-NCCustomerList

foreach ($Customer in $CustomerList){
    Set-NCCustomerProperty -CustomerIDs $Customer.customerid -PropertyLabel Reporting – Customer Name -PropertyValue $Customer.customername

}
```
Another advantage of the Set-NCCustomerProperty cmdlet is that it can distinguish between the default _customer_ properties, ie. zip/postalcode, street1, externalid, externalid2 and will use the appropriate method to update that. You can find the list of key names in the Java Docs, or refer to **Appendix D – Customer Property variables**.

For our example, you may have a PSA or perhaps a spreadsheet, and we want to refresh the information from that data source into _customer_ properties. In our example we'll use a spreadsheet/CSV as our datasource, and assume we have already matched the CustomerID with the company name and we have a dataset as below:

| **customerid** | **firstname** | **lastname** | **email** |
| --- | --- | --- | --- |
| 1 | Claire | Young | Claire.Young@email.com |
| 2 | Benjamin | Metcalfe | Bmetacalfe@usa.com |
| 3 | Kimberly | King | kk@asia.com |
| 4 | Michael | Mills | Mmills@engineer.com |
| 5 | Anthony | Jackson | 008Jac@mail.com |

You could then update the respective values in N-Central
```powershell
$custData = Import-CSV C:\Temp\customerData.csv

foreach ($Customer in $custData){
    #Gather properties to update
    $Properties = $Customer.psobject.properties | ?{$_.Name -ne 'customerid'}

    foreach ($Property in $Properties){
        #Update the property
        Set-NCCustomerProperty -CustomerIDs $Customer.customerid -PropertyLabel $Property -PropertyValue $Property.Value
    }
}
```

## PowerShell WebserviceProxy

Updating customer properties and without the PS-NCentral cmdlets can take several additional steps as PS-NCentral takes care of some busy work underneath.

### Registration token injection

Let's first take the example of injecting taking the registration token from the **customerList** method and injecting it via the **organizationPropertyModify** method. As above we'll assume we have a customer already and our list of customers is in the variable $CustomerList, take note of the line where gathering the value of the custom property with the id **123456789**.
```powershell
ForEach ($Entity in $CustomerList) {
    $CustomerAssetInfo = @{}
    ForEach ($item in $Entity.items) { $CustomerAssetInfo[$item.key] = $item.Value }
    
    #Create a custom object for the data
    $CustomerObject = [Ordered]@{
        ID                = $CustomerAssetInfo[customer.customerid]
        Name              = $CustomerAssetInfo[customer.customername]
        parentID          = $CustomerAssetInfo[customer.parentid]
        RegistrationToken = $CustomerAssetInfo[customer.registrationtoken]
    }

    #Skip any Registration tokens that are null/empty
    if ($null -eq $CustomerObject.RegistrationToken -or  -eq $CustomerObject.RegistrationToken) {continue}
    
    #Gather the property value for the specific property ID
    $CustomerProperty = ($OrgPropertiesList | ?{$_.customerId -eq $CustomerObject.ID}).properties | ?{$_.propertyid -eq 123456789}
    $CustomerProperty.value = $CustomerObject.RegistrationToken
    
    #Create a new OrganizationProperties object and populate it
    $ModOrgProperties = New-Object $NWSNameSpace.OrganizationProperties
    $ModOrgProperties.customerId = $CustomerObject.ID
    $ModOrgProperties.customerIdSpecified = $true
    $ModOrgProperties.properties = $CustomerProperty

    #Inject the property
    $nws.organizationPropertyModify("",$JWT,$ModOrgProperties)
}
```
### Gather organization property ID

While the above code works in updating the specific customer org property, one must first interrogate the properties and their associated values in advance. While this is fine for scripts where you will always be updating the same **propertyid** , you may wish to implement a function that takes care of searching and retrieving the **propertyid**.

PS-NCentral cmdlets use a class to retrieve this, we can convert it to a function for our use:
```powershell
function Get-OrganizationPropertyID(
    [Int]$OrganizationID,
    [String]$PropertyName,
    [String]$JWT,
    [System.Web.Services.Protocols.SoapHttpClientProtocol]$NcConnection){

    ## Returns 0 (zero) if not found.
    $OrganizationPropertyID = 0
    $results = $null
    Try{
        #Gets the organization and all custom properties
        $results = $NcConnection.OrganizationPropertyList("", $JWT, $OrganizationID, $false)
    }
    Catch {
        $_.ErrorHandler
    }

    #Search through all properties and match the one with the same name
    ForEach ($OrganizationProperty in $results.properties){
        If($OrganizationProperty.label -eq $PropertyName){
            $OrganizationPropertyID = $OrganizationProperty.PropertyID
        }
    }
    Return $OrganizationPropertyID
}
```
We can then use the function to gather the property ID

```powershell
#Get the property id for org 123 with the property name Agent – Registration token
Get-OrganizationPropertyID -OrganizationID 123 -PropertyName "Agent - Registration Token" -JWT $JWT -NcConnection $nws
```

The author notes that at time of writing, the **propertyid** appears to be the same for all customers/sites created at the same hierarchical level (System/SO/Customer/Site). For cases where you have multiple service organizations with the same named custom property created at the SO level they should be a different propertyid.

For single SO deployments where the custom properties are created at the SO level they are globally unique, you could also create a lookup table for optimising your code to avoid performing a propertyid lookup for each update of a custom property, though we won't be covering that in this document.

### Update customer property

Updating a _customer_ property such as the contact details or externalid values is done through the **customerModify** method, the method is called with the form:

```customerModify([string]username,[string]password,[ListEiKeyValue]settings)```

You may note in the PS-NCentral example it can update one property, either custom or customer, in a single call; whereas the EiKeyValue list can contain one or all of the customer values shown in **Appendix D – Customer Property variables** to be updated in a single call.

We'll use an expanded data set from the PS-NCentral as we have more mandatory fields **customername** , **customerid** and **parentid** that are otherwise looked up by an internal helper function in PS-NCentral:

| parentid | customerid | customername | firstname | lastname | email |
| --- | --- | --- | --- | --- | --- |
| 50 | 1 | Contoso | Claire | Young | Claire.Young@email.com |
| 50 | 2 | Volcano Coffee | Benjamin | Metcalfe | Bmetacalfe@usa.com |
| 50 | 3 | Northwind Traders | Kimberly | King | kk@asia.com |
| 50 | 4 | WW Importers | Michael | Mills | Mmills@engineer.com |
| 50 | 5 | Blue Yonder | Anthony | Jackson | 008Jac@mail.com |

You can retrieve the mandatory fields mentioned in the above table by retrieving the customer information as demonstrated in section 3.2.

In the below example the data is imported, then we generate the appropriate EIKeyValue array and assuming use the same $nwsconnection variable and $NWSNameSpace from previous examples to connect and update the modified keys.
```powershell
$KeyPairType = $NWSNameSpace.eiKeyValue
$custData = Import-CSV C:\Temp\customerData.csv
foreach ($Customer in $custData){
    #Gather properties to update
    $Properties = $Customer.psobject.properties
    #Create an Arraylist of EIKeyValues to update
    $ModifiedKeyList = New-Object System.Collections.ArrayList
    $Properties | ForEach-Object{
        $KeyPair = New-Object -TypeName $KeyPairType
        $KeyPair.key = $_.Name
        $KeyPair.value = $_.Value
        $ModifiedKeyList.Add($KeyPair)
    }
    $nws.customerModify("",$JWT,$ModifiedKeyList)
}
```
### Add new a new Customer

Not every cmdlet is currently available in PS-NCentral, one such cmdlet that could be useful is the automation of the creation of customer accounts. In the below example we use the ```$nws``` connection from before and pass through a hashtable of some of the customer properties in in Appendix D – Customer Property variables, note there are two required fields: **customername** and **partentid**

Combining the hashtable $newcustomer with the $JWT and $nws to the cmdlet it will create the customer. Note there is no output from the command but you should see the customer/site appear in N-Central.
```powershell
$newcustomer = @{
    customername = "New Customer"
    parentid = "50"
    firstname = "john"
    lastname = "doe"
    email = "john.doe@contoso.com"
    city = "Melbourne"
    telephone = "0312345678"
    country = "AU"
}

function Add-NCCustomer(
    [Hashtable]$CustomerTable,
    [String]$JWT,
    [System.Web.Services.Protocols.SoapHttpClientProtocol]$NcConnection) {
    $internalNameSpace = $NcConnection.ToString().Split('.')[0]
    $CustomerEiKeyList = New-Object System.Collections.ArrayList
    foreach ($key in $CustomerTable.Keys){
        $KeyPairType = $internalNameSpace.eiKeyValue
        $KeyPair = New-Object -TypeName $KeyPairType
        $KeyPair.key = $key
        $KeyPair.value = $CustomerTable[$key]
        $CustomerEiKeyList.Add($KeyPair)
    }
    $NcConnection.customerAdd("", $JWT, $CustomerEiKeyList)
}

Add-NCCustomer -CustomerTable $newcustomer -JWT $JWT -NcConnection $nws
```
# Appendix A – N-Central Web Service members

|Name |MemberType|
|---- |----------|
|accessGroupAddCompleted | Event |
|accessGroupGetCompleted | Event |
|accessGroupListCompleted | Event |
|acknowledgeNotificationCompleted | Event |
|activeIssuesListCompleted | Event |
|customerAddCompleted | Event |
|customerDeleteCompleted | Event |
|customerListChildrenCompleted | Event |
|customerListCompleted | Event |
|customerModifyCompleted | Event |
|deviceAssetInfoExportDeviceCompleted | Event |
|deviceAssetInfoExportDeviceWithSettingsCompleted | Event |
|deviceGetCompleted | Event |
|deviceGetStatusCompleted | Event |
|deviceListCompleted | Event |
|devicePropertyListCompleted | Event |
|devicePropertyModifyCompleted | Event |
|Disposed | Event |
|jobStatusListCompleted | Event |
|lastExportResetCompleted | Event |
|organizationPropertyListCompleted | Event |
|organizationPropertyModifyCompleted | Event |
|psaCreateCustomTicketCompleted | Event |
|psaCredentialsValidateCompleted | Event |
|psaGetCustomTicketCompleted | Event |
|psaReopenCustomTicketCompleted | Event |
|psaResolveCustomTicketCompleted | Event |
|SOAddCompleted | Event |
|taskPauseMonitoringCompleted | Event |
|taskResumeMonitoringCompleted | Event |
|userAddCompleted | Event |
|userRoleAddCompleted | Event |
|userRoleGetCompleted | Event |
|userRoleListCompleted | Event |
|versionInfoGetCompleted | Event |
|Abort | Method |
|accessGroupAdd | Method |
|accessGroupAddAsync | Method |
|accessGroupGet | Method |
|accessGroupGetAsync | Method |
|accessGroupList | Method |
|accessGroupListAsync | Method |
|acknowledgeNotification | Method |
|acknowledgeNotificationAsync | Method |
|activeIssuesList | Method |
|activeIssuesListAsync | Method |
|BeginaccessGroupAdd | Method |
|BeginaccessGroupGet | Method |
|BeginaccessGroupList | Method |
|BeginacknowledgeNotification | Method |
|BeginactiveIssuesList | Method |
|BegincustomerAdd | Method |
|BegincustomerDelete | Method |
|BegincustomerList | Method |
|BegincustomerListChildren | Method |
|BegincustomerModify | Method |
|BegindeviceAssetInfoExportDevice | Method |
|BegindeviceAssetInfoExportDeviceWithSettings | Method |
|BegindeviceGet | Method |
|BegindeviceGetStatus | Method |
|BegindeviceList | Method |
|BegindevicePropertyList | Method |
|BegindevicePropertyModify | Method |
|BeginjobStatusList | Method |
|BeginlastExportReset | Method |
|BeginorganizationPropertyList | Method |
|BeginorganizationPropertyModify | Method |
|BeginpsaCreateCustomTicket | Method |
|BeginpsaCredentialsValidate | Method |
|BeginpsaGetCustomTicket | Method |
|BeginpsaReopenCustomTicket | Method |
|BeginpsaResolveCustomTicket | Method |
|BeginSOAdd | Method |
|BegintaskPauseMonitoring | Method |
|BegintaskResumeMonitoring | Method |
|BeginuserAdd | Method |
|BeginuserRoleAdd | Method |
|BeginuserRoleGet | Method |
|BeginuserRoleList | Method |
|BeginversionInfoGet | Method |
|CancelAsync | Method |
|CreateObjRef | Method |
|customerAdd | Method |
|customerAddAsync | Method |
|customerDelete | Method |
|customerDeleteAsync | Method |
|customerList | Method |
|customerListAsync | Method |
|customerListChildren | Method |
|customerListChildrenAsync | Method |
|customerModify | Method |
|customerModifyAsync | Method |
|deviceAssetInfoExportDevice | Method |
|deviceAssetInfoExportDeviceAsync | Method |
|deviceAssetInfoExportDeviceWithSettings | Method |
|deviceAssetInfoExportDeviceWithSettingsAsync | Method |
|deviceGet | Method |
|deviceGetAsync | Method |
|deviceGetStatus | Method |
|deviceGetStatusAsync | Method |
|deviceList | Method |
|deviceListAsync | Method |
|devicePropertyList | Method |
|devicePropertyListAsync | Method |
|devicePropertyModify | Method |
|devicePropertyModifyAsync | Method |
|Discover | Method |
|Dispose | Method |
|EndaccessGroupAdd | Method |
|EndaccessGroupGet | Method |
|EndaccessGroupList | Method |
|EndacknowledgeNotification | Method |
|EndactiveIssuesList | Method |
|EndcustomerAdd | Method |
|EndcustomerDelete | Method |
|EndcustomerList | Method |
|EndcustomerListChildren | Method |
|EndcustomerModify | Method |
|EnddeviceAssetInfoExportDevice | Method |
|EnddeviceAssetInfoExportDeviceWithSettings | Method |
|EnddeviceGet | Method |
|EnddeviceGetStatus | Method |
|EnddeviceList | Method |
|EnddevicePropertyList | Method |
|EnddevicePropertyModify | Method |
|EndjobStatusList | Method |
|EndlastExportReset | Method |
|EndorganizationPropertyList | Method |
|EndorganizationPropertyModify | Method |
|EndpsaCreateCustomTicket | Method |
|EndpsaCredentialsValidate | Method |
|EndpsaGetCustomTicket | Method |
|EndpsaReopenCustomTicket | Method |
|EndpsaResolveCustomTicket | Method |
|EndSOAdd | Method |
|EndtaskPauseMonitoring | Method |
|EndtaskResumeMonitoring | Method |
|EnduserAdd | Method |
|EnduserRoleAdd | Method |
|EnduserRoleGet | Method |
|EnduserRoleList | Method |
|EndversionInfoGet | Method |
|Equals | Method |
|GetHashCode | Method |
|GetLifetimeService | Method |
|GetType | Method |
|InitializeLifetimeService | Method |
|jobStatusList | Method |
|jobStatusListAsync | Method |
|lastExportReset | Method |
|lastExportResetAsync | Method |
|organizationPropertyList | Method |
|organizationPropertyListAsync | Method |
|organizationPropertyModify | Method |
|organizationPropertyModifyAsync | Method |
|psaCreateCustomTicket | Method |
|psaCreateCustomTicketAsync | Method |
|psaCredentialsValidate | Method |
|psaCredentialsValidateAsync | Method |
|psaGetCustomTicket | Method |
|psaGetCustomTicketAsync | Method |
|psaReopenCustomTicket | Method |
|psaReopenCustomTicketAsync | Method |
|psaResolveCustomTicket | Method |
|psaResolveCustomTicketAsync | Method |
|SOAdd | Method |
|SOAddAsync | Method |
|taskPauseMonitoring | Method |
|taskPauseMonitoringAsync | Method |
|taskResumeMonitoring | Method |
|taskResumeMonitoringAsync | Method |
|ToString | Method |
|userAdd | Method |
|userAddAsync | Method |
|userRoleAdd | Method |
|userRoleAddAsync | Method |
|userRoleGet | Method |
|userRoleGetAsync | Method |
|userRoleList | Method |
|userRoleListAsync | Method |
|versionInfoGet | Method |
|versionInfoGetAsync | Method |
|AllowAutoRedirect | Property |
|ClientCertificates | Property |
|ConnectionGroupName | Property |
|Container | Property |
|CookieContainer | Property |
|Credentials | Property |
|EnableDecompression | Property |
|PreAuthenticate | Property |
|Proxy | Property |
|RequestEncoding | Property |
||Site | Property |
|SoapVersion | Property |
|Timeout | Property |
|UnsafeAuthenticatedConnectionSharing | Property |
|Url | Property |
|UseDefaultCredentials | Property |
|UserAgent | Property |

# Appendix B PS-NCentral cmdlets

| Command | Synopsis |
| --- | --- |
| Get-NCAccessGroupList | Returns the list of AccessGroups at the specified CustomerID level. |
| Get-NCActiveIssuesList | Returns the Active Issues on the CustomerID-level and below. |
| Get-NCCustomerList | Returns a list of all customers and their data. ChildrenOnly when CustomerID is specified. |
| Get-NCCustomerPropertyList | Returns a list of all Custom-Properties for the selected CustomerID(s). |
| Get-NCDeviceID | Returns the DeviceID(s) for the given DeviceName(s). Case Sensitive, No Wildcards. |
| Get-NCDeviceInfo | Returns the General details for the DeviceID(s). |
| Get-NCDeviceList | Returns the Managed Devices for the given CustomerID(s) and Sites below. |
| Get-NCDeviceLocal | Returns the DeviceID, CustomerID and some more Info for the Local Computer. |
| Get-NCDeviceObject | Returns a Device and all asset-properties as an object. |
| Get-NCDevicePropertyList | Returns the Custom Properties of the DeviceID(s). |
| Get-NCDevicePropertyListFilter | Returns the Custom Properties of the Devices within the Filter(s). |
| Get-NCDeviceStatus | Returns the Services for the DeviceID(s). |
| Get-NCHelp | Shows a list of available PS-NCentral commands and the synopsis. |
| Get-NCJobStatusList | Returns the Scheduled Jobs on the CustomerID-level and below. |
| Get-NCProbeList | Returns the Probes for the given CustomerID(s). |
| Get-NCServiceOrganizationList | Returns a list of all ServiceOrganizations and their data. |
| Get-NCTimeOut | Returns the max. time in seconds to wait for data returning from a (Synchronous) NCentral API-request. |
| Get-NCUserRoleList | Returns the list of Roles at the specified CustomerID level. |
| NcConnected | Checks or initiates the NCentral connection. |
| New-NCentralConnection | Connect to the NCentral server. |
| Set-NCCustomerProperty | Fills the specified property(name) for the given CustomerID(s). |
| Set-NCDeviceProperty | Fills the Custom Property for the DeviceID(s). |
| Set-NCTimeOut | Sets the max. time in seconds to wait for data returning from a (Synchronous) NCentral API-request. |

# Appendix C – GetAllCustomerProperties.ps1

```powershell
# Define the command-line parameters to be used by the script
[CmdletBinding()]
Param(
    [Parameter(Mandatory = $true)]$serverHost,
    [Parameter(Mandatory = $true)]$JWT
)
# Generate a pseudo-unique namespace to use with the New-WebServiceProxy and associated types.
$NWSNameSpace = NAble + ([guid]::NewGuid()).ToString().Substring(25)
$KeyPairType = $NWSNameSpace.eiKeyValue

# Bind to the namespace, using the Webserviceproxy
$bindingURL = "https://" + $serverHost + "/dms2/services2/ServerEI2?wsdl"
$nws = New-Webserviceproxy $bindingURL -Namespace ($NWSNameSpace)

# Set up and execute the query
$KeyPair = New-Object -TypeName $KeyPairType
$KeyPair.Key = 'listSOs'
$KeyPair.Value = false

#Attempt to connect
Try {
    $CustomerList = $nws.customerList("", $JWT, $KeyPair)
    $OrgPropertiesList = $nws.organizationPropertyList("", $JWT, $null, $false)
}
Catch {
    Write-Host Could not connect: $($_.Exception.Message)
    exit
}

$OrgPropertiesHashTable = @{}
foreach ($Org in $OrgPropertiesList) {
    $CustomerOrgProps = @{}
    foreach ($p in $Org.properties) { $CustomerOrgProps[$p.label] = $p.value }
    $OrgPropertiesHashTable[$Org.customerId] = $CustomerOrgProps

}

#Create customer report ArrayList
$CustomersReport = New-Object System.Collections.ArrayList
ForEach ($Entity in $CustomerList) {
    $CustomerAssetInfo = @{}
  
    #Custom select the required columns, us Ordered to keep them in the order we list them in
    ForEach ($item in $Entity.items) { $CustomerAssetInfo[$item.key] = $item.Value }
    $o = [Ordered]@{
        ID                = $CustomerAssetInfo[customer.customerid]
        Name              = $CustomerAssetInfo[customer.customername]
        parentID          = $CustomerAssetInfo[customer.parentid]
        RegistrationToken = $CustomerAssetInfo[customer.registrationtoken]
    }

    #Retrieve the properties for the given customer ID
    $p = $OrgPropertiesHashTable[[int]$CustomerAssetInfo[customer.customerid]]

    #Merge the two hashtables
    $o = $o + $p
    #Cast the hashtable to a PSCustomObject
    $o = [PSCustomObject]$o
    
    #Add the PSCustomObject to our CustomersReport ArrayLIst
    $CustomersReport.Add($o) > $null
}

#Output to the screen
$CustomersReport | Out-GridView
```
# Appendix D – Customer Property variables

- **zip/postalcode** - (Value) Customer's zip/ postal code.
- **street1** - (Value) Address line 1 for the customer. Maximum of 100 characters.
- **street2** - (Value) Address line 2 for the customer. Maximum of 100 characters.
- **city** - (Value) Customer's city.
- **state/province** - (Value) Customer's state/ province.
- **telephone** - (Value) Phone number of the customer.
- **country** - (Value) Customer's country. Two character country code, see http://en.wikipedia.org/wiki/ISO\_3166-1\_alpha-2 for a list of country codes.
- **externalid** - (Value) An external reference id.
- **firstname** - (Value) Customer contact's first name.
- **lastname** - (Value) Customer contact's last name.
- **title** - (Value) Customer contact's title.
- **department** - (Value) Customer contact's department.
- **contact\_telephone** - (Value) Customer contact's telephone number.
- **ext** - (Value) Customer contact's telephone extension.
- **email** - (Value) Customer contact's email. Maximum of 100 characters.
- **licensetype** - (Value) The default license type of new devices for the customer. Must be Professional or Essential. Default is Essential.
