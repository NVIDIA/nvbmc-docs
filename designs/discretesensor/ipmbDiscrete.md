# Ipmb Discrete Sensor Design

Author: Krishna Raj

Created: October 31, 2022

## Problem Description

Implements discrete sensor for Ipmb type for NVIDIA platforms

## Background and reference

The openbmc project currently has a phosphor-ipmi-host repository to support the VR type discrete sensor. The intent of this design is to
monitor and list ipmb type discrete sensor.

### References

- Cable interface - https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Inventory/Item/Cable.interface.yaml
- Entity-Manager - https://github.com/openbmc/entity-manager
- Dbus-Sensors - https://github.com/openbmc/dbus-sensors
- ipmbbridge - https://github.com/openbmc/ipmbbridge
- VR Sensor - https://gerrit.openbmc.org/c/openbmc/phosphor-host-ipmid/+/42776
- https://docs.nvidia.com/networking/display/BlueFieldDPUOSv370/Intelligent+Platform+Management+Interface#IntelligentPlatformManagementInterface-ListofIPMISupportedFRUsListofIPMISupportedFRUs

## Requirements

- Entity manager – JSON configuration
- Dbus-sensors – discrete sensor services to monitor device status
- Phosphor-ipmi-host – Construct IPMI SDR from dbus objects and properties

## Proposed Design

### Diagram:


```
                                                              ┌───────────────┐
                                                              │ME,IPMB Device │
                                                              └───┬─────▲─────┘
                                                                  │     │
                                                                  │     │
                                                                  │     │
                                                           ┌──────▼─────┴─────────┐
                                                           │ phosphor-ipmi-ipmb   │◄──────────┐
                                                           │     (ipmbbridge)     │           │
                                                           └────────┬───▲─────────┘           │
                                                                    │   │                     │
                                                         ┌──────────┼───┼───────────┐         │
┌──────────────────┐                                     │          │   │           │         │
│                  │         getsensorconfigs            │  ┌───────┼───┼───────┐   │         │
│   Entity Manager ├─────────────────────────────────────┼──►       │   │       │   │         │
│                  │                                     │  │   ....▼...│....   │   │         │
└───────▲──────────┘                                     │  │   .           .   │   │         │
        │                                                │  │   . monitor   .   │   │         │
        │                                                │  │   .....▲.......   │   │         │
┌───────┴────────┐                                       │  │ update │  │       │   │         │
│     json       │               ┌──────────────┐        │  │property│  │       │   │         │
│  configuration │   ┌───────────┤   Sensor     │        │  │  ┌─────┴──▼─────┐ │   │         │
└────────────────┘   │           │ Dbus-objects ├────────┼──│  │              │ │   │         │
                     │  ┌────────►              │        │  │  │ ipmb discrete│ │   │         │
            sensor list │        └──────▲─────┬─┘        │  │  │    sensor    │ │   │         │
                     │  │               │     │          │  │  └──────────────┘ │   │         │
                     │  │               │     │Get sensor│  │                   │   │         │
                     │  │               │     │  list    │  └───────────────────┘   │         │
                     │  │               │     │          │     IPMB Discrete App    │         │
              ┌──────▼──┴────────┐      │     │          └──────────────────────────┘         │
              │                  │   ┌──┴─────▼────┐             Dbus-Sensors                 │
              │phosphor-ipmi-host│   │             │                                          │
              │    (dbus-sdr)    │   │ Web UI/     │                                          │
              │                  │   │   Redfish   │                                          │
              └─────┬────▲──────┬┘   └─────────────┘                                          │
                    │    │      │                             Bridging Command                │
                    │    │      └─────────────────────────────────────────────────────────────┘
                ┌───▼────┴─────┐
                │              │
                │  ipmitool    │
                │              │
                └──────────────┘
```

The new code would be creating new services under dbus-sensor package.

- Sensor configuration (Slave Address, ipmb sensor class) would be part of the entity-manager JSON configuration.
- Inventory objects which are created by entity-manager is used by ipmb discrete services to create 
  sensor objects and monitor status.
- ipmb discrete service will be using ipmbbridge service to read sensor status.
- Phosphor-ipmi-host package is responsible for collecting sensors based on interface and converting into IPMI specification.

### Brief Design of ipmb discrete sensor

- Sensor Configuration would be part of entity-manager JSON configuration file.
- Entity Manager will be creating inventory dbus objects with properties from JSON configuration file. Interfaces are created by using
  Type configuration in json.
