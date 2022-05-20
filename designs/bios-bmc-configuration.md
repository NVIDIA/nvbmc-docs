# BIOS BMC Communication and Configuration

Author: Tejas Pratap Patil

Other contributors: Chandrasekhar T, Gayathri L

Created: May 20th, 2022

## Problem Description
Redfish BIOS-BMC Communication uses Host Interface for most of their communication though it uses IPMI Commands for the initial Authentication. It adheres Redfish Host Interface Specification 1.3.0.
Host Interface is the standardized interface specified by Redfish DMTF Community that enables Redfish based communication between the Host CPU and the Redfish service in the Mangement unit.
In this communication Redfish Host and the service in BMC does not require any external interface and is in addition to the Redfish services available in out of band network.
This interface is used in both Pre-boot(Firmware) stage and by the drivers and applications in the Host Operating System without use of external networking.
This document explains the protocol used in the communication flow, beginning with the initial handshake to the final stages, with intermediate Redfish calls to transfer the BIOS Attributes information from BIOS to BMC.

## Requirements
### Requirements for BIOS-BMC Communication
1) The BIOS should able to communicate with the BMC network stack (In this design BMC USB interface being used.).
2) The BIOS should able to GET/PATCH/POST the corrosponding Redfish Requests.
### Requirements for BIOS FWC
1) From Redfish Bios URI, able to perform actions like update the BIOS Password, Reset the BIOS.
2) Should be able to push/pull updates to BIOS settings.
3) The OOB user also should be able to modify the BIOS settings.

## Proposed Design

```
+-----------------------------------------------------------------------------------------------------------------+
|                                                                                                                 |
|                                                                                                                 |
|                                                                                                                 |
| +-------------+        +-------------+       +--------------------------------+      +-------+                  |
| |             |  LAN   |             |       |   RBC daemon                   |      |       |    +----------+  |
| |  BIOS/HOST  +<-over->+   REDFISH   +<Dbus->+                                |      |       |    |Web client|  |
| |             |  USB   |             |       |  Provide following Methods     |      |       |    |          |  |
| +-------------+        +-------------+       |     -SetAttribute()            |      |       |    +----^-----+  |
|                                              |     -GetAttribute()            |      |       |         |        |
|                                              |     -VerifyPassword()          |      |       |        LAN       |
|                                              |     -ChangePassword()          |      |       |         |        |
|                                              |                                |      |Redfish|    +----V-----+  |
|                                              | Properties                     +<Dbus>+  API  |    |Redfish & |  |
|                                              |     -BaseBIOSTable             |      |       +<-->+BMCWeb    |  |
|                                              |     -PendingAttributes         |      |       |    +----^-----+  |
|                                              |     -ResetBIOSSettings         |      |       |         |        |
|                                              |     -IsPasswordInitDone        |      |       |         |        |
|                                              |                                |      |       |    +----V-----+  |
|                                              |                                |      |       |    | Redfish  |  |
|                                              |                                |      |       |    |  Host    |  |
|                                              |                                |      |       |    | Interface|  |
|                                              +--------------------------------+      +-------+    +----------+  |
|                                                                                                                 |
+-----------------------------------------------------------------------------------------------------------------+
```

### BIOS BMC Communication
Redfish Host Interface supports Network HI-Redfish HTTPS requests/responses over a TCP/IP network connection between a Host and Redfish Service. The following main modules are involved in the communication as depicted in the block diagram below:

```
+------------------+                  +------------------+
|     +------+     |                  |     +------+     |
|     | BIOS |     |                  |     | BMC  |     |
|     +------+     <------------------>     +------+     |
|                  |  Host Interface  |                  |
|   Host Computer  <------------------>     Redfish      |
|     System       |                  |     Service      |
|                  |                  |                  |
+------------------+                  +------------------+
```
```
┌────┐                    ┌───────┐                       ┌───┐                ┌──────┐
│BIOS│                    │Redfish│                       │RBC│                │Client│
└──┬─┘                    └───┬───┘                       └─┬─┘                └───┬──┘
   │                          │                             │                      │
   │  HTTPS Request (Inband)  │                             │                      │
   ───────────────────────────>        Update Values        │                      │
   │                          ──────────────────────────────>                      │
   <─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─                             │                      │
   │         200 OK           │                             │                      │
   │                          │                  HTTPS Request (OOB)               │
   │                          <────────────────────────────────────────────────────
   │                          ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─>
   │                          │                         200 OK                     │
   │                          │                             │                      │
```
BIOS communicates with BMC over USB Ethernet interface IP and pushes the firmware configurations (BIOS Attributes and Attribute Registry) onto BMC. Once transferred, these data are available for BMC Out of band users.

