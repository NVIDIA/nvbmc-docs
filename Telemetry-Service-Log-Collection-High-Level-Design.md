# System Log collection
Author: Sudhakar Kuppusamy

Primary assignee: Sudhakar Kuppusamy

Other contributors: Sudhakar Kuppusamy, Pravinash Jeyapaul

Created: Nov 29, 2021

## Problem Description
Platform debug logs are needed to diagnose various problems. The following logs are generated as part of platform monitoring while the system is running. Phosphor Debug Collector already provides mechanisms to collect following type of debug logs and system parameters, however it does not have the following plugins for collection of system information log with password protection. Also, there is no provisioning to downloading of generated log file using webUI.

Type of Debug logs:
1.	Application Cored
2.	User Requested
3.	Internal Failure
4.	Checks top

### Background and References
This feature provides autonomous way of collecting the various types of logs from the platform through different interfaces.
### Requirements
* The WebUI/Redfish should allow the user to generate and download the logs as ZIP file with password protected.
* Password can be static one in system side; vendor can pass the password information to customer to unzip it.
* The log files should have the date and time appended to the prefix of the filename.
* The ZIP file should contain the following
   * Single integral HTML file includes all required log information. 
   * Separate file for CPU crash log, journal log, SEL log (raw and HEX).
   * All the collected information will be bundled and zipped at the backend. 
### Proposed Design
The proposal is to add following plugins on phosphor-debug-collector for log collection as a separate file. those logs are collected at on demand request from redfish or UI. Also, webUI will provide the menu to generate/download logs.

#### Plugins:
1.	General BMC Information 

    It will use the Linux command to collect the Kernel command line, Process list, Detailed process list, Free memory, Slab information, Interrupts, SoftIRQs, Filesystem information and Mount information.

2.	BMC kernel ring buffer

    It will use the dmesg command to collect the kernel ring buffer information

3.	BMC network information

    It will use the Linux command to collect the Network routes, Network connections, ARP Table and IP route information’s.

4.	BMC configurations Data

    It will use the Linux command to collect the Channel Access, Channel Config, ARPControl Configurations and LAN/VLAN Configurations

5.	BIOS POST Code readings

    It will get Post Codes information from phosphor-post-code-manager

6.	BMC Sensor Readings

    It will get from ipmi sensor list command

7.	BMC SEL Information

    It will get from rootfs(/var/lib/ipmi_sel); and text data from redfish: https://BMC_IP/redfish/v1/Systems/system/LogServices/EventLog/Entries
    Once circular SEL policy is implemented, all the SEL data can be retrieved from single file or from IPMI.

8.	BMC File Information

    It will use the Linux command to list all the files in /tmp and /var directory

9.	Watchdog Reset Logs

    It will separate watchdog reset log in current system event logs. 

10.	CPU Registers and PCI Configuration Space Dump

    The configuration details can be fetched through PECI interface


### Alternatives Considered
Not support the subfunction of Power_supply_blackbox_data & Power_supply_PEC_feature_support_status since it is not supported by openBmc low level driver
#### Impacts
The system log generation will consume near to 1 minutes of time and it will also increase the tar/zip file size.
#### Testing
### Steps to generate the System log:

#### 1.BMC Dump file creation

Make a post request with following URL and JSON. It will create the task at backend (BMC) then return the task ID and status as a response.

URL: https://<BMCIP>/redfish/v1/Managers/bmc/LogServices/Dump/Actions/LogService.CollectDiagnosticData

JSON:

{

   "DiagnosticDataType":"Manager",

   "OEMDiagnosticDataType":"BMC"

}

#### 2.Checking of task status with task ID

Make a get request with following URL with task ID to know the task status. It will return a JSON format detailed information about the task as a response.

URL: https://<BMCIP>/redfish/v1/TaskService/Tasks/<task_ID>

#### 3.Get the number of entries in the Dump

Make a get request using the following URL. It will return the number of entry in the Dump.

URL: https://<BMCIP>/redfish/v1/Managers/bmc/LogServices/Dump/Entries

#### 4.To view the dump entry information using dump entry ID.

Make a get request using the following URL with dump entry ID. It will return the Dump entry information as a response.

URL: https://<BMCIP>/redfish/v1/Managers/bmc/LogServices/Dump/Entries/<dump_ID>

#### 5.Download the Generated Log

Make a get request using following the URL with Dump Id on browser. It will download the log file after that we need to change the file name with .tar.xz file format. Then we need to extract the file with help of any tar file extractor. After extracting you could get the all log files.

URL: https://<BMCIP>/redfish/v1/Managers/bmc/LogServices/Dump/attachment/<dump_ID>

