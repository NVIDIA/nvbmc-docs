# Validation of Fru Device Information under Chassis Redfish URI 

**Document Purpose:** Validation of Fru Device Informations like Manufacturer, SerialNumber, ProductPartNumber and SerialNumber under Chassis  Redfish URI .

**Audience:** Developer/Tester familiar Web Redfish UI.

**Prerequisites:**  Web UI access.

**Author:** Sandeep Patil

**Created:** 2022-08-18

### Validation :
**1. Launch the below Redfish URI to see the list of Chassis boards**
https://{{IP}}}}/redfish/v1/Chassis

  **Example :** https://10.19.102.17/redfish/v1/Chassis  
  
  **Response:**
 
  ```
{
    "@odata.id": "/redfish/v1/Chassis",
    "@odata.type": "#ChassisCollection.ChassisCollection",
"Members": [
{
    "@odata.id": "/redfish/v1/Chassis/Cpld0"
},
{
    "@odata.id": "/redfish/v1/Chassis/Cpld1"
},
{
    "@odata.id": "/redfish/v1/Chassis/Luna_GPU_Baseboard"
},
{
    "@odata.id": "/redfish/v1/Chassis/Luna_IOE0"
},
{
    "@odata.id": "/redfish/v1/Chassis/Luna_IOE1"
},
{
    "@odata.id": "/redfish/v1/Chassis/Luna_Midplane"
},
{
    "@odata.id": "/redfish/v1/Chassis/Luna_Motherboard"
},
{
    "@odata.id": "/redfish/v1/Chassis/Luna_PDB"
},
{
    "@odata.id": "/redfish/v1/Chassis/Luna_Software"
}
],
"Members@odata.count": 9,
"Name": "Chassis Collection"
}
 ```

**2. Click on Any of the available ChassisID URI** 

### Request

 ```
 GET https://{{IP}}/redfish/v1/Chassis/{{ChassisID}}
Content-Type: application/json
https://{{IP}}/redfish/v1/Chassis/{{ChassisID}}
 ```
**Example :** Luna_GPU_Baseboard
**URI Example:**  https://10.19.102.17/redfish/v1/Chassis/Luna_GPU_Baseboard

Note : Here "Luna_GPU_Baseboard" is one of the ChassisID which contains the FRU Information. Hence the Fru Information is displayed under the ChassisID itself.

### Response :
  ```
{
"@odata.id": "/redfish/v1/Chassis/Luna_GPU_Baseboard",
"@odata.type": "#Chassis.v1_16_0.Chassis",
"Actions": {
"#Chassis.Reset": {
    "@Redfish.ActionInfo":
    "/redfish/v1/Chassis/Luna_GPU_Baseboard/ResetActionInfo",
    "target": 
    "/redfish/v1/Chassis/Luna_GPU_Baseboard/Actions/Chassis.Reset"
}
},
"Assembly": {
     "@odata.id": "/redfish/v1/Chassis/Luna_GPU_Baseboard/Assembly"
},
"ChassisType": "",
"EnvironmentMetrics": {
        "@odata.id": "/redfish/v1/Chassis/Luna_GPU_Baseboard/EnvironmentMetrics"
},
"Id": "Luna_GPU_Baseboard",
"Links": {
          "ComputerSystems": [
           {
               "@odata.id": "/redfish/v1/Systems/system"
           }
     ],
         "ManagedBy": [
          {
             "@odata.id": "/redfish/v1/Managers/bmc"
          }
     ]
        },
"Manufacturer": "NVIDIA",
"Model": "DGXA100",
"Name": "Luna_GPU_Baseboard",
"PCIeDevices": {
        "@odata.id": "/redfish/v1/Chassis/Luna_GPU_Baseboard/PCIeDevices"
    },
"PCIeSlots": {
        "@odata.id": "/redfish/v1/Chassis/Luna_GPU_Baseboard/PCIeSlots"
    },
"PartNumber": "920-23687-2530-000",
"Power": {
        "@odata.id": "/redfish/v1/Chassis/Luna_GPU_Baseboard/Power"
        },
"PowerState": "On",
"PowerSubsystem": {
        "@odata.id": "/redfish/v1/Chassis/Luna_GPU_Baseboard/PowerSubsystem"
    },
"Sensors": {
        "@odata.id": "/redfish/v1/Chassis/Luna_GPU_Baseboard/Sensors"
    },
"SerialNumber": "1663221600088",
"Status": {
"Health": "OK",
"HealthRollup": "OK",
"State": "Enabled"
},
"Thermal": {
        "@odata.id": 
        "/redfish/v1/Chassis/Luna_GPU_Baseboard/Thermal"
},
"ThermalSubsystem": {
        "@odata.id": 
        "/redfish/v1/Chassis/Luna_GPU_Baseboard/ThermalSubsystem"
}
}
  ```

The response of the request will be in JSON format. The properties are mentioned in the following table.

  *a)Manufacturer
  b)Model 
  c)PartNumber
  d)SerialNumber*
  
 **Properties table for Fru Information of the Chassis:**
  
|Name|Type|Attributes|Description|Dbus Command|
|-|-|-|-|-|
|Manufacturer|string|read-only (null)|The manufacturer of this chassis.|busctl get-property xyz.openbmc_project.EntityManager /xyz/openbmc_project/inventory/system/board/{{ChassisID}} xyz.openbmc_project.Inventory.Decorator.Asset Manufacturer|
|Model|string|read-only (null)|The model number of the chassis.|busctl get-property xyz.openbmc_project.EntityManager /xyz/openbmc_project/inventory/system/board/{{ChassisID}} xyz.openbmc_project.Inventory.Decorator.Asset Model|
|PartNumber|string|read-only (null)|The part number of the chassis.|busctl get-property xyz.openbmc_project.EntityManager /xyz/openbmc_project/inventory/system/board/{{ChassisID}} xyz.openbmc_project.Inventory.Decorator.Asset PartNumber|
|SerialNumber|string|read-only (null)|The serial number of the chassis.|busctl get-property xyz.openbmc_project.EntityManager /xyz/openbmc_project/inventory/system/board/{{ChassisID}} xyz.openbmc_project.Inventory.Decorator.Asset SerialNumber|

  
 
**3. Validate the Fru Information from the response comparing with the ipmitool fru list :**

### ipmitool fru list response :
 ```
 FRU Device Description : GB_FRU (ID 4)
 Chassis Type          : Rack Mount Chassis
 Chassis Part Number   : 920-23687-2530-000
 Board Mfg Date        : Tue Aug 17 06:35:00 2021 UTC
 Board Mfg             : NVIDIA
 Board Product         : DGXA100
 Board Serial          : 1573120520188
 Board Part Number     : 699-24612-1000-302
 Board Extra           : 01
 Product Manufacturer  : NVIDIA
 Product Name          : DGXA100
 Product Part Number   : 920-23687-2530-000
 Product Version       : AF
 Product Serial        : 1663221600088
 ```
**Observation** : Here in the above response we are able to match the fru information.
Fru device connected is **GB_FRU** 

*a)Manufacturer as Product Manufacturer
  b)Model as Product Name
  c)PartNumber as Product Part Number
  d)SerialNumber as Product Serial*

Note : Above Table information is based on the example ChassisID : "Luna_GPU_Baseboard", similarly we have different ChassisIDs 

