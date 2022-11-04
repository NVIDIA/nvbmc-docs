# RST_GPU Discrete Sensor Design

Author: Selvaganapathi M

Created: Nov 01, 2022

## Problem Description

Implements discrete sensor for RST_GPU OEM type for NVIDIA platforms

## Background and reference

The openbmc project currently has a phosphor-ipmi-host repository to support the VR type discrete sensor. The intent of this design is to monitor and list OEM type discrete sensor.

### References

- Entity manager, Dbus-Sensor
- VR Sensor - [https://gerrit.openbmc.org/c/openbmc/phosphor-host-ipmid/+/42776](https://gerrit.openbmc.org/c/openbmc/phosphor-host-ipmid/+/42776)
- SMBPI Document DG-06034-002_v05.1.pdf

## Requirements

- Entity manager – JSON configuration
- Dbus-sensors – discrete sensor services to monitor reset required data from GPU's
- Phosphor-ipmi-host – Construct IPMI SDR from dbus objects and properties

## Proposed Design

### Diagram:
```
+------------------------------------------------------------------------------+
|                                                                              |
|  +--------------------+    +--------------------+    +--------------------+  |
|  |                    |    |                    |    |                    |  |
|  | JSON Configuration +---->   Entity-Manager   <---->    Dbus-Objects    |  |
|  |                    |    |                    |    |                    |  |
|  +--------------------+    +--------------------+    +---------+----------+  |
|Entity-Manager                                                  |             |
+----------------------------------------------------------------+-------------+
                                                                 |
                          +--------------------------+-----------+
                          |                          |
               +----------+--------------------------+----------+
               |          |                          |          |
               |  +-------v--------+        +--------v-------+  | SMBPBI(I2C) Interface
+-----------+  |  |                |        |                |  |  +------------------+
|sysfs path <--+-->   HwmonTemp    |. . . . |   gpuStatus    <--+--+--+   +-------+   |
+-----------+  |  |    Sensor      |        |                |  |  |  |   | GPU1  |   |
               |  +-------+--------+        +--------+-------+  |  |  +---+       |   |
               |          |                          |          |  |  |   +-------+   |
               |          +-------------+------------+          |  |  |               |
               |                        |                       |  |  |   +-------+   |
               |  +---------------------v--------------------+  |  |  |   | GPU2  |   |
               |  |                                          |  |  |  +---+       |   |
               |  |              Dbus-Objects                |  |  |  |   +-------+   |
               |  |                                          |  |  |  |               |
               |  +---------------------+--------------------+  |  |  |   +-------+   |
               |dbus-sensors            |                       |  |  |   | GPUn  |   |
               +------------------------+-----------------------+  |  +---+       |   |
                                        |                          |      +-------+   |
                                        |                          | Baseboard        |
                          +-------------v--------------+           +------------------+
                          |                            |
                          |Phosphor-ipmi-host(dbus-sdr)|
                          |                            |
                          +-------------^--------------+
                                        |
                                        |
                               +--------v-----------+
                               |      IPMITool      |
                               +--------------------+
```
The new code would be creating new services under dbus-sensor package.

- Sensor configuration would be part of the entity-manager JSON configuration.
- Inventory objects which are created by entity-manager is used by dbus-sensor services to create 
  sensor objects and monitor.
- Dbus-sensor service will be using asynchronous timer to poll and get GPU status.
- Phosphor-ipmi-host package is responsible for collecting sensors based on interface and converting into IPMI specification.

### Brief Design Points of GPU OEM sensor

- Sensor Configuration would be part of entity-manager JSON configuration file
- Entity Manager will be creating inventory dbus objects with properties from JSON configuration file. Interfaces are created by using Type configuration in json
- GPU status service will be traversing entity manager inventory dbus objects and segregating by interface (Type) and creating sensor objects with OEM reset GPU interface
  - Once Dbus object created, it start reading GPU state flag using asynchronous timer with 2sec delay.
  - It will request access privilege as BMC to FPGA.
  - Once BMC get a privilege, will intiate SMBPI command to all the GPU to get Reset_required state flag
  - Based on the GPU Reset Required State flag, status(GPUStatusn) property gets updated as string(ResetRequired)
- Phosphor-ipmi-host dbus-sdr will be getting sensor objects using &quot;getsubtree&quot; dbus method call
- The dbus Interfaces are matched and sdr properties and sensor status constructed respectively

#### GPU SMBPI Command:

| Opcode | Arg1 | Arg2 | State Bit |
| --- | --- | --- | --- |
| 0x18 - Query GPU State Flags | 0x01 - Device State Flags Page 1 | Unused | Bit[0] 0 - GPU Reset Not Required |
|||| 1 - GPU Reset Required |

#### RST_GPU Configuration
```
    {
        "Name": "RST_GPU",
        "EntityId": "0x06",
        "Bus1": "0x01",
        "Bus2": "0x02",
        "Address1": ["0x44", "0x45", "0x46", "0x47"],
        "Address2": ["0x44", "0x45", "0x46", "0x47"],
        "Type": "rstgpu"
    }
```
### Representation of dbus interfaces

| Sensor Name | Object | Interfaces | Property | Value |
| --- | --- | --- | --- | --- |
| RST_GPU| xyz/openbmc\_project/<br />sensors/gpuboard/<br />GPU/RST_GPU | xyz.openbmc\_project.<br />inventory.item.<br />GPU | GPU<br />Status[n]|00h – <br />ResetRequired|

### GPU Interface yaml
```
description: >
    Implement to provide GPU sensor attributes.
properties:
    - name: GPUStatus
      type: enum[self.Status]
      description: >
          The status of GPU
      default: "None"
      
enumerations:
    - name: Status
      description: >
          The type of status performed.
      values:
          - name: "None"
            description: >
                Nothing done.
          - name: "ResetRequired"
            description: >
                GPU reset Required.
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
| Entity Instance | Can be get from sensor name(n) |
| Sensor Type | Can be based on sensor interface |
| Event/Reading Type code | Can be based on sensor interface |
| Assertion Event Mask/Lower Threshold Reading Mask | can be get from available interfaces.<br />E.g: bitn - xyz.openbmc\_project.Inventory.Item.GPU |
| Deassertion Event Mask/Upper Threshold Reading Mask | can be get from available interfaces.<br />E.g: bitn - xyz.openbmc\_project.Inventory.Item.GPU |
| Discrete Reading Mask/Settable Threshold Mask, Readable Threshold Mask | can be get from available interfaces.<br />E.g: bitn - xyz.openbmc\_project.Inventory.Item.GPU |
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
| ID String Bytes | Can be get from sensor path. <br />(/xyz/openbmc\_project/<br />sensors/gpuboard/<br />gpu/RST_GPU) |

## Alternatives Considered

None

## Testing

- Unit tests to make sure the dbus-objects are converting to IPMI from phosphor-ipmi-host and working.
- Run time test to make sure sensor objects getting constructed and status properties updated as per GPU Reset_required status bit.
- Test to sensors and SDR, listing at part of IPMITool sensor list and SDR list without affecting sensor listing mechanism.
