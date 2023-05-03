# DPU Manager

Author: Ben Peled

Created: Feb 24, 2023

## Problem Description
DPU BMC need to support access different to units, BMC platform already supports access to many BMC generic units, mainly through IPMI commands and Redfish schemas.
the objective is to support DPU specific units the same way.
This design will focus on the DPU Manager which will provide Redfish and IPMI with read and write access to the DPU specific units by creating D-Bus services. 

## Background and References
DPU has BMC features that require access to units like the DPU ARM, NIC and Eth SW those units are unique to the DPU and often not supported by generic BMC D-Bus objects.
DPU Manager will support DPU unique units access by implementing DPU specific D-bus objects.
It will enable support Redfish and Ipmi features separate DPU specific logic implemention from bmcweb and nvidia-impi-oem.

### DPU Manager current features
- DPU Ethernet Interface and DPU Chassis Networkadapters using D-bus EthernetInterface, IP interface and MAC Interface
- DPU SW Versions using D-bus Version interface

### DPU Manager future table
|Feature|Source |Content Protocol |D-Bus interfaces |Redfish interfaces|
|:----|:----|:----|:----|:----|
|Dpu Ethernet|DPU Ipmb  Eth<n>|OEM: Propriety file read from the dpu|Ethernet, IP , Mac|EthernetInterface, Chassis Ports, chassis network function|
|Dpu Versions|DPU Ipmb  fw info|OEM: Propriety file read from the dpu|Version|FirmwareInventory|
|Dpu Processors |TBD|TBD|CPU|Processors|
|Disable/Enable DPU eth port in 3-port switch|Switch i2c|OEM:TBD|OEM:TorSwitchPortsMode(wip)|OEM:TBD|
|enable rshim over USB|TBD|OEM:TBD|Reset|OEM:TBD|
|Support for the BMC to query the strap values of the NIC|NIC MCTP/I2c|OEM:TBD|TBD|TBD|
|Redfish based FW configuration to switch between BF DPU mode and enhanced NIC mode|TBD|OEM:TBD|Reset|OEM:TBD|
|DPU Dram|TBD|TBD|DIMM|memory|

## Requirements
- DPU units should be availble on Redfish and IPMI
- DPU Manager will provide read/write access to different DPU units using on D-Bus interfaces
- DPU Unit will communicate with DPU Manager using D-Bus services to access the DPU units
- DPU Manager should only include services that does not exist in generic BMC D-Bus services and aren’t needed by other platforms!
 
## Proposed Design
DPU Manager will be split in to three layers D-Bus object, FRU Parser and protocol enabling addition or change of message, protocol, or interface with minimal change and reuse each layer for other services.
### Protocol layer
Protocol layer is a wrapper to the interfaces communication protocols.
Giving generic interface to the different protocols. 
Protocol layer will support iteration, identification through StringId, read and write of data raw data.
Iteration and identification will enable us to create object dynamically.
Read and Write of raw data keeps separation between protocol and data itself.
Currently support IPMB, future should support i2c, and others

### FRU parser layer
Parser layer is parsing the the fru content manly using protocols that are specific to DPU platform.
FRU parser is created per FRU data message.
FRU parser can be used by more than one D-Bus, when FRU data needed by more than one object
FRU Parser will provide everything related to internal protocol from recognizing the right FRU by the String Id to parsing of the fields form raw data according to the protocol.
FRU Parser will enable fields read/write access and support dynamic allocation if needed.

### D-Bus object layer
D-Bus object is the implementation of the D-Bus interface methods or 
Using the FRU parser to access the specific fields.
D-Bus object can use more than one FRU parser, when object need to access data from more than one FRU

```
   ┌───────────┐    ┌───────────┐
   │  Redfish  │    │   IPMI    │
   │           │    │           │
   └──┬─────▲──┘    └──┬─────▲──┘
      │     │          │     │
┌─────▼─────┴──────────▼─────┴──────┐
│          D-Bus Interfaces         │
│                                   │
└───────────┬───────▲───────────────┘
            │       │
┌───────────▼───────┴───────────────┐
│          DPU Manager              │
│                                   │
│   ┌───────────────────────────┐   │
│   │      D-bus Objects        │   │
│   │                           │   │
│   └───────────────────────────┘   │
│                                   │
│   ┌───────────────────────────┐   │
│   │      FRU Parsers          │   │
│   │                           │   │
│   └───────────────────────────┘   │
│                                   │
│   ┌───────────────────────────┐   │
│   │       Protocols           │   │
│   │                           │   │
│   └───────────────────────────┘   │
│                                   │
│                                   │
└──┬────▲──────┬────▲──────┬────▲───┘
   │    │      │    │      │    │
 ┌─▼────┴─┐  ┌─▼────┴─┐  ┌─▼────┴─┐
 │  DPU   │  │  NIC   │  │ Switch │
 └────────┘  └────────┘  └────────┘

## Example case
Ethernet Manager will use IPMB protocol, it will iterate over all FRUs and check string ids.
For each fru with string id == eth# a eth parser object and a D-bus object will be created.
Eth parser will read the fru and parse the fields IP, MAC, MTU .. etc.
The dbus object will use EthernetInterface to read/write the fields from the parser to read the fields.

## Redfish current features paths
/redfish/v1/UpdateService/FirmwareInventory
/redfish/v1/Chassis/Card1/NetworkAdapters/ConnectX/NetworkDeviceFunctions/eth<n>
/redfish/v1/Chassis/Card1/NetworkAdapters/NvidiaNetworkAdapter/Ports/eth<n>

## Alternatives Considered
Implement logic under bmcweb/nvidia-impi-oem
Implement D-Bus object on other common project


## Testing
1. Check objects exist under the new service path 
2. Check objects exist under the object mapper
3. Use Redfish/Ipmi to read/write

