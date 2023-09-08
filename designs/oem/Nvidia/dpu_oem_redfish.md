# Bluefield System Dpu Oem via Redfish

Author: Ben Peled

Created: September 7, 2023

## Problem Description
The objective of this OEM features is to enable user to manage dpu over redfish.
Examples: such as enable/disable host rshim, enable/disable external host privileges, DPU OS state, Smart NIC mode

## Background and References
Using Ncsi over mctp over smbus read, enbale or disable NIC features
Reading and setting value sequence attached

1. mctp set eid on the nic using mctp-ctrl

2. ncsi search for ncsi package

3. ncsi clear state for used channels

4. dbus init initial values

5. read values and set values

6. reset NIC in order to update settings

7. mctp set eid once again

## Limitations
1. Backend must handle nic reset
2. Related privileges cannot contradict each other

## Proposed Design
The general flow is described in the following sequence diagram:
```
┌───────────────────────────────────────────────────────────────────────────────┐
│                                                                               │
│                                                                               │
│                                   BMC                                         │
│                                                                               │
│                                                                               │
│                 ┌─────────────────────────────────────────────┐               │
│                 │                                             │               │
│                 │                 redfish                     │               │
│                 │                                             │               │
│                 └──────────────┬─────────▲────────────────────┘               │
│                                │         │                                    │
│                                │         │                                    │
│                 ┌──────────────▼─────────┴────────────────────┐               │
│                 │                                             │               │
│                 │              dbus object                    │               │
│                 │                                             │               │
│                 └──────────────┬─────────▲────────────────────┘               │
│                                │         │                                    │
│                 ┌──────────────▼─────────┴────────────────────┐               │
│                 │                                             │               │
│                 │                nic oem                      │               │
│                 │                                             │               │
│                 └──────────────┬──────────▲───────────────────┘               │
│                                │          │                                   │
│                                │          │                                   │
│                 ┌──────────────▼──────────┴───────────────────┐               │
│                 │                                             │               │
│                 │               nsci mctp                     │               │
│                 │                                             │               │
│                 └─────────────┬───────────▲───────────────────┘               │
│                               │           │                                   │
│                               │           │                                   │
│                 ┌─────────────▼───────────┴───────────────────┐               │
│                 │                                             │               │
│                 │                  mctp                       │               │
│                 │                                             │               │
│                 └─────────────┬───────────▲───────────────────┘               │
│                               │           │                                   │
│                 ┌─────────────▼───────────┴───────────────────┐               │
│                 │                                             │               │
│                 │                 smbus                       │               │
│                 │                                             │               │
│                 └───────────┬──────────────────▲──────────────┘               │
│                             │                  │                              │
└─────────────────────────────┼──────────────────┼──────────────────────────────┘
                              │                  │
           ┌──────────────────▼──────────────────┴──────────────────────┐
           │                                                            │
           │                                                            │
           │                                                            │
           │                                                            │
           │                         NIC                                │
           │                                                            │
           │                                                            │
           │                                                            │
           │                                                            │
           └────────────────────────────────────────────────────────────┘
```

### Reading Oem DPU properties
A new OEM properties will be added to the Systems schema under Bluefield/Oem/Nvidia:
```
https://<bmc_ip>/redfish/v1/Systems/Bluefield
{
  ....
  "Oem": {
    "Nvidia": {
      "Connectx": {
        "ExternalHostPrivilege": {
          "@odata.id": "/redfish/v1/Systems/Bluefield/Oem/Nvidia/Connectx/ExternalHostPrivileges"
        },
        "StrapOptions": {
          "@odata.id": "/redfish/v1/Systems/Bluefield/Oem/Nvidia/Connectx/StrapOptions"
        }
      },
      "HostRshim": "Disabled",
      "Mode": "NicMode",
      "OsState": "OsStarting"
    }
  },
  ...
}
```
curl -k -H "X-Auth-Token: <token>" -X GET https://$bmc/redfish/v1/Systems/Bluefield

### Actions Set DpuMode and HostRshim
A new OEM Actions will be added to the Systems schema under Bluefield/Actions/Oem/:
```
https://<bmc_ip>/redfish/v1/Systems/Bluefield
{
  ...
  "Actions": {
    ...
    "Oem": {
      "Nvidia": {
        "#HostRshim.Set": {
          "Parameters": [
            {
              "AllowableValues": [
                "Disabled",
                "Enabled"
              ],
              "DataType": "String",
              "Name": "HostRshim",
              "Required": true
            }
          ],
          "target": "/redfish/v1/Systems/Bluefield/Oem/Nvidia/Actions/HostRshim.Set"
        },
        "#Mode.Set": {
          "Parameters": [
            {
              "AllowableValues": [
                "NicMode",
                "DpuMode"
              ],
              "DataType": "String",
              "Name": "Mode",
              "Required": true
            }
          ],
          "target": "/redfish/v1/Systems/Bluefield/Oem/Nvidia/Actions/Mode.Set"
        }
      }
    }
  ...
```
curl -k -H "X-Auth-Token: <token>"" -X GET https://$bmc/redfish/v1/Systems/Bluefield

Host Rshim Enable
curl -k -H "X-Auth-Token: <token>" -H "Content-Type: application/json" -X POST -d
'{"HostRshim":"Enabled"}'
https://$bmc/redfish/v1/Systems/Bluefield/Oem/Nvidia/Actions/HostRshim.Set

