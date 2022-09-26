# Event Logging Disbled Discrete Sensor Design

Author: Selvaganapathi M

Created: September 16, 2022

## Problem Description

Implements discrete sensor for Event Logging Disabled type for NVIDIA platforms

## Background and Reference

The openbmc project currently has a phosphor-ipmi-host repository to support the VR type discrete sensor. The intent of this design is to model and list Event Logging Disabled type discrete sensor.

### References

- Entity manager, Dbus-Sensor, Phosphor-logging
- VR Sensor - [https://gerrit.openbmc.org/c/openbmc/phosphor-host-ipmid/+/42776](https://gerrit.openbmc.org/c/openbmc/phosphor-host-ipmid/+/42776)

## Requirements

- Entity manager – JSON configuration
- Dbus-sensors – discrete sensor services to monitor phosphor-logging SEL full, cleared signal
- Phosphor-ipmi-host – Construct IPMI SDR from dbus objects and properties
- Phosphor-logging - Add signal to notify SEL full and cleared status

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
               |  +-------v--------+        +--------v-------+  |
+-----------+  |  |                |        |                |  |  +------------------+
|sysfs path <--+-->   Hwmon Temp   |. . . . |  System Sensor <--+--+ phosphor-logging |
+-----------+  |  |    Sensor      |        |                |  |  +------------------+
               |  +-------+--------+        +--------+-------+  |
               |          |                          |          |
               |          +-------------+------------+          |
               |                        |                       |
               |  +---------------------v--------------------+  |
               |  |                                          |  |
               |  |              Dbus-Objects                |  |
               |  |                                          |  |
               |  +---------------------+--------------------+  |
               |dbus-sensors            |                       |
               +------------------------+-----------------------+
                                        |
                                        |
                          +-------------v--------------+
                          |                            |
                          |Phosphor-ipmi-host(dbus-sdr)|
                          |                            |
                          +-------------^--------------+
                                        |                       .
                                        |
                               +--------v-----------+
                               |      IPMITool      |
                               +--------------------+
```
The new code would be creating new class under dbus-sensor package.

- Sensor configuration would be part of the entity-manager JSON configuration.
- Inventory objects which are created by entity-manager is used by SEL services to create 
  sensor objects and monitor signal.
- System sensor service will be using dbus signal monitor to get SEL status.
- Phosphor-ipmi-host package is responsible for collecting sensors based on interface and converting into IPMI specification.

### Brief Design Points of Sel Logging Disabled sensor

- Sensor configuration would be part of entity-manager JSON configuration file
- Entity Manager will be creating inventory dbus objects with properties from JSON configuration file. Interfaces are created by using type configuration in json
- almost full, full, cleared signal will be send by phosphor-logging service, whenever SEL usage size reaches 75% or 100% and SEL cleared.
- System sensor service will be traversing entity manager inventory dbus objects and segregating by interface (Type) and constructing sensor objects with SEL interface
- It will monitor SEL status signal, which is available in phosphor-logging service and state updated accordingly
- Phosphor-ipmi-host dbus-sdr will be getting sensor objects using &quot;getsubtree&quot; dbus method call
- The dbus Interfaces (SEL.interface.yaml) are matched and sdr properties and sensor status constructed respectively

### Phosphor-logging

- Phosphor-logging is the service to add SEL logs into d-bus objects
- SEL signal will be added as part of IPMI.interface.yaml
- CreateEntry method called to create SEL. When adding SEL log, used size compared with total size and if usage reaches 75 percent, partially full signal emitted
- If reaches 100 percent, full signal emitted
- EraseAll method used to clear the logs, When logs are cleared successfully, clear signal emitted
- Signal's are captured by SEL sensor and appropriate status will be updated

#### SEL Configuration
```
    {
        "Name": "SEL_FULLNESS",
        "EntityId": "0x06",
        "Type": "SEL"
    }
```
### Representation of dbus interfaces

| Sensor Name | Object | Interfaces | Property | Value |
| --- | --- | --- | --- | --- |
| SEL FULLNESS | xyz/openbmc\_project/<br />sensors/motherboard/<br />EventLogging/SEL\_FULLNESS | xyz.openbmc\_project.<br />inventory.item.<br />SEL | Status|02h – <br />Log Area Reset|
|||||04h – <br />SEL FULL|
|||||05h – <br />SEL Almost Full|

### SEL Signal
```
signals:
    - name: SEL
      description: >
          SEL event signal
      properties:
          - name: Event
            type: enum[self.Event]
            description: >
                The event emit during the SEL action

enumerations:
    - name: Event
      description: >
          The type of events.
      values:
          - name: "None"
            description: >
                nothing.
          - name: "Full"
            description: >
                SEL 100 percent Full.
          - name: "Partially"
            description: >
                SEL 75 percent full.
          - name: "Cleared"
            description: >
                SEL cleared succesfully.
```
### SEL Interface yaml
```
description: >
    Implement to provide SEL sensor attributes.
properties:
    - name: Status
      type: enum[self.Status]
      description: >
          The status of SEL
      default: "None"
      
enumerations:
    - name: Status
      description: >
          The type of status performed.
      values:
          - name: "LogCleared"
            description: >
                Log Area Reset/Cleared.
          - name: "SELFull"
            description: >
                SEL Area full.
          - name: "SELAlmostFull"
            description: >
                SEL Area 75 percent used.
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
| Assertion Event Mask/Lower Threshold Reading Mask | can be get from available interfaces.<br />E.g: bitn - xyz.openbmc\_project.Inventory.Item.SEL |
| Deassertion Event Mask/Upper Threshold Reading Mask | can be get from available interfaces.<br />E.g: bitn - xyz.openbmc\_project.Inventory.Item.SEL |
| Discrete Reading Mask/Settable Threshold Mask, Readable Threshold Mask | can be get from available interfaces.<br />E.g: bitn - xyz.openbmc\_project.Inventory.Item.SEL |
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
| ID String Bytes | Can be get from sensor path. <br />(/xyz/openbmc\_project/<br />sensors/motherboard/<br />EventLogging/SEL_FULLNESS) |

## Alternatives Considered

None

## Testing

- Unit tests to make sure the dbus-objects are converting to IPMI from phosphor-ipmi-host and working.
- Run time test to make sure sensor objects getting constructed and status properties updated as per phosphor-logging status signal.
- Test sensors and SDR, listing at part of IPMITool sensor list and SDR list without affecting sensor list mechanism.
