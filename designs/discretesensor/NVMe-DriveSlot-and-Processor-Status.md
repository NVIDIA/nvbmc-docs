# NVMe Drive Slot and Processor Discrete Sensor Design

Author: Selvaganapathi M

Created: July 13, 2022

## Problem Description

Implements discrete sensor for NVMe Drive Slot status and Processor status for NVDIA platforms

## Background and reference

The openbmc project currently has a phosphor-ipmi-host repository to support the VR type discrete sensor. The intent of this design is to monitor and list Drive slot, Processor type discrete sensor.

### References

- Drive Slot interface – [https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc\_project/Inventory/Item/Drive.interface.yaml](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Inventory/Item/Drive.interface.yaml)
- CPU interface - [https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc\_project/Inventory/Item/Cpu.interface.yaml](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Inventory/Item/Cpu.interface.yaml)
- Entity manager, Dbus-Sensor
- VR Sensor - [https://gerrit.openbmc.org/c/openbmc/phosphor-host-ipmid/+/42776](https://gerrit.openbmc.org/c/openbmc/phosphor-host-ipmid/+/42776)

## Requirements

- Entity manager – JSON configuration
- Dbus-sensors – discrete sensor services to monitor device status
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
+-----------+  |  |                |        |                |  |  +-----------+
|I2C driver <--+-->   Drive Sensor |. . . . |   CPU Sensor   <--+--+  LibGpiod |
+-----^-----+  |  |                |        |                |  |  +-----^-----+
      |        |  +-------+--------+        +--------+-------+  |        |
      |        |          |                          |          |        |
+-----v-----+  |          +-------------+------------+          |  +-----v-----+
|CPLD/NVMeMI|  |                        |                       |  |   GPIO    |
+-----------+  |  +---------------------v--------------------+  |  +-----------+
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

- Sensor configuration (Bus, Slave Address, GPIO) would be part of the entity-manager JSON configuration.
- Inventory objects which are created by entity-manager is used by Drive and Cpu services to create 
  sensor objects and monitor status.
- Drive and Cpu service will be using system calls or libraries to read sensor status.
- Phosphor-ipmi-host package is responsible for collecting sensors based on interface and converting into IPMI specification.

### Brief Design Points of Drive and Processor sensor

- Sensor Configuration would be part of entity-manager JSON configuration file.
- Entity Manager will be creating inventory dbus objects with properties from JSON configuration file. Interfaces are created by using Type configuration in json.
- Drivestatus sensor and Cpustatus sensor service will be traversing entity manager inventory dbus objects and segregating by interface (Type) and creating sensor objects with drive interface and cpu interface respectively.
- Both services start&#39;s monitoring by one of the methods, either event based, or poll (Timer) based.
  - &quot;Drive presence&quot; status of Nvme U2 is available in CPLD (0x3C) register (0x02)
  - bit0 – bitn represents &quot;Drive Presence&quot; respectively
  - status of NVMe M.2 is available via NVMe Basic management command over I2C(Address: 0xD4) from M.2 Drive.
    Refer NVM-Express-Management-Interface-1.1-Ratified.pdf.
  - If NVMe MI basic management command to read block of data from 0th offset success, &quot;Drive presence&quot; detected
  - If bit[5] of data[1] is not set, &quot;Drive Fault&quot; occured
  - If all bits of data[2] is not set, &quot;Predictive failure&quot; occured
  - &quot;Processor Presence Detected&quot; status of processor based on GPIO events

| Register Address (HEX) | bit 7 | bit 6 | bit 5 | bit 4 | bit 3 | bit 2 | bit 1 | bit 0 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 02 | SSD7 IFDET | SSD6 IFDET | SSD5 IFDET | SSD4 IFDET | SSD3 IFDET | SSD2 IFDET | SSD1 IFDET | SSD0 IFDET |

| Data[1]Bit[5] | Not Set | Drive Fault |
| Data[2] | All bit not set | Predictive Failure|

| GPIOI0 | CPU1\_PRESENT\_N | Active-low | CPU1 present |
| GPIOI1 | CPU2\_BMC\_PRESENT\_N | Active-low | CPU2 present |

- Phosphor-ipmi-host dbus-sdr will be getting sensor objects using &quot;getsubtree&quot; dbus method call.
- The dbus Interfaces (Drive.interface.yaml, Cpu.interface.yaml) are matched and sdr properties and sensor status constructed respectively.

#### Status U2 n
```
    {
        "Name": "Status u2",
        "Bus": 2,
        "Address": "0x3c",
        "Register": "0x02",
        "EntityId": "0x1a",
        "Type": "Nvmecpld"
    }
```
#### Status M2 n
```
    {
        "Name": "STATUS_M2_0",
        "Bus": 214,
        "Address": "0x6a",
        "EntityId": 4,
        "Type": "Nvmem2"
    }
```
#### Status CPUn
```
    {
        "Name": "Status cpu0",
        "GpioName": "cpu1_present_n",
        "Polarity": "Low",
        "EntityId": 3,
        "Type": "Cpustatus"
    }
```
### Representation of dbus interfaces

| Sensor Name | Object | Interfaces | Property | IPMI Offset |
| --- | --- | --- | --- | --- |
| STATUS\_U.2\_n| xyz/openbmc\_project/<br />sensors/motherboard/<br />drive/status\_u2\_n | xyz.openbmc\_project.<br />inventory.item.<br />Drive |
 ||| xyz.openbmc\_project.<br />inventory.item | Present | 00h – <br />Drive presence |
 |||| Pretty<br />name | - |
| STATUS\_M.2\_n| xyz/openbmc\_project/<br />sensors/motherboard/<br />drive/STATUS\_M2\_n | xyz.openbmc\_project.<br />inventory.item.<br />Drive |
 ||| xyz.openbmc\_project.<br />inventory.item | Present | 00h – <br />Drive presence |
 |||| Pretty<br />name | - |
 ||| xyz.openbmc\_project.<br />State.Decorator.<br />OperationalStatus | Fault | 01h - <br />Drive Fault |
 |||| Functional | 02h - <br />Predictive <br /> Failure | 
| STATUS\_CPUn| xyz/openbmc\_project/<br />sensors/motherboard/<br />cpu/status\_cpun | xyz.openbmc\_project.<br />inventory.item.<br />cpu |
 ||| xyz.openbmc.project.<br />inevtory.item | Present | 07h – Processor presence detected |
 |||| Pretty<br />name | - |

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
| Entity Instance | Can be get from sensor name(n). <br />E. g: status\_cpu **0**...status\_cpu **n** |
| Sensor Type | Can be based on sensor interface |
| Event/Reading Type code | Can be based on sensor interface |
| Assertion Event Mask/Lower Threshold Reading Mask | can be get from available interfaces.<br />E.g: bit0 - xyz.openbmc\_project.Inventory.Item |
| Deassertion Event Mask/Upper Threshold Reading Mask | can be get from available interfaces.<br />E.g: bit0 - xyz.openbmc\_project.Inventory.Item |
| Discrete Reading Mask/Settable Threshold Mask, Readable Threshold Mask | can be get from available interfaces.<br />E.g: bit0 - xyz.openbmc\_project.Inventory.Item |
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
| ID String Bytes | Can be get from sensor path. <br />(/xyz/openbmc\_project/<br />sensors/motherboard/<br />cpu/status\_cpu0) |

## Alternatives Considered

None

## Testing

- Unit tests to make sure the dbus-objects are converting to IPMI from phosphor-ipmi-host and working.
- Run time test to make sure sensor objects getting created and status properties updated as per CPLD register value and GPIO value.
- Test to sensors and SDR, listing at part of IPMITool sensor list and SDR list without affecting sensor listing mechanism.
