## Appendix 1

# Building and Running the Sample Code (RI)

This appendix describes how to obtain, build and run the RI.

# Prerequisites

Before you begin, you should install the following pre-requisites:
* Visual Studio 2010 or later
* IIS
* SQL Server 2008 Express or later
* ASP.NET MVC 3 (Visual Studio 2010)
* ASP.NET NVC 4 Installer (Visual Studio 2010)

You can download and install all of these except for Visual Studio and IIS by using the Web Platform Installer 4.0.

If you plan to deploy the RI to Windows Azure or use the local Windows Azure compute and storage emulators, you should also install the Windows Azure SDK for .NET. This can also be downloaded and installed by using the Web Platform Installer 4.0.

If you plan to deploy the RI to Windows Azure, you must have a Windows Azure subscription, a SQL Azure subscription, and a Windows Azure Service Bus subscription. 

At the time of writing, you can sign-up for a Windows Azure free trial that enables you to run the RI in Windows Azure.

**Note:** You can run the RI locally without using the Windows Azure compute and storage emulators.

# Obtaining the Code

The source code is hosted in GitHub at [https://github.com/mspnp/cqrs-journey-code][source].
On this page, by clicking on the **Zip** button, you can download a zip file that contains the complete repository.
After you have downloaded the code, you should un-zip it to a suitable location on your hard drive.

# Creating the Databases

# Adding Windows Azure Configuration Settings

# Building the RI

Open the **Conference** Visual Studio solution file in the code repository that you downloaded and un-zipped.

If you did _not_ install the Windows Azure SDK, the **Conference.Azure** project will not be available. This is to be expected and will not prevent you from building and running the solution on your local machine.

## Build Configurations

The solution includes a number of build configurations. These are described in the following sections:

### Release

Use the **Release** build configuration if you plan to deploy your application to Windows Azure.

This solution uses the Windows Azure Service Bus to provide the messaging infrastructure.

### Debug

Use the **Debug** build configuration if you plan either to deploy your application locally to the Windows Azure compute and storage emulators or run the application locally and stand-alone in IIS without using the Windows Azure compute and storage emulators.

This solution uses the Windows Azure Service Bus to provide the messaging infrastructure.

### DebugLocal

Use the **DebugLocal** build configuration if you plan either to deploy your application locally to the Windows Azure compute and storage emulators or run the application locally and stand-alone in IIS without using the Windows Azure compute and storage emulators.

This solution uses a local messaging infrastructure built on SQL Server.

# Running the RI Locally

# Deploying the RI to Windows Azure


[source]: https://github.com/mspnp/cqrs-journey-code