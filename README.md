# How To: N-Central API Automation

## Table of Content

- [How To: N-Central API Automation](#how-to--n-central-api-automation)
  * [Table of Content](#table-of-content)
- [Overview](#overview)
- [Connecting](#connecting)
  * [PS-NCentral](#ps-ncentral)
    + [Multiple PS-NCentral server connections](#multiple-ps-ncentral-server-connections)
  * [PowerShell WebserviceProxy](#powershell-webserviceproxy)
- [Performing Queries](#performing-queries)
  * [PS-NCentral](#ps-ncentral-1)
    + [Advanced PS-NCentral querying](#advanced-ps-ncentral-querying)
  * [PowerShell WebserviceProxy](#powershell-webserviceproxy-1)
    + [Bind to the namespace, using the Webserviceproxy](#bind-to-the-namespace--using-the-webserviceproxy)
- [Updating a Value](#updating-a-value)
  * [PS-NCentral](#ps-ncentral-2)
    + [Updating with pipelining](#updating-with-pipelining)
    + [Updating Custom Device Properties](#updating-custom-device-properties)
    + [Custom Property options](#custom-property-options)
      - [Encoding](#encoding)
      - [Comma-Separated Values](#comma-separated-values)
      - [Format-properties](#format-properties)
  * [PowerShell WebserviceProxy](#powershell-webserviceproxy-2)
    + [Registration token injection](#registration-token-injection)
    + [Gather organization property ID](#gather-organization-property-id)
    + [Update customer property](#update-customer-property)
    + [Add new a new Customer](#add-a-new-customer)
    + [Add new a new User](#add-a-new-user)
- [Appendix A – N-Central Web Service members](#appendix-a---n-central-web-service-members)
- [Appendix B - PS-NCentral cmdlets](#appendix-b---ps-ncentral-cmdlets)
- [Appendix C – GetAllCustomerProperties.ps1](#appendix-c---getallcustomerpropertiesps1)
- [Appendix D – Customer Property variables](#appendix-d---customer-property-variables)
- [Appendix E - All PS-Central Methods](#appendix-e---all-ps-central-methods)
- [Appendix F - Common Error Codes](#appendix-f---common-error-codes)
- [Appendix G - Issue Status](#appendix-g---issue-status)
- [Appendix H - WebserviceProxy Alternative](#appendix-h---webserviceproxy-alternative)
- [Credits](#credits)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>


# Overview

N-Central's API is a flexible, programmatic, object oriented, Java based interface by which developers can achieve integration and automation via native SOAP API calls.

For the purposes of this guide we'll be covering connectivity and basic usage with PowerShell based automation through the PS-NCentral module, as well as native WebserviceProxy cmdlet.

The information covering the PS-NCentral is useful for those with starting some experience with PowerShell or need to quickly put together code where module dependency isn't an issue, while the usage of the WebserviceProxy method is for those more familiar with object oriented coding or need code portatability. As of version 1.2 PS-NCentral uses PowerShell 7 for cross compatability to be able to run Windows/Linux or in an Azure function.

PS-NCentral provides cmdlets for 17 Get cmdlets and 4 Set cmdlets (See Appendix B) that cover the majority, so should cover the majority of automation. This can be downloaded from: [https://github.com/ToschAutomatisering/PS-NCentral](https://github.com/ToschAutomatisering/PS-NCentral)

Or installed from PS-Gallery with the cmdlet

```powershell
Install-Module PS-NCentral
```

# Connecting

The first step required before connecting is to create a new automation account with appropriate role permissions. With N-Central 2020 or 12.3 HF4 and later you must **disable the MFA** requirement for the account, so use a long and complex password. The (preferred) API-Only account in N-Central 2021 needs to be set to \'mfa not needed\'.

Once the account is created, select the API Authentication tab and click on the ' **Generate JSON Web Token**' button, save this **JWT** token somewhere secure, if you lose your JWT, you can generate another one at any time, but it will invalidate the previous one. If you update/change role permissions for the account automation account you will need to regenerate the token, as the asserted permissions are in the JWT.

Note: Activating the MFA-setting after generating the token still blocks JWT-access, as well as an expired password for the originating account (90 days default).

## PS-NCentral

Connecting to your N-Central service with PS-NCentral only needs to be done once per session. You first require the following information:

- The fqdn of your N-Central server, ie: `n-central.mydomain.com`
- The JWT from above
- Username/Password (no MFA) for versions **before 1.2**.

Then enter the following:

**All versions**

```powershell
#Import the PS-NCentral module
import-module .\PS-NCentral.psm1 -Verbose

#$credential = Get-Credential		## This line can be used for a dialog. Skip 2 below.
$password = ConvertTo-SecureString "<Password>" -AsPlainText -Force
$credential = New-Object System.Management.Automation.PSCredential ("<User>", $password)

#Connect to NC
New-NCentralConnection -ServerFQDN "YOUR SERVER FQDN" -PSCredential $credential
```

\* Might generate an error on some complex passwords. Avoid using a ^.

**Version 1.2 and later**

```powershell
#Import the PS-NCentral module
import-module .\PS-NCentral.psm1 -Verbose

#Connect to NC using the JWT directly
New-NCentralConnection -ServerFQDN "YOUR SERVER FQDN" -JWT "YOUR JWT TOKEN"
```

which is equivalent to:

```powershell
#Import the PS-NCentral module
import-module .\PS-NCentral.psm1 -Verbose

#Credentials with JWT
$password = ConvertTo-SecureString "YOUR JWT TOKEN" -AsPlainText -Force
$credential = New-Object System.Management.Automation.PSCredential ("_JWT", $password)

#Connect to NC
New-NCentralConnection -ServerFQDN "YOUR SERVER FQDN" -PSCredential $credential
```
If successful you will get an output similar to the below:

|Property | Value|
|--------|-----|
| ConnectionURL | `n-central.mydomain.com` |
| BindingURL        | `https://n-central.mydomain.com/dms2/services2/ServerEI2?wsdl' |
| AllProtocols      | tls12,tls13                                                  |
| IsConnected | True |
| RequestTimeOut | 100 |
| NCVersion | 2020.1.5.425 |
| DefaultCustomerID | 50 |
| Error |  |

The session is now stored in the global **$\_NCSession** variable, and will automatically be used as default for following PS-NCentral commands.

### Multiple PS-NCentral server connections

If you are an MSP with multiple N-Central servers, or have an NFC server for testing you can leverage the **-NCSession** parameter available on PS-NCentral cmdlets to quickly call other servers, this is available in all versions of PS-NCentral, but we'll use 1.2 as the example for brevity.

Then enter the following:
```powershell
#Connect to NC
$Connection1 = New-NCentralConnection "$NCentralFQDN1" -JWT "$JWT1"
$Connection2 = New-NCentralConnection "$NCentralFQDN2" -JWT "$JWT2"

#Get the customer list from each server for later processing
$NC1Customers = Get-NCCustomerList -NcSession $Connection1
$NC2Customers = Get-NCCustomerList -NcSession $Connection2
```

Another useful pameter when connecting is the **DefaultCustomerID**, this sets the default scope for when calling cmdlets such as Get-NCDeviceList without a parameter, so if I were to perform the following connection and function call it would only give me all devices associated with CustomerID 333

``` powershell
New-NCentralConnection "$NCentralFQDN" -JWT "$JWT1" -DefaultCustomerID 333
$Customer333Devices = Get-NCDeviceList
```

You can use **Set-NCCustomerDefault** to change the value afterwards.

## PowerShell WebserviceProxy

Notice the WebserviceProxy is discontinued after PowerShell version 5.1. You can find alternative code in [Appendix H - WebserviceProxy Alternative](#appendix-h---webserviceproxy-alternative)


As a preface to the usage of the New-WebserviceProxy cmdlet, we will focus on the v2 rather than v1 legacy API as the v1 endpoint maybe deprecated at some point.

The main differences between the v1 and v2 endpoints are:

- The WSDL endpoint
- Different classes, including the KeyPair constructor class used for adding custom settings for queries and update/change methods
- V2 has all the newer methods available

It will be necessary to review the Javadocs provided on your server for the lastest information on the classes and constructors, you can find them under your own N-Central server under `https://n-central.mydomain.com/dms/`

If reviewing other WebserviceProxy powershell code on the internet, you can identify **v1**/legacy code as it will have the following in the binding URL string: **/dms/services/ServerEI?wsdl** while **v2** has **/dms2/services2/ServerEI2?wsdl**

For connecting to webservice you will need the same information as with the PS-NCentral which connects in the same way underneath:

- The fqdn of your N-Central server, ie: n-central.myserver.com
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


The above output has been shortened, see [Appendix A – N-Central Web Service members](#appendix-a---n-central-web-service-members) members for the complete output. In addition you will have a **Definition** column, and you will observe that your `$NWSNameSpace` is seen prefixed to the methods/classes noted in them. All classes/methods/constructors available in the Javadocs can be created and called upon through the `$nws` variable. Eg. The customerListmethod would be called with

```powershell
$nws.customerList("", $JWT, $settings)
```

As you will note when connecting with the `$nws` variable, at no point did you use your username or JWT, as you will observe in the `$nws.customerList` method called above, the  $JWT is used in every get or set, and the username is simply `""` as the username is inside of the JWT string.

Underneath the PS-NCentral module it saves these variables with each connection and re-uses each time a cmdlet is used.

# Performing Queries

## PS-NCentral

Performing queries with the PS-Central module is quick and easy, and several examples are provided in it's own documentation [here](https://github.com/ToschAutomatisering/PS-NCentral/blob/master/PS-NCentral_Examples.ps1) . The outcomes of the examples are fairly self explanatory. For our example we'll take a common query like the Customer table and join it with the Organisation properties table using the PS-NCentral cmdlets, then in the advanced section we'll give the same example using native cmdlets.
```powershell
Import-Module PS-NCentral.psm1 -Verbose

$ServerFQDN = n-central.myserver.com
$JWT = "JWT TOKEN"

#Connect to NC
New-NCentralConnection -ServerFQDN $ServerFQDN -JWT $JWT

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

The important parts of this example are the simple one line calls for the **New-CentralConnection** , **Get-NCCustomerList** and **Get-NCCustomerPropertyList**. With very little effort we can connect, retrieve the data then process into a single table for review.

### Advanced PS-NCentral querying
The PS-NCentral module provides ease of access to N-Central API calls with normal **verb-noun** functions, but you can also perform a direct call through the internal connection class. We could replace the above function calls with these methods:

```powershell
# Connect to NC
$NCSession = New-NCentralConnection -ServerFQDN $ServerFQDN -JWT $JWT

# Grab the customer list/details
$CustomerList = $NCSession.CustomerList()

# Get the customer properties
$CustomerPropertyList = $NCSession.OrganizationPropertyList()
```

We can get the list of all the underlying class connection methods by enumerating the members with `$_NCSession | Get-Member -MemberType Method` to see all 'inside' methods, which reflect the API-methods of N-Central ('http://mothership.n-able.com/dms/javadoc_ei2/com/nable/nobj/ei2/ServerEI2_PortType.html') where applicable.

Most methods have 'Overloads'. These are selected based on the parameter-pattern eg. `([String], [String])` or `([String],[Int])`.

```powershell
#Get the devices for customerid 100
$Customer100Devices = $NCSession.DeviceList(100)
#or
$Customer100Devices = $NCSession.DeviceList(100,$true,$false)

#Get the probes for customerid 100
$Customer100Probes = $NCSession.DeviceList(100,$false,$true)
```

For a list of all methods see [Appendix E - All PS-Central Methods](#appendix-e---all-ps-central-methods)

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
```

### Bind to the namespace, using the Webserviceproxy
```powershell
$bindingURL = "https://" + $serverHost + "/dms2/services2/ServerEI2?wsdl"
$nws = New-Webserviceproxy $bindingURL -Namespace ($NWSNameSpace)
```
For many API calls a list of settings are required, in the case of the `CustomerList()` method we need to specify if the service organisation should be listed or not. The JavaDocs specify you have to use the `EiKeyValue` KeyPair type or array of `EiKeyValuesList` per the Javadocs, but it is simpler to create an ArrayList and add a generic hashtable Key/Pair that will be automatically cast to the `EiKeyValue`.

```powershell
$settings = New-Object System.Collections.ArrayList
$settings.Add(@{key = "listSOs"; value = "True" })
```

Next we wrap the steps of retrieving the Customer List and Organisation properties list with a try/catch block that exits if there is an error with retrieving the data.
```powershell
#Attempt to connect
Try {
    $CustomerList = $nws.customerList("", $JWT, $Settings)
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

For the complete script see [Appendix C – GetAllCustomerProperties.ps1](#appendix-c---getallcustomerpropertiesps1)



Note that **WebServiceProxy** is **discontinued** by Microsoft in PowerShell versions after 5.1. You can find alternative code in [Appendix H - WebserviceProxy Alternative](#appendix-h---webserviceproxy-alternative)


# Updating a Value

A common case for updating a value would be automating the update/change of Organisation or Device properties. Examples of Organisation properties could be: tokens/keys for MSP applications deployed to devices, the customer name to pass through to a script for output or the N-Central API registration token for installation of the agent. Updating these properties is straightforward with either the web proxy or PS-NCentral cmdlets.

At time of writing the normal way for a Registration Token to be generated is through the UI, requiring an administrator to navigate to every _customer_ and every _site_ and click on the **Get Registration Token button** under **Download Agent/Probe** ; this will be changed in future.

If you do need to perform mass registration token updating/refreshing there is an AMP provided as a part of the InstallAgent 6 suite that has a workaround and can be found at [https://github.com/AngryProgrammerInside/InstallAgent/tree/master/AMPs](https://github.com/AngryProgrammerInside/InstallAgent/tree/master/AMPs)

## PS-NCentral

In the example for PS-NCentral we'll take the Customer name from the Get-NCCustomerList and inject it into _custom_ property called  **Agent – Registration Token**, this is useful if we need to programmatically inject token information into files or registry settings for later use. We'll assume we already have a connection to N-Central:

```powershell
$CustomerList = Get-NCCustomerList
foreach ($Customer in $CustomerList){
    Set-NCCustomerProperty -CustomerIDs $Customer.customerid -PropertyLabel "Agent - RegistrationToken" -PropertyValue $Customer.registrationtoken
}
```

Or if we wanted to take the customer's name and inject it into a custom property like  **Reporting – Customer Name** to inject it into an AMP's output for easier to identify the device AMPs run across a service organization:

```powershell
$CustomerList = Get-NCCustomerList

foreach ($Customer in $CustomerList){
    Set-NCCustomerProperty -CustomerIDs $Customer.customerid -PropertyLabel "Reporting – Customer Name" -PropertyValue $Customer.customername
}
```
An advantage of the Set-NCCustomerProperty cmdlet is that it can distinguish between the default _customer_ properties, ie. zip/postalcode, street1, externalid, externalid2 and will use the appropriate method to update that. You can find the list of key names in the Java Docs, or refer to **Appendix D – Customer Property variables**.

For our example, you may have a PSA or perhaps a spreadsheet, and we want to refresh the information from that data source into _customer_ properties. In our example we'll use a spreadsheet/CSV as our datasource, and assume we have already matched the CustomerID with the company name and we have a dataset as below:

| **customerid** | **firstname** | **lastname** | **email** |
| --- | --- | --- | --- |
| 1 | Claire | Young | Claire.Young@email.com |
| 2 | Benjamin | Metcalfe | Bmetacalfe@usa.com |
| 3 | Kimberly | King | kk@asia.com |
| 4 | Michael | Mills | Mmills@engineer.com |
| 5 | Anthony | Jackson | 008Jac@mail.com |

\
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
### Updating with pipelining
Another advantage of PS-NCentral is that you can easily pipeline information through and set it as a customer property, in the first example we will update the **Reporting - Customer Name** again except this time utilising the pipe:
```powershell
Get-NCCustomerList | Set-NCCustomerProperty -PropertyLabel 'Reporting – Customer Name' -PropertyValue $_.customername
```

In the second example we may have a custom table from a CSV or other source that has the following properties and values:

| **customerid** | **CustomerSLA** |
| --- | --- |
| 123 | 1H |
| 124 | 4H |
| 221 | 8H |
| 233 | 8H |
| 321 | 8H |

\
We then have this table in a variable `$CustomerProps` and use it to populate a custom property called **'Reporting - Customer SLA'**
```powershell
Get-NCCustomerList |
Select-Object customerid, @{n="CustomerSLA"; e={$CustomerID = $_.customerid; (@($CustomerProps).where({ $_.customerID -eq $CustomerID })).CustomerSLA}} `
| Where-Object {$_.CustomerSLA} `
| % { Set-NCCustomerProperty -CustomerIDs $_.CustomerID -PropertyLabel 'Reporting – Customer SLA' -PropertyValue ($_.CustomerSLA -join ',') }
```
When multiple records for a customerid are found in $Customerprops all values will be added comma-separated.

The important parts of this example are the table-lookup 

```
(@(<TableObject>).where(<LookupColumn> -eq <LookupValue>)).<ResultsColumn>
```

and the filter to only return objects which had values added

```
Where-Object {$_.<AddedCustomField>}
```

before setting the properties.

### Updating Custom Device Properties
Another example would be where we may want to populate a Custom Device Property, in this case **'External ID'** based upon the CustomerID using in a customer table `$Customers`
| **customerid** | **ExternalID** |
| --- | --- |
| 123 | 78409377 |
| 124 | 78405890 |
| 221 | 78404905 |
| 233 | 78402984 |
| 321 | 38940384 |

```powershell
Get-NCDeviceList | `
Select-Object DeviceID, `
@{n="ExternalID"; e={$CustomerID = $_.customerid; (@($Customers).where({ $_.customerID -eq $CustomerID })).ExternalID}} | `
Where-Object {$_.ExternalID} | %{Set-NCDeviceProperty -DeviceIDs $_.DeviceID -PropertyLabel 'ExternalID' -PropertyValue ($_.ExternalID -join ',')}
```



### Custom Property options

These options are introduced in PS-NCentral version **1.3**.

#### Encoding

Use the **-Base64** option to Encode/Decode a CustomProperty when using Set or Get. Unicode (utf16) by default, -utf8 option available.

**Convert-Base64** is available as a seperate command too.



#### Comma-Separated Values

When a Custom Property holds a string of (unique) Comma-seperated values you can easily Add or Remove a single value with the commands:

- Add-NCCustomerPropertyValue
- Add-NCDevicePropertyValue
- Remove-NCCustomerPropertyValue
- Remove-NCDevicePropertyValue   

Use Get-help \<command\> to see the options.



#### Format-properties

Sometimes not all objects in a list have the same properties. This can become an issue when using **Format-Table** or **Output-CSV**, which only use the properties of the first object in the list.

The **Format-Properties** cmdlet ensures all unique properties are added to all objects in a list. It is integrated in several PS-NCentral list-commands but also available as a seperate command.



## PowerShell WebserviceProxy

Updating customer properties and without the PS-NCentral cmdlets can take several additional steps as PS-NCentral takes care of some busy work underneath.

### Registration token injection

Let's first take the example of injecting taking the registration token from the **customerList** method and injecting it via the **organizationPropertyModify** method. As above we'll assume we have a connection `$nws` already and our list of customers is in the variable $CustomerList, take note of the line where gathering the value of the custom property with the id **123456789**.
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

For single SO deployments where the custom properties are created at the SO level they are globally unique, you could also create a lookup table for optimising your code to avoid performing a **propertyid** lookup for each update of a custom property, though we won't be covering that in this document.

### Update customer property

Updating a _customer_ property such as the contact details or externalid values is done through the **customerModify** method, the method is called with the form:

```customerModify([string]username,[string]password,[ListEiKeyValue]settings)```

You may note in the PS-NCentral example it can update one property, either *custom* or customer, in a single call; whereas the KeyValue list can contain one or all of the customer values shown in [Appendix D – Customer Property variables](#appendix-d---customer-property-variables) to be updated in a single call.

We'll use an expanded data set from the PS-NCentral as we have more mandatory fields **customername** , **customerid** and **parentid** that are otherwise looked up by an internal helper function in PS-NCentral:

| parentid | customerid | customername | firstname | lastname | email |
| --- | --- | --- | --- | --- | --- |
| 50 | 1 | Contoso | Claire | Young | `Claire.Young@email.com` |
| 50 | 2 | Volcano Coffee | Benjamin | Metcalfe | `Bmetacalfe@usa.com` |
| 50 | 3 | Northwind Traders | Kimberly | King | `kk@asia.com` |
| 50 | 4 | WW Importers | Michael | Mills | `Mmills@engineer.com` |
| 50 | 5 | Blue Yonder | Anthony | Jackson | `008Jac@mail.com` |

You can retrieve the mandatory fields mentioned in the above table by using the `CustomerList()` covered previously.

In the below example the data is imported, then we generate the appropriate KeyValue array and assuming use the same $nwsconnection variable and $NWSNameSpace from previous examples to connect and update the modified keys.
```powershell
$custData = Import-CSV C:\Temp\customerData.csv
foreach ($Customer in $custData){
    #Gather properties to update
    $Properties = $Customer.psobject.properties
    #Create an Arraylist of HashTables to update
    $ModifiedKeyList = New-Object System.Collections.ArrayList
    $Properties | ForEach-Object{
        $KeyPair = @{}
        $KeyPair.key = $_.Name
        $KeyPair.value = $_.Value
        $ModifiedKeyList.Add($KeyPair)
    }
    $nws.customerModify("",$JWT,$ModifiedKeyList)
}
```
### Add a new Customer

Not every cmdlet is currently available in PS-NCentral, one such cmdlet that could be useful is the automation of the creation of customer accounts. In the below example we use the ```$nws``` connection from before and pass through a hashtable of some of the customer properties in in [Appendix D – Customer Property variables](#appendix-d---customer-property-variables), note there are two required fields: **customername** and **parentid**

Combining the hashtable `$newcustomer` with the `$JWT` and `$nws` to the cmdlet it will create the customer. It will return the new CustomerID value once the job is completed.
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
    $CustomerAttributeList = New-Object System.Collections.ArrayList
    foreach ($key in $CustomerTable.Keys){
        $setting = @{key = $key; value = $CustomerTable[$key]}
        $CustomerAttributeList.Add($setting) > $null
    }
    $NcConnection.customerAdd("", $JWT, $CustomerAttributeList)
}

Add-NCCustomer -CustomerTable $newcustomer -JWT $JWT -NcConnection $nws
```

At time of writing with PS-NCentral version 1.2 it is possible to use the CustomerAdd() method as it is exists inside the core class object now. While there is currently no Powershell function to call this, create a customer with it in the following way:

```powershell
#Connect to NC
$NCSession = New-NCentralConnection -ServerFQDN n-central.myserver.com -JWT $JWT
$ParentId = 50
$NewCustomerAttributes = @{
    firstname = "john"
    lastname = "doe"
    email = "john.doe@contoso.com"
    city = "Melbourne"
    telephone = "0312345678"
    country = "AU"
}

$NCSession.CustomerAdd("NewCustomerName",$ParentId,$NewCustomerAttributes)
```
As of version 1.6 you can add all fields (inclusding the mandatory customername and parentid) directly to $NewCustomerAttributes and use this as a single parameter for the CustomerAdd() method. 

Similar to the UserAdd-command documented below it is also possible to import a csv with multiple customers, which has the fieldnames as the columnheader. The following command can be used to create a csv with all possible  (standard) headers first.

```powershell
$NCSession.CustomerValidation -join "," | set-content .\CustomerTemplate.csv
```

You can also create the customer without attributes and fill them out later if you wish by simply calling 

```powershell
$NCSession.CustomerAdd("NewCustomerName",$ParentId)`
```

The CustomerAdd function will return the value for the new Customer ID, you can then use that Id to perform further automation if needed.

### Add a new User

Not every cmdlet is currently available in PS-NCentral, one such cmdlet that could be useful is the automation of the creation of user accounts.

At time of writing with PS-NCentral version 1.6 it is possible to use the CustomerAdd() method as it is exists inside the core class object now. While there is currently no Powershell function to call this, create a user with it in the following way:

```powershell
#Connect to NC
$NCSession = New-NCentralConnection -ServerFQDN n-central.myserver.com -JWT $JWT
$NewUserAttributes = @{
    email = "john.doe@contoso.com"
    customerID = 50
    password = "ComplexityCompliant"
    firstname = "John"
    lastname = "Doe"
}

$NCSession.UserAdd($NewUserAttributes)
```

Above are the required attributes only. To see the additional attributes you can type

```powershell
$NCSession.UserValidation
```

It is also possible to import a csv with multiple accounts, which has the fieldnames as the columnheader. The command above can be used to create a csv with all possible headers first.

```powershell
$_ncsession.UserValidation -join "," | set-content .\UserTemplate.csv
```

After modifying the CSV and saving it as AccountsList.csv you can import 

```powershell
#Connect to NC
$NCSession = New-NCentralConnection -ServerFQDN n-central.myserver.com -JWT $JWT
#Import list of accounts
Import-CSV C:\Temp\AccountsList.csv | $NCSession.UserAdd($_)
```



The UserAdd function will return the value for the new User ID, you can then use that Id to perform further automation if needed.

It will return **-1** when the addition fails. Make sure the email is unique across the system (even if deleted) and the password meets the complexity requirements.

**Note:** It is not possible to change or add information for the user-account by API after creation.



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

# Appendix B - PS-NCentral cmdlets

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
<br>

# Appendix C – GetAllCustomerProperties.ps1

```powershell
# Define the command-line parameters to be used by the script
[CmdletBinding()]
Param(
    [Parameter(Mandatory = $true)]$serverHost,
    [Parameter(Mandatory = $true)]$JWT
)
# Generate a pseudo-unique namespace to use with the New-WebServiceProxy
$NWSNameSpace = NAble + ([guid]::NewGuid()).ToString().Substring(25)

# Bind to the namespace, using the Webserviceproxy
$bindingURL = "https://" + $serverHost + "/dms2/services2/ServerEI2?wsdl"
$nws = New-Webserviceproxy $bindingURL -Namespace ($NWSNameSpace)

# Set up and execute the query
$Settings = New-Object System.Collections.ArrayList
$Settings.Add(@{key = "listSOs"; value = "True" })

#Attempt to connect
Try {
    $CustomerList = $nws.customerList("", $JWT, $Settings)
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

These are the default properties returned in the customerlist. After connecting using PS-NCentral this list can also be found in the customervalidation property. 

```powershell
$_ncsession.customervalidation
```

- **postalcode** - (Value) Customer's zip/ postal code.
- **street1** - (Value) Address line 1 for the customer. Maximum of 100 characters.
- **street2** - (Value) Address line 2 for the customer. Maximum of 100 characters.
- **city** - (Value) Customer's city.
- **stateprov** - (Value) Customer's state/ province.
- **phone** - (Value) Phone number of the customer.
- **country** - (Value) Customer's country. Two character country code, see http://en.wikipedia.org/wiki/ISO\_3166-1\_alpha-2 for a list of country codes.
- **externalid** - (Value) An external reference id.
- **externalid2** - (Value) A second external reference id.
- **contactfirstname** - (Value) Customer contact's first name.
- **contactlastname** - (Value) Customer contact's last name.
- **contacttitle** - (Value) Customer contact's title.
- **contactdepartment** - (Value) Customer contact's department.
- **contactphonenumber** - (Value) Customer contact's telephone number.
- **contactext** - (Value) Customer contact's telephone extension.
- **contactemail** - (Value) Customer contact's email. Maximum of 100 characters.
- **registrationtoken** - (ReadOnly) For agent/probe-install validation.
- **licensetype** - (Value) The default license type of new devices for the customer. Must be Professional or Essential. Default is Essential.

# Appendix E - All PS-Central Methods
| Name |
| --- |
|AccessGroupGet|
|AccessGroupList|
|ActiveIssuesList|
|Connect|
|CustomerAdd|
|CustomerList|
|CustomerListChildren|
|CustomerModify|
|DeviceAssetInfoExportDevice|
|DeviceAssetInfoExportDeviceWithSettings|
|DeviceGet|
|DeviceGetAppliance|
|DeviceGetStatus|
|DeviceList|
|DevicePropertyID|
|DevicePropertyList|
|DevicePropertyModify|
|ErrorHandler|
|GetHashCode|
|JobStatusList|
|OrganizationPropertyID|
|OrganizationPropertyList|
|OrganizationPropertyModify|
|ProcessData1|
|ProcessData2|
|UserAdd|
|UserRoleGet|
|UserRoleList|

# Appendix F - Common Error Codes

\#    Connection-error (https): There was an error downloading ..

\#    1012 - Thrown when mandatory settings are not present in "settings".

\#    2001 - Required parameter is null - Thrown when null values are entered as inputs.

\#    2001 - Unsupported version - Thrown when a version not specified above is entered as input.

\#    2001 - Thrown when a bad username-password combination is input, or no PSA integration has been set up.

\#    2100 - Thrown when invalid MSP N-central credentials are input.

\#    2100 - Thrown when MSP-N-central credentials with MFA are used.

\#    3010 - Maximum number of users reached.

\#    3012 - Specified email address is already assigned to another user.

\#    3014 - Creation of a user for the root customer (CustomerID 1) is not permitted.

\#    3014 - When adding a user, must not be an LDAP user.

\#    3020 - Account is locked

\#    3022 - Customer/Site already exists.

\#    3026 - Customer name length has exceeded 120 characters.

\#    4000 - SessionID not found or has expired.

\#    5000 - An unexpected exception occurred.

\#    5000 - Query failed.

\#    5000 - javax.validation.ValidationException: Unable to validate UI session	--> often an expired password, even when using JWT.

\#    9910 - Service Organization already exists.



# Appendix G - Issue Status

These codes can be used with the **-IssueStatus** option of the **Get-NCActiveIssuesList** command. In v1.3 code 1 and 5 naming was incorrect (swapped)

```
1	No Data
2	Stale
3	Normal        --> Nothing returned
4	Warning
5	Failed
6	Misconfigured
7	Disconnected

11	Unacknowledged
12	Acknowledged
```

The API does not allow combinations of these filters.

- **1-7** are reflected in the **notifstate**-property.
- **11** and **12** relate to the properties **numberofactivenotification** and **numberofacknowledgednotification**.



# Appendix H - WebserviceProxy Alternative

The command **WebServiceProxy** is **discontinued** by Microsoft in PowerShell versions after 5.1. Below you can find an example of retrieving data with **Invoke-RestMethod**  and compare it to the use of WebserviceProxy. 

```PowerShell
## Retrieving data from N-Central by SOAP-API

# Settings
$Servername = "ServerFQDN"
$UserName = "API-User-noMFA"    ## Or "" when using JWT
$Password = "P@ssword"          ## Or JWT (preferred)

# Function for retrieving data without Webserviceproxy.
# Works for most (but not all!!) Methods in 
#      https://mothership.n-able.com/dms/javadoc_ei2/com/nable/nobj/ei2/ServerEI2_PortType.html
# Note: Complex passwords may contain characters that invalidate the SOAP-request.
Function GetNCData([String]$APIMethod,[String]$Username,[String]$PassOrJWT,$KeyPairs){

    ## Process Keys
    $MyKeys=""
    ForEach($KeyPair in $KeyPairs){ 
        $MyKeys = $MyKeys + ("
        <ei2:settings>
            <ei2:key>{0}</ei2:key>
            <ei2:value>{1}</ei2:value>
        </ei2:settings>" -f ($KeyPair.Key),($KeyPair.Value))
    }
    
    ## Build SoapRequest (last line must be left-lined for terminating @")
    $MySoapRequest =(@"
    <soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope" xmlns:ei2="http://ei2.nobj.nable.com/">
    <soap:Header/>
    <soap:Body>
        <ei2:{0}>
            <ei2:username>{1}</ei2:username>
            <ei2:password>{2}</ei2:password>{3}
        </ei2:{0}>
    </soap:Body>
    </soap:Envelope>
"@ -f $APIMethod, $Username, $PassOrJWT, $MyKeys)
    #Write-Host $MySoapRequest       ## For Debug/Educational purpose
    
    ## Request DataSet
    $FullResponse = $null
    Try{
            $FullResponse = Invoke-RestMethod -Uri $BindingURL -body $MySoapRequest -Method POST
        }
    Catch{
            Write-Host ("Could not connect: {0}." -f $_.Exception.Message )
            exit
        }
        
    ## Process Returned DataSet
    $ReturnClass = $FullResponse.envelope.body | Get-Member -MemberType Property
    $ReturnProperty = $ReturnClass[0].Name
            
    Return  $FullResponse.envelope.body.$ReturnProperty.return
}
# End of Function GetNCData

#Start of Main Script
# Extract local configuration Info.
$ApplianceConfig = ("{0}\N-able Technologies\Windows Agent\config\ApplianceConfig.xml" -f ${Env:ProgramFiles(x86)})
# Get appliance id
$ApplianceXML = [xml](Get-Content -Path $ApplianceConfig)
$ApplianceID = $ApplianceXML.ApplianceConfig.ApplianceID

# Prepare Connection
$BindingURL = ("https://{0}/dms2/services2/ServerEI2?wsdl" -f $ServerName)

## Add one or more keypairs to the array, depending on method parameter needs.
$KeyPairs = @()
$KeyPair = [PSObject]@{Key='applianceID'; Value=$ApplianceID;}
$KeyPairs += $KeyPair

## Pre Powershell 7 : Method using WebserviceProxy
#$Connection = New-Webserviceproxy $BindingURL

# Get data from N-Central. Old and New line for comparison and easy conversion of existing scripts.
#$rc = $Connection.deviceGet($UserName, $Password, $KeyPairs)
$rc = GetNCData 'deviceGet' $UserName $Password $KeyPairs

# Extract the Customer-ID and display.
$CustomerID = ($rc.info | where-object {$_.key -eq "device.customerid"}).value
$CustomerID
```





# Credits
Special Thanks go to the following Partners and Community Members for their contributions to the **NC-API-Documentation**
*   David Brooks of Premier Technology Solutions
*   Adriaan Sluis of Tosch for PS-NCentral and notes
*   Joshua Bennet of Impact Networking for notes on EiKeyValue usage
