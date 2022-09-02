# Watchdog2 Discrete Sensor Design

Author: Selvaganapathi M

Created: Augest 29, 2022

## Problem Description

Implements discrete sensor for Watchdog2 type for NVIDIA platforms

## Background and reference

The openbmc project currently has a phosphor-ipmi-host repository to support the VR type discrete sensor. The intent of this design is to monitor and list Watchdog2 type discrete sensor.

### References

- Watchdog interface - [https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/State/Watchdog.interface.yaml](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/State/Watchdog.interface.yaml)
- Entity manager, Dbus-Sensor, Phosphor-watchdog
- VR Sensor - [https://gerrit.openbmc.org/c/openbmc/phosphor-host-ipmid/+/42776](https://gerrit.openbmc.org/c/openbmc/phosphor-host-ipmid/+/42776)

## Requirements

- Entity manager – JSON configuration
- Dbus-sensors – discrete sensor services to monitor phosphor-watchdog timeout action signal
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
               |  +-------v--------+        +--------v-------+  |
+-----------+  |  |                |        |                |  |  +------------------+
|sysfs path <--+-->   Hwmon Temp   |. . . . |  System sensor <--+--+ phosphor-watchdog|
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
The new code would be creating new services under dbus-sensor package.

- Sensor configuration would be part of the entity-manager JSON configuration.
- Inventory objects which are created by entity-manager is used by watchdog services to create 
  sensor objects and monitor signal.
- System sensor service will be using dbus signal monitor to get watchdog action status.
- Phosphor-ipmi-host package is responsible for collecting sensors based on interface and converting into IPMI specification.

### Brief Design Points of Drive and Processor sensor

- Sensor Configuration would be part of entity-manager JSON configuration file
- Entity Manager will be creating inventory dbus objects with properties from JSON configuration file. Interfaces are created by using Type configuration in json
- System sensor service will be traversing entity manager inventory dbus objects and segregating by interface (Type) and creating sensor objects with watchdog interface
- It will monitor Timeout signal, which is available in phosphor-watchdog service and state updated accordingly
- Phosphor-ipmi-host dbus-sdr will be getting sensor objects using &quot;getsubtree&quot; dbus method call
- The dbus Interfaces (Watchdog.interface.yaml) are matched and sdr properties and sensor status constructed respectively

#### Watchdog2
```
    {
        "Name": "WATCHDOG2",
        "EntityId": "0x06",
        "Type": "watchdog"
    }
```
### Representation of dbus interfaces

| Sensor Name | Object | Interfaces | Property | Value |
| --- | --- | --- | --- | --- |
| WATCHDOG2| xyz/openbmc\_project/<br />sensors/motherboard/<br />watchdog/watchdog2 | xyz.openbmc\_project.<br />inventory.item.<br />watchdog | Status|00h – <br />TimerExpired|
|||||01h – <br />HardReset|
|||||02h – <br />PowerOff|
|||||03h – <br />PowerCycle|
|||||04h – <br />TimerInterrupt|

### Watchdog Interface yaml
```
description: >
    Implement to provide watchdog sensor attributes.
properties:
    - name: Status
      type: enum[self.Status]
      description: >
          The status of watchdog
      default: "None"
      
enumerations:
    - name: Status
      description: >
          The type of status performed.
      values:
          - name: "None"
            description: >
                Nothing done.
          - name: "TimerExpired"
            description: >
                Timer expired but no action performed.
          - name: "HardReset"
            description: >
                System reset performed.
          - name: "PowerOff"
            description: >
                System power off performed.
          - name: "PowerCycle"
            description: >
                System power cycle performed.
          - name: "TimerInterrup"
            description: >
                Timer interrupted.
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
| Assertion Event Mask/Lower Threshold Reading Mask | can be get from available interfaces.<br />E.g: bitn - xyz.openbmc\_project.Inventory.Item.Watchdog |
| Deassertion Event Mask/Upper Threshold Reading Mask | can be get from available interfaces.<br />E.g: bitn - xyz.openbmc\_project.Inventory.Item.Watchdog |
| Discrete Reading Mask/Settable Threshold Mask, Readable Threshold Mask | can be get from available interfaces.<br />E.g: bitn - xyz.openbmc\_project.Inventory.Item.Watchdog |
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
| ID String Bytes | Can be get from sensor path. <br />(/xyz/openbmc\_project/<br />sensors/motherboard/<br />watchdog/watchdog2) |

## Alternatives Considered

None

## Testing

- Unit tests to make sure the dbus-objects are converting to IPMI from phosphor-ipmi-host and working.
- Run time test to make sure sensor objects getting constructed and status properties updated as per phosphor-watchdog timeout signal.
- Test to sensors and SDR, listing at part of IPMITool sensor list and SDR list without affecting sensor listing mechanism.
