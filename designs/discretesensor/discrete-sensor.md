# PSU-discrete-sensor:
Author: Selvaganapathim, Vinodhinij
# Block diagram:

```
+-------------------+   +-------------------+   +-------------------+
|                   |   |                   |   |                   |
|     CPLD Daemon   |   |     GPIO Daemon   |   |      VR Daemon    |
|                   |   |                   |   |                   |
+---------^---------+   +---------^---------+   +---------^---------+
          |                       |                       |
          |                       |                       | Updates property based on hardware
          |             +---------v---------+             |
          +------------->                   <-------------+
                        | D-Bus Objects     |
          +------------->                   |
          |             +---------^---------+
+---------+---------+             | retrive discrete sensors
|                   |   +---------v---------+   +-------------------+
|  Inventory App    |   |phosphor-ipmi-host |   |                   |
|                   |   +---------+         <--->  IPMItool         |
+-------------------+   |dbus-sdr |         |   |                   |
                        +---------+---------+   +-------------------+
```

# Brief:
- GPIO, CPLD and VR are examples of services fetching sensor status from hardware. These values are mapped & updated to the corresponding d-bus properties
- All d-bus sensors are aggregated by phosphor-ipmi-host:

    - **Discrete sensor:** Sensors are filtered using specific interfaces and sdr info (Sensor type, Entity_ID, Sensor name) constructed from object paths and interfaces.
    Sensor states are determined by converting appropriate dbus properties into event offsets
    - **Threshold sensor:** Threshold sensors are configured dynamically, with a specific formatted sensor object path (xyz/openbmc_project/sensors/\<sensortype\>/\<sensorname\>) and interfaces "xyz.openbmc_project.Sensor.Value", "xyz.openbmc_project.Sensor.Threshold.Warning",
    "xyz.openbmc_project.Sensor.Threshold.Critical".
    If sensor value interface matches, sensor info and reading is returned as response

# Phosphor-ipmi-host:

- Phosphor-ipmi-host is the service responsible for IPMI command handling
- Sensor command handlers are implemented for dynamically constructing sdr from sensor object paths
- Discrete sensors are filtered by d-bus interface, specific to sensor type
- Getsdr command handler filters discrete sensor by dbus interface and constructs sdr from object path and interfaces, for example sensor name, sensor type, Entity ID
- Get Event status command filters discrete sensor by dbus interface and asserts corresponding bits in the discrete event byte based on property

# An Illustration - PSU sensor:

- Psu(n) sensor object path, interface and properties will be created and default initialized by inventory application. Psu\_drop\_to\_2, Psu\_drop\_to\_1 and mbon\_gboff object paths, interface will be created and updated by psu monitor application
- Psu(n) sensor object filtered using "powersupply" interface and properties updated by psu monitor application based on sensor status
- Phosphor-host-ipmi service filters psu sensor using "powersupply" interface and
constructs sdrinfo(sensor type, event type, sensor name, entity_Id..) and returns response for ipmitool getsdr command
- Same as above, sensors appropriate state bits are asserted from state
interfaces(item.inventory, powerstate, operational status) and response returned
for getevent status, get sensor reading command's

**Representation of sensor as dbus objects:**
```
                                          +--------------------------+
                                      +-->|xyz.openbmc_project.      |
                                      |   |Inventory.Item.PowerSupply|
                                      |   +--------------------------+       +----------+
                                      |                                  +-->|PrettyName|
                                      |    +---------------------+       |   +----------+
                                      +----> xyz.openbmc_project.+-------+
                                      |    | Inventory.Item      |       |
            +---------------------+   |    +---------------------+       |   +-------+
+-------+   |xyz/openbmc_project/ |   |                                  +-->|Present|
| Psu(n)+---+inventory/power/     +---+                                      +-------+
+-------+   |power_supply/psu(n)  |   |
            +---------------------+   |    +---------------------------+     +----------+
                                      +--->| xyz.openbmc_project.State.+---->|PowerState|
                                      |    | Decorator.PowerState      |     +----------+
                                      |    +---------------------------+
                                      |
                                      |
                                      |  +---------------------------+      +-----------+
                                      +->|xyz.openbmc_project.State. +----->|Functional |
                                         |Decorator.OperationalStatus|      +-----------+
                                         +---------------------------+
                                         
                     +--------------------------+    +--------------------+   +-------+
+---------------+    |xyz/openbmc_project/      |    |xyz.openbmc_project.+-->|Enabled|
| Psu_drop_to_2 +--->| sensors/power/           +--->|   Object.Enable    |   +-------+
+---------------+    | psu_drop_to_2            |    +--------------------+
                     +--------------------------+

                     +--------------------------+    +--------------------+   +-------+
+---------------+    |xyz/openbmc_project/      |    |xyz.openbmc_project.+-->|Enabled|
| Psu_drop_to_1 +--->| sensors/power/           +--->|   Object.Enable    |   +-------+
+---------------+    | psu_drop_to_1            |    +--------------------+
                     +--------------------------+

                     +-----------------------+    +--------------------+   +-------+
+---------------+    |xyz/openbmc_project/   |    |xyz.openbmc_project.+-->|Enabled|
| mbon_gboff    +--->| sensors/power/        +--->|   Object.Enable    |   +-------+
+---------------+    | mbon_gboff            |    +--------------------+
                     +-----------------------+
```
The below interfaces and properties are used to filter Powersupply discrete sensor and assert bits of the discrete event byte