- Ipmb discrete sensor service will be traversing entity manager inventory dbus objects and segregating by interface (Type) and creating
  sensor objects with cable interface.
- Reading for Discrete sensor:-
	 Using Get Sensor Reading Command via ipmb(bridging command) :-
      
        Request byte:-
        Byte 0:- Sensor ID
        Response byte :- 
        Byte 0:- Completion Code
        Byte 1:- Sensor Reading 
        Byte 3:- [7]-All Event Messages disabled from the sensor ,[6] sensor scanning disabled ,
                 [5]reading state,[4:0] reserved 
        Byte 4:- For Threshold-based sensors/discrete reading sensors 
                 for discrete reading sensors
                 [7]- 1b = state 7 asserted
                 [6]- 1b = state 6 asserted
                 [5]- 1b = state 5 asserted
                 [4]- 1b = state 4 asserted
                 [3]- 1b = state 3 asserted
                 [2]- 1b = state 2 asserted
                 [1]- 1b = state 1 asserted
                 [0]- 1b = state 0 asserted
        Byte 5:-  For discrete reading sensors only(Optional)
                 [7]- 1b = state 15 asserted
                 [6]- 1b = state 14 asserted
                 [5]- 1b = state 13 asserted
                 [4]- 1b = state 12 asserted
                 [3]- 1b = state 11 asserted
                 [2]- 1b = state 10 asserted
                 [1]- 1b = state 9 asserted
                 [0]- 1b = state 8 asserted

	 we can update sensor reading property accordingly by response byte of Get sensor reading  Byte4 and Byte5.
    
- Ipmb discrete sensor service wil be monitoring the status of sensor by one of the methods, either event based, or poll(Timer) based.
- Phosphor-ipmi-host dbus-sdr will be getting sensor objects using &quot;getsubtree&quot; dbus method call.
- The dbus Interfaces (Cable.interface.yaml) are matched and sdr properties and sensor status constructed respectively.
- phosphor-ipmi-ipmb gets the request and constructs I2C command from the request, then sends the I2C commend to
  ME(Intel Management Engine)/IPMB Device for getting information. 

#### JSON Configuration

##### P0_link
```
{
            "Address": "0x7",
            "Class": "MECable",
            "Name": "p0_link",
            "Type": "Ipmbstatus"
}
```
##### P1_link
```
{
            "Address": "0x8",
            "Class": "MECable",
            "Name": "p1_link",
            "Type": "Ipmbstatus"
}
```
### Compact SDR mapping

| Field Name | Mapping |
| --- | --- |
| Record ID | Input argument from ipmitool |
| SDR Version | Hardcoded as per IPMI2.0 spec, version 0x51h |
| Record Type | Hardcoded as per spec, 0x02h - compact sdr |
| Record Length | Size of compact sensor |
| Sensor Owner ID | Hardcoded bmci2caddr - 0x20h |
| Sensor Owner LUN | SensorNum << 8 |
| Sensor Number | Getting sensor number from object path<br />(sensorNumMapPtr) |
| Entity ID | can be based on sensor interfaces |
| Entity Instance | Can be get from sensor name(n).
| Sensor Type | Can be based on sensor interface |
| Event/Reading Type code | Can be based on sensor interface |
| Assertion Event Mask/Lower Threshold Reading Mask | can be get from available interfaces.
| Deassertion Event Mask/Upper Threshold Reading Mask | can be get from available interfaces.
| Discrete Reading Mask/Settable Threshold Mask, Readable Threshold Mask | can be get from available interfaces.
| Sensor Units 1 | not required for discrete-0x00h |
| Sensor Units 2 – Base Unit | not required for discrete-0x00h |
| Sensor Units 3 – Modifier Unit | not required for discrete-0x00h |
| Positive – going Threshold Hysteresis Value | not required for discrete-0x00h |
| Negative – going Threshold Hysteresis Value | not required for discrete-0x00h |
| Reserved | write as 0x00h |
| Reserved | write as 0x00h |
| Reserved | write as 0x00h |
| OEM | reserved for OEM use |
| ID String Type/Length Code | sizeof of sensor name |
| ID String Bytes | Can be get from sensor path. <br />(/xyz/openbmc\_project/<br />sensors/motherboard/<br />cable/p1\_link) |



## Alternatives Considered

None

## Testing

- Unit tests to make sure the dbus-objects are converting to IPMI from phosphor-ipmi-host and working.
- Run time test to make sure sensor objects getting created and status properties updated properly.
- Test to sensors and SDR,IPMITool sensor list and SDR list without affecting sensor listing mechanism.
