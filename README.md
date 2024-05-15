# QuickBooks Desktop Enterprise 22.0/23.0 Intune Management Guide
## Overview
I have been maintaining QuickBooks Desktop Enterprise for several years now and have put together this guide on how to deploy the base app and the webpatches to users using Intune.

- This works for versions **22.0** & **23.0**.
	- You may reference this guide in an attempt to manage versions prior to 22.0. 

Before you build your installation and patching packages you will want to make sure you have some necessary tools ready and that you are familiar with how to use them.

## Required Tools
### PSAppDeployToolkit (PSADT)
You will be using PSADT to do the bulk of the deployment work. You will want to get familiar with this tool if you are not already.

- The PSAppDeployToolkit is a framework for deploying applications in a business or corporate environment.
- It offers a set of functions for common application deployment tasks and user interface elements for end-user interaction during a deployment.
- The toolkit aims to simplify the complex scripting challenges of deploying applications and to provide a consistent deployment experience.
- Download PSAppDeployToolkit at [https://github.com/PSAppDeployToolkit/PSAppDeployToolkit/releases/](https://github.com/PSAppDeployToolkit/PSAppDeployToolkit/releases/).

### Microsoft Deployment Toolkit (MDT)
- The Microsoft Deployment Toolkit (MDT) is a free tool for automating Windows and Windows Server operating system deployment, leveraging the Windows Assessment and Deployment Kit (ADK) for Windows 10.
- Version 8456 was released on January 25th 2019 and is the latest current version.
- Download the MDT at [https://www.microsoft.com/en-us/download/details.aspx?id=54259](https://www.microsoft.com/en-us/download/details.aspx?id=54259)

For managing QuickBooks with Intune, we are after a specific tool within this toolkit. We will be using the **ServiceUI.exe** tool.

### Microsoft Win32 Content Prep Tool
You should already be familiar with this tool. If not, it is a simple tool to understand and use, so do not feel intimidated. This is what you will be using to package everything into a single **.intunewin** file that will be uploaded to Intune.

- Requires .NET Framework 4.7.2
- Download at [https://github.com/microsoft/Microsoft-Win32-Content-Prep-Tool](https://github.com/microsoft/Microsoft-Win32-Content-Prep-Tool)

## Building The Deployment Package
### Download QuickBooks Installer
We need to download a current installer for our QuickBooks application. Visit the QuickBooks download center at [https://downloads.quickbooks.com/app/qbdt/products](https://downloads.quickbooks.com/app/qbdt/products)

Select your version and download. This is also where you will download the QBwebpatch.exe patch installer that you need for installing updates.

![Image](https://scsim4ges.blob.core.windows.net/guides/2024-05-14_11_50_28.jpg)

### Prepare PSADT
If you haven't already, download the current version of PSADT, extract, and stage the files to repackage for Intune.

Place the **QuickBooksEnterprise2X.exe** into the **Files** folder of the PSADT structure.

![Image](https://scsim4ges.blob.core.windows.net/guides/2024-05-14_13_08_14.jpg)

### Modify Deploy-Application.ps1
You need to make some modificatons to the deploy script of PSADT. Enter the following commands under each corresponding section of the script.

#### Installation Tasks
Here you need to enter the following installation command for the **QuickBooksEnterprise23.exe** installer.

```
 ## <Perform Installation tasks here>
 Execute-Process -Path "$dirFiles\QuickBooksEnterprise22.exe" -Arguments "-s", "-a", "QBMIGRATOR=1", "MSICOMMAND=/s", "QB_PRODUCTNUM=XXXXXX", "QB_LICENSENUM=XXXXXXXXXXXXXX"
```

#### Post-Installation Tasks
Here we will enable the XPS document writer with the following command.

```
## <Perform Post-Installation tasks here>
Enable-WindowsOptionalFeature -FeatureName “Printing-XPSServices-Features” -Online -NoRestart
```

#### Uninstallation Tasks
Enter the following command to trigger the uninstall script.

```
## <Perform Uninstallation tasks here>
Execute-Process -Path "$dirFiles\uninstall.cmd"
```

#### Post-Uninstallation Tasks
Enter the following command to disable the XPS document writer.

```
## <Perform Post-Uninstallation tasks here>
Disable-WindowsOptionalFeature -FeatureName "Printing-XPSServices-Features" -Online -NoRestart
```


You may make any other modifications you choose if you are comfortable working with the deploy script. Depending on your environment, there may be other tasks you need to perform to get the install to complete successfully.

### Create Uninstall Script
You will need to navigate through the registry to get the uninstallation string that is used during the uninstall process for your version of QuickBooks. 

Install QuickBooks on a computer temporarily if you do not have it installed somewhere already.

The uninstall string for QuickBooks Desktop Enterprise can be found at:

`HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall`

The registry entry for it will have a product ID for its name. Look through them until you find the one for QuickBooks.

![Image](https://scsim4ges.blob.core.windows.net/guides/2024-05-14_13_22_44.jpg)

Copy the **UninstallString** entry value and paste it onto a notepad editor to build the script.

>Use the "/passive" parameter at the end of the uninstall string. This will present the user with a small progress bar during the uninstallation process, but requires no additional interaction.

Your complete uninstall command should look something like this below. Save it as a .cmd file to the **Files** fold of the PSADT structure.

```
msiexec.exe /x {B9BE758E-50B5-4BA7-987B-63184123AA1A} UNIQUE_NAME="belcontractor" QBFULLNAME="QuickBooks Enterprise Solutions: Contractor Edition 22.0" ADDREMOVE=1 /passive
```


### Add ServiceUI.exe
This package will be running the the SYSTEM context. In order for the uninstall process to work properly the **ServiceUI.exe** tool needs to be added to the root of your PSADT structure.

This tool is part of **MDT** and can be downloaded from the link provided at the top of this guide. 

After you download and install the MDT, you will find the **ServiceUI.exe** in `C:\Program Files\Microsoft Deployment Toolkit\Templates\Distribution\Tools\x64`

Copy and paste it into your PSADT pacakge where your **Deploy-Application.ps1** script is.

![Image](https://scsim4ges.blob.core.windows.net/guides/2024-05-14_16_59_01.jpg)

### Wrap with Win32 Content Prep Tool
Now you are ready to wrap it all up into the .intunewin file format so it can be uploaded to Intune.

Select the **Deploy-Application.exe** as the setup file when packaging with the tool. Depending on your computers hardware, it can take a few minutes for it to complete and the app to be ready to upload.

## Upload to Intune
### Configure Application
Navigate to the Intune portal to create a new Windows application and upload the freshly packaged app.

Enter details to name the application and give it a description. Included with this guide is a description markdown file that you can use if you wish. I have also included an app logo .png image file.

When configuring the app make sure to set install behavior to **System**.

![Image](https://scsim4ges.blob.core.windows.net/guides/2024-05-14_19_32_54.jpg)

Use the following Install and uninstall commands.

```
# Install Command
Deploy-Application.exe -DeploymentType "Install" -DeployMode "Silent"

# Uninstall command
.\ServiceUI.exe -Process:explorer.exe Deploy-Application.exe -DeploymentType "Uninstall" -DeployMode "Silent"
```

### Detection Method
The detection method can be done a number of ways. For ease and reliability I have only provided the basic detection of the .exe after installation/uninstallation.

- Rule Type: File
- Path: `C:\Program Files\Intuit\QuickBooks Enterprise Solutions 22.0\`
- File: `QBW.EXE`
- Detection method: `File or folder exists`

That's it. You're all set to assign the app and test installing it on a computer. 

Please note, depending on your bandwidth and network environment, it can take several minutes for the package to download and install. 

If you run into failures, make sure you have logging enabled for PSADT to make troubleshooting failures a breeze.

## QBWebpatch.exe
### Prepare Files

I came up with a simpe file creation command that places a "log" file on the computer after successfully running the update installer.

This can be used for future detections if you ever update the app with a newer QBwebpatch.exe. Simply set your detection to look for the log file and its last modified date.

```
@echo off
for /f "tokens=2 delims==" %%I in ('wmic os get localdatetime /format:list') do set datetime=%%I
set datetime=%datetime:~0,4%-%datetime:~4,2%-%datetime:~6,2% %datetime:~8,2%:%datetime:~10,2%:%datetime:~12,2%
echo Update installed %datetime% > "C:\Users\Public\qbupdate.log"

```


Intuit offers the **QBwebpatch.exe** installer that installs all updates for that version that are availalbe at time of installation. 

This .exe can be downloaded from the QuickBooks downloads page mentioned at the beginning of this guide.

You will package it the same way we did the parent app. Use the below commands for the **Deploy-Application.ps1** script and **Intune** install/uninstall tasks and commands.

```
# Install tasks
## <Perform Installation tasks here>
Execute-Process -Path "en_qbwebpatch.exe" -Parameter "/silent" -IgnoreExitCodes "1"



## <Perform Post-Installation tasks here>
Execute-Process -Path "log.cmd"

```


### Upload to Intune
Create another app for the update installer. Make sure to set it to System install.

Use the following install/uninstall commands and detection method.

```
# Install command
.\ServiceUI.exe -Process:explorer.exe Deploy-Application.exe -DeploymentType "Install" -DeployMode "Silent"


# Uninstall Command
Deploy-Application.exe -DeploymentType "Uninstall" -DeployMode "Silent"

```


**File Detection Method**

- Path: `C:\Users\Public`
- File: `qbupdate.log`
- Detection Method: `Date modified`
- Operator: `Greater than or equal to`

You're ready to test. The update installer has a "silent" mode, but it is not truly silent. Running it with the `/silent` parameter will hide the first screen, but the user still needs to click on a prompt to install the update when it's ready.
