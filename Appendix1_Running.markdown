## Appendix 1

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
Azure subscription, a SQL Azure subscription, and a Windows Azure 
Service Bus subscription. You should be aware, that depending on your 
Windows Azure subscription type, you may incur usage charges when you 
use the Windows Azure Service Bus, Windows Azure table storage, and when 
you deploy and run the RI in Windows Azure. 

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

For scenarios 1,2,3 and 4 you can create a local SQL Express database 
called **Conference** by running the script **Install-Database.ps1** in 
the scripts folder. 

For scenario 5, you must create a SQL Azure database called
**Conference** by running the script **Install-Database.ps1** in 
the scripts folder.

The follow command will populate a SQL Azure database called 
**Conference** with the tables and views required to support the RI: 

```
.\Install-Database.ps1 -ServerName [your-sql-azure-server].database.windows.net -DoNotCreateDatabase -DoNotAddNetworkServiceUser -UseSqlServerAuthentication -UserName [your-sql-azure-username]
```

The projects in the solution use this database to store application 
data. The SQL-based message bus and event store also use this database. 

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
> database in SQL Azure using the **Install-Database.ps1** in the scripts
> folder. You must also modify the connection strings in all
> configuration files in the solution to point to your SQL Azure
> **Conference** database instead of your local SQL Express
> **Conference** database.

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

[source]:   https://github.com/mspnp/cqrs-journey-code
[xunit]:    http://xunit.codeplex.com/
[specflow]: http://www.specflow.org/