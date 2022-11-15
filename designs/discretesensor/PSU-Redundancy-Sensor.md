# PSU\_REDUNDANCY Discrete Sensor Design

Author: Selvaganapathi M

Created: Nov 14, 2022

## Problem Description

Implements PSU REDUNDANCY discrete sensor for NVIDIA platforms

## Background and reference

The openbmc project currently has a phosphor-ipmi-host repository to support the VR type discrete sensor. 
The intent of this design is to monitor PSU state and list PSU REDUNDANCY discrete sensor.

### References

- Entity manager, Dbus-Sensor, nvidia-power-manager
- VR Sensor - [https://gerrit.openbmc.org/c/openbmc/phosphor-host-ipmid/+/42776](https://gerrit.openbmc.org/c/openbmc/phosphor-host-ipmid/+/42776)
- OpenBMC power management-v74-20211208\_072943.pdf document


## Requirements

- Entity manager – JSON configuration
- Dbus-sensors – discrete sensor services to monitor PSU state changes
- Phosphor-ipmi-host – Construct IPMI SDR from dbus objects and properties

## Proposed Design

### Diagram:
```
+-----------------------------------------------------+          +--------------------+ 
|                                                     |          |      IPMITool      |
|  +--------------------+    +--------------------+   |          +----------^---------+
|  |                    |    |                    |   |                     |
|  | JSON Configuration +---->   Entity-Manager   |   |      +--------------V-------------+      
|  |                    |    |                    |   |      |                            |
|  +--------------------+    +----------^---------+   |      |Phosphor-ipmi-host(dbus-sdr)|
|Entity-Manager                         |             |      |                            |
+---------------------------------------+-------------+      +-------^--------------------+
                       Create Inventory |                            | Get sensor objects
                       Objects          |                            | using getsubtree
+---------------------------------------v----------------------------+---------+
|                                                                              |
|                                 D-Bus                                        |
|                                                                              |
+---------------------------------------^-----------------------------------^--+
                                        |                                   |
                             Set Sensor |                                   |
                             Status     |              D-bus signal for     |
                          +-------------+------------+ PSU state change     |
                          |                          | from power-manager   |
               +----------+--------------------------+----------+           |Set PSU state
               |          |                          |          |           |
               |  +-------v--------+        +--------v-------+  |           |
+-----------+  |  |                |        |                |  |  +--------+---------+
|sysfs path <--+-->   HwmonTemp    |. . . . | psuRedundancy  |  |  |                  |
+-----------+  |  |    Sensor      |        |                |  |  |    Nvidia-       |
               |  +----------------+        +----------------+  |  |  Power-Manager   |
               |                                                |  |                  |
               |dbus-sensors                                    |  +------------------+
               +------------------------------------------------+
```
The new code would be creating new services under dbus-sensor package.

- Sensor configuration would be part of the entity-manager JSON configuration.
- Inventory objects which are created by entity-manager is used by dbus-sensor services to create 
  sensor objects and monitor.
- Dbus-sensor service will be using properties changed signal to get PSU Status to assert PSU redundancy state.
- Phosphor-ipmi-host package is responsible for collecting sensors based on interface and converting into IPMI specification.

### Brief Design Points of PSU Redundancy sensor

- Sensor Configuration would be part of entity-manager JSON configuration file
- Entity Manager will be creating inventory dbus objects with properties from JSON configuration file. Interfaces 
are created by using Type configuration in json
- psuredundancy service will be traversing entity manager inventory dbus objects and segregating by interface (Type) and 
creating sensor objects with PowerSupplyRedundancy interface
  - Once Dbus object created, it will create propertieschanged signal monitor to powersupply status sensor interfaces
xyz.openbmc\_project.Inventory.Item(Present property), xyz.openbmc\_project.State.Decorator.PowerState(PowerState property)
under /xyz/openbmc\_project/inventory/system/chassis/motherboard/powersupplyn objects
  - When powersupply sensor status changes to not present or poweroff, PSU redundunt count compared with workable PSU 
  - If workable PSU less than redundunt count, redundancy lost
  - If workable PSU count greater than redundant count, redundancy regained or fully redundant
- Phosphor-ipmi-host dbus-sdr will be getting sensor objects using &quot;getsubtree&quot; dbus method call
- The dbus Interfaces are matched and sdr properties and sensor status constructed respectively

| Active PSU | Redundancy State |
| --- | --- | 
| 6PSUs | Fully Redundant |
| 5PSUs | Redundancy Degraded |
| 4PSUs | Redundancy Degraded |
| 3PSUs | Redundancy Lost |
| <3PSUs | Redundancy Lost |

#### PSU\_REDUNDANCY Configuration
```
    {
        "Name": "PSU_REDUNDANCY",
        "EntityId": "0x13",
	"TotalPSU": 6,
	"RedundantPSU": 3,
        "Type": "PSUredundancy"
    }
```
### Representation of dbus interfaces

| Sensor Name | Object | Interfaces | Property | Value |
| --- | --- | --- | --- | --- |
| PSU_<br />REDUNDANCY | xyz/<br />openbmc\_project/<br />sensors/power/<br />PSU/PSU_REDUNDANCY | xyz.openbmc\_projcet.<br />Control.<br />PowerSupply<br />Redundancy | Status | Full - <br />Fully redundant(00h) |
|||||Degraded - <br />Redundancy degraded(02h)|
|||||Failed – <br />Redundancy lost(01h)|

### PSU\_REDUNDANCY Interface yaml
```
properties:
    - name: Status
      type: enum[self.State]
      flags:
          - const
      description: >
          The current redundancy status.
      
enumerations:
    - name: State
      description: >
          Possible redundancy states.
      values:
          - name: Full
            description: >
                PSU is in a fully redundant state.
          - name: Degraded
            description: >
                PSU is in a degraded state.
          - name: Failed
            description: >
                PSU has lost redundancy.
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
| Assertion Event Mask/Lower Threshold Reading Mask | can be get from available interfaces.<br />E.g: bitn - xyz.openbmc\_project.Control.PowerSupplyRedundancy |
| Deassertion Event Mask/Upper Threshold Reading Mask | can be get from available interfaces.<br />E.g: bitn - xyz.openbmc\_project.Control.PowerSupplyRedundancy |
| Discrete Reading Mask/Settable Threshold Mask, Readable Threshold Mask | can be get from available interfaces.<br />E.g: bitn - xyz.openbmc\_project.Control.PowerSupplyRedundancy |
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
| ID String Bytes | Can be get from sensor path. <br />(/xyz/openbmc\_project/<br />sensors/power/<br />PSU/PSU\_REDUNDANCY) |

## Alternatives Considered

None

## Testing

- Unit tests to make sure the dbus-objects are converting to IPMI from phosphor-ipmi-host and working.
- Run time test to make sure sensor objects getting constructed and status properties updated as per PSU\_REDUNDANCY status bit.
- Test to sensors and SDR, listing at part of IPMITool sensor list and SDR list without affecting sensor listing mechanism.
