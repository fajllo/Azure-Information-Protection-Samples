# Migrate analytics from Azure Information Protection to Microsoft Purview Information Protection 

Azure Information Protection analytics, a pipeline that brought data for Azure Information Protection into log analytics, was retired on September 30, 2022. In order to modernize the Azure Information Protection stack, logs for Azure Information Protection were added into the Office 365 Unified Audit Log and can be accessed through Microsoft Purview Activity Explorer, unified audit log search tool, Search-UnifiedAuditLog powershell commandlet, and Office 365 Management Activity API.

Azure Information Protection events within the Unified Audit Log:
- [AipDiscover](https://learn.microsoft.com/office/office-365-management-api/aipdiscover)
- [AipSensitivityLabelAction](https://learn.microsoft.com/office/office-365-management-api/aipsensitivitylabelaction)
- [AipProtectionAction](https://learn.microsoft.com/office/office-365-management-api/aipprotectionaction)
- [AipFileDeleted](https://learn.microsoft.com/office/office-365-management-api/aipfiledeleted)
- [AipHeartBeat](https://learn.microsoft.com/office/office-365-management-api/aipheartbeat)

In addition, Azure Information Protection events within the Unified Audit Log can be sent to log analytics using the Microsoft Purview Information Protection connector, and this guide walks through the steps to do that.
1) [Enable the Microsoft Purview Information Protection connector in Sentinel.](#enable-the-microsoft-purview-information-protection-connector-in-sentinel)
2) [Migrate queries to use the Microsoft Purview Information Protection connector.](#migrate-queries-to-use-the-microsoft-purview-information-protection-connector)
3) [Get label names with Microsoft Purview Information Protection Logs.](#get-label-names-with-microsoft-purview-information-protection-logs) 

## Enable the Microsoft Purview Information Protection connector in Sentinel
The Microsoft Purview Information Protection connector was introduced into Sentinel on January 9, 2023. The Microsoft Purview Information Protection connector streams data to a log analytics table (MicrosoftPurviewInformationProtection) and contains events related to Azure Information Protection. These events are similiar to what used to show up within the Azure Information Protection log analytics table (InformationProtectionLogs_CL) and can be stored in the same log analytics workspace. The Microsoft Purview Information Protection connector must be enabled within Microsoft Sentinel in order to see events populate in log analytics going forward.

Guidance on how to set up this connector can be found here: [Stream data from Microsoft Purview Information Protection to Microsoft Sentinel](https://learn.microsoft.com/azure/sentinel/connect-microsoft-purview)

## Migrate queries to use the Microsoft Purview Information Protection connector

The next step is to compare the data fields within the Azure Information Protection table (InformationProtectionLogs_CL) and the Microsoft Purview Information Protection table (MicrosoftPurviewInformationProtection) and adjust existing queries.

A Sentinel workbook has been created to show how to transition queries from the the Azure Information Protection table (InformationProtectionLogs_CL) to the Microsoft Purview Information Protection table (MicrosoftPurviewInformationProtection). It replicates visualizations that used to appear within the Azure Information Protection workbook. 

- [Comparison Workbook](ComparisonWorkbook.json) showing queries from the Azure Information Protection connector and Microsoft Purview Information Protection connector side-by-side
-  [Microsoft Purview Information Protection workbook](MicrosoftPurviewInformationProtectionWorkbook.json)

### KQL query for data fields with JSON

There are some data fields that use JSON, and these data fields require an extra step in order to retrieve the data with KQL.

Example Data Field
```
[{"SensitiveInfoTypeId":"50842eb7-edc8-4019-85dd-5a5c1f2bb085","Count":1,"Confidence":85,"SensitiveInfoTypeName":"Credit Card Number"}]
```

Query to extract SensitiveInfoTypeId
```powershell
let Logs = MicrosoftPurviewInformationProtection 
| extend SensitiveInfo = parse_json(SensitiveInfoTypeData) 
| project SensitiveInfo.SensitiveInfoTypeId
```
### Comparison of Azure Information Protection and Microsoft Purview Information Protection events
Azure Information Protection (InformationProtectionLogs_CL) | Example | Microsoft Purview Information Protection (MicrosoftPurviewInformationProtection) | Example
---|---|---|---
**Tenant Information** | | |
TenantId	| ff614dc9-a3e5-4c1b-b360-4bdcb342230d | TenantId | ff614dc9-a3e5-4c1b-b360-4bdcb342230d
AadTenantId_g | 52a31004-8251-47e1-93f2-f0430045979c | OrganizationId | 52a31004-8251-47e1-93f2-f0430045979c
**User Information** | | |
UserId_s | admin@sarahdemolab.onmicrosoft.com | UserId | admin@SarahDemoLab.onmicrosoft.com
IPv4_s | 20.240.128.99 | ClientIP | 20.240.128.99
| | | UserType | Application
| | |  UserKey | 52c88d0c-dcbc-4b4e-a4f0-7f8b9484dd94
**Device and Application Information** | | |
SourceSystem | OpsManager | SourceSystem | |
ApplicationName_s | AIP scanner | Common | {"ApplicationId":"c00e9d32-3c8d-4a7d-832b-029040e7db99","ApplicationName":"Microsoft Azure Information Protection Scanner","ProcessName":"MSIP.Scanner","Platform":1,"DeviceName":"Scanner","ProductVersion":"2.16.16.1","Location":"On-premises file shares"}
ApplicationId_g | c00e9d32-3c8d-4a7d-832b-029040e7db99 | MgtRuleId | |
ManagementGroupName | | MachineName | |
MachineName_s | SCANNER | | |
Platform_s | Windows Desktop | Platform | Windows
DeviceId_g | 07468200-3795-40d1-9cb1-5ac556d6417d | Application | |
ProcessName_s | msip.scanner | Scope | Onprem
ProductVersion_s | 2.16.16.1 | DeviceName | |
Location_s | Endpoint | | |
ProcessVersion_s | | |
Computer | | | 
MG | | | 
**File Information** | | |
ObjectId_s	| c:\users\sarah\downloads\scannerfiles\folder (17)\random files\copy_1c_2.pptx	 | 	ObjectId | C:\Users\Sarah\Downloads\ScannerFiles\Folder (26)\Random Files\Copy_1C_19.xlsx
DataState_s	| Rest |	DataState |	Rest
LastModifiedBy_s | sarah seftel | | 			
LastModifiedDate_t [UTC] | 10/3/2022, 9:20:16.986 AM | |	
**Sensitive Information Type Information** | | |
DiscoveredInformationTypes_s | [{"Confidence": 85,"Count": 1,"SensitiveType": "50842eb7-edc8-4019-85dd-5a5c1f2bb085","UniqueCount": null, "SensitiveInformationDetections": null,"Name": ""Credit Card Number"}]	|	SensitiveInfoTypeData |[{"SensitiveInfoTypeId":"50842eb7-edc8-4019-85dd-5a5c1f2bb085","Count":1,"Confidence":85,"SensitiveInfoTypeName":"Credit Card Number"}]
InformationTypesAbove55_s	| ["Credit Card Number"]" | |			
InformationTypesAbove65_s	| ["Credit Card Number"] | |			
InformationTypesAbove75_s	| ["Credit Card Number"]
InformationTypesAbove85_s	| ["Credit Card Number"] | |		
InformationTypesAbove95_s	| []			
InformationTypes_s |	["Credit Card Number"] | |			
SensitivityChange_s | | |
**Protection Information** | | |
ContentId_g | IrmContentId | |	
ProtectionType_s	|	CurrentProtectionType | |	
ProtectionTypeBefore_s | PreviousProtectionType | | 	
Protected_b |	FALSE	|	ProtectionEventData	| {"IsProtected":false} 
ResultStatus_s	| ResultStatus | |	
ActionSource_s |	ActionSource | |	
ContentIdBefore_g	|	CurrentProtectionTypeName | |	
ProtectedBeforeAction_b	|	PreviousProtectionTypeName | |	
TemplateId_g	|	ProtectionEventTypeName | |	
ProtectionTime_t [UTC]	| ActionSourceDetail | |	
TemplateIdBefore_g	| CorrelationId	| |
ProtectionOwnerBefore_s | | |				
IsProtectionChanged_b	| | |				
ProtectionOwner_s	| | |				
RawData	| | |				
ErrorMessage_s | | |					
ActionId_g | | |					
ActionIdBefore_g | | |	
**Log Event Information** | | |
TimeGenerated [UTC] |	3/23/2023, 7:24:23.990 AM	|	TimeGenerated [UTC] |	3/23/2023, 7:22:53.000 AM
LogId_g |	dc0b25a4-1b7f-475b-9f7f-f1569056d2fa | Id	| 9608d13b-5e9f-4611-8395-28d6ae9628c1
Workload_s	| AIP	|	Workload | Aip
Operation_s |	Discover	|	Operation |	Discover
Type |	InformationProtectionLogs_CL	|	Type |	MicrosoftPurviewInformationProtection
Version_s |	1.1 |	RecordType | 93
Activity_s |	Discover |	RecordTypeName |	AipDiscover
_ResourceId | | |
**Related to other events** | | |
| | | ExecutionRuleId	
| | | ExecutionRuleName	
| | | ExecutionRuleVersion	
| | | RuleMode	
| | | Severity	
| | | SharePointMetaData	
| | | ExchangeMetaData	
| | | ConditionMatch	
| | | RuleActions	
| | | WorkLoadItemId	
| | | OverriddenActions	
| | | SensitiveInfoDetectionIsIncluded	
| | | IsViewableByExternalUsers	
| | | OverRideType	
| | | ItemCreationTime [UTC]	
| | | ItemLastModifiedTime [UTC]	
| | | ItemSize	
| | | OverRideReason	
| | | AppAccessContext	
| | | EmailInfo	
| | | ContentType	
| | | TargetLocation	
| | | Sender	
| | | Receivers	
| | | ItemName	
| | | ApplicationMode	
| | | LabelVersion	
| | | ScopedLocationId	
| | | PolicyId	
| | | PolicyName	
| | | PolicyVersion	

## Get label names with Microsoft Purview Information Protection Logs 

To retrieve label names to use with the Microsoft Purview Information Protection logs, use the [Export-LabelData PowerShell script](Export-LabelData.ps1). Before running the script, edit the script to include your log analytics workspace ID and the primary authentication key. Then, follow the PowerShell commandlets.  

```powershell
# If needed to run PowerShell scripts on machine, adjust the Execution Policy 
Set-ExecutionPolicy -ExecutionPolicy Unrestircted -Scope Process
```

```powershell
# Connect to Exchange Online 
Connect-IPPSSession
Import-Module ExchangeOnlineManagement
Connect-ExchangeOnline
```

```powershell
# Run the PowerShell script
./Export-LabelData.ps1
```

After running the script, a table will be created in the log analytics workspace that is specified in the script. The table will be called Labels_CL. The table can be joined with the MicrosoftPurviewInformationProtection table in log analytics by joining the tables on label id. Here is an example on how to join the two tables:

```
let Logs = MicrosoftPurviewInformationProtection 
| join kind=leftouter Labels_CL on $left.SensitivityLabelId==$right.Guid_g 
```
## Additional Resources

- [Integrate Log Analytics with Power BI](https://learn.microsoft.com/azure/azure-monitor/logs/log-powerbi)
- [Office 365 Management Activity API schema](https://learn.microsoft.com/office/office-365-management-api/office-365-management-activity-api-schema)