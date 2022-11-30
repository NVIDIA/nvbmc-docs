# Redfish APIs Design for ThermalSubsystem Management

This document describes the design for ThermalSubsystem Management under Chassis URI.

Author:
  sandeepap@ami.com

Created:
  October 10, 2022
## 1. ThermalSubsystem 
This ThermalSubsystem  contains the definition for the thermal subsystem of a chassis.

### Request
```
GET https://{{ip}}/​redfish/​v1/​Chassis/​{ChassisId}/​ThermalSubsystem
Content-Type: application/json
```

### Response
The response of the request will be in JSON format. The properties are mentioned in the following table.

### Properties table for ThermalSubsystem
|Name|Type |Read only|Description|
| - | - | - | - |
| Fans { | object| - | The link to the collection of fans within this subsystem. Contains a link to a resource |
| @odata.id | string | read-only | Link to Collection of Fan. It shall be of the form defined in the Redfish specification (or Refer 2.Fans Collection)|
| } 

## 2.Fans Collection
The Fan Collection describes a cooling fan unit for a chassis.

### Request
```
GET https://{{ip}}/​redfish/​v1/​Chassis/​{ChassisId}/​ThermalSubsystem/​Fans/​{FanId}
Content-Type: application/json
```

### Response
The response of the request will be in JSON format. The properties are mentioned in the following table.

### Properties table for Fans Collection :
|Name|Type|Read Only|Description
|-|-|-|-|
|@odata.id|string|read-only|Link to the Fan ID|
|@odata.type|||The value of this property shall be an absolute URL that specifies the type of the resource and it shall be of the form defined in the Redfish specification.|	
|MemberId|String|True|Resource Identifier|
|MaxReadingRange|	integer	|read-only(null)|Maximum value for this sensor.|
|MinReadingRange|	integer	|read-only(null)|Minimum value for this sensor.|
|Name|String|True|Name of the Resource|
|Reading|number|read-only (null)|The sensor value.|
|ReadingUnits |	string (enum)|read-only(null)|The units in which the fan reading and thresholds are measured. For the possible property values, see ReadingUnits in Property details.|
|SpeedPercent {	|object(excerpt)||The fan speed reading. This object is an excerpt of the Sensor resource located at the URI shown in DataSourceUri.|
|DataSourceUri|string(URI)|read-only (null)|The value of this property shall be an absolute URL  that provides the data for this sensor. And this sensor can be derived from EntityManager.|
|}|

### Example Response
```
{
    @odata.id": "/redfish/v1/Chassis/1u/ThermalSubsystem#/Fans/0",
    "@odata.type": "#Fan.v1_1_0.Fan",
    "MaxReadingRange": 100,
    "MinReadingRange": 0,
    "MemberId": "Bay1",
    "Name": "Fan Bay 1",
    "Reading": 45,
    "ReadingUnits": "Percent",
    "Status": {
        "State": "Enabled",
        "Health": "OK"
    },
    "SpeedPercent": {
        "DataSourceUri": "/redfish/v1/Chassis/1U/Sensors/FanBay1"
    },
}
```
```
{
"@odata.id": "/redfish/v1/Chassis/Luna_Motherboard/ThermalSubsystem",
"@odata.type": "#ThermalSubsystem.v1_0_0.ThermalSubsystem",
"Fans": [
{
"@odata.id": "/redfish/v1/Chassis/Luna_Motherboard/ThermalSubsystem#/Fans/0",
"@odata.type": "#Fan.v1_1_0.Fan",
"MaxReadingRange": 100,
"MemberId": "Pwm_PSU0_Fan_1",
"MinReadingRange": 0,
"Name": "Pwm PSU0 Fan 1",
"Reading": 80,
"ReadingUnits": "Percent",
"SpeedPercent": {
"DataSourceUri": "/redfish/v1/Chassis/Luna_Motherboard/Sensors/Pwm_PSU0_Fan_1"
}
}
```
In the the above example response "Bay1" Describes the example for the SensorId and "Fan Bay 1" Describes the example for SensorName.
And ThermalSubsystem# contains the different fan sensors contained within the chassis.

