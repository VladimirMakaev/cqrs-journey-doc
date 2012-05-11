## Microsoft patterns & practices
# CQRS Journey sample application

http://cqrsjourney.github.com

## Appendix 1

4th May 2012

These release notes apply to the Pseudo-Production Release (V1) of the 
Contoso Conference Management System.

# Building and Running the Sample Code (RI)

This appendix describes how to obtain, build and run the RI.

These instructions describe five different scenarios for running the RI:

1. Running the application on a local web server and using a local
   message bus and event store.
2. Running the application on a local web server and using the Windows
   Azure Service Bus and an event store that uses Windows Azure table
   storage.
3. Deploying the application to the local Windows Azure compute emulator
   and using a local message bus and event store.
4. Deploying the application to the local Windows Azure compute emulator
   and using the Windows Azure Service Bus and an event store that uses
   Windows Azure table storage.
5. Deploying the application to Windows Azure and using the Windows
   Azure Service Bus and an event store that uses Windows Azure table
   storage.

> **Note 1:** The local message bus and event store use SQL Express and
> are intended to help you run the application locally for demonstration
> purposes. They are not intended to illustrate a production-ready
> scenario.

> **Note 2:** Scenarios 1,2,3 and 4 use SQL Express for other data
> storage requirements. Scenario 5 requires you to use SQL Azure instead
> of SQL Express.

# Prerequisites

Before you begin, you should install the following pre-requisites:

* Visual Studio 2010 or later
* SQL Server 2008 Express or later
* ASP.NET MVC 3 (Visual Studio 2010)
* ASP.NET MVC 4 Installer (Visual Studio 2010)
* Windows Azure SDK for .NET - November 2011 or later.

> **Note:** Currently the RI requires the Windows Azure runtime
> libraries in order to compile. This is true even for scenario 1. The
> Windows Azure SDK includes these libraries. 

You can download and install all of these except for Visual Studio by
using the Web Platform Installer 4.0. 

You can install the remaining dependencies from NuGet by running the
script **install-packages.ps1** included with the downloadable source.

If you plan to deploy the RI to Windows Azure, you must have a Windows 
Azure subscription. You will need to configure a Windows Azure storage 
account, a Windows Azure Service Bus namespace, and a SQL Azure
database (they do not necessarily need to be in the same Windows Azure 
subscription). You should be aware, that depending on your Windows Azure 
subscription type, you may incur usage charges when you use the Windows 
Azure Service Bus, Windows Azure table storage, and when you deploy and 
run the RI in Windows Azure. 

At the time of writing, you can sign-up for a Windows Azure free trial 
that enables you to run the RI in Windows Azure. 

> **Note:** Scenario 1 enables you to run the RI locally without using
> the Windows Azure compute and storage emulators. 

If you want to run the SpecFlow acceptance tests, you should also 
install [SpecFlow][specflow]. 


# Obtaining the Code

