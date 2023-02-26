# System firmware SEL logging sensor

Author: Adi Fogel

Created:  September 1, 2023

## Problem Description
Implements Boot Progress sensor for NVIDIA platforms.
The existing implementation of this sensor relies on a fixed sensor number when sending SEL events on the
sensor. This fixed sensor number may already be in use by other sensors, causing potential conflicts. Furthermore, 
the phosphor-Ipmi-Host require that all sensors must be dynamically allocated.

## Background and Reference
During the boot process of DPU UEFI, it sends the boot progress SEL event to a designated sensor number. The objective of
this proposal is to modify the designated sensor number to a dynamic sensor. This means that the UEFI will iterate through
the SDR list, dentify the specific record number based on the sensor type, and subsequently send the SEL event accordingly. 
The creation of the sensor list in phosphor-Impi-Host involves searching for objects that possess a specific interface 
within a predefined path. By introducing this new boot progress sensor, a new object will be generated and appended to the
sensor list. 
The sensor is presently a placeholder and does not contain any real-time data updates.

UEFI logging information: https://docs.nvidia.com/networking/display/BlueFieldDPUOSv392/Logging#heading-IPMILogginginUEFI

## Requirements
- Dbus-sensors - Create the sensor object
- Phosphor-ipmi-host – Construct IPMI SDR from D-Bus objects 

## Proposed Design

### Diagram:
```
 ┌───────────────────────────────────────────────────────────────────────────────────┐
 │   ┌───────────────────────┐                                                       │
 │   │  Dbus-sensors         │                                                       │
 │   │  ┌──────────────────┐ │                                                       │
 │   │  │  Dbus-Objects    │ │                                                       │
 │   │  │                  │ │                                                       │
 │   │  └───────────▲──────┘ │                                                       │  
 │   └──────────────┼────────┘                                                       │
 │                  │                                                                │
 │      ┌───────────┼──────────┐                                                     │   ┌──────────────────────────┐
 │      │     ┌─────┴─────┐    │                       IPMB                          │   │   ┌──────────────────┐   │
 │      │     │ SDR-List  ◄────┼─────────────────────────────────────────────────────┼───┼───┼─  UEFI Logging   │   │
 │      │     │           ┼────┼─────────────────────────────────────────────────────┼───┼───┤►                 │   │
 │      │     └───────────┘    │                                                     │   │   │                  │   │
 │      │     ┌───────────┐    │                       IPMB                          │   │   │                  │   │
 │      │     │SEL-Event ◄├────┼─────────────────────────────────────────────────────┼───┼───┼─                 │   │
 │      │     │           │    │                                                     │   │   │                  │   │
 │      │     └───────────┘    │                                                     │   │   └──────────────────┘   │
 │      │ Phosphor-Ipmi-host   │                                                     │   │  DPU                     │
 │ BMC  └──────────────────────┘                                                     │   └──────────────────────────┘
 └───────────────────────────────────────────────────────────────────────────────────┘                    

```
The new code would be creating new boot-progress service under dbus-sensor package.

- boot-progress services (under the D-bus sensors) will create sensor object under the path of:
   /xyz/openbmc_project/state/boot_progress/boot_progress_sensor
   and with BootProgress property under interface: xyz.openbmc_project.State.Boot.Progress 
- Phosphor-ipmi-host package is responsible for collecting sensors based on interface and converting into IPMI 
  specification. This process involves searching for an object that contains a particular interface, 
  such as "xyz.openbmc_project.State.Boot.Progress" within a designated path, 
  like "/xyz/openbmc_project/state/boot_progress/". 
  Once the relevant object is located, the package allocates a position in the Sensor Data Record (SDR) list. 
  The placement of this sensor in the list is determined by the order in which other sensors were previously 
  discovered.
- The SDR of this sensor will have the following information:
    -	Record Type = 0x03 (SENSOR_DATA_EVENT_RECORD)
    -	Owner ID = 0x01
    -	Owner LUN = 0x00
    -	Sensor Type = 0x0F (system Firmware Progress)
    -	Event / Reading Type Code = 0x6F (sensorSpecified)
