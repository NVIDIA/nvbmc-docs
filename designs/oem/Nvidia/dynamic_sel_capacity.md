# Dynamic SEL Capacity

Author: Sean Zhang

Created: November 17, 2023

## Problem Description
Currently, the capacity of SEL records is predefined to 3K. Once the records number is large or close to the capacity, the SEL list command will take long time to show the records, and the list operation will impact other operations. To improve the function, need to make the capacity of SEL records configurable so that customers can reduce the capacity to avoid the SEL list command taking too long.

## Requirements
1.	The user can configure the SEL records capacity with ipmi command/redfish.
2.	The user can get the SEL records capacity with ipmi command/redfish.

## Proposed Design
1.  Add ipmitool raw command and redfish interface for user to configure the capacity of SEL records. The new command will trigger DBUS Capacity methods errorInfoCapacity to set the capacities of SEL records. The new methods will be under “xyz/openbmc_project/Logging/Capacity”
2.  The new logging interface will include the new methods.
3.  The new method will be implemented by new functions to change the errorInfoCap values in map binNameMap[“SEL”].
4.  The implementation in method errorInfoCapacity need to check the existing entries number and remove the older entries if the entries number exceed the new capacity.
5.  Add implementation for ipmitool raw command and redfish interface for getting the capacity.

### Actions Set SEL Capacity
A new IPMI raw command will be added to set SEL Capacity:
```
ipmitool raw 0xa 0x4a number
```

A new OEM link will be added to the Managers schema under Bluefield_BMC/Actions/Oem/Nvidia:
```
https://<bmc_ip>/redfish/v1/Managers/Bluefield_BMC/Actions/Oem/Nvidia/SelCapacity
{
  "ErrorInfoCap": 300
}
```
curl -k -H "X-Auth-Token: <token>" -X POST https://$bmc/redfish/v1/Managers/Bluefield_BMC/Actions/Oem/Nvidia/SelCapacity -d '{"ErrorInfoCap":300 }'

### Reading SEL Capacity
A new IPMI raw command will be added to get SEL Capacity:
```
ipmitool raw 0xa 0x4b
```

A new OEM link will be added to the Managers schema under Bluefield_BMC/Oem/Nvidia:
```
https://<bmc_ip>/redfish/v1/Managers/Bluefield_BMC/Oem/Nvidia/SelCapacity
```
curl -k -H "X-Auth-Token: <token>" -X GET https://$bmc/redfish/v1/Managers/Bluefield_BMC/Oem/Nvidia/SelCapacity

## Testing
1. Use Ipmi/Redfish to set/get the SEL capacity
2. Check if the SEL entries wrap with the configured capacity

## Limitation
1. The ErrorInfoCap to be set should not be larger than build option "-Derror_info_cap".
