## Microsoft patterns & practices
# CQRS Journey sample application

http://cqrsjourney.github.com

## Appendix 1

22nd May 2012

These release notes apply to the Pseudo-Production Release (V2) of the 
Contoso Conference Management System.

# Building and Running the Sample Code (RI)

This appendix describes how to obtain, build, and run the RI.

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

> **Note 2:** Scenarios 1, 2, 3 and 4 use SQL Express for other data
> storage requirements. Scenario 5 requires you to use SQL Database instead
> of SQL Express.

# Prerequisites

Before you begin, you should install the following pre-requisites:

* Visual Studio 2010 or later
* SQL Server 2008 Express or later
* ASP.NET MVC 4 Installer (Visual Studio 2010)
* Windows Azure SDK for .NET - June 2012 or later

> **Note:** Currently the RI requires the Windows Azure runtime
> libraries in order to compile. This is true even for scenario 1. The
> Windows Azure SDK includes these libraries. 

> **Note:** The V1 and V2 releases of the sample application used
> ASP.NET MVC 3 in addition to ASP.NET MVC 4. As of the V3 release all
> of the web applications in the project use ASP.NET MVC 4.

You can download and install all of these except for Visual Studio by
using the Microsoft Web Platform Installer 4.0. 

You can install the remaining dependencies from NuGet by running the
script **install-packages.ps1** included with the downloadable source.

If you plan to deploy the RI to Windows Azure, you must have a Windows 
Azure subscription. You will need to configure a Windows Azure storage 
account, a Windows Azure Service Bus namespace, and a SQL Database
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

For scenarios 1, 2, 3 and 4 you can create a local SQL Express database 
called **Conference** by running the script **Install-Database.ps1** in 
the scripts folder. 

The projects in the solution use this database to store application 
data. The SQL-based message bus and event store also use this database. 

## SQL Database Database

For scenario 5, you must create a SQL Database database called
**Conference** by running the script **Install-Database.ps1** in 
the scripts folder.

The follow command will populate a SQL Database database called 
**Conference** with the tables and views required to support the RI
(this script assumes that you have already created the **Conference**
database in SQL Database): 

```
.\Install-Database.ps1 -ServerName [your-sql-azure-server].database.windows.net -DoNotCreateDatabase -DoNotAddNetworkServiceUser -UseSqlServerAuthentication -UserName [your-sql-azure-username]
```

You must then modify the **ServiceConfiguration.Cloud.cscfg** file in the Conference.Azure project to use the following connection strings.

**SQL Database Connection String**

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
      <Setting name="Diagnostics.ScheduledTransferPeriod" value="00:02:00" />
      <Setting name="Diagnostics.LogLevelFilter" value="Warning" />
      <Setting name="Diagnostics.PerformanceCounterSampleRate" value="00:00:30" />
      <Setting name="DbContext.ConferenceManagement" value="[your-sql-azure-connection-string]" />
      <Setting name="DbContext.SqlBus" value="[your-sql-azure-connection-string] />
    </ConfigurationSettings>
  </Role>
  <Role name="Conference.Web.Public">
    <Instances count="1" />
    <ConfigurationSettings>
      <Setting name="Microsoft.WindowsAzure.Plugins.Diagnostics.ConnectionString" value="[your-windows-azure-connection-string]" />
      <Setting name="Diagnostics.ScheduledTransferPeriod" value="00:02:00" />
      <Setting name="Diagnostics.LogLevelFilter" value="Warning" />
      <Setting name="Diagnostics.PerformanceCounterSampleRate" value="00:00:30" />
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
      <Setting name="Diagnostics.ScheduledTransferPeriod" value="00:02:00" />
      <Setting name="Diagnostics.LogLevelFilter" value="Information" />
      <Setting name="Diagnostics.PerformanceCounterSampleRate" value="00:00:30" />
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

> **Note:** The **LogLevelFilter** values for these roles is set to
> either **Warning** or **Information**. If you want to capture logs
> from the application into the **WADLogsTable**, you should change
> these values to **Verbose**.

# Creating the Settings.xml File

Before you can build the solution, you must create a **Settings.xml** 
file in the **Infrastructure Projects\Azure** solution folder. You can 
copy the **Settings.Template.xml** in this solution folder to create a 
**Settings.xml** file. 

> **Note:** You only need to create the **Settings.xml** file if you
> plan to use either the **Debug** or **Release** build configurations.

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

> **Note:** In the V2 release, there is an additional entry in the 
> **Settings.Template.xml** file to specify the Windows Azure table
> storage table name for the message log used to store all event and
> command messages.

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
application locally to the Windows Azure compute emulator or to run the 
application locally and stand-alone without using the Windows Azure 
compute emulator. 

This solution uses the Windows Azure Service Bus to provide the 
messaging infrastructure and the event store based on Windows Azure 
table storage (scenarios 2 and 4). 

### DebugLocal

Use the **DebugLocal** build configuration if you plan to either deploy 
your application locally to the Windows Azure compute emulator or run 
the application on a local web server without using the Windows 
Azure compute emulator. 

This solution uses a local messaging infrastructure and event store 
built using SQL Server (scenarios 1 and 3). 

# Running the RI

When you run the RI, you should first create a conference, add at least
one seat type, and then publish the conference using the 
**Conference.Web.Admin** site.

After you have published the conference, you will then be able to use 
the site to order seats and use the simulated the payment process using 
the **Conference.Web** site. 

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
> database in SQL Database using the **Install-Database.ps1** in the
> scripts folder as described above. You must also ensure that you have 
> modified the connection strings in the
> configuration files in the solution to point to your SQL Database
> **Conference** database instead of your local SQL Express
> **Conference** database as described above.