* The source code is hosted in GitHub at
  [https://github.com/mspnp/cqrs-journey-code][source].
* On this page, by clicking on the **Zip** button, you can download a
  zip file that contains the complete repository.
* After you have downloaded the code, you should un-zip it to a suitable
  location on your hard drive.

# Creating the Databases

## SQL Express Database

For scenarios 1,2,3 and 4 you can create a local SQL Express database 
called **Conference** by running the script **Install-Database.ps1** in 
the scripts folder. 

The projects in the solution use this database to store application 
data. The SQL-based message bus and event store also use this database. 

## SQL Azure Database

For scenario 5, you must create a SQL Azure database called
**Conference** by running the script **Install-Database.ps1** in 
the scripts folder.

The follow command will populate a SQL Azure database called 
**Conference** with the tables and views required to support the RI
(this script assumes that you have already created the **Conference**
database in SQL Azure): 

```
.\Install-Database.ps1 -ServerName [your-sql-azure-server].database.windows.net -DoNotCreateDatabase -DoNotAddNetworkServiceUser -UseSqlServerAuthentication -UserName [your-sql-azure-username]
```

You must then modify the ServiceConfiguration.Cloud.cscfg file in the Conference.Azure project to use the following connection strings.

**SQL Azure Connection String**

```
Server=tcp:[your-sql-azure-server].database.windows.net;Database=myDataBase;User ID=[your-sql-azure-username]@[your-sql-azure-server];Password=[your-sql-azure-password];Trusted_Connection=False;Encrypt=True; MultipleActiveResultSets=True;
```

**Windows Azure Connection String**

```
DefaultEndpointsProtocol=https;AccountName=[your-windows-azure-storage-account-name];AccountKey=[your-windows-azure-storage-account-key]
```

**Conference.Azure\ServiceConfiguration.Cloud.cscfg**

```
<?xml version="1.0" encoding="utf-8"?>
<ServiceConfiguration serviceName="Conference.Azure" xmlns="http://schemas.microsoft.com/ServiceHosting/2008/10/ServiceConfiguration" osFamily="1" osVersion="*">
  <Role name="Conference.Web.Admin">
    <Instances count="1" />
    <ConfigurationSettings>
      <Setting name="Microsoft.WindowsAzure.Plugins.Diagnostics.ConnectionString" value="[your-windows-azure-connection-string]" />
      <Setting name="DbContext.ConferenceManagement" value="[your-sql-azure-connection-string]" />
      <Setting name="DbContext.SqlBus" value="[your-sql-azure-connection-string] />
    </ConfigurationSettings>
  </Role>
  <Role name="Conference.Web.Public">
    <Instances count="1" />
    <ConfigurationSettings>
      <Setting name="Microsoft.WindowsAzure.Plugins.Diagnostics.ConnectionString" value="[your-windows-azure-connection-string]" />
      <Setting name="DbContext.Payments" value="[your-sql-azure-connection-string]" />
      <Setting name="DbContext.ConferenceRegistration" value="[your-sql-azure-connection-string]" />
      <Setting name="DbContext.SqlBus" value="[your-sql-azure-connection-string]" />
      <Setting name="DbContext.BlobStorage" value="[your-sql-azure-connection-string]" />
    </ConfigurationSettings>
  </Role>
  <Role name="WorkerRoleCommandProcessor">
    <Instances count="1" />
    <ConfigurationSettings>
      <Setting name="Microsoft.WindowsAzure.Plugins.Diagnostics.ConnectionString" value="[your-windows-azure-connection-string]" />
      <Setting name="DbContext.Payments" value="[your-sql-azure-connection-string]" />
      <Setting name="DbContext.EventStore" value="[your-sql-azure-connection-string]" />
      <Setting name="DbContext.ConferenceRegistrationProcesses" value="[your-sql-azure-connection-string]" />
      <Setting name="DbContext.ConferenceRegistration" value="[your-sql-azure-connection-string]" />
      <Setting name="DbContext.SqlBus" value="[your-sql-azure-connection-string]" />
      <Setting name="DbContext.BlobStorage" value="[your-sql-azure-connection-string]" />
      <Setting name="DbContext.ConferenceManagement" value="your-sql-azure-connection-string]" />
    </ConfigurationSettings>
  </Role>
</ServiceConfiguration>
```

# Creating the Settings.xml File

Before you can build the solution you must create a **Settings.xml** file 
in the **Infrastructure Projects\Azure** solution folder. You can copy the 
**Settings.Template.xml** in this solution folder to create a 
**Settings.xml** file. 

If you plan to use the Windows Azure Service Bus and the Windows Azure 
table storage based event store then you must edit the **Settings.xml** 
file in the **Infrastructure Projects\Azure** solution folder to include 
details of your Windows Azure storage account and a Windows Azure 
Service Bus namespace. 

> **Note:** See the contents of the **Settings.Template.xml** for details
> of the configuration information that is required.

> **Note:** You cannot currently use the Windows Azure storage emulator
> for the event store. You must use a real Windows Azure storage
> account.

# Building the RI

Open the **Conference** Visual Studio solution file in the code 
repository that you downloaded and un-zipped. 

You can use NuGet to download and install all of the dependencies by
running the script **install-packages.ps1** before building the
solution.

## Build Configurations

The solution includes a number of build configurations. These are 
described in the following sections: 

### Release

Use the **Release** build configuration if you plan to deploy your 
application to Windows Azure. 

This solution uses the Windows Azure Service Bus to provide the 
messaging infrastructure. 

Use this build configuration if you plan to deploy the RI to Windows 
Azure (scenario 5). 

### Debug

Use the **Debug** build configuration if you plan either to deploy your 
application locally to the Windows Azure compute emulator or run the 
application locally and stand-alone without using the Windows Azure 
compute emulator. 

This solution uses the Windows Azure Service Bus to provide the 
messaging infrastructure and the event store based on Windows Azure 
table storage (scenarios 2 and 4). 

### DebugLocal

Use the **DebugLocal** build configuration if you plan either to deploy 
your application locally to the Windows Azure compute emulator or run 
the application on a local web server without using the Windows 
Azure compute emulator. 

This solution uses a local messaging infrastructure and event store 
built using SQL Server (scenarios 1 and 3). 

# Running the RI

When you run the RI, you should first create a conference, add at least
one seat type, and then publish the conference using the 
**Conference.Web.Admin** site.

After you have published the conference you will then be able to use the
site to order seats and use the simulated the payment process using the 
**Conference.Web** site.

The following sections describe how to run the RI using in the different 
scenarios. 

## Scenario 1. Local Web Server, SQL Event Bus, SQL Event Store

To run this scenario you should build the application using the 
**DebugLocal** configuration. 

Run the **WorkerRoleCommandProcessor** project as a console application. 

Run the **Conference.Web.Public** and **Conference.Web.Admin** (located 
in the **ConferenceManagement** folder) as web applications. 

## Scenario 2. Local Web Server, Windows Azure Service Bus, Table Storage Event Store

To run this scenario you should build the application using the 
**Debug** configuration. 

Run the **WorkerRoleCommandProcessor** project as a console application. 

Run the **Conference.Web.Public** and **Conference.Web.Admin** (located 
in the **ConferenceManagement** folder) as web applications. 

## Scenario 3. Compute Emulator, SQL Event Bus, SQL Event Store

To run this scenario you should build the application using the 
**DebugLocal** configuration. 

Run the **Conference.Azure** Windows Azure project. 

> **Note:** To use the Windows Azure compute emulator you must launch
> Visual Studio as an administrator.

## Scenario 4. Compute Emulator, Windows Azure Service Bus, Table Storage Event Store

To run this scenario you should build the application using the 
**Debug** configuration. 

Run the **Conference.Azure** Windows Azure project. 

> **Note:** To use the Windows Azure compute emulator you must launch
> Visual Studio as an administrator.

## Scenario 5. Windows Azure, Windows Azure Service Bus, Table Storage Event Store 

Deploy the **Conference.Azure** Windows Azure project to your Windows 
Azure account. 

> **Note:** You must also ensure that you have created **Conference**
> database in SQL Azure using the **Install-Database.ps1** in the
> scripts folder as described above. You must also ensure that you have 
> modified the connection strings in the
> configuration files in the solution to point to your SQL Azure
> **Conference** database instead of your local SQL Express
> **Conference** database as described above.

# Running the Tests

The following sections describe how to run the unit, integration, and 
acceptance tests. 

## Running the Unit and Integration Tests

The unit and integration tests in the **Conference** solution are 
created using **xUnit.net**. 

For more information about how you can run these tests please visit the 
[xUnit.net][xunit] site on Codeplex. 

## Running the Acceptance Tests

The acceptance tests are located in the Visual Studio solution in the 
**Conference.AcceptanceTests** folder. 

You can use NuGet to download and install all of the dependencies by
running the script **install-packages.ps1** before building this
solution.

The acceptance tests are created using SpecFlow. For more information 
about SpecFlow, please visit [SpecFlow][specflow]. 

The SpecFlow tests are implemented using **xUnit.net**.

The **Conference.AcceptanceTests** solution uses the same build 
configurations as the **Conference** solution to control whether the 
acceptance tests are run against either the local SQL-based messaging 
infrastructure and event store or the Windows Azure Service Bus
messaging infrastructure and Windows Azure table storage based event
store. 

# Known Issues

## Runtime Activation Error in Debug Mode

When you run the application in debug mode you will see an error in the
**Conference.Web.Public** web application:

```
Activation error occured while trying to get instance of type IControllerFactory
```

Click in the **Continue** button and the application will run as
expected.

## Server Error in '/' Application

When you run the application locally and you are using a proxy server you see:

```
Server Error in '/' Application.
A connection attempt failed because the connected party did not properly respond after a period of time, or established connection failed because connected host has failed to respond ...

Description: An unhandled exception occurred during the execution of the current web request. Please review the stack trace for more information about the error and where it originated in the code.

Exception Details: System.Net.Sockets.SocketException: A connection attempt failed because the connected party did not properly respond after a period of time, or established connection failed because connected host has failed to respond ...
```

For help resolving this issue see [Azure: A connection attempt failed...][connectionerror]

## Missing Validation

Currently, there is no validation to:

* Check that the end date for a conference is later than the start date.
* Check that email addresses are in a valid format.

## Switching between Debug and DebugLocal Builds

If you run the application after building using the Debug configuration 
and create some data and then re-build using the DebugLocal 
configuration you will see errors when you run the application. This 
scenario is not supported. 

The problem arises because the two build configurations only share some 
data sources, so after the switch there are inconsistencies in the data. 
You should re-create all the data sources if you switch from one build 
configuration to another. 

## Other Known Issues

* No security features have been implemented yet. 
* No performance or localizability tests have been performed yet.
* The UI is still a work in progress.
* Validation in the UI is not yet complete. 
* You see a list of outstanding issues for the V1 release [here][v1outstanding].


[source]:          https://github.com/mspnp/cqrs-journey-code
[xunit]:           http://xunit.codeplex.com/
[specflow]:        http://www.specflow.org/
[connectionerror]: http://blogs.msdn.com/b/narahari/archive/2011/12/21/azure-a-connection-attempt-failed-because-the-connected-party-did-not-properly-respond-after-a-period-of-time-or-established-connection-failed-because-connected-host-has-failed-to-respond-x-x-x-x-x-quot.aspx
[v1outstanding]:   https://github.com/mspnp/cqrs-journey-code/issues/search?utf8=%E2%9C%93&q=v1