#### Authentication
The communication between the Redfish (BMC) and BIOS are done in below way:-
* Basic Auth:-
BIOS Initiates BootStrap IPMI Command to get the Credentials.
BIOS communicates to BMC with these account credentials for subsequent HTTPS Requests.

#### Initialization

```
┌────┐                                            ┌───┐
│BIOS│                                            │BMC│
└──┬─┘                                            └─┬─┘
   │                                                │    ╔════════════════════════════╗
   │               Intial Handshake                 │    ║ BIOS gets the BMC network  ║
   ─────────────────────────────────────────────────>    ║ interface information and  ║
   <─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─    ║ establish basic BIOS-BMC   ║
   │      <<return>> HI Interface Config Info       │    ║ communication with that    ║
   │                                                │    ║ interface.                 ║
   │                                                │    ╚════════════════════════════╝
   │                                                │    ╔════════════════════════════╗
   │Send IPMI BootStrap Account Credentials command │    ║ BMC generates credentials  ║
   ─────────────────────────────────────────────────>    ║ (UserName & Password), and ║
   <─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─    ║ send it back to BIOS.      ║
   │         <<return>> Credentials Info            │    ╚════════════════════════════╝
   │                                                │
```

1. BIOS checks for the SMBIOS type 42 related infromation as well as it checks for network interface configuration details through inital handshake mechanism which are required to establish connection with BMC network interface. (As per this design it will be mapped  to BMC USB interface.)

2. BIOS first reads the BMC USB interface information like VendorId, ProductId with the help of Get USB description IPMI Oem command (NetFn 3Ch, Command 30h).

3. Then BIOS reads the serial number of USB interface with the help of Get Virtual USB Serial Number IPMI Oem command (NetFn 3Ch, Command 31h).

4. Then BIOS reads the Redfish Hostname with the help of Get Redfish Hostname IPMI Oem command (NetFn 3Ch, Command 32h).

5. Then BIOS gets the channel number for the BMC network interface for which HI is configured with the help of Get IPMI Channel Number Oem command (NetFn 3Ch, Command 33h). And with that channel number it will read the network configuration details with the help of standard IPMI command Get LAN Configuration Parameters.

6. Once the handshaking is completed and basic BIOS-BMC communiction established, then BIOS sends Get BootStrap Account Credentials IPMI command (NetFn 2Ch, Command 02h) to get auto generated credentials.

7. BMC generates username and password to be used for further authentication and sends it back to BIOS. This account will only be used by host interface alone and it will not be shown under IPMI user list as well as Redfish AccountService schema. And this account should only be used to push the BIOS attributes to Bios schema.

8. Upon normal completion of this command, the Enabled property within the CredentialBootstrapping property of the host interface resource will be set to false , unless the "Disable credential bootstrapping control" parameter of the command contains the value A5h.

9. And whenever there is host or service reset, then this account will be deleted.


### BIOS-BMC Firmware Configuration
#### First Boot
```
+----------------------------------------+                    +----------------------------------------+                    +-----------------------------------+
|                BIOS                    |                    |                  BMC                   |                    |              REDFISH              |
|                                        |                    |                                        |                    |                                   |
|  +----------------------------------+  |                    | +------------------------------------+ |                    | +-------------------------------+ |
|  | Send the Redfish host interface  |  |                    | |1.Get the complete atttributes data.| |                    | | BIOS Attributes and Attribute | |
|  | BIOS configuration data          |  |-----LanOverUSB---->| |2.Validate and convert into         | |------------------->| | Registry available on         | |
|  | (PUT request to /redfish/v1      |  |-------Redfish----->| |  native to D-bus format.           | |------------------->| | GET /redfish/v1/Systems       | |
|  |  /Systems/system/Bios URI)       |  |                    | |3.Expose the D-bus interface        | |                    | | /system/Bios and              | |
|  +----------------------------------+  |                    | |  (RBC Daemon)                      | |                    | | /redfish/v1/Registries        | |
|                                        |                    | +------------------------------------+ |                    | | /BiosAttributeRegistry URI's  | |
|                                        |                    |                                        |                    | +-------------------------------+ |
|  +----------------------------------+  |                    |                                        |                    |                                   |
|  |  Continue the BIOS boot          |  |                    |                                        |                    |                                   |
|  +----------------------------------+  |                    |                                        |                    |                                   |
+----------------------------------------+                    +----------------------------------------+                    +-----------------------------------+
```

