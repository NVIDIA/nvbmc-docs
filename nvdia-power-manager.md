# Nvidia power Manager

Author:
  hemanthkumarm@ami.com

Primary assignee:
  hemanthkumarm@ami.com


Created:
  February 1, 2022

## Problem Description
This is a proposal to provide a set of enhancements to the current BMC firmware 
as part of power management has the ability to monitor PSUs for faults and log 
errors and take evasive action on particular fault conditions.also enhance the
abiity to do power capping based on power policy mode.


## Requirements
The PSU status dbus sensors should be populated by the inventory application and
nvdia-gpu-manager should be available to take power capping action when there is
change in mode or power limit properties.

## Proposed Design
As part of fault monitoring power manager application will monitor the PSU status 
sensors and  take eavsive action to power of GPU when there is < 3 operating PSU.
below events will be loged as part of fault monitoring.
 1. System Power ON.
 2. Mode Change.
 3. Power Limit is different from previous value.
 4. System shutdown the GB or Chassis when PSU < 3.  
 as part of fault monitoring power manger will be monitoring  psu status sensors and
 trigger the gpio pin to intiate shutdown sequence if PSU count is less than 3.

 below sensors are sensors are monitored/updated by power manager app, inventory and sensor objects. will be created by “psu inventory app”.
 1. r_MBON_GBOFF_event
 2. w_PSU_pwrok_drop_to_2_event
 3. w_PSU_pwrok_drop_to_1_event
 4. powerGD  

 BMC is responsible for logging OEM sel events for use cases mentioned in OpenBMC power management-v74-20211208_072943.pdf as part of Power manager  using phosphor sel logging service.
 
┌──────────────┬─────────────────┬─────────────┬──────────────────────┐
│  Sensor Name │    Sensor Type  │Event offset │  Event description   │
├──────────────┼─────────────────┼─────────────┼──────────────────────┤
│              │                 │             │                      │
│ Nvdia Maxp/Q │    c9h          │     00h     │  Host power on MODE  │
│  sensor      │                 │             │                      │
│              │                 │             │                      │
├──────────────┼─────────────────┼─────────────┼──────────────────────┤
│              │                 │             │                      │
│              │                 │     01h     │     Mode change      │
│              │                 │             │                      │
├──────────────┼─────────────────┼─────────────┼──────────────────────┤
│              │                 │             │                      │
│              │                 │     02h     │  power limit change  │
│              │                 │             │                      │
├──────────────┼─────────────────┼─────────────┼──────────────────────┤
│              │                 │             │                      │
│              │                 │     03h     │  cpld shut down GPU  │
│              │                 │             │  for PSU=1           │
├──────────────┼─────────────────┼─────────────┼──────────────────────┤
│              │                 │             │                      │
│              │                 │     04h     │  cpld shutdown GPU   │
│              │                 │             │  for PSU<2 when host │
│              │                 │             │  is powered  up      │
├──────────────┼─────────────────┼─────────────┼──────────────────────┘
│              │                 │     05h     │  bmc graceful shutdown
│              │                 │             │  host when psu drops │
│              │                 │             │  to 2                │
├──────────────┼─────────────────┼─────────────┼──────────────────────┤
│              │                 │     06h     │  bmc force shutdown  │
│              │                 │             │  host when psu=1     │
├──────────────┼─────────────────┼─────────────┼──────────────────────┤
│              │                 │     07h     │  failed to set GPU   │
│              │                 │             │  power limit.        │
└──────────────┴─────────────────┴─────────────┴──────────────────────┘ 