- During the UEFI boot process, it will iterate through the sensor list in search of the boot progress sensor. 
Once found, it will retrieve the sensor number associated with the matching record and then transmit the SEL 
(System Event Log) boot event to that specific sensor number.
- This feature support a single DPU.

#### Changes needed
- Phosphor-ipmi-host - Add the sensor type and code and progress interface for sensor list
- Dbus-sensor - Add boot progress sensor

#### SEL events
- Sensor Event Type Codes: sensorSpecified = 0x6f
- Sensor type: system_firmware_progress = 0xF

SEL event are taken from ipmitool/include/ipmitool/ipmi_sel.h:
 /* System Firmware Progress */
 ```
    { 0x0f, 0x02, 0x00, "Unspecified" },
    { 0x0f, 0x02, 0x01, "Memory initialization" },
    { 0x0f, 0x02, 0x02, "Hard-disk initialization" },
    { 0x0f, 0x02, 0x03, "Secondary CPU Initialization" },
    { 0x0f, 0x02, 0x04, "User authentication" },
    { 0x0f, 0x02, 0x05, "User-initiated system setup" },
    { 0x0f, 0x02, 0x06, "USB resource configuration" },
    { 0x0f, 0x02, 0x07, "PCI resource configuration" },
    { 0x0f, 0x02, 0x08, "Option ROM initialization" },
    { 0x0f, 0x02, 0x09, "Video initialization" },
    { 0x0f, 0x02, 0x0a, "Cache initialization" },
    { 0x0f, 0x02, 0x0b, "SMBus initialization" },
    { 0x0f, 0x02, 0x0c, "Keyboard controller initialization" },
    { 0x0f, 0x02, 0x0d, "Management controller initialization" },
    { 0x0f, 0x02, 0x0e, "Docking station attachment" },
    { 0x0f, 0x02, 0x0f, "Enabling docking station" },
    { 0x0f, 0x02, 0x10, "Docking station ejection" },
    { 0x0f, 0x02, 0x11, "Disabling docking station" },
    { 0x0f, 0x02, 0x12, "Calling operating system wake-up vector" },
    { 0x0f, 0x02, 0x13, "System boot initiated" },
    { 0x0f, 0x02, 0x14, "Motherboard initialization" },
    { 0x0f, 0x02, 0x15, "reserved" },
    { 0x0f, 0x02, 0x16, "Floppy initialization" },
    { 0x0f, 0x02, 0x17, "Keyboard test" },
    { 0x0f, 0x02, 0x18, "Pointing device test" },
    { 0x0f, 0x02, 0x19, "Primary CPU initialization" },
    { 0x0f, 0x02, 0xff, "Unknown Progress" },
```
### Representation of dbus interfaces
```
root@dpu-bmc:~# busctl introspect xyz.openbmc_project.bootProgress /xyz/openbmc_project/sensors/boot_progress/BOOT_PROGRESS
NAME                                             TYPE      SIGNATURE RESULT/VALUE                             FLAGS
org.freedesktop.DBus.Introspectable              interface -         -                                        -
.Introspect                                      method    -         s                                        -
org.freedesktop.DBus.Peer                        interface -         -                                        -
.GetMachineId                                    method    -         s                                        -
.Ping                                            method    -         -                                        -
org.freedesktop.DBus.Properties                  interface -         -                                        -
.Get                                             method    ss        v                                        -
.GetAll                                          method    s         a{sv}                                    -
.Set                                             method    ssv       -                                        -
.PropertiesChanged                               signal    sa{sv}as  -                                        -
xyz.openbmc_project.State.Boot.Progress          interface -          -                                        -
.BootProgress                                    property  s          "Unspecified"                            emits-change writable
```

## Testing
- Run time test to make sure sensor objects getting constructed.
- Test sensors and SDR, listing at part of IPMITool sensor list and SDR list without affecting the sensor list mechanism.
- Create SEL event with the record number, making sure it parsed correctly.