- The first boot can be determined as, there will be empty response for "Attributes" and "AttributeRegistry" redfish properties undder /redfish/v1/System/system/Bios URI.

- BIOS has to send all the Attributes to BMC via Lan Over USB interface with respect to Redfish Host Interface specification. BIOS will make PUT request with BIOS configuration attributes to "/redfish/v1/Systems/system/Bios" URI.

    a. Once all the attributes received, then Bios PUT handler will convert JSON data into DBus format and then it will be exposed to DBus via bios-config-manager on "xyz.openbmc_project.BIOSConfig.Manager" DBus interface and update the "BaseBIOSTable" DBus property.

    b. The "BaseBIOSTable" DBus property expects the input likely in the BIOS Attribute Registry format (eg: busctl set-property xyz.openbmc_project.BIOSConfigManager /xyz/openbmc_project/bios_config/manager xyz.openbmc_project.BIOSConfig.Manager BaseBIOSTable a{s\(sbsssvva\(sv\)\)} 2 "DdrFreqLimit" "xyz.openbmc_project.BIOSConfig.Manager.AttributeType.String" false "Memory Operating Speed Selection" "Force specific Memory Operating Speed or use Auto setting." "Advanced/Memory Configuration/Memory Operating Speed Selection" s "0x00" s "0x0B" 5 "xyz.openbmc_project.BIOSConfig.Manager.BoundType.OneOf" s "auto" "xyz.openbmc_project.BIOSConfig.Manager.BoundType.OneOf" s "2133" "xyz.openbmc_project.BIOSConfig.Manager.BoundType.OneOf" s "2400" "xyz.openbmc_project.BIOSConfig.Manager.BoundType.OneOf" s "2664" "xyz.openbmc_project.BIOSConfig.Manager.BoundType.OneOf" s"2933" "BIOSSerialDebugLevel" "xyz.openbmc_project.BIOSConfig.Manager.AttributeType.Integer" false "BIOS Serial Debug level" "BIOS Serial Debug level during system boot." "Advanced/Debug Feature Selection" x 0x00 x 0x01 1 "xyz.openbmc_project.BIOSConfig.Manager.BoundType.ScalarIncrement" x 1).

    c. Redfish will make use of this dbus interface and property and shows all the BIOS Attributes and BiosAttributeRegistry under "/redfish/v1/Systems/system/Bios/" and "/redfish/v1/Registries/BiosAttributeRegistry" Redfish URI dynamically.


