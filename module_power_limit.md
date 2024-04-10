# Redfish aggregation Stack Architecture
Author: Mars Yang  
Created: April 10, 2024

<!-- TOC -->
- [Platform Enablement - module power limit](#))
    - [Contents](#contents)
    - [Features/capabilities](#featurescapabilities)
    - [Standard spec versions](#standard-spec-versions)
        - [Nvidia power manager](#nvidia-power-manager)
    - [High Level Architecture](#high-level-architecture)
    - [Low-Level design](#low-level-architecture)
        - [Dbus objects for the module power](#dbus-objects-for-the-module-power)
        - [Module power configuration](#module-power-configuration)
<!-- /TOC -->
## Features/capabilities
1. enable the module power limit for Grace Hopper platforms
2. generate redfish event for power limit setting. 

## Standard spec versions 

###  Nvidia power manager
- https://gitlab-master.nvidia.com/dgx/bmc/docs/-/blob/module_power/nvidia/nvdia-power-manager.md

<!-- /TOC -->
## High Level Architecture
Grace Hopper platform has a variety of configurations, CG1 , CG4 and C2. In CG1 or CG4 systems,there is a pair of GPU and CPU in the module. The system offers CPU, GPU as well as the module-level power limit. CPU and GPU power limit are existing solutions and the module power limit is new technology. SMBPBI commands will be the interface to configure the module power budget. `nvidia-power-manager` will be utilized to provide the different power configurations and methods for CG1 or CG4. `nvidia-gpu-manager` will be backend service to deliver power settings to GPU firmware through SMBPBI interface.
In C2 system, there are two CPUs in the processor module without any GPU on the system. PLDM service as the backend service provides the module-level power capability/reporting and deliver the power settings to satMC. `nvidia-power-manager` is not used any longer for C2 systems.

<!-- /TOC -->
## Low Level Architecture
`nvidia-power-manager` is the middle layer between Redfish APIs and `nvidia-gpu-manager`. It holds the system's power policy and can deliver the power settings to te backend service via Dbus , generate redfish event message indicating the power setting changes and populate mnodule power Dbus objects.
Once the power policy is received by `nvidia-power-manager`, it will send SMPBPI commands to GPU and the module power limit will be executed.

```                                                                                                                                                                                             
 ┌────────────────────────────┐                                                    
 │                            │                                                    
 │  Redfish EnvironmentMetric/│                                                    
 │  Control                   │                                                    
 └────────────┬───────────────┘                                                    
              │                                                                    
              │  Power setting                                                     
              │                                                                    
              │                                                                    
  ┌───────────▼───────────────┐              ┌───────────────┐                     
  │                           │              │               │                     
  │   nvidia-power-manager    ├────────────► │redfish message│                     
  │                           │              │               │                     
  │     (power policy)        │              └───────────────┘                     
  └───────────┬───────────────┴┐                                                   
              │                │                                                   
              │                │             ┌────────────────────────────┐        
              │                │             │                            │        
              │  Dbus call     └────────────►│ Dbus power control objects │        
              │                              │                            │        
              │                              └────────────────────────────┘        
              │                                                                    
              ▼                                                                    
   ┌───────────────────────────┐                       ┌─────────────────────────┐ 
   │                           │      SMPBPI           │                         │ 
   │                           │                       │                         │ 
   │ nvidia-power-manager      │ ───────────────────►  │    GPU firmware(VBIOS)  │ 
   │   (Backend Service)       │                       │                         │ 
   │                           │                       │                         │ 
   └───────────────────────────┘                       └─────────────────────────┘ 
                                                                                                                         
                                                                                    
```

### Dbus objects for the module power

Following the Dbus tree and objects for the module power control
```
#busctl tree  com.Nvidia.Powermanager
`-/xyz
`-/xyz/openbmc_project
`-/xyz/openbmc_project/inventory
  `-/xyz/openbmc_project/inventory/system
    `-/xyz/openbmc_project/inventory/system/board
      |-/xyz/openbmc_project/inventory/system/board/ProcessorModule_0
      | `-/xyz/openbmc_project/inventory/system/board/ProcessorModule_0/power
      |   `-/xyz/openbmc_project/inventory/system/board/ProcessorModule_0/power/control
      |     `-/xyz/openbmc_project/inventory/system/board/ProcessorModule_0/power/control/ProcessorModule_0_PowerNonvolatile_0

#busctl introspect com.Nvidia.Powermanager /xyz/openbmc_project/inventory/system/board/ProcessorModule_0/power/control/ProcessorModule_0_PowerNonvolatile_0 
NAME                                        TYPE      SIGNATURE RESULT/VALUE                             FLAGS
org.freedesktop.DBus.Introspectable         interface -         -                                        -
.Introspect                                 method    -         s                                        -
org.freedesktop.DBus.Peer                   interface -         -                                        -
.GetMachineId                               method    -         s                                        -
.Ping                                       method    -         -                                        -
org.freedesktop.DBus.Properties             interface -         -                                        -
.Get                                        method    ss        v                                        -
.GetAll                                     method    s         a{sv}                                    -
.Set                                        method    ssv       -                                        -
.PropertiesChanged                          signal    sa{sv}as  -                                        -
xyz.openbmc_project.Association.Definitions interface -         -                                        -
.Associations                               property  a(sss)    1 "chassis" "power_controls" "/xyz/op... emits-change
xyz.openbmc_project.Control.Power.Cap       interface -         -                                        -
.MaxPowerCapValue                           property  u         1000                                     emits-change
.MinPowerCapValue                           property  u         200                                      emits-change
.PowerCap                                   property  u         350                                      emits-change writable 
```
### Module power Configuration
`powermanger.json` is located in the platform recipes describing the power settings or methods to configure the limit.
`powerCappingConfigs` is the array so the scalability is considered in nvidia-power-manager to expand the support of the processor modules, like CG4 system. More than one module power configuration can be supported. 

| property       | description     | 
|----------------|-----------------------|
|objectName      | dbus path of the module power control|
|interface       | the interface of power capping. |
| flags          | the permission of the property, like ReadOnly or readWrite|
|MinPowerCapValue| Maximum power capping value of the module|
|MaxPowerCapValue| Minimum power capping value of the module |
|Property object     | the actions to issue Dbus call for redfish event or power limit|
|action.methodName   | Dbus method,e.g. Create or Set|
|action.objectpath   | Dbus object path of the backend service|
|action.serviceName  | the backend service name|
|action.interfaceName| the power capping interface in the backend service|
|appendData          | the flexible or non-fixed size options for Dbus call|
|Associations        | build the association of the processor module|
|PhysicalContext     | Redfish Context of the power control resource|

* Redfish event

The backend service will be `xyz.openbmc_project.Logging`.
Generally speaking, redfish event will be generated through Dbus all to Logging Service.

|data                             |dataType|description|
|---------------------------------|----------------|---------|
|ResourceEvent.1.0.ResourceChanged|string| redfish event message|
|xyz.openbmc_project.Logging.Entry.Level.Informational|string| the event severity|
|array collection                |dict| Dbus arguments. e.g.Resolution or message ID|

Following is Redfish event setting.
```
 {
    "methodName": "Create",
    "serviceName": "xyz.openbmc_project.Logging",
    "objectpath": "/xyz/openbmc_project/logging",
    "interfaceName": "xyz.openbmc_project.Logging.Create",
    "appendData": [
        {
            "dataType": "s",
            "data": "ResourceEvent.1.0.ResourceChanged"
        },
        {
            "dataType": "s",
            "data": "xyz.openbmc_project.Logging.Entry.Level.Informational"
        },
        {
            "dataType": "e",
            "data": [
                    {"REDFISH_MESSAGE_ID": "ResourceEvent.1.0.ResourceChanged"},
                    {"xyz.openbmc_project.Logging.Entry.Resolution": "None"}
                ]
        }
    ]
}
```
* execute power policy

The backend service is `xyz.openbmc_project.GpuMgr`


| data                                |dataType|description|
|-------------------------------------|--------|---------------|
|xyz.openbmc_project.Control.Power.Cap|s       |the power capping interface|
|ModulePowerCap                       |s       |the module power property|
|PropertyValue                        |s       |the module power value|
````
{
    "methodName": "Create",
    "serviceName": "xyz.openbmc_project.Logging",
    "objectpath": "/xyz/openbmc_project/logging",
    "interfaceName": "xyz.openbmc_project.Logging.Create",
    "appendData": [
        {
            "dataType": "s",
            "data": "ResourceEvent.1.0.ResourceChanged"
        },
        {
            "dataType": "s",
            "data": "xyz.openbmc_project.Logging.Entry.Level.Informational"
        },
        {
            "dataType": "e",
            "data": [
                {"REDFISH_MESSAGE_ID": "ResourceEvent.1.0.ResourceChanged"},
                {"xyz.openbmc_project.Logging.Entry.Resolution": "None"}
                ]
        }
    ]
},
````
* following is the module power configuration from `powermanger.json`
```
  "powerCappingConfigs":[
        {
            "objectName": "/xyz/openbmc_project/inventory/system/board/ProcessorModule_0/power/control/ProcessorModule_0_PowerNonvolatile_0",
            "interfaceName": "xyz.openbmc_project.Control.Power.Cap",
            "module":"Module",
            "property": [
                {
                    "propertyName": "PowerCap",
                     "value": 1000,
                     "flags": "sdbusplus::asio::PropertyPermission::readWrite" ,
                     "action":[
                        {
                            "methodName": "Create",
                            "serviceName": "xyz.openbmc_project.Logging",
                           ... 
                        },{
                            "methodName": "Set" ,
                            "serviceName": "xyz.openbmc_project.GpuMgr",
                            ...
                        }
                     ]
                },
                {
                    "propertyName": "MinPowerCapValue",
                     "value": 200,
                     "flags": "sdbusplus::asio::PropertyPermission::readOnly"
                },
                {
                    "propertyName": "MaxPowerCapValue",
                     "value": 1000,
                     "flags": "sdbusplus::asio::PropertyPermission::readOnly"
                }
            ]
        },
        {
            "objectName": "/xyz/openbmc_project/inventory/system/board/ProcessorModule_0/power/control/ProcessorModule_0_PowerNonvolatile_0",
            "interfaceName": "xyz.openbmc_project.Association.Definitions",
            "module":"Module",
            "property": [
                {
                    "propertyName": "Associations",
                     "value": "chassis power_controls /xyz/openbmc_project/inventory/system/board/ProcessorModule_0",
                     "flags": "sdbusplus::asio::PropertyPermission::readOnly"
                },
                {
                    "propertyName": "PhysicalContext",
                     "value": "xyz.openbmc_project.Inventory.Decorator.Area.PhysicalContextType.SystemBoard",
                     "flags": "sdbusplus::asio::PropertyPermission::readOnly"
                }
            ]
        },

```