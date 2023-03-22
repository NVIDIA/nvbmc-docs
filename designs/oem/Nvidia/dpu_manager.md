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
- DPU Ethernet Interface
- DPU SW Versions
- DPU Chassis Networkadapters

### DPU Manager future features
- DPU Processors
- DPU DRAM
- DPU Nic Flash

## Requirements
- DPU units should be availble on Redfish and IPMI
- DPU Manager will provide read/write access to different DPU units using on D-Bus interfaces
- DPU Unit will communicate with DPU Manager using D-Bus services to access the DPU units
- DPU Manager should only include services that aren’t exist in generic BMC D-Bus services and aren’t needed by other platforms
 
## Proposed Design
DPU Manager will be split in to three layers D-Bus object, FRU Parser and protocol enabling addition or change of message, protocol, or interface with minimal change and reuse each layer for other services.
### Protocol layer
Protocol layer is a wrapper to the interfaces communication protocols.
Giving generic interface to the different protocols. 
Protocol layer will support iteration, identification through StringId, read and write of data raw data.
Iteration and identification will enable us to create object dynamically.
Read and Write of raw data keeps separation between protocol and data itself. 

### FRU parser layer
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

## Alternatives Considered
Implement logic under bmcweb/nvidia-impi-oem
Implement D-Bus object on other common project


## Testing
1. Check objects exist under the new service path 
2. Check objects exist under the object mapper
3. Use Redfish/Ipmi to read/write