#### Second Boot
```
+----------------------------------------+                    +----------------------------------------+                    +-----------------------------------+
|                BIOS                    |                    |                  BMC                   |                    |              REDFISH              |
|                                        |                    |                                        |                    |                                   |
|  +----------------------------------+  |                    |                                        |                    |                                   |
|  | Get the config status            |  |                    |                                        |                    |                                   |
|  | (Get updated Settings)           |  |                    |  +-----------------------------------+ |                    |  +-----------------------------+  |
|  | - Any config changed or not      |  |<-Get config status-|  | Update the Pending Attributes list| |<----Update BIOS----|  | Update the BIOS Attribute   |  |
|  | - New attribute values exist     |  |<-----from BMC------|  | (PendingAttributes DBus property) | |     Attributes     |  | values (PATCH /redfish/v1   |  |
|  | (GET /redfish/v1/Systems/system  |  |                    |  +-----------------------------------+ |                    |  | /Systems/system/Settings    |  |
|  |   /Bios/Settings)                |  |                    |                                        |                    |  +-----------------------------+  |
|  +----------------------------------+  |                    |                                        |                    |                                   |
|                                        |                    |  +-----------------------------------+ |                    |                                   |
|  +----------------------------------+  |                    |  |                                   | |                    |                                   |
|  | If new attribute value exist     |<-|-----------------------|  Send the new value attributes    | |                    |                                   |
|  |           then                   |  |                    |  |  (Pending Attributes list)        | |                    |                                   |
|  | Get & Update the BIOS variables  |--|------+             |  |                                   | |                    |                                   |
|  |                                  |  |      |             |  +-----------------------------------+ |                    |                                   |
|  +---------------+------------------+  |      |             |                                        |                    |                                   |
|                  |                     |      |             |                                        |                    |                                   |
|                 YES                    |      |             |                                        |                    | +-------------------------------+ |
|                  |                     |      |             |  +----------------------------------+  |                    | | BIOS Attributes and Attribute | |
|   +--------------V------------------+  |      |             |  |                                  |  |                    | | Registry available on         | |
|   | Send the updated data to BMC    |  |      |             |  | Update the BIOS attributes       |  |                    | | GET /redfish/v1/Systems       | |
|   | (PUT request to /redfish/v1/    |------------------------->| (BaseBIOSTable)                  |  |----Get Updated---->| | /system/Bios and              | |
|   |  Systems/system/Bios)           |  |      |             |  | Clear Pending Attributes list    |  |    Attributes      | | /redfish/v1/Registries        | |
|   +---------------------------------+  |      |             |  |                                  |  |                    | | /BiosAttributeRegistry URI's  | |
|                                        |      |             |  +----------------------------------+  |                    | +-------------------------------+ |
|                                        |      |             |                                        |                    |                                   |
|   +---------------------------------+  |      |             |                                        |                    |                                   |
|   | Reset the BIOS for BIOS conf    |  |     NO             |                                        |                    |                                   |
|   | update                          |  |      |             |                                        |                    |                                   |
|   +---------------------------------+  |      |             |                                        |                    |                                   |
|                                        |      |             |                                        |                    |                                   |
|  +----------------------------------+  |      |             |                                        |                    |                                   |
|  |  Continue the BIOS boot          |<--------+             |                                        |                    |                                   |
|  +----------------------------------+  |                    |                                        |                    |                                   |
+----------------------------------------+                    +----------------------------------------+                    +-----------------------------------+
```

- The OOB user can update the BIOS Attribute values with PATCH request to "/redfish/v1/Systems/system/Bios/Settings" Redfish URI.

- In Redfish Bios/Settings PATCH request handler, it will first validate the newly requested attribute values with the help of RBC application for the current attribute types and the valid values. If it matches then only PATCH will be allowed, if in case it doesn't match then PATCH will be failed.

- On PATCH success, the newly requested attribute values will be set to "PendingAttributes" DBus property. The RBC application will persist the "BaseBIOSTable" and "PendingAttributes" DBus properties on BMC/service reset.

- The "PendingAttributes" DBus property will be cleared, whenever the new "BaseBIOSTable" is received from the BIOS.

- On next boot onwards, BIOS checks for any changes like

    a. Check for BIOS Configuration reset (/redfish/v1/Systems/system/Bios/Actions/Bios.ResetBios) action is perfomed or not, by reading the "ResetBiosToDefaultsPending" Redfish property under "/redfish/v1/Systems/system/Bios" Redfish URI.

    b. Check for Password change (/redfish/v1/Systems/system/Bios/Actions/Bios.ChangePassword) acton is performed or not.

    c. Check for BIOS attributes changed or not (GET /redfish/v1/Systems/system/Bios/Settings).

    d. If BIOS find that any of the above mentioned checks has performed, then it will update the BIOS variables accordingly, and then send the updated BIOS configurations to BMC by making PUT call to "/redfish/v1/Systems/system/Bios" Redfish URI.

    (As per this design, BIOS will always make PUT request to /redfish/v1/Systems/system/Bios URI to push the BIOS Attributes and Attribute Registry. Even if there is only single attribute changed by OOB user. But this can also be achievable with PATCH request to "/redfish/v1/Systems/system/Bios" URI for "Attributes" Redfish property.)

    d. Reset system in any one of the above actions applied.

    e. If there are no changes, then BIOS boots normally.

- In some case, even if the OOB user updates the value correctly and also if PATCH Bios/Settings success, but then if anyhow BIOS didn't update it with some failure at BIOS side, then the same old attribute value will be displayed at "Attributes" Redfish property under /redfish/v1/Systems/system/Bios" Redfish URI.

- There can be race condition, where BIOS is booting and OOB user might try to update the Bios/Settings. But this situation can be avoided by adding the condition check at Redfish Bios/Settings PATCH handler to not perform the operation whenever the BIOS is booting by getting current host state.

- If someone changes the BIOS settings directly through BIOS, then it will be communicated to the BMC only on the next BIOS boot cycle.

