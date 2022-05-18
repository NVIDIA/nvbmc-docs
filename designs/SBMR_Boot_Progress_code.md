# SBMR Boot Progress Code Design Document
Author: Adarsh G R

Created: 2022/05/12
# Problem Description
The phosphor-host-postd supports postcode access through LPC interface currently. For ARM based platforms SSIF interface can be used to receive the boot progress code which is based on the SBMR specification.

## Background and References
1.  Arm Server Base Manageability Requirements 2.0
2.  IPMI Specification, V2.0.
3.  UEFI Platform Integration Specification, version 1.7 Errata A
4.  DSP8010 Redfish Schema. DMTF.

## Requirements

 - Provide an IPMI SBMR command to store the Boot progress code in the BMC. 
 - Map the Boot progress code to the respective dbus properties.
 - Provide an IPMI SBMR command to get last received Boot progresss code.
 - Add support to display the last BootProgress codes received with timestamp and payload in Redfish. 
 - Store all the bootprogress codes received in BMC filesystem and display the Bootprogress codes in Redfish log service and Web GUI.The BMC file system will store the bootprogress code upto 100 bootcycle.   
 - Seperate the Bootprogress code received into Error/Progress and Debug and perform necessary actions based on the Bootprogress types.
 
## SBMR Boot progress Codes

- The IPMI commands for Boot progress codes are:
  1. Send boot progress code (NetFn 2Ch, Command 02h)
  2. Get boot progress code (NetFn 2Ch, Command 03h)

- The above commands use group extension 0x2C and defining body 0xAE since the commands are under non IPMI groups and requests.

### Send boot progress code 

This command is used to send the Boot Progress Code to the BMC


|| |
| - | - |
|NetFn|0x2C|
|Command|0x02|


**Request Data:**

|Byte|Data field|
| - | - |
|1|Group extension defining body (AEh) |
|2-10|Boot Progress Code record (9 bytes). |

**Response Data:**

|Byte|Data field|
| - | - |
|1|Completion Code 0x0 -Command completed success, 0x80 -Command completed with error|
|2 |Group extension defining body (AEh)|

The Boot progress code of 9 bytes will be written into respective Dbus objects hosted by phosphor-host-postd.  



### Get boot progress code

This command is used to read the last Boot Progress Code that was received by the BMC from the command “Send Boot Progress Code”.


|| |
| - | - |
|NetFn|0x2C|
|Command|0x03

**Request Data:**

|Byte|Data field|
| - | - |
|1|Group extension defining body (AEh)|

**Response Data:**

|Byte|Data field|
| - | - |
|1|Completion Code    0x00 -Command completed success  ,0x80 -Command completed with error|
|2 |Group extension defining body (AEh)|
|3-11|Boot progress Code (9 Bytes)|

The 9 bytes of last Boot progress code received will be retrieved from the respective Dbus objects and send to the user through IPMI. 


##  Boot progress record format

- BMC will receive a 9 bytes of boot progress code record from Send boot progress command.


|**Byte offset**|**Size(Bytes)**|**Description**|
| :- | :- | :- |
|0|1|Status code type|
|1|2|Status code reserved|
|3|1|Status code severity|
|4|2|EFI Status code operation|
|6|1|EFI Status code Subclass|
|7|1|EFI Status code Class|
|8|1|Instance|



 ## DESIGN
**Interface Diagram**

Diagram Legend

|Label|Signifies|
|-----|---------|
|`I:` |D-Bus interface|
|`S:` |D-Bus service name (well-known bus name)|
|`R:` |Repository name|
|`P:` |Property|
|`M:` |Method|