As part of Power policy power manger will provide below dbus properties.
 1. Mode -> (MaxP/MaxQ/Custom).
 2. ChassisPowerLimit_Curr -> (Set is Valid only when Mode is Custom).
 3. ChassisPowerLimit_P -> (this value will be loaded into ChassisPowerLimit_Curr when Mode is MaxP. Not configurable at runtime).
 4. ChassisPowerLimit_Q -> (this value will be loaded into ChassisPowerLimit_Curr when Mode is MaxQ. Not configurable at runtime).
 5. ChassisPowerLimit_Min -> (this value is the minimum value that can be loade into ChassisPowerLimit_Curr).
 6. ChassisPowerLimit_Max -> (this value is the maximum value that can be loade into ChassisPowerLimit_Curr).
 7. RestOfSystemPower -> (GPU power capping is dependent on ChassisPowerLimit_Curr and RestOfSystemPower property).

power mangaer will load the default values of above properties from the NV storage. monitor the mode and power limit
properties . any change in the power policy properties are logged and in custom mode the GPU power capping is calculated
and set using GPU manager Dbus API.User settings values should also be in written into file and should be preserved while FW update.

pwer capping for GPU is derived from below formula.

ChassisPowerLimit_Curr = ((Max Power of GA100 x Total GA100 in Luna) + System power other than GPU). 
ex: for MaxP 
6500 = ((400 x 8) + 3300).
ChassisPowerLimit_P - 6500 W
Max Power of GA100 - 400 W
Total GA100 in Luna - 8 W 
System power other than GPU - 3300 W.
so according to above max power of each GPU is capped to 400 W.

 
  

                                                                                                                        Dbus
                                                           ┌─────────────────┐                                            │
                                                           │                 │                                            │
                                                           │nvidia-oem-cmds  ├────────────────────────────────────────────►
                                                           │                 │                                            │
                                                           └─────────────────┘                                            │
                                                                                                                          │
                              ┌────────────────────────────────────────────────────────────────────────┐  ┌─────────────┐ │
                              │ Power-Manager                                       PSU<3 Redundancy   │  │   Host      │ │
                              │                                         ┌──────────────────────────────┴──► Power oFF   │ │
                              │                                         │           Lost Powering off host│   GPIO      │ │
                              │        ┌────────────────────────────────┴──────┐                       │  └─────────────┘ │
                              │        │Fault Monitoring & Logging:            │                       │                  │
                              │        │1.System Power On.                     │           Monitoring  │ Dbus Sensors     │
                              │        │2.Mode Change.                         ◄───────────────────────┼──────────────────┤
                              │        │3.Power Limit Change.                  │                       │                  │
                              │        │4.System or GB shutdown when PSU < 3.  │        Dbus SEL logging  API             │      ┌────────────────┐
                              │        │                                       ├───────────────────────┬──────────────────►      │                │
                              │        └───────────────────────────────────────┘                       │                  ┌──────►  phosphor sel  │
┌───────────────────┐         │                                                                        │                  │      │    logger      │
│  Power Policy     │         │       ┌─────────────────────────────────────────┐                      │                  │      └────────────────┘
│   Configuration   │         │       │Power Policy:                            │                      │                  │
│     file.         │         │       │1.Mode                                   │                      │                  │
└────────┬──────────┘         │       │2.ChassisPowerLimit_Curr                 │                      │                  │
         │                    │       │3.ChassisPowerLimit_P                    │          Providing   │ Dbus Properties  │
         │   load default values      │4.ChassisPowerLimit_Q                    ├──────────────────────┼──────────────────►
         └────────────────────┬───────►5.ChassisPowerLimit_Min                  │                      │                  │
                              │       │6.ChassisPowerLimit_Max                  │                      │                  │
                              │       │7.RestOfSystemPower                      │                      │                  │
                              └───────┴─────────────────────────────────────────┴──────────────────────┘                  │
                                                                                                                          │
                                                                                                                          │
                                                                                                                          │
                                                                                                                          │
                                                       ┌──────────────────────────────┐                                   │
                                                       │                              │    Dbus GPU Manger API            │
                                                       │        GPU  Manager          │◄──────────────────────────────────┤
                                                       │                              │     For Power Capping             │
                                                       │                              │                                   ▼
                                                       └──────────────────────────────┘




 # TODO: GPU manager Dbus API to set Power Capping need to be provided by NVDIA.