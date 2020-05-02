---
title: "How to access/upload to Windows Azure Files in PowerShell"
date: 2019-09-08T16:47:11+08:00
draft: false
---
### Install Az Module for PwShell
```powershell
Install-Module -Name Az -AllowClobber -Scope AllUsers
```
### Import Az.Storage
```powershell
PS C:\Users\admin> Import-Module Az.Storage
PS C:\Users\admin> Get-Module

ModuleType Version    Name                                ExportedCommands
---------- -------    ----                                ----------------
Script     1.6.2      Az.Accounts                         {Add-AzEnvironment, Clear-AzContext, Clear-AzDefault, Conn...
Script     1.6.0      Az.Storage                          {Add-AzRmStorageContainerLegalHold, Add-AzStorageAccountMa...
...
```
### Login to Azure account
```powershell
PS C:\Users\admin> Connect-AzAccount

Account                     SubscriptionName         TenantId                             Environment
-------                     ----------------         --------                             -----------
AndrewBlue_1988@hotmail.com Visual Studio Enterprise 3a58e326-6e27-4b00-bf8c-13c1711f6a2a AzureCloud
```
### Initialize storage access
1. Get storage access key
2. Create storage context

```powershell
$key=PS C:\Users\admin> $key=Get-AzStorageAccountKey -ResourceGroupName Default -Name vtkmnck123
PS C:\Users\admin> $key.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     Object[]                                 System.Array
PS C:\Users\admin> $key[0].GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     False    StorageAccountKey                        System.Object
```
----------
```powershell
$context=New-AzStorageContext -StorageAccountName vtkmnck123 -StorageAccountKey $key[0].Value
```
### Azure Files share operation
```powershell
PS C:\Users\admin> $context |Get-AzStorageShare


   File End Point: https://vtkmnck123.file.core.windows.net/

Name                                                            LastModified       IsSnapshot         SnapshotTime
----                                                            ------------       ----------         ------------
vmtkndi348c20                                                   6/18/2019 2:19:... False
PS C:\Users\admin> $context | Remove-AzStorageFile -ShareName vmtkndi348c20 ubuntu-18.04.3-live-server-amd64.iso
PS C:\Users\admin> $context | Get-AzStorageFile -ShareName vmtkndi348c20


   Directory: https://vtkmnck123.file.core.windows.net/vmtkndi348c20

Type                Length Name
----                ------ ----
                         1 data
                         1 LogFiles
                         1 site

PS C:\Users\admin> $context | Set-AzStorageFileContent -ShareName vmtkndi348c20 -Source "C:\Users\admin\Downloads\example_pcap.pcap"
PS C:\Users\admin> $context | Get-AzStorageFile -ShareName vmtkndi348c20


   Directory: https://vtkmnck123.file.core.windows.net/vmtkndi348c20

Type                Length Name
----                ------ ----
                         1 data
                         1 example_pcap.pcap
                         1 LogFiles
                         1 site
PS C:\Users\admin> $context | Get-AzStorageFile -ShareName vmtkndi348c20 -Path config/ | Get-AzStorageFile


   Directory: https://vtkmnck123.file.core.windows.net/vmtkndi348c20/config

Type                Length Name
----                ------ ----
                         1 elasticsearch.yml
```
