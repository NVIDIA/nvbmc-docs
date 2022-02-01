# System Log collection
Author: Sudhakar Kuppusamy

Primary assignee: Sudhakar Kuppusamy

Other contributors: Sudhakar Kuppusamy, Pravinash Jeyapaul

Created: Nov 29, 2021

## Problem Description
Platform debug logs are needed to diagnose various problems. The following logs are generated as part of platform monitoring while the system is running. Phosphor Debug Collector already provides mechanisms to collect following type of debug logs and system parameters, however it does not have the following plugins for collection of system information log with password protection. Also, there is no provisioning to downloading of generated log file using webUI/Redfish/Busctl command.

Type of Debug logs:
1.	Application Cored
2.	User Requested
3.	Internal Failure
4.	Checks top

missing plugins:

- General BMC Information
	- Kernel Command line
    - Detailed process list
    - Free memory
    - Slab information
    - Interrupts
    - SoftIRQs
    - Mount information
- Kernel Ring Buffer Information
- BMC Network Interfaces
    - Network connections Information
    - ARP Table Information
    - Network routes Information
    - IP route information’s
- BMC File Information
	- TMP file list
    - VAR file list
- BMC Configuration Data
    - IPMI Channel Access
    - IPMI Channel Config
    - ARPControl Configurations
    - LAN/VLAN Configurations
- BMC Sensor Readings
- BMC SEL Information
- BIOS POST Code
- Watchdog Reset Logs
- CPU Registers and PCI Configuration Space Dump


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

    It will use the Linux command to collect the Network routes, Network connections, ARP Table, IP address and IP route information’s.

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

    -TBD

### Steps to generate the telemetry log:
Web server or UI can able to access the file under www only. But www is read only file system. We need to have some mechanism to generate and make the file available under www. The below is one of the option:

Step 1: Create empty telemetrylogs.zip file under /usr/share/www/ at build time

Step 2: When user click the download button, it uses REST API to generate the log files

Step 3: It stores all the collected files and create zip file under /var/lib/phosphor-debug-collector/dumps/ telemetrylogs.zip

Step 4: Unmount the /usr/share/www/telemetrylogs.zip if it is already mounted

Step 4: Create mount point to /usr/share/www/telemetrylogs.zip

Step 5: Return the response to UI and then link to download the telemetrylogs.zip file will be shown in UI

Step 6: User can click the link to download the zip file

Step 2 should overwrite file each time to generate the logs to get the current snapshot.

### WebUI/redfish:

It will use the Dbus Create function call to generate the logs and it will trigger to build zip file. For Download, it will download the generated log file from www directory.

### Alternatives Considered
Not support the subfunction of Power_supply_blackbox_data & Power_supply_PEC_feature_support_status since it is not supported by openBmc low level driver
#### Impacts
None
#### Testing
In WebUI, there is a pull-down menu for telemetry, click the button then it will download the tar/zip file.
