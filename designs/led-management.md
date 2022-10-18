# LED Design and Porting Guide

Author	: Krishna Raj
Other contributors: Vinodhini J

## References

1. https://github.com/openbmc/docs/blob/master/architecture/LED-architecture.md
2. https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Led/README.md
3. https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Led/Group.interface.yaml
4. https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Led/Physical.interface.yaml


## Requirements

The document describe how to monitor and enable LED management in OpenBMC .

## Proposed Design

The openbmc project currently has a phosphor-led-sysfs and phosphor-led-manager repository to support the LED.


### Brief Design Points of Phosphor-led-sysfs

phosphor-led-sysfs application, design exposes one service per LED configuration
in device tree.
In kernel dts, LEDs shall be described, example 

```
  leds {
    compatible = "gpio-leds";

    boot{
      gpios = <&gpio ASPEED_GPIO(N, 2) GPIO_ACTIVE_LOW>;
    };

    LED2 {
      gpios = <&gpio ASPEED_GPIO(N, 4) GPIO_ACTIVE_HIGH>;
    };

    LEDN {
      gpios = <&gpio ASPEED_GPIO(R, 5) GPIO_ACTIVE_LOW>;
    };
  };
```
#### Diagram:

```
   KERNEL                                           PHOSPHOR-LED-SYSFS SERVICE

   +------------+  Event 1   +----------+  boot    +-----------------------------+
   |            |----------->|          |-------->|                             |
   |    boot    |  Event 2   |          |  LED 2  | Service 1 :                 |
   |            |----------->|   UDEV   |-------->|   xyz.openbmc_project.      |
   |    LED 2   |   .....    |  Events  |  ....   |        Led.Controller.boot  |
   |            |   .....    |          |  ....   | Service 2 :                 |
   |    LED 3   |  Event N   |          |  LED N  |   xyz.openbmc_project.      |
   |            |----------->|          |-------->|        Led.Controller.LED2  |
   |    LED 4   |            +----------+         |          .......            |
   |            |                                 |          .......            |
   |  .......   |                                 | Service N :                 |
   |  .......   |                                 |   xyz.openbmc_project.      |
   |  .......   |                                 |        Led.Controller.LEDN  |
   |            |                                 +-----------------------------+
   |    LED N   | 
   +------------+ 
   
```
phosphor-led-sysfs uses sysfs entry as udev events for particular LED and
triggers the necessary action to generate the dbus object path and interface
for that specified service. It exposes one service per LED.

#### Dbus-objects

a D-bus service "xyz.openbmc_project.led.controller@.service" with object paths for each sensor: "/xyz/openbmc_project/led/physical/ledN"

```
     busctl tree xyz.openbmc_project.LED.Controller.boot
     `-/xyz
       `-/xyz/openbmc_project
         `-/xyz/openbmc_project/led
           `-/xyz/openbmc_project/led/physical
             `-/xyz/openbmc_project/led/physical/boot

     busctl tree xyz.openbmc_project.LED.Controller.led2
     `-/xyz
       `-/xyz/openbmc_project
         `-/xyz/openbmc_project/led
           `-/xyz/openbmc_project/led/physical
             `-/xyz/openbmc_project/led/physical/led2

      
```

### Brief Design Points of Phosphor-led-manager

"led manager" is responsible translating group state to physical LED state. It provides dbus objects for each group to allow them to be asserted / deasserted.

#### led configuration file
create led.yaml file under "led-manager-config". This is specific to plateform.
In machine layer, LEDs needs to be configured via yaml to describe how it functions.

example

bmc_booted:
    boot:
        Action: 'Blink'
        DutyOn: 50
        Period: 1000
        Priority: 'On'

This configuration file here sets the LED blinking cycle to 1s after the bmc starts.

1.There are total three "Action".
 "Off", "On", and "Blink", which are off, on, and flashing. Only when Action is "Blink", "DutyOn" and "Period" will take effect. And there will be "delay_off" and "delay_on" files created under leds in the file system.


2.At runtime, LED manager automatically set LEDs on/off/blink based on the above yaml config.


```
┌────────────────────────┐        ┌─────────────────────────┐
│                        │        │                         │
│                        │        │                         │
│phosphor-led-sysfs      │◄───────┤  phosphor-led-manager   │
│     (low level)        │        │     (high level)        │
│                        │        │                         │
│                        │        │                         │
└────────────────────────┘        └─────────────────────────┘

```
LED actions are triggered by phosphor-led-manager and phosphor-led-sysfs will be responsible for controling 
Action.



## Testing

The following instructions can be used to verify if the LED management is working good or not.

### using DBUS

Run the dbus commands in OpenBMC’s console (either serial or ssh). 
By setting and getting "Asserted" propert of interface xyz.openbmc_project.Led.Group , LED action 
can be tested

example:-
1.BMC boot
  Get the bmc_booted group Asserted property 
  
   root@dgx-a100-dp:~# busctl get-property xyz.openbmc_project.LED.GroupManager      /xyz/openbmc_project/led/groups/bmc_booted xyz.openbmc_project.Led.Group "Asserted"
b true









