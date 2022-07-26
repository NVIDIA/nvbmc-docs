# Nvidia power Manager

Author:
  hemanthkumarm@ami.com
  nibinc@ami.com

Primary assignee:
  hemanthkumarm@ami.com
  nibinc@ami.com

Created:
  February 1, 2022

## Problem Description
This is a proposal to provide a set of enhancements to the current BMC firmware and 
as a part of power management has the ability to monitor the PSUs for faults and log 
errors and take evasive action on particular fault conditions. This also enhances the
abiity to do power capping based on power policy mode.


## Requirements
The dbus sensors should be populated by the inventory application.

## Proposed Design
As part of fault monitoring, power manager application will monitor the sensors configured in PowerRedundancyConfigs section of powermanager.json.sensors can be added as part of "PowerRedundancyConfigs" section of the powermanager.json configuration file. all the sensors monitored by power manager and there respective corrective actions are configurable using "PowerRedundancyConfigs"  section of the powermanager.json

###example:
    
	
	"PowerRedundancyConfigs": [
        {
            "objectName": "/xyz/openbmc_project/sensors/power/psu_drop_to_1_event",
            "interfaceName": "xyz.openbmc_project.Object.Enable",
            "propertyName": "Enabled",
            "action": [
                {
                    "trigger": true,
                    "methodName": "IpmiSelAddOem",
                    "serviceName": "xyz.openbmc_project.Logging.IPMI",
                    "objectpath": "/xyz/openbmc_project/Logging/IPMI",
                    "interfaceName": "xyz.openbmc_project.Logging.IPMI",
                    "appendData": [
                        {
                            "dataType": "s",
                            "data": "OEM psuDropTo1EventTriggered SEL Entry"
                        },
                        {
                            "dataType": "y",
                            "data": [6, 255 , 255]
                        },
                        {
                            "dataType": "y",
                            "data": 201
                        }
                    ]
                },
                {
                    "trigger": true,
                    "conditionBlock": [
                        {
                            "serviceName": "com.Nvidia.PsuEvent",
                            "objectpath": "/xyz/openbmc_project/sensors/power/MBON_GBOFF_event",
                            "interfaceName": "xyz.openbmc_project.Object.Enable" ,
                            "propertyName": "Enabled",
                            "timeDelay": 10,
                            "propertyValue": true

                        }
                    ],
                    "methodName": "Set",
                    "serviceName": "xyz.openbmc_project.State.Chassis",
                    "objectpath": "/xyz/openbmc_project/state/chassis0",
                    "interfaceName": "org.freedesktop.DBus.Properties",
                    "appendData": [
                        {
                            "dataType": "s",
                            "data": "xyz.openbmc_project.State.Chassis"
                        },
                        {
                            "dataType": "s",
                            "data": "RequestedPowerTransition"
                        },
                        {
                            "dataType": "s",
                            "data": "xyz.openbmc_project.State.Chassis.Transition.Off" ,
                            "variant": true
                        }
                    ]
                }
            ]
        },
		{
			.
			. can configure multiple sensors monitered by power manager
			.
		}
        
	]


 BMC is responsible for logging OEM sel events as part of Power manager using phosphor sel logging service. this can be configured in the action block of the respective sensor in PowerRedundancyConfigs

	"action": [
                {
                    "trigger": true,
                    "methodName": "IpmiSelAddOem",
                    "serviceName": "xyz.openbmc_project.Logging.IPMI",
                    "objectpath": "/xyz/openbmc_project/Logging/IPMI",
                    "interfaceName": "xyz.openbmc_project.Logging.IPMI",
                    "appendData": [
                        {
                            "dataType": "s",
                            "data": "OEM psuDropTo1EventTriggered SEL Entry"
                        },
                        {
                            "dataType": "y",
                            "data": [6, 255 , 255]
                        },
                        {
                            "dataType": "y",
                            "data": 201
                        }
                    ]
                },


As a part of Power policy, power manger will provide dbus properties for the properties configured in "powerCappingConfigs" section of powermanager.json.
currents the configuration will support only string and uint32 dbus types as properties.all the dbus properties and their trigger actions are configurable using the "powerCappingConfigs" section of powermanager.json

###Example

	"powerCappingConfigs":[
        {
            "objectName": "/xyz/openbmc_project/control/power/CurrentChassisLimit",
            "interfaceName": "xyz.openbmc_project.Control.Power.Cap",
            "module":"System",
            "property": [
                {
                    "propertyName": "PowerCap",
                     "value": 6500000,
                     "flags": "sdbusplus::asio::PropertyPermission::readWrite" ,
                     "action":[
                        {
                            "methodName": "IpmiSelAddOem",
                            "serviceName": "xyz.openbmc_project.Logging.IPMI",
                            "objectpath": "/xyz/openbmc_project/Logging/IPMI",
                            "interfaceName": "xyz.openbmc_project.Logging.IPMI",
                            "appendData": [
                                {
                                    "dataType": "s",
                                    "data": "OEM current chassis limit changed SEL Entry"
                                },
                                {
                                    "data": "OEM",
                                    "key": "AddSel"
                                },
                                {
                                    "dataType": "y",
                                    "data": 201
                                }
                            ]
                        }
                     ]    
                },
                {
                    "propertyName": "MinPowerCapValue",
                     "value": 4900000,
                     "flags": "sdbusplus::asio::PropertyPermission::readOnly"
                },
                {
                    "propertyName": "MaxPowerCapValue",
                     "value": 6500000,
                     "flags": "sdbusplus::asio::PropertyPermission::readOnly"
                },
                {
                    "propertyName": "Associations",
                     "value": " ",
                     "flags": "sdbusplus::asio::PropertyPermission::readOnly"
                }
            ]

        },
        {
            "objectName": "/xyz/openbmc_project/control/power/CurrentChassisLimit",
            "interfaceName": "xyz.openbmc_project.Control.Power.Mode",
            "module":"System",
             "property":[
                 {
                    "propertyName": "PowerMode",
                     "value": "xyz.openbmc_project.Control.Power.Mode.PowerMode.MaximumPerformance",
                     "flags": "sdbusplus::asio::PropertyPermission::readWrite",
                     "action":[
                        {
                            "methodName": "IpmiSelAddOem",
                            "serviceName": "xyz.openbmc_project.Logging.IPMI",
                            "objectpath": "/xyz/openbmc_project/Logging/IPMI",
                            "interfaceName": "xyz.openbmc_project.Logging.IPMI",
                            "appendData": [
                                {
                                    "dataType": "s",
                                    "data": "OEM current PowerMode changed SEL Entry"
                                },
                                {
                                    "dataType": "y",
                                    "data": [1, 255 , 255]
                                },
                                {
                                    "dataType": "y",
                                    "data": 201
                                }
                            ]
                        }
                     ] 
                }
                ]

        },
        {
            "objectName": "/xyz/openbmc_project/control/power/ChassisLimitP",
            "interfaceName": "xyz.openbmc_project.Control.Power.Cap",
            "module":"System",
             "property":[
                 {
                    "propertyName": "MaxPowerCapValue",
                     "value": 6500000,
                     "flags": "sdbusplus::asio::PropertyPermission::readOnly"
                 }
                ]

        },
        {
            "objectName": "/xyz/openbmc_project/control/power/ChassisLimitQ",
            "interfaceName": "xyz.openbmc_project.Control.Power.Cap",
            "module":"System",
             "property":[
                 {
                    "propertyName": "MaxPowerCapValue",
                     "value": 4900000,
                     "flags": "sdbusplus::asio::PropertyPermission::readOnly"
                 }
                ]

        },
        {
            "objectName": "/xyz/openbmc_project/control/power/RestOfSystemPower",
            "interfaceName": "xyz.openbmc_project.Sensor.Value",
            "module":"RestOFSystem",
             "property":[
                 {
                    "propertyName": "Value",
                     "value": 3300000,
                     "flags": "sdbusplus::asio::PropertyPermission::readOnly"
                 }
                ]

        },{
            "objectName": "/xyz/openbmc_project/control/power/GPUPowerLimit",
            "interfaceName": "xyz.openbmc_project.Control.Power.Cap",
            "module":"GPU",
             "property":[
                 {
                    "propertyName": "PowerCap",
                     "value": 400000,
                     "flags": "sdbusplus::asio::PropertyPermission::readOnly",
                     "action":[
                        {
                            "methodName": "IpmiSelAddOem",
                            "serviceName": "xyz.openbmc_project.Logging.IPMI",
                            "objectpath": "/xyz/openbmc_project/Logging/IPMI",
                            "interfaceName": "xyz.openbmc_project.Logging.IPMI",
                            "appendData": [
                                {
                                    "dataType": "s",
                                    "data": "OEM GPU PowerLimit changed SEL Entry"
                                },
                                {
                                    "dataType": "y",
                                    "data": [1, 255 , 255]
                                },
                                {
                                    "dataType": "y",
                                    "data": 201
                                }
                            ]
                        },{
                            "conditionBlock": [
                                {
                                    "serviceName": "com.Nvidia.Powermanager",
                                    "objectpath": "/xyz/openbmc_project/control/power/CurrentChassisLimit",
                                    "interfaceName": "xyz.openbmc_project.Control.Power.Mode",
                                    "propertyName": "PowerMode",
                                    "propertyValue": "xyz.openbmc_project.Control.Power.Mode.PowerMode.OEM"
                                }
                            ],
                            "methodName": "Set" ,
                            "serviceName": "xyz.openbmc_project.GpuMgr",
                            "objectpath": "/xyz/openbmc_project/inventory/system/chassis/HGX_Baseboard_0",
                            "interfaceName":"org.freedesktop.DBus.Properties",
                            "appendData": [
                                {
                                    "dataType": "s",
                                    "data": "xyz.openbmc_project.Control.Power.Cap"
                                },
                                {
                                    "dataType": "s",
                                    "data": "SetPoint"
                                },
                                {
                                    "dataType": "s",
                                    "data": "PropertyValue"
                                }
                            ]
                        }
                     ] 
                 }
                ]

        }
    ]


	below are the dbus object Paths created according to above configuration.
	
	# busctl  tree com.Nvidia.Powermanager
	`-/xyz
	  `-/xyz/openbmc_project
    `-/xyz/openbmc_project/control
      `-/xyz/openbmc_project/control/power
        |-/xyz/openbmc_project/control/power/ChassisLimitP
        |-/xyz/openbmc_project/control/power/ChassisLimitQ
        |-/xyz/openbmc_project/control/power/CurrentChassisLimit
        |-/xyz/openbmc_project/control/power/GPUPowerLimit
        `-/xyz/openbmc_project/control/power/RestOfSystemPower
	# busctl introspect com.Nvidia.Powermanager /xyz/openbmc_project/control/power/CurrentChassisLimit
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
	xyz.openbmc_project.Association             interface -         -                                        -
	.Endpoints                                  property  ao        0                                        emits-change writable
	xyz.openbmc_project.Association.Definitions interface -         -                                        -
	.Associations                               property  a(sss)    1 "power_control" "chassis" "/xyz/ope... emits-change writable
	xyz.openbmc_project.Control.Power.Cap       interface -         -                                        -
	.MaxPowerCapValue                           property  u         6500000                                  emits-change
	.MinPowerCapValue                           property  u         4900000                                  emits-change
	.PowerCap                                   property  u         6000000                                  emits-change writable
	xyz.openbmc_project.Control.Power.Mode      interface -         -                                        -
	.PowerMode                                  property  s         "xyz.openbmc_project.Control.Power.Mo... emits-change writable
	# busctl introspect com.Nvidia.Powermanager /xyz/openbmc_project/control/power/GPUPowerLimit
	NAME                                  TYPE      SIGNATURE RESULT/VALUE FLAGS
	org.freedesktop.DBus.Introspectable   interface -         -            -
	.Introspect                           method    -         s            -
	org.freedesktop.DBus.Peer             interface -         -            -
	.GetMachineId                         method    -         s            -
	.Ping                                 method    -         -            -
	org.freedesktop.DBus.Properties       interface -         -            -
	.Get                                  method    ss        v            -
	.GetAll                               method    s         a{sv}        -
	.Set                                  method    ssv       -            -
	.PropertiesChanged                    signal    sa{sv}as  -            -
	xyz.openbmc_project.Control.Power.Cap interface -         -            -
	.PowerCap                             property  u         2940000      emits-change
	# busctl introspect com.Nvidia.Powermanager /xyz/openbmc_project/control/power/ChassisLimitP
	NAME                                  TYPE      SIGNATURE RESULT/VALUE FLAGS
	org.freedesktop.DBus.Introspectable   interface -         -            -
	.Introspect                           method    -         s            -
	org.freedesktop.DBus.Peer             interface -         -            -
	.GetMachineId                         method    -         s            -
	.Ping                                 method    -         -            -
	org.freedesktop.DBus.Properties       interface -         -            -
	.Get                                  method    ss        v            -
	.GetAll                               method    s         a{sv}        -
	.Set                                  method    ssv       -            -
	.PropertiesChanged                    signal    sa{sv}as  -            -
	xyz.openbmc_project.Control.Power.Cap interface -         -            -
	.MaxPowerCapValue                     property  u         6500000      emits-change
	# busctl introspect com.Nvidia.Powermanager /xyz/openbmc_project/control/power/ChassisLimitQ
	NAME                                  TYPE      SIGNATURE RESULT/VALUE FLAGS
	org.freedesktop.DBus.Introspectable   interface -         -            -
	.Introspect                           method    -         s            -
	org.freedesktop.DBus.Peer             interface -         -            -
	.GetMachineId                         method    -         s            -
	.Ping                                 method    -         -            -
	org.freedesktop.DBus.Properties       interface -         -            -
	.Get                                  method    ss        v            -
	.GetAll                               method    s         a{sv}        -
	.Set                                  method    ssv       -            -
	.PropertiesChanged                    signal    sa{sv}as  -            -
	xyz.openbmc_project.Control.Power.Cap interface -         -            -
	.MaxPowerCapValue                     property  u         4900000      emits-change
	# busctl introspect com.Nvidia.Powermanager /xyz/openbmc_project/control/power/RestOfSystemPower
	NAME                                TYPE      SIGNATURE RESULT/VALUE FLAGS
	org.freedesktop.DBus.Introspectable interface -         -            -
	.Introspect                         method    -         s            -
	org.freedesktop.DBus.Peer           interface -         -            -
	.GetMachineId                       method    -         s            -
	.Ping                               method    -         -            -
	org.freedesktop.DBus.Properties     interface -         -            -
	.Get                                method    ss        v            -
	.GetAll                             method    s         a{sv}        -
	.Set                                method    ssv       -            -
	.PropertiesChanged                  signal    sa{sv}as  -            -
	xyz.openbmc_project.Sensor.Value    interface -         -            -
	.Value                              property  u         3300000      emits-change


power manager will follow below sequence in order to create association with the system chassis object.


1. Search all objects which implement xyz.openbmc_project.Inventory.Item.Chassis and pick the one whose chassisID match with the chassisID in the request.


2. Find this association end point - /xyz/openbmc_project/inventory/system/chassis/<chassis_id>/power_controls 
3. The end point should point to the object path which implement the below interface.

	

> xyz.openbmc_project.Control.Power.Cap


below is how redfish can use this association to access the powermanger object path.

###Example:

	root@dgx:~# busctl introspect xyz.openbmc_project.ObjectMapper /xyz/openbmc_project/inventory/system/board/Luna_Motherboard/power_controls
	NAME                                TYPE      SIGNATURE RESULT/VALUE                             FLAGS
	org.freedesktop.DBus.Introspectable interface -         -                                        -
	.Introspect                         method    -         s                                        -
	org.freedesktop.DBus.Peer           interface -         -                                        -
	.GetMachineId                       method    -         s                                        -
	.Ping                               method    -         -                                        -
	org.freedesktop.DBus.Properties     interface -         -                                        -
	.Get                                method    ss        v                                        -
	.GetAll                             method    s         a{sv}                                    -
	.Set                                method    ssv       -                                        -
	.PropertiesChanged                  signal    sa{sv}as  -                                        -
	xyz.openbmc_project.Association     interface -         -                                        -
	.endpoints                          property  as        1 "/xyz/openbmc_project/control/power... emits-change
		root@dgx:~# busctl get-property xyz.openbmc_project.ObjectMapper /xyz/openbmc_project/inventory/system/board/Luna_Motherboard/power_controls xyz.openbmc_project.Association endpoints
	as 1 "/xyz/openbmc_project/control/power/CurrentChassisLimit"

	root@dgx:~# busctl get-property com.Nvidia.Powermanager /xyz/openbmc_project/control/power/CurrentChassisLimit xyz.openbmc_project.Association.Definitions Associations
	a(sss) 1 "chassis" "power_controls" "/xyz/openbmc_project/inventory/system/board/Luna_Motherboard"



The power mangager will load the default values of configured properties from the NV storage, monitor the mode and power limit properties. Any change in the power policy properties will triggere their respective action block and the configured Dbus API will be called.

Power capping for various power modules will be configured using "powerCappingAlgorithm" section of the power manager.json
###example:

    "powerCappingAlgorithm":[
        {
            "powerModule":"GPU",
            "powerCapPercentage":49,
            "numOfDevices": 1
            
        }
    ],
	# busctl introspect com.Nvidia.Powermanager /xyz/openbmc_project/control/power/GPUPowerLimit
	NAME                                  TYPE      SIGNATURE RESULT/VALUE FLAGS
	org.freedesktop.DBus.Introspectable   interface -         -            -
	.Introspect                           method    -         s            -
	org.freedesktop.DBus.Peer             interface -         -            -
	.GetMachineId                         method    -         s            -
	.Ping                                 method    -         -            -
	org.freedesktop.DBus.Properties       interface -         -            -
	.Get                                  method    ss        v            -
	.GetAll                               method    s         a{sv}        -
	.Set                                  method    ssv       -            -
	.PropertiesChanged                    signal    sa{sv}as  -            -
	xyz.openbmc_project.Control.Power.Cap interface -         -            -
	.PowerCap                             property  u         2940000      emits-change

	

###Powermanager flow

	
	                                                                                                        Dbus Api
	
	
	                                                                                                             │
	                                                                                                             │
	                                                                                                             │
	                                                                                                             │
	                                                                                                             │
	                                                                                                             │
	                                                                                                             │
	                     ┌───────────────────────────────────────────────────────────────────────────────┐       │
	                     │                                                                               │       │
	                     │   Power Manager                                                               │       │
	                     │                                                                               │       │
	                     │                                                                               │       │
	                     │                                                                               │       │
	                     │         ┌────────────────────────────────────┐                                │       │
	                     │         │                                    │                                │       │
	                     │         │  Fault monitoring & logging:       │                                │       │
	                     │         │1.sensors configured in configuration                                │       │
	                     │         │ file .                             ─┐          Action Block         │       │
	                     │         │2. perform action configured when cha└───────────────────────────────┼──────►│
	                     │         │   detected.                        │          configured            │       │
	                     │         │                                    │                                │       │
	                     │         │                                    │                                │       │
	                     │         │                                    │                                │       │
	                     │         │                                    │                                │       │
	                     │         └────────────────────────────────────┘                                │       │
	┌────────────────┐   │                                                                               │       │
	│                │   │                                                                               │       │
	│ configuration  │   │         ┌──────────────────────────────────────┐                              │       │
	│    file        │   │         │                                      │                              │       │
	│                │   │         │ Power-policy:                        │                              │       │
	│                │   │         │1. Provide the dbus properties configured                            │       │
	│                │   │         │   in the configuration file and monitor                             │       │
	│                │   │         │   any change and perform the configured                             │       │
	│                │   │         │   action.                            │           Action Block       │       │
	└─────┬──────────┘   │         │                                      ├──────────────────────────────┼──────►│
	      │              │         │                                      │                              │       │
	      │              │         │2. calculate power limit based on the │            configuted        │       │
	      └──────────────┼────────►│   power capping algorithm.           │                              │       │
	                     │         │                                      │                              │       │
	                     └─────────┴──────────────────────────────────────┴──────────────────────────────┘       │
	                                                                                                             │
	                                                                                                             │
	                                                                                                             │
	                                                                                                             │
	                                                                                                             │
	                                                                                                             │
	                                                                                                             │
	                                                                                                             │





More details on json are present in the README file of nvidia-power-manager repo.
 

# Redfish API's for Power Management

 ## 1.PowerSubsystem 
This resource contains the PowerSubsystem for the Chassis.
### Request
	GET /redfish/v1/Chassis/{chassisID}/PowerSubsystem/
	Content-Type: application/json
### Response
The response of the request will be in JSON format. The properties are mentioned in the following table.
### Properties table for PowerSubsystem
	|Name|Type |Read only|Description|
	| - | - | - | - |
	|@odata.id|String|True|The value of this property shall be the unique identifier for the resource and it shall be of the form defined in the Redfish specification.
	|@odata.type|String|True|The value of this property shall be an absolute URL that specifies the type of the resource and it shall be of the form defined in the Redfish specification. 
	|Id|String|True|Resource Identifier
	|Name|String|True|Name of the Resource
	|PowerSupplies|Object|True|The link to the collection of power supplies within this subsystem.

## 2.PowerSupplies Collection
It displays the collection of power supplies. 
### Request

	GET /redfish/v1/Chassis/{chassisID}/PowerSubsystem/PowerSupplies/
	Content-Type: application/json

### Response
The response of the request will be in JSON format. The properties are mentioned in the following table.
### Properties table for PowerSupplies Collection
	|Name|Type|Read Only|Description
	|-|-|-|-|
	|@odata.id|String|True|The value of this property shall be the unique identifier for the resource and it shall be of the form defined in the Redfish specification.
	|@odata.type|String|True|The value of this property shall be an absolute URL that specifies the type of the resource and it shall be of the form defined in the Redfish specification. 
	|Members|Array|True|Contains the members of this collection.|
	|Members@odata.count|Number|True|Collection members count.|
	|Name|String|True|Name of the Collection|

## 3.PowerSupply Instance
The PowerSupply schema describes a power supply unit.
### Request
	GET /redfish/v1/Chassis/{chassisID}/PowerSubsystem/PowerSupplies/{PowerSupplyId}
	Content-Type: application/json
### Response
The response of the request will be in JSON format. The properties are mentioned in the following table
### Properties table for PowerSupply
	|Name|Type|Read Only|Description|Dbus Command|
	|-|-|-|-|-|
	|@odata.id|String|True|The value of this property shall be the unique identifier for the resource and it shall be of the form defined in the Redfish specification.
	|@odata.type|String|True|The value of this property shall be an absolute URL that specifies the type of the resource and it shall be of the form defined in the Redfish specification.
	|Id|String|True|Resource Identifier
	|Name|String|True|Name of the Resource
	|Location|Object|True|The location of the power supply|busctl call com.Nvidia.Powersupply  /xyz/openbmc_project/inventory/system/chassis/motherboard/<powersupplyId> org.freedesktop.DBus.Properties GetAll s "xyz.openbmc_project.Inventory.Decorator.LocationCode"
	|Manufacturer|String|True|The manufacturer of this power supply.|busctl get-property com.Nvidia.Powersupply /xyz/openbmc_project/inventory/system/chassis/motherboard/<powerSupplyId> xyz.openbmc_project.Inventory.Decorator.Asset Manufacturer
	|Model|String|True|The model number for this power supply.|busctl get-property com.Nvidia.Powersupply /xyz/openbmc_project/inventory/system/chassis/motherboard/<powerSupplyId> xyz.openbmc_project.Inventory.Decorator.Asset Model
	|PartNumber|String|True|The part number for this power supply.|busctl get-property com.Nvidia.Powersupply /xyz/openbmc_project/inventory/system/chassis/motherboard/<powerSupplyId> xyz.openbmc_project.Inventory.Decorator.Asset PartNumber
	|Status|Object|True|The status and health of the resource and its subordinate or dependent resources.|State = busctl get-property com.Nvidia.Powersupply /xyz/openbmc_project/inventory/system/chassis/motherboard/<powerSupplyId> xyz.openbmc_project.Inventory.Item Present, Health = busctl get-property com.Nvidia.Powersupply /xyz/openbmc_project/inventory/system/chassis/motherboard/<powerSupplyId> xyz.openbmc_project.State.Decorator.OperationalStatus Functional
	|SerialNumber|String|True|The serial number for this power supply.|busctl get-property com.Nvidia.Powersupply /xyz/openbmc_project/inventory/system/chassis/motherboard/<powerSupplyId> xyz.openbmc_project.Inventory.Decorator.Asset SerialNumber
	|SparePartNumber|String|True|The spare part number for this power supply.|busctl get-property com.Nvidia.Powersupply /xyz/openbmc_project/inventory/system/chassis/motherboard/<powerSupplyId> xyz.openbmc_project.Inventory.Decorator.Asset SparePartNumber
	|EfficiencyRatings|array|True|The efficiency ratings of this power supply.|busctl get-property xyz.openbmc_project.Settings /xyz/openbmc_project/control/power_supply_attributes xyz.openbmc_project.Control.PowerSupplyAttributes DeratingFactor

## 4.Control
The Control schema describes a control point and its properties.
### Request
```
GET /redfish/v1/Chassis/{ChassisId}/Controls/{ControlId}
Content-Type: application/json
```
```
POST /redfish/v1/Chassis/{ChassisId}/Controls/{ControlId}
Content-Type: application/json
```
### Sample request body for set power limit
```
{
    "ControlMode" : "Override",
    "SetPoint" : 5000
}
```
```
{
    "ControlMode" : "Automatic"
}
```
```
{
    "ControlMode" : "Manual"
}
```

### Response 
The response of the request will be in JSON format. The properties are mentioned in the following table.
### Properties table for Control Power limit
	|Name|Type|Read Only|Description|Dbus Command|
	|-|-|-|-|-|
	|@odata.type|String|True|The value of this property shall be an absolute URL that specifies the type of the resource and it shall be of the form defined in the Redfish specification. The type values for each Redfish Entity gives the schema it follows and is mentioned in Redfish API List under Schema column.
	|@odata.id|String|True|The value of this property shall be the unique identifier for the resource and it shall be of the form defined in the Redfish specification.
	|Id|String|True|Resource Identifier.
	|Name|String|True|Name of the Resource.
	|PhysicalContext|String|True|This property shall contain a description of the affected component or region within the equipment to which this control applies.
	|ControlType|String|True|This property shall contain the type of the control.
	|ControlMode|String|False|This property shall contain the operating mode of the control.|busctl get-property com.Nvidia.Powermanager /xyz/openbmc_project/control/power/CurrentChassisLimit xyz.openbmc_project.Control.Power.Mode PowerMode|
	|SetPoint|Number|False|This property shall contain the desired set point control value. The units shall follow the value of SetPointUnits.|busctl get-property com.Nvidia.Powermanager /xyz/openbmc_project/control/power/CurrentChassisLimit xyz.openbmc_project.Control.Power.Cap PowerCap|
	|SetPointUnits|String|True|This property shall contain the units of the control's set point.
	|AllowableMax|Number|True|The maximum possible setting for this control.|busctl get-property com.Nvidia.Powermanager /xyz/openbmc_project/control/power/CurrentChassisLimit xyz.openbmc_project.Control.Power.Cap MaxPowerCapValue|
	|AllowableMin|Number|True|The minimum possible setting for this control.|busctl get-property com.Nvidia.Powermanager /xyz/openbmc_project/control/power/CurrentChassisLimit xyz.openbmc_project.Control.Power.Cap MinPowerCapValue|
	|Sensor|object|True|This property shall contain the Sensor excerpt directly associated with this control. The value of the DataSourceUri property shall reference a resource of type Sensor. This property shall not be present if multiple sensors are associated with a single control.|busctl get-property xyz.openbmc_project.VirtualSensor /xyz/openbmc_project/sensors/power/total_power xyz.openbmc_project.Sensor.Value Value|

### Flow Diagram



	+--------------------------------------------+                                                     +----------------------------------------+
	|                Redfish                     |                                                     |                    BMC                 |
	|                                            |                                                     |                                        |
	|  +-------------------------------------+   |                                                     |                                        |
	|  | Get /redfish/v1/Chassis/<str>/      |   |               Dbus Get Call                         |    +-------------------------------+   |
	|  | Controls/PowerLimit                 |<--+-----------------------------------------------------+----+  Nvidia-Power-Manager         |   |
	|  |                                     |   |                                                     |    |                               |   |
	|  | Get the modes and power related     |   |                                                     |    |ObjectPath                     |   |
	|  | Values(MaxP,MaxQ,PowerCap,PowerMode,|<--+--------------+                             +--------+--->|   /xyz/openbmc_project/control|   |
	|  | Total Power)                        |   |              |                             |        |    |    /power/CurrentChassisLimit |   |
	|  |                                     |   |              |                             |        |    |                               |   |
	|  +-------------------------------------+   |              |                             |        |    |Properties                     |   |
	|                                            |              |                             |        |    |    MaxPowerCapValue           |   |
	|                                            |              |                             |        |    |    MinPowerCapValue           |   |
	|  +-------------------------------------+   |              |                             |        |    |    PowerCap                   |   |
	|  |                                     |   |              |                             |        |    |    PowerMode                  |   |
	|  |                                     |   |              |                             |        |    |                               |   |
	|  |Post /redfish/v1/Chassis/<str>/      |   |              |                             |        |    +-------------------------------+   |
	|  | Controls/PowerLimit                 |   |              |                             |        |                                        |
	|  |                                     |   |              |                             |        |                                        |
	|  +----+---------------------+----------+   |              |                             |        |    +-------------------------------+   |
	|       |                     |              |              |                             |        |    |  Phsosphor-Virtual-Sensor     |   |
	|       |                     |              |              |      Dbus Get Call          |        |    |                               |   |
	|       |                     |              |              +-----------------------------+--------+----+                               |   |
	| +-----v---------+ +---------v---------+    |                                            |        |    |                               |   |
	| |               | | Mode = Manual     |    |                                            |        |    |ObjectPath                     |   |
	| |Mode = Override| |         or        |    |                   Dbus Set Call            |        |    |   /xyz/openbmc_project/sensors|   |
	| |               | |        Automatic  +----+------------------------------------------->|        |    |    /power/total_power         |   |
	| +-----+---------+ +-------------------+    |                                            |        |    |                               |   |
	|       |                                    |                                            |        |    |Properties                     |   |
	|       |                                    |                                            |        |    |     Value                     |   |
	|  +----v---------+  Valid set  Point        |                   Dbus Set Call            |        |    |                               |   |
	|  |              +--------------------------+--------------------------------------------+        |    |                               |   |
	|  | Set Point    |                          |                                                     |    |                               |   |
	|  +----+---------+                          |                                                     |    |                               |   |
	|       |                                    |                                                     |    |                               |   |
	|    Not valid                               |                                                     |    |                               |   |
	|       |                                    |                                                     |    +-------------------------------+   |
	|  +----v----------+                         |                                                     |                                        |
	|  |               |                         |                                                     |                                        |
	|  |  Failure      |                         |                                                     |                                        |
	|  +---------------+                         |                                                     |                                        |
	|                                            |                                                     |                                        |
	+--------------------------------------------+                                                     +----------------------------------------+

- User can change the power mode and power cap value through redfish POST call (/redfish/v1/Chassis/Luna_Motherboard/Controls/PowerLimit).
- Supporting modes are Override,Automatic and Manual.
- If the modes are not valid POST call will fail.
- If user setting the mode as Automatic, It will set the "MaxPowerCapValue" to "powercap" property (xyz.openbmc_project.Control.Power.Mode.PowerMode.MaximumPerformance). If the user setting the mode as Manual, It will set the "MinPowerCapValue" to "powercap" property (xyz.openbmc_project.Control.Power.Mode.PowerMode.PowerSaving).
- For setting mode as override two arguments mandatory(ControlMode and Setpoint) otherwise POST call will fail. Set point should be between "MaxPowerCapValue" and "MinPowerCapValue" otherwise POST call will fail.

### Control Modes:

	|Mode|Description|
	|-|-|
	|Automatic|Automatically adjust control to meet the set point.
	|Disabled|The control has been disabled.
	|Manual|No automatic adjustments are made to the control.
	|Override|User override of the automatic set point value.