# Running the Tests

The following sections describe how to run the unit, integration, and 
acceptance tests. 

## Running the Unit and Integration Tests

The unit and integration tests in the **Conference** solution are 
created using **xUnit.net**. 

For more information about how you can run these tests, please visit the 
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
configurations as the **Conference** solution to control whether you run 
the acceptance tests against either the local SQL-based messaging 
infrastructure and event store or the Windows Azure Service Bus 
messaging infrastructure and Windows Azure table storage based event 
store. 

# Migrating from the V1 to the V2 Release

If you have been running the V1 release and have data that you would 
like to preserve as you migrate to the V2 release, the following steps 
describe how you can perform this migration if you are hosting the V1
release in Windows Azure. 

1. Deploy the V2 release to your Windows Azure staging environment. The
   V2 release has a global **MaintenanceMode** property that is
   initially set to **true**. In this mode, the application displays a
   message to the user that site is currently undergoing maintenance.
2. When you are ready, swap the V2 release (still in maintenance mode)
   into your Windows Azure production environment.
3. Leave the V1 release (now running in the staging environment) to run
   for a few minutes to ensure that all in-flight messages complete
   their processing.
4. Run the migration program to migrate the data (see below).
5. After the data migration completes successfully, change the
   **MaintenanceMode** property to **false**.
6. The V2 release is now live in Windows Azure.

> **Note:** You can change the value of the **MaintenanceMode** property
> in the Windows Azure management console.

## Running the Migration Program to Migrate the Data

_Before beginning the data migration process, ensure that you have a 
backup of the data from your SQL Database database._ 

The **MigrationToV2** utility uses the same **Settings.xml** file as the 
other projects in the **Conference** solution in addition to its own 
**App.config** file to specify the Windows Azure storage account 
and SQL connection strings.

The **Settings.xml** file contains the names of the new Windows Azure 
tables that the V2 release uses. If you are migrating data from V1 to V2 
ensure that the name of the **EventSourcing** table is different to the 
name of the table used by the V1 release. The name of the table used by 
the V1 release is hardcoded in the **Program.cs** file in the MigrationToV2 
project: 

```
var originalEventStoreName = "ConferenceEventStore";
```

The name of the new table for V2 is in the **Settings.xml** file:

```
<EventSourcing>
	<ConnectionString>...</ConnectionString>
	<TableName>ConferenceEventStoreApplicationDemoV2</TableName>
</EventSourcing>
```

> **Note:** The migration utility assumes that the V2 event sourcing
> table is in the same Windows Azure storage account as the V1 event
> sourcing table. If this is not the case, you will need to modify the
> MigrationToV2 application code.

The **App.config** file contains the **DbContext.ConferenceManagement** 
connection string. The migration utility uses this connection string to 
connect to the SQL Database database that contains the SQL tables used by 
the application. Ensure that this connection string points to the SQL 
Azure database that contains your production data. You can verify which 
SQL Database database your production environment uses by looking in the 
active **ServiceConfiguration.csfg** file. 

> **Note:** If you are running the application locally using the
> **Debug** configuration, the **DbContext.ConferenceManagement**
> connection string will point to local SQL Express database.

> **Note:** To avoid data transfer charges, you should run the migration
> utility inside a Windows Azure worker role instead of on-premise. The
> solution includes an empty Windows Azure worker role in the
> **MigrationToV2.Azure** with diagnostics already configured that you
> can use for this purpose. For information about how to run an
> application inside a Windows Azure role instance, see [Using Remote
> Desktop with Windows Azure Roles][azurerdp]. 

> **Note:** Migration from V1 to V2 is not supported if you are using
> the **DebugLocal** configuration.

### If the Data Migration Fails

If the data migration process fails for any reason, then before you retry the migration you should:

1. Restore the SQL database back to its state before you ran the
   migration utility.
2. Delete the two new Windows Azure tables defined in **Settings.xml**
   in the **EventSourcing** and **MessageLog** sections.


# Known Issues

## Localizability is not in scope

The RI is not designed with localizability in mind. For example, it 
currently contains hardcoded strings, fixed number formats, etc. 

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

## Switching between Debug and DebugLocal Builds

If you run the application after building using the Debug configuration, 
create some data, and then re-build using the DebugLocal configuration 
you will see errors when you run the application. This scenario is not 
supported. 

The problem arises because the two build configurations only share some 
data sources, so after the switch there are inconsistencies in the data. 
You should re-create all the data sources if you switch from one build 
configuration to another.

## Other Known Issues

* No security features have been implemented yet. 
* No performance or localizability tests have been performed yet.
* The UI is still a work in progress.
* Validation in the UI is not yet complete. 
* You see a list of outstanding issues for the V2 release [here][v2outstanding].


[source]:          https://github.com/mspnp/cqrs-journey-code
[xunit]:           http://xunit.codeplex.com/
[specflow]:        http://www.specflow.org/
[connectionerror]: http://blogs.msdn.com/b/narahari/archive/2011/12/21/azure-a-connection-attempt-failed-because-the-connected-party-did-not-properly-respond-after-a-period-of-time-or-established-connection-failed-because-connected-host-has-failed-to-respond-x-x-x-x-x-quot.aspx
[v2outstanding]:   https://github.com/mspnp/cqrs-journey-code/issues/search?utf8=%E2%9C%93&q=v2
[azurerdp]:        http://msdn.microsoft.com/en-us/library/windowsazure/gg443832.aspx