```
 +-----------------------------------------------------------------------------------------------------------+
 | +-------------------------+             +----------------------------------------------------------------+|
 | |                         |             |   BMC                                                          ||
 | |                         |             |  +-----------------------+       +---------------------------+ ||
 | | RBC Web tool - POSTMAN  |             |  |Redfish Daemon         |       |RBC Daemon Manager         | ||
 | |                         |             |  |-Responsible for handle|       |-Parse Bios Data,convert to| ||
 | |                         |             |  |all Redfish request    |       | required format & return  | ||
 | |                         |             |  +-----------------------+       +---------------------------+ ||
 | +-------------------------+             |  +-----------------------+       +---------------------------+ ||
 | |                         |             |  |                       |       |                           | ||
 | |1.Get Current attributes |<---Req-/Res--> | Read BaseBIOSTable    |<-dbus-| BaseBIOSTable             | ||
 | |   name & value list     |             |  |                       |       |                           | ||
 | |                         |             |  |                       |       |                           | ||
 | |2.Get Attribute Registry |<---Req-/Res--> | Read BaseBIOSTable    |<-dbus-| BaseBIOSTable             | ||
 | |                         |             |  |                       |       |                           | ||
 | |3.Change BIOS Password   |<---Req-/Res--> | Call RBC D-bus Method |-dbus->| ChangePassword()          | ||
 | |                         |             |  |                       |       |                           | ||
 | |4.Reset To default       |<---Req-/Res--> | Set ResetBIOSSettings |-dbus->| ResetBiosSettings         | ||
 | |            settings     |             |  |                       |       |     -ResetFlag            | ||
 | |5.Update new BIOS setting|<---Req-/Res--->| Call RBC D-bus Method |-dbus->| SetAttribute()            | ||
 | |  (For single attribute) |             |  |                       |       |                           | ||
 | |                         |             |  |                       |       |                           | ||
 | |6.Get Pending attributes |<---Req-/Res--->| Get PendingAttributes |<-dbus-| PendingAttributes         | ||
 | |           list          |             |  |                       |       |                           | ||
 | |7.Update new BIOS setting|<---Req-/Res--->| Set PendingAttributes |<-dbus-| PendingAttributes         | ||
 | |           list          |             |  |                       |       |                           | ||
 | |  For multiple attributes|             |  |                       |       |                           | ||
 | +-------------------------+             |  +-----------------------+       +---------------------------+ ||
 |                                         +---------------------------------------------------------------+||
 +-----------------------------------------+-----------------------------------------------------------------+
```

- URI's involved
    1. /redfish/v1/Registries/BiosAttributeRegistry
    2. /redfish/v1/Systems/system/Bios
    3. /redfish/v1/Systems/system/Bios/Settings
    4. /redfish/v1/Systems/system/Bios/Actions/Bios.Reset
    5. /redfish/v1/Systems/system/Bios/Actions/Bios.ChangePassword


## Testing

- BIOS can send the Attributes/AttributeRegistry to BMC with PUT Redfish request.

  PUT /redfish/v1/Systems/system/Bios/
  Request Data:
  ```json
  {
      "Attributes" : [
          {
              "AttributeName" : "DdrFreqLimit",
              "CurrentValue" : "2133",
              "DefaultValue" : "auto",
              "DisplayName" : "Memory Operating Speed Selection",
              "Description" : "Force specific Memory Operating Speed or use Auto setting.",
              "MenuPath" : "Advanced/Memory Configuration/Memory Operating Speed Selection",
              "ReadOnly" : false,
              "Type" : "String",
              "Values" : ["auto", "2133", "2400", "2664", "2933"]
          },
          {
              "AttributeName" : "BIOSSerialDebugLevel",
              "CurrentValue" : "0",
              "DefaultValue" : "1",
              "DisplayName" : "BIOS Serial Debug level",
              "Description" : "BIOS Serial Debug level during system boot.",
              "MenuPath" : "Advanced/Debug Feature Selection",
              "ReadOnly" : false,
              "Type" : "Integer",
              "Values" : [1, 0]
          }
      ]
  }
  ```

- BMC will convert this data into DBus format and update the "BaseBIOSTable" DBus property.