xyz.openbmc\_project.Inventory.Item.PowerSupply([https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Inventory/Item/PowerSupply.interface.yaml](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Inventory/Item/PowerSupply.interface.yaml)) 
    
xyz. openbmc\_project. Inventory.Item ([https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc\_project/Inventory/Item.interface.yaml](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Inventory/Item.interface.yaml))

xyz. openbmc\_project.State.Decorator.PowerState ( [https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc\_project/State/Decorator/PowerState.interface.yaml](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/State/Decorator/PowerState.interface.yaml))

xyz.openbmc\_project.State.Decorator.OperationalStatus ([https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc\_project/State/Decorator/OperationalStatus.interface.yaml](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/State/Decorator/OperationalStatus.interface.yaml))
    
xyz.openbmc_project.Object.Enable (https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Object/Enable.interface.yaml)

| **Sensor\_name** | **Object** | **interfaces** | **Properties** | **IPMI offset** |
| --- | ---| --- | --- | --- |
| psu(n) | xyx/openbmc\_project/<br />inventory/power/<br />power\_supply/psu(n)| xyz.openbmc\_projcet.<br />Inventory.Item.<br />PowerSupply |
 || | xyz.openbmc\_projcet.<br />Inventory.Item | Present | 00h - Presense detected |
 |||| PrettyName |-
||| xyz.openbmc\_project.<br />State.Decorator.<br />OperationalState | Functional | 01h - Power supply Failure detected
||| xyz.openbmc\_project.<br />State.Decorator.<br />PowerState | PowerState | 03h - Power supply input lost |
| psu<br />\_drop\_to\_2 | xyx/openbmc\_project/<br />sensors/power/<br />psu\_drop\_to\_2 | xyz.openbmc\_project.<br />Object.Enable | Enabled | - |
| psu<br />\_drop\_to\_1 | xyx/openbmc\_project/<br />sensors/power/<br />psu\_drop\_to\_1 |  xyz.openbmc\_project.<br />Object.Enable | Enabled | - |
| mbon\_gboff | xyx/openbmc\_project/<br />sensors/power/<br />mbon\_gboff | xyz.openbmc\_project.<br />Object.Enable | Enabled | - |

# Compact SDR mapping
| **Field Name** | **Mapping** |
| --- | --- |
| Record ID | Input argument from ipmitool |
| SDR Version | Hardcoded as per IPMI2.0 spec, version 0x51h |
| Record Type | Hardcoded as per spec, 0x02h - compact sdr |
| Record Length | Size of compactsensor |
| Sensor Owner ID | Hardcoded bmci2caddr - 0x20h |
| Sensor Owner LUN | sensorNum >> 8 |
| Sensor Number | getting sensor number from object path(sensorNumMapPtr) |
| Entity ID | Can be get from sensor path. E.g: 0x0Ah - power supply (/xyz/openbmc\_project/inventory/**power**/power\_supply/psu1)|
| Entity Instance | Can be get from sensorname(n). E.g: psu**1**..psu**n** |
| Sensor Type | Can be get from sensor path. E.g: 0x08h - powersupply (/xyz/openbmc\_project/inventory/power/**power\_supply**/psu1) |
| Event / Reading Type Code | Can be get from sensor path. E.g: 0x6Fh - sensor specific (/xyz/openbmc\_project/inventory/power/**power\_supply**/psu1)   |
| Assertion Event Mask / Lower Threshold Reading Mask | can be get from available interfaces.<br />E.g: bit0 - xyz.openbmc\_project.Inventory.Item,<br />bit1 - xyz.openbmc\_project.State.Decorator.OperationalState,<br />bit3 - xyz.openbmc_project.State.Decorator.PowerState |
| Deassertion Event Mask / Upper Threshold Reading Mask | can be get from available interfaces.<br />E.g: bit0 - xyz.openbmc\_project.Inventory.Item,<br />bit1 - xyz.openbmc\_project.State.Decorator.OperationalState,<br />bit3 - xyz.openbmc_project.State.Decorator.PowerState |
| Discrete Reading Mask / Settable Threshold Mask, Readable Threshold Mask | can be get from available interfaces.<br />E.g: bit0 - xyz.openbmc\_project.Inventory.Item,<br />bit1 - xyz.openbmc\_project.State.Decorator.OperationalState,<br />bit3 - xyz.openbmc_project.State.Decorator.PowerState |
| Sensor Units 1 | not required for discrete-0x00h |
| Sensor Units 2 - Base Unit | not required for discrete-0x00h |
| Sensor Units 3 - Modifier Unit | not required for discrete-0x00h |
| Positive-going Threshold Hysteresis value | not required for discrete-0x00h |
| Negative-going Threshold Hysteresis value | not required for discrete-0x00h |
| reserved | write as 0x00h |
| reserved | write as 0x00h |
| reserved | write as 0x00h |
| OEM | reserved for OEM use |
| ID String Type/Length Code | sizeof of sensor name |
| ID String Bytes | Can be get from sensor path. (/xyz/openbmc\_project/inventory/power/power\_supply/**psu1**) |