```
+---------------------------------------------------------------+
|                                                               |
|     +------------------------------+                          |         +---------------------+
|     |     SSIF BRIDGE              |                          |         |                     |
|     |     (ssif bridge)            <------------- SSIF--------+---------+       HOST          |
|     |                              |                          |         |                     |
|     +------------+-----------------+                          |         |                     |
|                  |                                            |         +---------------------+
|                  |                                            |
|     +------------v-----------------+                          |
|     |   IPMI OEM HANDLER           |                          |
|     |   (nvidia-ipmi-oem)          |                          |
|     |                              |                          |
|     +--------------+---------------+                          |
|                    |                                          |
|                    |                                          |
|    +---------------v-----------------------------------+      |
|    | R:Phosphor-host-postd                             |      |   
|    |    +-------------------------------------------+  |      |
|    |    |boot-progress-manger                       |  |      |
|    |    | (feature:sbmr-boot-progress)              |  |      |
|    |    |                                           |  |      |
|    |    | S:xyz.openbmc_project.State.Boot.Raw      |  |      |
|    |    | /xyz/openbmc_project/state/boot/raw0      |  |      |
|    |    | I:xyz.openbmc_project.State.Boot.Raw      |  |      |
|    |    | P: Value                                  |  |      |
|    |    |                                           |  |      |
|    |    |                                           |  |      |
|    |    | S:xyz.openbmc_project.State.Host          |  |      |
|    |    | /xyz/openbmc_project/state/host0          |  |      |
|    |    |I:xyz.openbmc_project.State.Boot.Progress  |  |      |
|    |    | P:BootProgress                            |  |      |
|    |    | P:BootProgressLastUpdate                  |  |      |
|    |    | P:BootProgressOem                         |  |      |
|    |    +-------------------------------------------+  |      |
|    |                                                   |      |
|    +---------------+--------------------^--------------+      |
|                    |                    |                     |
|                    |                    + ---------+          |
|  +-----------------v--------------------------+    |          |
|  |R:phosphor-post-code-manager                |    |          |
|  |                                            |    |          |
|  |S:xyz.openbmc_project.State.Boot.PostCode0  |    |          |
|  |/xyz/openbmc_project/State/Boot/PostCode0   |    |          |
|  | I:xyz.openbmc_project.State.Boot.PostCode  |    |          |
|  | M:GetPostCodesWithTimeStamp                |    |          |
|  +-----------------^-------------------------+     |          |
|                    |                 +-------------+          |
|                    |                 |                        |
|      +-------------+-----------------+------------+           |
|      | Redfish                                    |           |
|      |                                            |           |
|      | Get BootProgress Code:                     |           |
|      |                                            |           |
|      | S:xyz.openbmc_project.State.Host           |           |
|      |  /xyz/openbmc_project/state/host0          |           |
|      | I:xyz.openbmc_project.State.Boot.Progress  |           |
|      | P:BootProgress                             |           |
|      | P:BootProgressLastUpdate                   |           |
|      | P:BootProgressOem                          |           |
|      |                                            |           |
|      |Get Log Service                             |           |
|      |                                            |           |
|      |S:xyz.openbmc_project.State.Boot.PostCode0  |           |
|      |/xyz/openbmc_project/State/Boot/PostCode0   |           |
|      |I:xyz.openbmc_project.State.Boot.PostCode   |           |
|      |M :GetPostCodesWithTimeStamp                |           |
|      +--------------------------------------------+           |
|                                                               |
|                                                               |
|                                          BMC                  |
|                                                               |
+---------------------------------------------------------------+
```
**Bootprogress Code Flow:**

- BMC Power on the host. 
- Host starts sending the boot progress IPMI message to the BMC through SSIF Interface.
- This IPMI Message will be in SBMR IPMI Bootprogress code format.
- The SSIF Bridge pass along the message to IPMI daemon(ipmid).
- The ipmid writes this boot progress code to appropriate D-Bus raw objects hosted by phosphor-host-postd.
- The phosphor-post-code-manager will get the PropertiesChanged signals from this  DBus Raw object. 
- The phosphor-post-code-manager will store the boot progress code in persistant memory.
- The phosphor-post-code-manager will store all the bootprogress codes received in a boot cycle in BMC filesystem and it can store the bootprogress data upto 100 bootcycle.
- In phosphor-host-postd a boot-progress-manger service is created . This service will be enabled only when the feature sbmr-boot-progress is enabled. 
- The boot-progress-manager will update the following properties along with the Dbus Raw.Value property(xyz.openbmc_project.State.Boot.Raw.Value).
    - BootProgress : This property will hold the bootprogress stages .The boot-progress-manager will decode the 9 bytes of boot progress data and map this into standard Bootprogress stages (https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/State/Boot/Progress.interface.yaml). If the data does not map to one of the standard bootprogress stages the Bootprogress property will be set as an OEM bootprogress stage. 
    - BootProgressLastUpdate : This property will hold the timestamp of last boot progress code received.
    - BootProgressOem : This property will hold the 9 byte hex values of boot progress code in string format.
-  boot-progress manager will also have a boot_progress_code.json file which contain boot progress codes and their meaingful texts.
- If the bootprogress code received is an Error type and a match is found in the boot_progress_code.json ,then boot-progress-manager will create an event logging for this code. 
- The boot_progress_code.json will be also used in Redfish Postcode entries(/redfish/v1/Systems/system/LogServices/PostCodes/Entries) to display as a meessage if a match is found for the boot progress code. 
- When BMC reboots the boot-progress-manager will retrieve last received bootprogess with timestamp from the persistant BMC filesytem and update the appropriate Dbus properties(BootProgress/BootProgressOem/BootProgressLastUpdate). 