- Then the BIOS Attributes will be available for the BMC OOB user under Bios Redfish schema.

  GET /redfish/v1/Systems/system/Bios/
  Response:
  ```json
  {
      "@odata.id": "/redfish/v1/Systems/system/Bios",
      "@odata.type": "#Bios.v1_1_0.Bios",
      "Actions": {
          "#Bios.ResetBios": {
              "target": "/redfish/v1/Systems/system/Bios/Actions/Bios.ResetBios"
          }
      },
      "AttributeRegistry": "BiosAttributeRegistry",
      "Attributes": {
          "BIOSSerialDebugLevel": 0,
          "DdrFreqLimit": "2133"
      },
      "Description": "BIOS Configuration Service",
      "Id": "BIOS",
      "Links": {
          "ActiveSoftwareImage": {
              "@odata.id": "/redfish/v1/UpdateService/FirmwareInventory/bios_active"
          },
          "SoftwareImages": [
              {
                  "@odata.id": "/redfish/v1/UpdateService/FirmwareInventory/bios_active"
              }
          ],
          "SoftwareImages@odata.count": 1
      },
      "Name": "BIOS Configuration"
  }
  ```

- The OOB user can modify the BIOS Attributes with PATCH request to Bios/Settings Redfish URI.

  PATCH /redfish/v1/Systems/system/Bios/Settings
  Request Data:
  ```json
  {
      "Attributes":{
          "DdrFreqLimit" : "auto",
          "BIOSSerialDebugLevel" : 1
      }
  }
  ```

- Those attributes will be displayed under Bios/Settings Redfish URI.

  GET /redfish/v1/Systems/system/Bios/Settings
  Response
  ```json
  {
      "@odata.id": "/redfish/v1/Systems/system/Bios/Settings",
      "@odata.type": "#Bios.v1_1_0.Bios",
      "AttributeRegistry": "BiosAttributeRegistry",
      "Attributes": {
          "DdrFreqLimit" : "auto",
          "BIOSSerialDebugLevel" : 1
      },
      "Id": "BiosSettings",
      "Name": "Bios Settings"
  }
  ```

- The BIOS Attribute Registry will be displayed under Redfish Registries Schema.

  GET /redfish/v1/Registries/BiosAttributeRegistry/BiosAttributeRegistry
  Response
  ```json
  {
      "@odata.id": "/redfish/v1/Registries/BiosAttributeRegistry/BiosAttributeRegistry",
      "@odata.type": "#AttributeRegistry.v1_4_0.AttributeRegistry",
      "Id": "BiosAttributeRegistry",
      "Language": "en",
      "Name": "Bios Attribute Registry",
      "RegistryEntries": {
          "Attributes": [
              {
                  {
                      "AttributeName" : "DdrFreqLimit",
                      "CurrentValue" : "2133",
                      "DefaultValue" : "auto",
                      "DisplayName" : "Memory Operating Speed Selection",
                      "HelpText" : "Force specific Memory Operating Speed or use Auto setting.",
                      "MenuPath" : "Advanced/Memory Configuration/Memory Operating Speed Selection",
                      "ReadOnly" : false,
                      "Type" : "String",
                      "Value" : [
                          {
                              "ValueName" : "auto"
                          },
                          {
                              "ValueName" : "2133"
                          },
                          {
                              "ValueName" : "2400"
                          },
                          {
                              "ValueName" : "2664"
                          },
                          {
                              "ValueName" : "2933"
                          },
                      ]
                  },
                  {
                      "AttributeName" : "BIOSSerialDebugLevel",
                      "CurrentValue" : "0",
                      "DefaultValue" : "1",
                      "DisplayName" : "BIOS Serial Debug level",
                      "Description" : "BIOS Serial Debug level during system boot.",
                      "MenuPath" : "Advanced/Debug Feature Selection",
                      "ReadOnly" : false,
                      "Type" : "Integer",
                      "Value" : [
                          {
                              "ValueName" : 1
                          },
                          {
                              "ValueName" : 0
                          }
                      ]
                  }
              }
          ]
      },
      "RegistryVersion": "1.0.0"
  }

## Footnotes
1. https://github.com/openbmc/docs/blob/master/designs/remote-bios-configuration.md
2. https://www.dmtf.org/sites/default/files/standards/documents/DSP0270_1.3.0.pdf
3. https://redfish.dmtf.org/schemas/v1/Bios.v1_1_0.json
4. https://redfish.dmtf.org/schemas/v1/AttributeRegistry.v1_3_2.json
5. https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/BIOSConfig/Manager.interface.yaml
