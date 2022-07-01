
# PLX Sensor Design 

Author	: Krishna Raj, Vinodhini J
Other contributors: Selvaganapathi M
Created: June 29, 2022


## Requirements

This design will allow to generate a new PLX sensors.This implementation should provide
* a daemon(service) to create and monitor each sensor defined in json condiguration file and update them based on value change in each given I2C register.


## Proposed Design

### Block diagram

```
                                                       ┌─────────────┐          ┌────────────┐
                                                       │     PLX     │          │   ADC      │
                                                       │   Register  │          │  Driver    │
                                                       │             │          │            │
                                                       └─────▲───┬───┘          └─────▲──────┘
                                                             │I2C│Read/write          │
                                                  ┌──────────┼───┼────────────────────┼─────────────────┐
                                                  │          │   │                    │                 │
                                                  │  ┌───────┼───┼───────┐   ┌────────┼────────────┐    │
                                                  │  │       │   │       │   │        │            │    │
┌───────────────────┐                             │  │ ┌─────┴───▼───┐   │   │  ┌─────┴───────┐    │    │
│                   │     getsensorconfigs        │  │ │ monitor     │   │   │  │   monitor   │    │    │
│   Entity Manager  ├─────────────────────────────►  │ │             │   │   │  │             │    │    │
│                   │                             │  │ └──────▲─┬────┘   │   │  └──────▲──┬───┘    │    │
└────────▲──────────┘                             │  │ update │ │        │...│         │  │        │    │
         │                                        │  │property│ │        │   │         │  │        │    │
         │                                        │  │ ┌──────┴─▼────┐   │   │  ┌──────┴──▼───┐    │    │
         │                                        │  │ │    PLX      │   │   │  │   ADC       │    │    │
         │                    ┌────────────┐      │  │ │  Sensor     │   │   │  │  Sensor     │    │    │
┌────────┴────────┐    ┌──────┤   Sensor   │      │  │ └─────────────┘   │   │  └─────────────┘    │    │
│                 │    │      │Dbus-objects├──────┤  │                   │   │                     │    │
│       json      │    │  ┌───►            │      │  └───────────────────┘   └─────────────────────┘    │
│  configuration  │    │  │   └─────┬──▲───┘      │      PLX App                     ADC APP            │
└─────────────────┘    │  │         │  │Get sensor└─────────────────────────────────────────────────────┘
                       │  │         │  │ list                        Dbus-Sensors
                       │  │   ┌─────▼──┴────┐
                       │  │   │  Web UI /   │
                       │  │   │ redfish     │
              sensor list │   └─────────────┘
                       │  │
                       │  │
                       │  │
                  ┌────▼──┴───┐
                  │           │
                  │   ipmid   │
                  └───┬──▲────┘
                      │  │
                      │  │
                ┌─────▼──┴──────┐
                │  ipmitool     │
                │               │
                └───────────────┘
```

### Entity Manager 
The entity-manager process parses the json configuration files present in /usr/share/entity-manager/configurations/ it will have details for I2c Bus number and Register address. If probe successful , it uses those .json files to create system.json file in /var/configurations which is a combination of same platform specific .json file, then creates the sensor dbus objects.

JSON configuration file example,

```
        {
            "Address": "72",
            "Bus": "8",
            "Name": "TEMP_PLX_1",
            "Thresholds": [
                {
                    "Direction": "greater than",
                    "Name": "upper critical",
                    "Severity": 1,
                    "Value": 118.0
                },
                {
                    "Direction": "greater than",
                    "Name": "upper non critical",
                    "Severity": 0,
                    "Value": 115.0
                },
                {
                    "Direction": "less than",
                    "Name": "lower non critical",
                    "Severity": 0,
                    "Value": 5
                }
            ],
            "Type": "PLX"
        }
        
```



### PLX Sensor

Create a D-bus service "xyz.openbmc_project.plxtempsensor.service" with object paths for each sensor: "/xyz/openbmc_project/sensor/temperature/sensor_name"

```
root@dgx:~# busctl tree xyz.openbmc_project.PLXTempSensor
`-/xyz
  `-/xyz/openbmc_project
    `-/xyz/openbmc_project/sensors
      `-/xyz/openbmc_project/sensors/temperature
        |-/xyz/openbmc_project/sensors/temperature/TEMP_PLX_1
        |-/xyz/openbmc_project/sensors/temperature/TEMP_PLX_2
        |-/xyz/openbmc_project/sensors/temperature/TEMP_PLX_3
        `-/xyz/openbmc_project/sensors/temperature/TEMP_PLX_4

```

### Representation of dbus interfaces


|  Sensor Name  |                  Object                                | Interfaces                                     |    Property          |
|  --------     |                 --------                               | --------                                       |    --------          |
| TEMP_PLX_n    | /xyz/openbmc_project/sensors/temperature/TEMP_PLX_n    | xyz.openbmc_project.Sensor.Threshold.Critical  | CriticalAlarmHigh    |
|               |                                                        |                                                | CriticalHigh         |
|               |                                                        | xyz.openbmc_project.Sensor.Threshold.Warning   | WarningAlarmHigh     |
|               |                                                        |                                                | WarningAlarmLow      |
|               |                                                        |                                                | WarningHigh          |
|               |                                                        |                                                | WarningLow           |
|               |                                                        | xyz.openbmc_project.Sensor.Value               | MaxValue             |
|               |                                                        |                                                | MinValue             |
|               |                                                        |                                                | Unit                 |
|               |                                                        |                                                | Value                |
|               |                                                        |                                                |                      |

LIBI2C will be used to read and write PLX register values and update the reading value .polling mechanism  will be followed to monitor the register value changes. 

As per Luna Specification
There are 4 PLX SW on SW Board having slave address as 0x48 and 0x49 on bus number 7 & 9.

PLX Sensor with crossponding slave address and Bus number

   1.TEMP_PLX_1  (0x48/7)
   2.TEMP_PLX_2  (0x49/7)
   3.TEMP_PLX_3  (0x48/9)
   4.TEMP_PLX_4  (0x49/9)

Reading PLX sensors required 3 steps
        1. initialization
        2. setting register to read
        3. Getting temperature readings

using write/read system call can set and get  register value.

Function used for getting PLX reading are :-

1. hwInit()     :-  defination present in PLXTempSensor.cpp file ,
                    This function is used for initialization of PLX sensor's using Register's.

2. updateReading():- defination present in PLXTempSensor.cpp file ,
                    This function is used for monitoring the register value and update the reading .First it's check
                    if the valid bit is set(bit 16) or not of register MXS IPCtrl ADC Temperature Sensor 0 Status Register 0(offset 0xFFE7_8538),if valid bit set
                    then check reading value is signed or unsigned if signed then peform 2's complement and update the reading.

Reference documents :-  "How to Use Hardware I2C Slave in Atlas.pdf" for framing i2c packets and "Read Die Temperature in Atlas".

## Footnotes
1. https://github.com/openbmc/entity-manager/blob/master/README.md
2. https://github.com/openbmc/entity-manager/blob/master/docs/my_first_sensors.md
3. https://github.com/openbmc/dbus-sensors/blob/master/README.md
4. https://github.com/openbmc/docs/blob/master/architecture/ipmi-architecture.md