### Request
The following URI is already implemented and the Patch support is implemented yet to Merge(MR:75) for the "Thresholds" on Redfish  (Refer Sample request body)
```
PATCH https://{{ip}}/​redfish/​v1/​Chassis/​{ChassisId}/​Sensors/​{SensorId}
Content-Type: application/json  
```
### Properties to Patch
|Name|Type|Read Only|Description
|-|-|-|-|
|"Thresholds": {|object||The set of thresholds defined for this sensor.|
| LowerCaution {}|object||The value at which the reading is below normal range. For more information about this property, see Threshold in Property Details.|
|  LowerCautionUser (v1.2+) {}|object||The value at which the reading is below normal range. For more information about this property, see Threshold in Property Details.|
|  LowerCritical {}|object||The value at which the reading is below normal range but not yet fatal. For more information about this property, see Threshold in Property Details.|
|LowerCriticalUser (v1.2+) {}|object||The value at which the reading is below normal range but not yet fatal. For more information about this property, see Threshold in Property Details.|
|LowerFatal {}|object||The value at which the reading is below normal range and fatal. For more information about this property, see Threshold in Property Details.|
|UpperCaution {}|object||The value at which the reading is above normal range. For more information about this property, see Threshold in Property Details.|
|UpperCautionUser (v1.2+) {}|object||The value at which the reading is above normal range. For more information about this property, see Threshold in Property Details.|
|UpperCritical {}|object||The value at which the reading is above normal range but not yet fatal. For more information about this property, see Threshold in Property Details.|
|UpperCriticalUser (v1.2+) {}|object||The value at which the reading is above normal range but not yet fatal. For more information about this property, see Threshold in Property Details.|
|UpperFatal {}|object||The value at which the reading is above normal range and fatal. For more information about this property, see Threshold in Property Details.|
|}||||
### Sample request body for setting the Sensor(i.e FanID) :
Here the threshold value to be modified for the particular sensor(i.e FanID)
### Example 1 
```
 {
 "Thresholds": {
        "LowerCritical":{
            "Reading": 2000
                         }
              }
}
```
### Example 2 
```
{
"Thresholds": {
    "LowerCaution": {
                "Reading": 5.0
                    },
    "UpperCaution": {
                "Reading": 55.0
                    },
    "UpperCritical": {
                "Reading": 58.0
                     }
}
```
### Reason to patch Threshold value and backend effect :
Sensors and fans are categorized on zones.
Zone 0, Zone 1, Zone 2 as per config file in bmc. Each Zone will have respective Sensors and fans but there are set of Temperature Sensors that is present in all the Zones.

Sensor state        -       action
If Sensor is UpperNonCritical       - PWM value for respective zone will get to 80%  and other Zone PWM values should not get changed instead event should gets generated.

If Sensor is UpperCritical         - Host should gets powered off, till Sensors state changes Host should not gets Powered ON. Event should gets generated.

If sensor is lowenNonCritical        - Only event should gets generated.

For UpperNonCritical Sensor state   -  Sensors those were present in all the Zones from these sensors if any of the Sensor's state reaches to UpperNonCritical state then all PWM value should be increased in all the Zones as these Sensors were configured in all three zones.

#### Dbus Command to patch Threshold value
```
Command :

Example :
busctl get-property xyz.openbmc_project.PSUSensor  /xyz/openbmc_project/sensors/power/PWR_PSU0 xyz.openbmc_project.Sensor.Threshold.Critical CriticalLow

DBUS Introspect (Ex : Luna_Motherboard)  :

busctl introspect xyz.openbmc_project.PSUSensor /xyz/openbmc_project/sensors/power/PWR_PSU0
```

Similarly we have properties 
2. CriticalHigh
3. WarningHigh
4. WarningLow