## Phosphor-host-postd

- Phosphor-host-postd contain LPC snoop broadcast daemon which reads postcodes from an lpc-snoop driver and broadcasts the values on DBus.
- We have introduced a new Boot-progress-manager service in phosphor-host-postd to support ARM SBMR Bootprogress codes.
- If the platform is x86 and support postcodes then snoop feature can be used . By default snoop feature will be enabled in the recipe
- The snoop feature can be disabled by adding below changes in the bbappend file. 
   PACKAGECONFIG:remove += "snoop-feature" 
   EXTRA_OEMESON += "-Dsnoop-feature=disabled"
- If the platform is ARM and support boot progress codes then the sbmr-boot-progress feature can be enabled in the bbappend file by adding below changes.
 PACKAGECONFIG:append += "sbmr-boot-progress" 
 EXTRA_OEMESON += "-Dsbmr-boot-progress=enabled"
- The lpc snoop service and boot-progress-manager service changes are kept separately to distinguish between server architecture(ARM/x86) 

## Platform Specific OEM Handler (nvidia-ipmi-oem)
This library is part of the phosphor-ipmi-host and gets the postcode from host through phosphor-ipmi-ssif.

- Register IPMI OEM SBMR Boot progress handler.
- Extract postcode from IPMI message (phosphor-ipmi-host/phosphor-ipmi-ssif).
- Sets Value property on appropriate D-Bus Raw object hosted by phosphor-host-postd. Other programs (e.g. phosphor-post-code-manager) can subscribe to PropertiesChanged signals on this object to get the updates.

## Redfish
To display boot progress codes in Redfish a new BootProgressOem property is created under xyz.openbmc_project.State.Host. This property will hold the 9 byte hex values of bootprogress  as a string.

- The Bootprogress stage will be mapped to redfish **LastState** property.The DMTF schema defines a handful of standard boot progress codes.
- If the Boot Progress Code does not map to one of the DMTF defined codes, the LastState property will be set to OEM and the **OEMLastState** property will be set to the 9-byte hex values of bootprogress data.
- The BootProgressOem proeprty value will be mapped to the OEMLastState.
- BootProgressLastUpdate property which hold the timestamp of last bootprogress code received  and this will be mapped to the **LastStateTime** property.