Host Rshim Disable
curl -k -H "X-Auth-Token: <token>" -H "Content-Type: application/json" -X POST -d
'{"HostRshim":"Disabled"}'
https://$bmc/redfish/v1/Systems/Bluefield/Oem/Nvidia/Actions/HostRshim.Set

Set DPU Mode
curl -k -H "X-Auth-Token:<token>" -H "Content-Type: application/json" -X POST -d
'{"Mode":DpuMode"}'
https://$bmc/redfish/v1/Systems/Bluefield/Oem/Nvidia/Actions/Mode.Set


Set Nic Mode
curl -k -H "X-Auth-Token:<token>" -H "Content-Type: application/json" -X POST -d
'{"Mode":NicMode"}'
https://$bmc/redfish/v1/Systems/Bluefield/Oem/Nvidia/Actions/Mode.Set
### Reading Strap options
```
https://<bmc_ip>/redfish/v1/Systems/Bluefield/Oem/Nvidia/Connectx/StrapOptions
{
  "Mask": {
    "2PcoreActive": "Enabled",
    "CoreBypassN": "Enabled",
    "DisableInbandRecover": "Disabled",
    "Fnp": "Enabled",
    "OscFreq0": "Enabled",
    "OscFreq1": "Enabled",
    "PciPartition0": "Enabled",
    "PciPartition1": "Enabled",
    "PciReversal": "Enabled",
    "PrimaryIsPcore1": "Enabled",
    "SocketDirect": "Enabled"
  },
  "StrapOptions": {
    "2PcoreActive": "Disabled",
    "CoreBypassN": "Enabled",
    "DisableInbandRecover": "Disabled",
    "Fnp": "Enabled",
    "OscFreq0": "Enabled",
    "OscFreq1": "Enabled",
    "PciPartition0": "Disabled",
    "PciPartition1": "Disabled",
    "PciReversal": "Disabled",
    "PrimaryIsPcore1": "Disabled",
    "SocketDirect": "Disabled"
  }
}
```
curl -k -H "X-Auth-Token: <token>"  -X GET https://${bmc}/redfish/v1/Systems/Bluefield/Oem/Nvidia/Connectx/StrapOptions

### Reading External Host privileges
```
https://<bmc_ip>/redfish/v1/Systems/Bluefield/Oem/Nvidia/Connectx/ExternalHostPrivileges
{
  "Actions": {
    "#ExternalHostPrivilege.Set": {
      "Parameters": [
        {
          "AllowableValues": [
            "Default",
            "Enabled",
            "Disabled"
          ],
          "DataType": "String",
          "Name": "HostPrivPccUpdate",
          "Required": false
        },
        {
          "AllowableValues": [
            "Default",
            "Enabled",
            "Disabled"
          ],
          "DataType": "String",
          "Name": "HostPrivNvPort",
          "Required": false
        },
        {
          "AllowableValues": [
            "Default",
            "Enabled",
            "Disabled"
          ],
          "DataType": "String",
          "Name": "HostPrivNvGlobal",
          "Required": false
        },
        {
          "AllowableValues": [
            "Default",
            "Enabled",
            "Disabled"
          ],
          "DataType": "String",
          "Name": "HostPrivNvInternalCpu",
          "Required": false
        },
        {
          "AllowableValues": [
            "Default",
            "Enabled",
            "Disabled"
          ],
          "DataType": "String",
          "Name": "HostPrivNicReset",
          "Required": false
        },
        {
          "AllowableValues": [
            "Default",
            "Enabled",
            "Disabled"
          ],
          "DataType": "String",
          "Name": "HostPrivNvHost",
          "Required": false
        },
        {
          "AllowableValues": [
            "Default",
            "Enabled",
            "Disabled"
          ],
          "DataType": "String",
          "Name": "HostPrivFwUpdate",
          "Required": false
        },
        {
          "AllowableValues": [
            "Default",
            "Enabled",
            "Disabled"
          ],
          "DataType": "String",
          "Name": "HostPrivFlashAccess",
          "Required": false
        }
      ],
      "target": "/redfish/v1/Systems/Bluefield/Oem/Nvidia/Connectx/ExternalHostPrivileges/Actions/ExternalHostPrivileges.Set"
    }
  },
  "ExternalHostPrivilege": {
    "HostPrivFlashAccess": "Default",
    "HostPrivFwUpdate": "Enabled",
    "HostPrivNicReset": "Enabled",
    "HostPrivNvGlobal": "Enabled",
    "HostPrivNvHost": "Enabled",
    "HostPrivNvInternalCpu": "Enabled",
    "HostPrivNvPort": "Enabled",
    "HostPrivPccUpdate": "Default"
  }
}
```
curl -k -H "X-Auth-Token: <token>"  -X GET https://${bmc}/redfish/v1/Systems/Bluefield/Oem/Nvidia/Connectx/ExternalHostPrivileges
### Actions Set External Host privileges

curl -k -H "X-Auth-Token:<token>" -H "Content-Type: application/json" -X POST -d
'{"HostPrivFwUpdate":"Default","HostPrivNvGlobal":"Enabled" ...}'
https://$bmc/redfish/v1/Systems/Bluefield/Oem/Nvidia/Connectx/ExternalHostPrivileges/Actions/ExternalHostPrivileges.Set