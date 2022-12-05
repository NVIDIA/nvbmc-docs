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
- Reset status interface - [https://gitlab-collab-01.nvidia.com/viking-team/phosphor-dbus-interfaces/-/blob/develop/yaml/xyz/openbmc_project/State/ResetStatus.interface.yaml](https://gitlab-collab-01.nvidia.com/viking-team/phosphor-dbus-interfaces/-/blob/develop/yaml/xyz/openbmc_project/State/ResetStatus.interface.yaml)
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
               |          |                          |          | D-bus signal
               |  +-------v--------+        +--------v-------+  |  +------------------+
+-----------+  |  |                |        |                |  |  |                  |
|sysfs path <--+-->   HwmonTemp    |. . . . |   gpuStatus    <--+--+     gpumgrd      |
+-----------+  |  |    Sensor      |        |                |  |  |                  |
               |  +-------+--------+        +--------+-------+  |  |                  |
               |          |                          |          |  +--^---------------+
               |          +-------------+------------+          |     |
               |                        |                       |     |
               |  +---------------------v--------------------+  |     | SMBPBI(I2C) Interface
               |  |                                          |  |  +--+---------------+
               |  |              Dbus-Objects                |  |  |  |   +-------+   |
               |  |                                          |  |  |  |   | GPU1  |   |
               |  +---------------------+--------------------+  |  |  +---+       |   |
               |dbus-sensors            |                       |  |  |   +-------+   |
               +------------------------+-----------------------+  |  |               |
                                        |                          |  |   +-------+   |
                                        |                          |  |   | GPU2  |   |
                          +-------------v--------------+           |  +---+       |   |
                          |                            |           |  |   +-------+   |
                          |Phosphor-ipmi-host(dbus-sdr)|           |  |               |
                          |                            |           |  |   +-------+   |
                          +-------------^--------------+           |  |   | GPUn  |   |
                                        |                          |  +---+       |   |
                                        |                          |      +-------+   |
                               +--------v-----------+              | Baseboard        |
                               |      IPMITool      |              +------------------+
                               +--------------------+              

```
The new code would be creating new services under dbus-sensor package.

- Sensor configuration would be part of the entity-manager JSON configuration.
- Inventory objects which are created by entity-manager is used by dbus-sensor services to create 
  sensor objects and monitor.
- Dbus-sensor service will be using dbus signal monitor to get GPU status.
- Phosphor-ipmi-host package is responsible for collecting sensors based on interface and converting into IPMI specification.

### Brief Design Points of GPU OEM sensor

- Sensor Configuration would be part of entity-manager JSON configuration file
- Entity Manager will be creating inventory dbus objects with properties from JSON configuration file. Interfaces are created by using Type configuration in json
- GPU status service will be traversing entity manager inventory dbus objects and segregating by interface (Type) and creating sensor objects with OEM reset GPU interface
- gpustatus service will be looking for properties changed signal from dbus on ResetRequired property under xyz.openbmc_projec.
State.ResetStatus interface
- Based on the ResetRequired property boolean value(true/false)change, status(GPUStatus[n]) property gets updated
- Phosphor-ipmi-host dbus-sdr will be getting sensor objects using &quot;getsubtree&quot; dbus method call
- The dbus interfaces are matched and sdr properties and sensor status constructed respectively

#### RST_GPU Configuration
```
    {
        "Name": "RST_GPU",
        "EntityId": "0x06",
        "Type": "rstgpu",
        "TotalGpu": 4
    }
```

### GPU Manager ResetStatus interface
```
description: >
   Implement to provide GPU Reset Requirements.

properties:
    - name: ResetRequired
      type: boolean
      description: >
            Represents Reset Requirement.
flags:
          - readonly
    - name: DrainAndResetRequired
      type: boolean
      description: >
            Represents Drain and Reset Requirement.
flags:
          - readonly
```

### Representation of sensor dbus interfaces

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