The Bootprogress codes stored in the BMC file system can be accessed from Web GUI and from redfish log entries (https://<BMC_IP>/redfish/v1/Systems/system/LogServices/PostCodes/Entries).The **GetPostCodesWithTimeStamp** Method can be used retrive the boot progress codes stored in the BMC file system.The Redfish will also map the bootprogress codes with the boot_progress_code.json file. If a match is present for bootprogress codes then the corresponding text will be shown as a message data in PostCodes entries. 

## Sample Mapping

The Sample mapping of Host Processor Subclass is given below:

The first four bytes of Boot Progress Code indicates the Status of the Boot progress.

|**Byte offset**|**Size(Bytes)**|**Description**|**Supported Bytes**|
| :- | :- | :- | :- |
|0|1|Status code type| 0x1 -Progress Code, 0x2-Error code, 0x3 Debug code| 
|1|2|Status code reserved||
|3|1|Status code severity|0x40-Minor ,0x80-Major,0x90-Unrecoverd error,0xa0-Error Contained|


Which will give the information if the Boot progress code received is  progress/Error/Debug code and in case of error the status code severity indicates the Error severity.

|**EFI Status Code Class**|**EFI Status Code Subclass**|
| :- | :- | 
|0x00|0x01|

The EFI Status code and EFI Status code subclass will help to identify the component in the system. Example given in the above table indicates that the component is Host processor. 

The Host processor operation data for Progress codes are :

|Operation|Description| Operation Byte|
|:----|:----|:----|
|EFI_CU_PC_INIT_BEGIN|General computing unit initialization begins|Ox0000|
|EFI_CU_PC_INIT_END|General computing unit initialization ends|Ox0001|
|EFI_CU_HP_PC_POWER_ON_INIT|Power-on initialization|Ox1000|
|EFI_CU_HP_PC_CACHE_INIT|Embedded cache initialization including cache controller hardware and cache memory.|0x1001|
|EFI_CU_HP_PC_RAM_INIT|Embedded RAM initialization|0x1002|
|EFI_CU_HP_PC_MEMORY_ CONTROLLER_INIT|Embedded memory controller initialization|0x1003|
|EFI_CU_HP_PC_IO_INIT|Embedded I/O complex initialization|0x1004|
|EFI_CU_HP_PC_BSP_SELECT |BSP selection|0x1005|
|EFI_CU_HP_PC_BSP_RESELECT|BSP reselection|0x1006|
|EFI_CU_HP_PC_AP_INIT|AP initialization (this operation is performed by the current BSP)|0x1007|
|EFI_CU_HP_PC_SMM_INIT|SMM initialization|0x1008|
|0x000B–0x7FFF |Reserved for future use by this specification| |

The Host processor operation data for Error codes are :

|Operation|Error Description| Operation Byte|
|:----|:----|:----|
|EFI_CU_EC_NON_SPECIFIC|No error details available.|0x0000|
|EFI_CU_EC_DISABLED|Instance is disabled.|0x0001|
|EFI_CU_EC_NOT_SUPPORTED|Instance is not supported|0x0002|
|EFI_CU_EC_NOT_DETECTED|Instance not detected when it was expected to be present|0x0003|
|EFI_CU_EC_NOT_CONFIGURED|Instance could not be properly or completely initialized or configured.|0x0004|
|EFI_CU_HP_EC_INVALID_TYPE|Instance is not a valid type.|0x1000|
|EFI_CU_HP_EC_INVALID_SPEED|Instance is not a valid speed.|0x1001|
|EFI_CU_HP_EC_MISMATCH|Mismatch detected between two instances.|0x1002|
|EFI_CU_HP_EC_TIMER_EXPIRED|A watchdog timer expired.|0x1003|
|EFI_CU_HP_EC_SELF_TEST|Instance detected an error during BIST|0x1004|
|EFI_CU_HP_EC_INTERNAL|Instance detected an IERR.|0x1005|
|EFI_CU_HP_EC_THERMAL|An over temperature condition was detected with this instance.|0x1006|
|EFI_CU_HP_EC_LOW_VOLTAGE|Voltage for this instance dropped below the low voltage threshold|0x1007|
|EFI_CU_HP_EC_HIGH_VOLTAGE|Voltage for this instance surpassed the high voltage threshold|0x1008|
|EFI_CU_HP_EC_CACHE|The instance suffered a cache failure.|0x1009|
|EFI_CU_HP_EC_MICROCODE_ UPDATE|instance microcode update failed|0x100A|
|EFI_CU_HP_EC_CORRECTABLE|Correctable error detected|0x100B|
|EFI_CU_HP_EC_UNCORRECTABLE|Uncorrectable ECC error detected|0x100C|
|EFI_CU_HP_EC_NO_MICROCODE_UPDATE|No matching microcode update is found|0x100D|
|0x100D–0x7FFF|Reserved for future use by this specification| |

**Sample Mapping of Boot progress code :**

|Bytes Received |Boot Progress String |
|:----|:----|
|0x01 0x00 0x00 0x00 0x00 0x00 0x01 0x00 0x00|xyz.openbmc_project.State.Boot.Progress.ProgressStages.PrimaryProcInit|
|0x01 0x00 0x00 0x00 0x00 0x00 0x05 0x00 0x00|xyz.openbmc_project.State.Boot.Progress.ProgressStages.MemoryInit|
|0x01 0x00 0x00 0x00 0x00 0x00 0x01 0x02 0x00|xyz.openbmc_project.State.Boot.Progress.ProgressStages.PCIInit|



If the user received the bootprogress code 0x01 0x00 0x00 0x00 0x00 0x00 0x01 0x02 0x00 then after mapping the Boot Progress string (xyz.openbmc_project.State.Boot.Progress) will be **xyz.openbmc_project.State.Boot.Progress.ProgressStages.PCIInit** 


Similarly for each subclass separate mapping is present in the  UEFI Platform Integration Specification Document.

## Testing

For testing the Send boot progress command are enabled through network medium. 
The Sample request response is shown below:

1. Send boot progress code

   cmd : ipmitool -H <bmc_ip> -U <user_name> -P <password> -I lanplus -C 17 raw 0x2c 0x02 0xae 0x01 0x00 0x00 0x00 0x00 0x10 0x01 0x00 0x00

    respone : ae

2. Get boot progress code
 
   cmd : ipmitool -H <bmc_ip> -U <user_name> -P <password> -I lanplus -C 17 raw 0x2c 0x03 0xae
 
    response : ae 01 00 00 00 00 10 01 00 00

3. Check the progress code from the Redfish interface

   URI :/redfish/v1/Systems/system

   "BootProgress": {

        "LastState": "OEM",
        "OemLastState": "0x010000000010010000",
        "LastStateTime ": "2022 -06 -01 T10 :14:13+06:00" 
    
    }

