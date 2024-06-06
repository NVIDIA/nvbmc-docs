# Platform Enablement - PLDM T2 Telemetry

Author: Gilbert Chen(gilbertc@nvidia.com)
Created: Feb 6, 2024

## Contents

<!-- TOC -->

- [Platform Enablement - PLDM T2 Telemetry](#platform-enablement---pldm-t2-telemetry)
    - [Contents](#contents)
    - [Features/capabilities](#featurescapabilities)
    - [Standard spec versions](#standard-spec-versions)
        - [DSP0236 Management Component Transport Protocol Base Specification](#dsp0236-management-component-transport-protocol-base-specification)
        - [DSP0237 MCTP SMBus/I2C Transport Binding Specification](#dsp0237-mctp-smbusi2c-transport-binding-specification)
        - [DSP0238 MCTP PCIe VDM Transport Binding Specification](#dsp0238-mctp-pcie-vdm-transport-binding-specification)
        - [DSP0240 Platform Level Data Model Base Specification](#dsp0240-platform-level-data-model-base-specification)
        - [DSP0248 PLDM for Platform Monitoring and Control Specification](#dsp0248-pldm-for-platform-monitoring-and-control-specification)
        - [DSP2054 NIC PLDM NIC Modeling](#dsp2054-nic-pldm-nic-modeling)
        - [DSP0245 PLDM IDs and Codes](#dsp0245-pldm-ids-and-codes)
        - [SBMR](#sbmr)
        - [Grace PDR reference](#grace-pdr-reference)
        - [NVIDIA Networking Devices Host Management Support Application Note](#nvidia-networking-devices-host-management-support-application-note)
    - [High-Level architecture](#high-level-architecture)
        - [PLDM T2 over PCIe Grace](#pldm-t2-over-pcie-grace)
        - [PLDM T2 over I2C ConnectX-7](#pldm-t2-over-i2c-connectx-7)
        - [PLDM Terminus discovery](#pldm-terminus-discovery)
        - [PLDM T2 sensor monitoring](#pldm-t2-sensor-monitoring)
    - [Low-Level design](#low-level-design)
        - [Terminus Initialing](#terminus-initialing)
        - [Sensor polling](#sensor-polling)
        - [sensor polling algorithm](#sensor-polling-algorithm)
            - [polling scheduler](#polling-scheduler)
            - [sensor polling Example 1](#sensor-polling-example-1)
            - [sensor polling Example 2](#sensor-polling-example-2)
    - [D-Bus APIs](#d-bus-apis)
        - [Numeric Sensor](#numeric-sensor)
        - [State Sensor](#state-sensor)
        - [Numeric Effecter](#numeric-effecter)
        - [State Effecter](#state-effecter)
        - [Updating Association PDI](#updating-association-pdi)
            - [PLDM Entity Type to Inventory PDI table](#pldm-entity-type-to-inventory-pdi-table)
    - [Platform Enablement](#platform-enablement)
        - [Enablement for Yocto recipes](#enablement-for-yocto-recipes)
            - [PLDM T2 meson options](#pldm-t2-meson-options)
            - [Example pldm bbappend](#example-pldm-bbappend)
            - [Priority Sensor Config JSON file](#priority-sensor-config-json-file)
                - [example pldm t2 config JSON](#example-pldm-t2-config-json)
        - [Enablement for Grace SatMC PLDM T2 Device](#enablement-for-grace-satmc-pldm-t2-device)
            - [Enabling SMBIOS for CPU inventory](#enabling-smbios-for-cpu-inventory)
                - [SMBIOS Type 4 record example](#smbios-type-4-record-example)
            - [Enabling EntityManager for ProcessorModule inventory](#enabling-entitymanager-for-processormodule-inventory)
                - [EM JSON for ProcessorModule example](#em-json-for-processormodule-example)
            - [Grace SatMC Entity Association PDRs](#grace-satmc-entity-association-pdrs)
        - [Enablement for ConnectX-7 PLDM T2 Device](#enablement-for-connectx-7-pldm-t2-device)
            - [ConnectX-7 Entity Manager JSON file](#connectx-7-entity-manager-json-file)
                - [Entity Manager Configuration PDIs](#entity-manager-configuration-pdis)
                - [Example ConnectX-7 EM JSON file](#example-connectx-7-em-json-file)
        - [Enablement for Unsupported PLDM Modeling Device](#enablement-for-unsupported-pldm-modeling-device)
            - [PLDM T2 Numeric Sensor](#pldm-t2-numeric-sensor)
            - [PLDMT T2 state sensor and numeric/state effecter](#pldmt-t2-state-sensor-and-numericstate-effecter)
                - [The steps needed by Developer for unsupported state sensor](#the-steps-needed-by-developer-for-unsupported-state-sensor)
                - [The steps needed by Developer for unsupported numeric effecter](#the-steps-needed-by-developer-for-unsupported-numeric-effecter)
                - [The steps needed by Developer for unsupported state effecter](#the-steps-needed-by-developer-for-unsupported-state-effecter)

<!-- /TOC -->
## Features/capabilities

- extend upstream pldmd repo. to support PLDM Type2 telemetry
  - Sensor monitoring(e.g. temp, power and voltage)
  - Change device setting(e.g. power capping)
  - Event Log
- The implementation is compatible with OpenBMC Sensor Architecture
  - The PLDM T2 sensor/effecter is exposed by PDI(phosphor-DBus-Interface) to interact with other OpenBMC service(e.g. bmcweb)
  - The PLDM T2 sensor/effecter is associated to device Inventory(D-Bus object with xyz.openbmc_project.inventory.item.X PDI)
  - The Numeric Sensors are exposed to Redfish /Chassis/<$ChassisID>/Sensors/\* by default
  - Which Chassis ID the sensor should be belonged to is decided by Association PDI
- Supported PDR types
  - Numeric sensor PDR
  - State sensor PDR
  - Sensor Auxiliary Name PDR
  - Numeric effecter PDR
  - State effecter PDR
  - Effecter Auxiliary Name PDR
- Supported Sensor/Effecter types
  - Numeric Sensor [table](#numeric-sensor)
  - State sensor [table](#state-sensor)
  - Numeric effecter [table](#numeric-effecter)
  - State effecter [table](#state-effecter)
- The implementation of pldmd depends on MCTP PDIs for communicating PLDM terminus
  over MCTP. So, the MCTP APIs should be available on platform.
  - mctp stack should expose xyz.openbmc_project.MCTP.Endpoint PDI at /xyz/openbmc_project/mctp/<NetworkId>/<EID>
  - mctp stack should expose xyz.openbmc_project.Common.UnixSocket PDI for the
    information to send PLDM command to terminus.
  - mctp stack should expose xyz.openbmc_project.Inventory.Decorator.I2CDevice
    PDI for the binding protocol is MCTP over I2C to expose its I2C bus# and
    address to the same D-Bus path

## Standard spec versions

### DSP0236 Management Component Transport Protocol Base Specification
- version 1.3.1
- https://www.dmtf.org/dsp/DSP0236

### DSP0237 MCTP SMBus/I2C Transport Binding Specification

- https://www.dmtf.org/dsp/DSP0237

### DSP0238 MCTP PCIe VDM Transport Binding Specification

- https://www.dmtf.org/dsp/DSP0238

### DSP0240 Platform Level Data Model Base Specification
- version 1.1.0
- https://www.dmtf.org/dsp/DSP0240

### DSP0248 PLDM for Platform Monitoring and Control Specification
- version 1.2.2
- https://www.dmtf.org/sites/default/files/standards/documents/DSP0248_1.2.2.pdf

### DSP2054 NIC PLDM NIC Modeling
- version 1.0.0
- https://www.dmtf.org/dsp/DSP2054

### DSP0245 PLDM IDs and Codes
- version 1.3.0
- https://www.dmtf.org/dsp/DSP0245

### SBMR
- version 2.0
- https://developer.arm.com/documentation/den0069/latest/

### Grace PDR reference

- https://nvidia.sharepoint.com/:w:/r/sites/SatMC/_layouts/15/doc2.aspx?sourcedoc=%7BE0C5D54B-7FAF-4C75-91FE-555645494528%7D&file=Grace-pldm-pdr-reference.adoc.docx&action=default&mobileredirect=true&cid=e31f9784-a5ea-411b-a86b-a64369a218de

### NVIDIA Networking Devices Host Management Support Application Note

- https://apps.nvidia.com/pid/contentlibraries/detail?id=1089697

## High-Level architecture

- The role of pldmd for PLDM T2 is requester.
- pldmd implements DSP0248 to discovery, initialize PLDM terminus
- pldmd fetch PLDM PDR from device and then expose sensor/effecter modeled by
  fetched PDRs to D-Bus

### PLDM T2 over PCIe (Grace)

- The Grace SatMC is behind the FPGA
- The FPGA is a bridge to forward MCTP message in between MCTP over PCIe
  binding and MCTP over I2C binding.

```
          ┌─────────────────────────────────────┐                       ┌────────────┐
          │ BMC AST2600                         │                       │ Grace      │
┌──────┐  │            ┌──────────┐             │                       │            │
│Client│◄─┼──────────► │  bmcweb  │             │                       │            │
└──────┘  │ OOB Redfish│          │             │                       │            │
          │            └──────────┘             │                       │            │
          │                 ▲                   │                       │            │
          │                 │ D-Bus             │                       │            │
          │                 ▼                   │                       │            │
          │            ┌──────────┐             │                       │            │
          │            │  pldmd   │             │                       │            │
          │            │          │             │                       │            │
          │            └──────────┘             │                       │            │
          │              ▲      ▲               │                       │            │
          │        D-Bus │      │ unix socket   │                       │            │
          │              ▼      ▼               │                       │            │
          │ ┌──────────────┐ ┌───────────────┐  │       ┌──────┐        │  ┌─────┐   │
          │ │mctp-pcie-ctrl│ │mctp-pcie-demux├──┼───────┤ FPGA │◄───────┼─►│SatMC│   │
          │ └──────────────┘ └───────────────┘  │MCTP   └──────┘MCTP    │  └─────┘   │
          │                                     │over PCIe      over I2C│            │
          └─────────────────────────────────────┘                       └────────────┘
```

- SatMC: The OOB interface of Grace described in [SBMR](#sbmr)
- FPGA: The MCTP bridge
- mctp-pcie-demux: A service implements [DSP0238](#dsp0238-mctp-pcie-vdm-transport-binding-specification).
- mctp-pcie-ctrl: A service implements MCTP control protocol described in section 8.14 of [DSP0236](#dsp0236-management-component-transport-protocol-base-specification)
- pldmd: A service implements [DSP0248](#dsp0248-pldm-for-platform-monitoring-and-control-specification)
- bmcweb: A service implements Redfish API
- client: A Remote client sends Redfish request to BMC

### PLDM T2 over I2C (ConnectX-7)

- The ConnectX-7 OOB interface is directly connect to BMC
- The FRU EEPROM and ConnectX-7 OOB interface should be on the same I2C bus

```
                ┌─────────────────────────────────────┐        ┌─────────────────┐
                │ BMC AST2600                         │        │Host             │
      ┌──────┐  │            ┌──────────┐             │        │                 │
      │Client│◄─┼──────────► │  bmcweb  │             │        │                 │
      └──────┘  │ OOB Redfish│          │             │        │                 │
                │            └──────────┘             │        │                 │
                │                 ▲                   │        │                 │
                │                 │ D-Bus             │        │                 │
                │                 ▼                   │        │                 │
                │            ┌──────────┐             │        │                 │
                │            │  pldmd   │             │        │                 │
                │            │          │             │        │                 │
                │            └──────────┘             │        │    ┌─────┐      │
                │              ▲      ▲               │        │ ┌─►│CX7  │      │
                │        D-Bus │      │ unix socket   │        │ │  └─────┘      │
                │              ▼      ▼               │        │ │               │
                │ ┌──────────────┐ ┌───────────────┐  │        │ │  ┌──────────┐ │
                │ │mctp-i2cX-ctrl│ │mctp-i2cX-demux├──┼────────┼─┴─►│FRU EEPROM│ │
                │ └──────────────┘ └───────────────┘  │MCTP    │    └──────────┘ │
                │                                     │over I2C│                 │
                └─────────────────────────────────────┘        └─────────────────┘
```

- CX7: OOB interface of ConnectX-7 Network Controller
- FRU EEPROM: The EEPROM contains FRU data of ConnectX-7
- mctp-i2cX-demux: A service implements [DSP0237](#dsp0237-mctp-smbusi2c-transport-binding-specification).
- mctp-i2cX-ctrl: A service implements MCTP control protocol described in section 8.14 of [DSP0236](#dsp0236-management-component-transport-protocol-base-specification)

### PLDM Terminus discovery

1. pldmd gets EID list which supports PLDM message type from MCTP ctrl service
   by ObjectMapper GetManagedObjects D-Bus method for the object under
   /xyz/openbmc_project/mctp
2. pldmd get the UNIX socket from xyz.openbmc_project.Common.UnixSocket PDI from
   the response of step 1 for sending PLDM command to the EID through MCTP demux
   service.
3. pldmd identifies the PLDM capable terminus by PLDMTypes fields of GetPLDMType
   response defined in section 8.3 of DSP0240
4. pldmd follows the section 14.1 of DSP0248 to assign TID and set Event
   Receiver to PLDM terminus

### PLDM T2 sensor monitoring

1. pldmd fetches all sensor data from PLDM terminus, parse the sensor data for
   sensor id, base unit, sensor thresholds, sensor name, ..etc., and then create
   the D-Bus objects to expose the PLDM T2 sensor reading
2. pldmd starts a timer to trigger coroutine to poll sensor reading periodically
3. pldmd implements a polling scheduler to ensure the freshness of priority sensor

## Low-Level design

The PLDM T2 implementation is hosted by pldm repo

| **Folder/files**                  | **comment**                                  |
| --------------------------------- | -------------------------------------------- |
| /platform-mc/\*                   | Files for the DSP0248 implementation         |
| /platform-mc/terminus_manager.cpp | PLDM Terminus discovery                      |
| /platform-mc/platform_manager.cpp | PLDM Terminus initialization                 |
| /platform-mc/terminus.cpp         | PDR parsing and sensor creating              |
| /platform-mc/sensor_manager.cpp   | Sensor Polling                               |
| /platform-mc/event_manager.cpp    | Event handler                                |
| /oem/nvidia/platform-mc/\*        | folder for OEM PDR and sensor implementation |

### Terminus Initialing

- pldmd gets EID list by D-Bus method of MCTP Control Daemon
- pldmd sends PLDM command to PLDM terminus through MCTP DeMux service
- pldmd performs necessary initialization steps described in section 14.1 of DSP0248
  to EID which supports PLDM Type2 message.
- pldmd fetches PDRs from terminus and then parses the PDR for the required sensor
  parameters and thenpldmd creates D-Bus objects for parsed sensor data with PDIs
  described in [D-Bus APIs section](#d-bus-apis)
- After done the initialization, pldmd starts a timer to trigger coroutine to poll
  sensor reading.

```

                         ┌─────────┐                           ┌───────────────────┐ ┌───────────┐ ┌─────────────┐
                         │  pldmd  │                           │MCTP Control Daemon│ │MCTP DeMux │ │PLDM terminus│
                         └────┬────┘                           └────────┬──────────┘ └─────┬─────┘ └─────┬───────┘
                              │                                         │                  │             │
                              │                                         │                  │             │
                              │ GetSubTree                              │                  │             │
        Discovery Terminus    │ /xyz/openbmc_project/mctp               │                  │             │
                             ┌┴┐xyz.openbmc_project.MCTP.Endpoint       │                  │             │
                             │ ├───────────────────────────────────────►│                  │             │
                             │ │                                        │                  │             │
                             │ │                                        │                  │             │
                             │ │   response                             │                  │             │
                             │ │ ◄──────────────────────────────────────┤                  │             │
                             └┬┘                                        │                  │             │
                              │                                         │                  │             │
                             ┌┴┐ GetPLDM Types                          │                  │             │
                             │ ├───────────────────────────────────────►│                  │             │
                             │ │                                        ├─────────────────►│             │
                             │ │                                        │                  ├───────────► │
                             │ │                                        │                  │ ◄───────────┤
                             │ │   response                             ├──────────────────┤             │
                             │ │ ◄──────────────────────────────────────┤                  │             │
                             └┬┘                                        │                  │             │
                              │                                         │                  │             │
                             ┌┴┐ Set TID                                │                  │             │
                             │ ├────────────────────────────────────────┤                  │             │
                             │ │                                        ├─────────────────►│             │
                             │ │                                        │                  ├───────────► │
                             │ │                                        │                  │ ◄───────────┤
                             │ │   response                             ├──────────────────┤             │
                             │ │ ◄──────────────────────────────────────┤                  │             │
                             └┬┘                                        │                  │             │
                              │                                         │                  │             │
                             ┌┴┐ Get PDR                                │                  │             │
                             │ ├────────────────────────────────────────┤                  │             │
                             │ │                                        ├─────────────────►│             │
                             │ │                                        │                  ├───────────► │
                             │ │                                        │                  │ ◄───────────┤
                             │ │   response                             ├──────────────────┤             │
                             │ │ ◄──────────────────────────────────────┤                  │             │
                             └┬┘                                        │                  │             │
                              │                                         │                  │             │
                             ┌┴┐ SetEventReceiver                       │                  │             │
                             │ ├────────────────────────────────────────┤                  │             │
                             │ │                                        ├─────────────────►│             │
                             │ │                                        │                  ├───────────► │
                             │ │                                        │                  │ ◄───────────┤
                             │ │   response                             ├──────────────────┤             │
                             │ │ ◄──────────────────────────────────────┤                  │             │
                             └┬┘                                        │                  │             │
                              │                                         │                  │             │
                             ┌┴┐                                        │                  │             │
                 Parse PDR   │ │                                        │                  │             │
                             │ │                                        │                  │             │
              Create Sensor  │ │                                        │                  │             │
                             └┬┘                                        │                  │             │
                              │                                         │                  │             │
                ┌────────────▲│                                         │                  │             │
                │            ┌┴┐ GetSensorReading                       │                  │             │
                │ Poll sensor│ ├────────────────────────────────────────┼─────────────────►│             │
                │ State      │ │                                        │                  ├────────────►│
                │            │ │                                        │                  │             │
                │            │ │  response                              │                  │◄────────────┤
                │            │ │◄───────────────────────────────────────┼──────────────────┤             │
                │            └┬┘                                        │                  │             │
                │             │                                         │                  │             │
                └─────────────┤                                         │                  │             │
                 Timer loop   │                                         │                  │             │
```

### Sensor polling

- PLDM T2 sensors are categorized into priority and round-robin sensor like the description in
  [sensor polling algorithm section](#sensor-polling-algorithm)
- pldmd starts a coroutine for a terminus when timer expired and previous coroutine is completed.
- The coroutine polls all priority sensors first and partial of round-robin sensors. the details
  of polling algorithm is described in the [section](#sensor-polling-algorithm)

```
┌─────────┐                                                                     ┌───────────────────┐ ┌───────────┐  ┌───────┐ ┌───────┐
│  pldmd  │                                                                     │MCTP Control Daemon│ │MCTP DeMux │  │ SatMC │ │ CX7_0 │
└────┬────┘                                                                     └────────┬──────────┘ └─────┬─────┘  └────┬──┘ └───┬───┘
     │                                                                                   │                  │             │        │
     │ start Timer┌───────┐                                                              │                  │             │        │
     ├──────────► │timerA │                                                              │                  │             │        │
     │            └───┬───┘                                                              │                  │             │        │
     │                │                                                                  │                  │             │        │
     │        ┌──────►│                                                                  │                  │             │        │
     │        │       │ Timer expired                                                    │                  │             │        │
     │        │  ┌────┴─────┐                                                            │                  │             │        │
     │        │  │previous  │ Yes               ┌──────────┐                             │                  │             │        │
     │        │  │coroutine ├──────────────────►│CoroutineA│                             │                  │             │        │
     │        │  │completed?│ Start CoroutineA  └────┬─────┘                             │                  │             │        │
     │        │  └────┬─────┘                        │                                   │                  │             │        │
     │        │       │No                        ┌──▲│                                   │                  │             │        │
     │        └───────┘                          │  ┌┴┐ GetSensorReading                 │                  │             │        │
     │                                 Polling   │  │ ├─────────────────────────────────►│                  │             │        │
     │                                 Priority  │  │ │                                  ├─────────────────►│             │        │
     │                                 Sensor    │  │ │                                  │                  ├───────────► │        │
     │                                           │  │ │                                  │                  │ ◄───────────┤        │
     │                                           │  │ │   response                       ├──────────────────┤             │        │
     │                                           │  │ │ ◄────────────────────────────────┤                  │             │        │
     │                                           │  └┬┘                                  │                  │             │        │
     │                                           │   │                                   │                  │             │        │
     │                                           └───┤                                   │                  │             │        │
     │                                               │                                   │                  │             │        │
     │                                               │                                   │                  │             │        │
     │                                           ┌──►│                                   │                  │             │        │
     │                                           │   │  GetStateSensorReadings           │                  │             │        │
     │                                 Polling   │  ┌┴┬─────────────────────────────────►│                  │             │        │
     │                                 Round-robin  │ │                                  ├─────────────────►│             │        │
     │                                 Sensor    │  │ │                                  │                  ├───────────► │        │
     │                                           │  │ │                                  │                  │ ◄───────────┤        │
     │start Timer ┌───────┐                      │  │ │   response                       ├──────────────────┤             │        │
     ├──────────► │timerB │                      │  │ │ ◄────────────────────────────────┤                  │             │        │
     │            └───┬───┘                      │  └┬┘                                  │                  │             │        │
     │                │                          │   │                                   │                  │             │        │
     │        ┌──────►│                          └───┤                                   │                  │             │        │
     │        │       │ Timer expired                │                                   │                  │             │        │
     │        │  ┌────┴─────┐                                                            │                  │             │        │
     │        │  │previous  │ Yes               ┌──────────┐                             │                  │             │        │
     │        │  │coroutine ├──────────────────►│CoroutineB│                             │                  │             │        │
     │        │  │completed?│ Start CoroutineB  └────┬─────┘                             │                  │             │        │
     │        │  └────┬─────┘                        │                                   │                  │             │        │
     │        │       │No                        ┌──▲│                                   │                  │             │        │
     │        └───────┘                          │  ┌┴┐ GetSensorReading                 │                  │             │        │
     │                                 Polling   │  │ ├─────────────────────────────────►│                  │             │        │
     │                                 Priority  │  │ │                                  ├─────────────────►│             │        │
     │                                 Sensor    │  │ │                                  │                  ├─────────────┼──────► │
     │                                           │  │ │                                  │                  │ ◄───────────┼────────┤
     │                                           │  │ │   response                       ├──────────────────┤             │        │
     │                                           │  │ │ ◄────────────────────────────────┤                  │             │        │
     │                                           │  └┬┘                                  │                  │             │        │
     │                                           │   │                                   │                  │             │        │
     │                                           └───┤                                   │                  │             │        │
     │                                               │                                   │                  │             │        │
     │                                               │                                   │                  │             │        │
     │                                           ┌──►│                                   │                  │             │        │
     │                                           │   │  GetStateSensorReadings           │                  │             │        │
     │                                 Polling   │  ┌┴┬─────────────────────────────────►│                  │             │        │
     │                                 Round-robin  │ │                                  ├─────────────────►│             │        │
     │                                 Sensor    │  │ │                                  │                  ├─────────────┼──────► │
     │                                           │  │ │                                  │                  │ ◄───────────┼────────┤
     │                                           │  │ │   response                       ├──────────────────┤             │        │
     │                                           │  │ │ ◄────────────────────────────────┤                  │             │        │
     │                                           │  └┬┘                                  │                  │             │        │
     │                                           │   │                                   │                  │             │        │
     │                                           └───┤                                   │                  │             │        │
     │                                               │                                   │                  │             │        │
```

### sensor polling algorithm

For polling sensor in background by single thread pldmd app, the pldmd starts a
timer expired after the period define in [meson_options](#pldm-t2-meson-options).
When timer expired, the pldmd will perform to priority scheduler and round-robin
scheduler described in [polling scheduler section](#polling-scheduler) to meet the
requirement of sensor reading age.

#### polling scheduler
- pldmd categorizes sensor into priority and non-priority sensor.
- The sensor is priority sensor if its sensor name space is specified in
  [PrioritySensorNameSpaces](#priority-sensor-config-json-file) and they
  updated by priority scheduler
- The sensor is non-priority sensor if its sensor name space is not specified
  in [PrioritySensorNameSpaces](#priority-sensor-config-json-file) and they
  are updated by round-robin scheduler.

1. priority scheduler
   - When timer timeout, priority scheduler simply updates all priority sensor
     so that max sensor age of priority sensor is less than timer period
2. round-robin scheduler
   - The round-robin scheduler is executed after priority scheduler
   - The round-robin scheduler will update as many sensor in round-robin queue
     as possible before next timer timeout
   - round-robin scheduler ensures at least one of the non-priority sensors is
     updated in every polling iteration.

#### sensor polling Example 1

There is 20 sensors in a PLDM terminus

- Sensor S1~S5 are Priority sensor(temp, power or energy) and updated by Priority
  scheduler.
- Sensor S6~S20 are State sensor by Round-Robin scheduler
- Polling S1~S5 are completed within a timer period(<250ms).

|            | Prio | Prio | Prio | Prio | Prio | RR  | RR  | RR  | RR  | RR  |
| ---------- | ---- | ---- | ---- | ---- | ---- | --- | --- | --- | --- | --- |
| Iteration1 | S1   | S2   | S3   | S4   | S5   | S6  | S7  | S8  | S9  | S10 |
| Iteration2 | S1   | S2   | S3   | S4   | S5   | S11 | S12 | S13 | S14 | S15 |
| Iteration3 | S1   | S2   | S3   | S4   | S5   | S16 | S17 | S18 | S19 | S20 |
| Iteration4 | S1   | S2   | S3   | S4   | S5   | S6  | S7  | S8  | S9  | S10 |

#### sensor polling Example 2

There are 20 sensors in a PLDM terminus.

- Sensor S1~S10 are Priority sensor(temp, power or energy) and updated by Priority
  scheduler.
- Sensor S11~S20 are State sensor by Round-Robin scheduler.
- Priority scheduler spent more than 250ms to update Priority sensors so pldmd just
  update one non-priority sensor in this iteration.

|            | Prio | Prio | Prio | Prio | Prio | Prio | Prio | Prio | Prio | Prio | RR  |
| ---------- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | --- |
| Iteration1 | S1   | S2   | S3   | S4   | S5   | S6   | S7   | S8   | S9   | S10  | S11 |
| Iteration2 | S1   | S2   | S3   | S4   | S5   | S6   | S7   | S8   | S9   | S10  | S12 |
| Iteration3 | S1   | S2   | S3   | S4   | S5   | S6   | S7   | S8   | S9   | S10  | S13 |
| Iteration4 | S1   | S2   | S3   | S4   | S5   | S6   | S7   | S8   | S9   | S10  | S14 |

## D-Bus APIs

### Numeric Sensor

- The reading of Numeric sensor is exposed to D-Bus via xyz.openbmc_project.Sensor.Value PDI
- The Base Unit field from the numeric sensor PDR determines the namespace of sensor
- The table of supported numeric sensor base units

| PLDM Sensor Base Unit       | D-Bus NameSpace                          | Redfish Reading Type |
| --------------------------- | ---------------------------------------- | -------------------- |
| PLDM_SENSOR_UNIT_DEGRESS_C  | /xyz/openbmc_project/sensors/temperature | Temperature          |
| PLDM_SENSOR_UNIT_VOLTS      | /xyz/openbmc_project/sensors/voltage     | Voltage              |
| PLDM_SENSOR_UNIT_AMPS       | /xyz/openbmc_project/sensors/current     | Current              |
| PLDM_SENSOR_UNIT_RPM        | /xyz/openbmc_project/sensors/fan_pwm     | Percent              |
| PLDM_SENSOR_UNIT_WATTS      | /xyz/openbmc_project/sensors/power       | Power                |
| PLDM_SENSOR_UNIT_JOULES     | /xyz/openbmc_project/sensors/energy      | EnergyJoules         |
| PLDM_SENSOR_UNIT_HERTZ      | /xyz/openbmc_project/sensors/frequency   | Frequency            |
| PLDM_SENSOR_UNIT_PERCENTAGE | /xyz/openbmc_project/sensors/utilization | Percent              |
| PLDM_SENSOR_UNIT_COUNTS     | /xyz/openbmc_project/sensors/counter     | N/A                  |

- The Numeric sensor expose xyz.openbmc_project.Association.Definition PDI to its D-Bus path
  1. The Associations property of is {chassis, all_sensors, <Inventory D-Bus path>}
  2. The sensor associates to an Inventory D-Bus path according to the steps described in
     [PLDM Entity Type vs inventory PDI mapping table](#pldm-entity-type-to-inventory-pdi-table)

### State Sensor

- The type of PDI for state sensor is based on the combination of StateSet ID and Entity Type.
- The table of supported state set IDs and Entity types

| PLDM State Set ID                     | PLDM Entity Type                                       | PDI                                                   | properties                                                                                                                | Note           |
| ------------------------------------- | ------------------------------------------------------ | ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | -------------- |
| PLDM_STATESET_ID_PRESENCE             | PLDM_ENTITY_PROC \|\|<br>PLDM_ENTITY_MEMORY_CONTROLLER | com.nvidia.MemorySpareChannel                         | memorySpareChannelPresence                                                                                                | downstream PDI |
| PLDM_NVIDIA_OEM_STATE_SET_NVLINK      | PLDM_ENTITY_SYS_BUS                                    | xyz.openbmc_project.Inventory.Item.Port               | linkState and linkStatus                                                                                                  | downstream PDI |
| PLDM_NVIDIA_OEM_STATE_SET_DEBUG_STATE |                                                        | xyz.openbmc_project.Control.Processor.RemoteDebug     | jtagDebug, deviceDebug, securePrivilegeNonInvasiveDebug, securePrivilegeInvasiveDebug, nonInvasiveDebug and invasiveDebug | downstream PDI |
| PLDM_STATESET_ID_PERFORMANCE          |                                                        | xyz.openbmc_project.State.ProcessorPerformance        | value                                                                                                                     | downstream PDI |
| PLDM_STATESET_ID_POWERSUPPLY          |                                                        | xyz.openbmc_project.State.Decorator.PowerSystemInputs | status                                                                                                                    |                |
| PLDM_STATESET_ID_LINKSTATE            | PLDM_ENTITY_PCI_EXPRESS_BUS                            | xyz.openbmc_project.Inventory.Item.Port               | linkState and linkStatus                                                                                                  | downstream PDI |
| PLDM_STATESET_ID_BOOT_REQUEST         |                                                        | xyz.openbmc_project.Control.ClearNonVolatileVariables | clear                                                                                                                     | downstream PDI |
| PLDM_STATESET_ID_LINKSTATE            | PLDM_ENTITY_ETHERNET                                   | xyz.openbmc_project.Inventory.Item.Port               | linkState and linkStatus                                                                                                  | downstream PDI |
| PLDM_STATESET_ID_HEALTHSTATE          |                                                        | xyz.openbmc_projcet.State.Decorator.Health            | health                                                                                                                    | downstream PDI |

### Numeric Effecter

- Only one type of numeric effecter is supported.

| PDR type         | PLDM SensorBase unit   | PDI                                   |
| ---------------- | ---------------------- | ------------------------------------- |
| Numeric effecter | PLDM_SENSOR_UNIT_WATTS | xyz.openbmc_project.Control.Power.Cap |

### State Effecter

- Only one type of numeric effecter is supported.

| PLDM State Set ID             | PLDM Entity Type | PDI                                                        | Note           |
| ----------------------------- | ---------------- | ---------------------------------------------------------- | -------------- |
| PLDM_STATESET_ID_BOOT_REQUEST | Any              | xyz.openbmc_project.Control.Boot.ClearNonVolatileVariables | downstream PDI |

### Updating Association PDI

The PLDM T2 sensor/effecter should expose Association PDI to associate itself to one of item.inventory
D-Bus object. The association is decided by sensor's entity type and instance number defined in its PDR.
Following steps show the method how pldm find the item.inventory D-Bus object for a PLDM T2 sensor.

- pldmd check the [table](#pldm-entity-type-to-inventory-pdi-table) to get all D-Bus path with
  inventory.Item.X PDI for sensor with Y PLDM Entity Type
- pldmd get xyz.openbmc_project.inventory.Decorator.Instance PDI from the same
  D-Bus path of inventory.item.X PDI
- pldmd compares the InstanceNumber field of sensor's PDR and the InstanceNumber
  property of Instance PDI.
- If multiple D-Bus path is matched and then comparing parent of inventory.item.X
  to sensor's container entity Type and instance Number
- The hierarchy of inventory.item.X is present in D-Bus path.
  - the parent folder of inventory.item.X is the parent inventory D-Bus path
  - example: /xyz/openbmc_project/system/board/ProcessorModule_0/CPU_0
    - The inventory D-Bus path of CPU is /xyz/openbmc_project/system/board/ProcessorModule_0/CPU_0
    - The parent inventory D-Bus path of CPU_0 is /xyz/openbmc_project/system/board/ProcessorModule_0

#### PLDM Entity Type to Inventory PDI table

| PLDM Entity Type                    | Inventory.item.X PDI                               |
| ----------------------------------- | -------------------------------------------------- |
| Overall system(container ID=0x0000) | xyz.openbmc_project.inventory.item.System          |
| PLDM_ENTITY_PROC_IO_MODULE          | xyz.openbmc_project.inventory.item.ProcessorModule |
| PLDM_ENTITY_PROC                    | xyz.openbmc_project.inventory.item.Cpu             |
| PLDM_ENTITY_ADD_IN_CARD             | xyz.openbmc_project.inventory.item.Board           |

## Platform Enablement

### Enablement for Yocto recipes

- The mandatory option for enabling PLDM T2 feature is set pldm-type2 meson
  option to enabled by a .bbappend file [example](#example-pldm-bbappend)

- The default sensor polling interval is 250ms and it can be configured by
  sensor-polling-time meson option. [example](#example-pldm-bbappend)

- The default sensor types polled in priority are temperature, energy and
  power. The setting can be overwrite by [pldm_t2_config.JSON](#example-pldm-t2-config-json)

- A [Entity Manager JSON file](#connectx-7-entity-manager-json-file) is needed for enabling PLDM terminus of which OOB
  is MCTP over SMBus.

#### PLDM T2 meson options

| **meson options**   | **value type**                            | **Description**                                                       |
| ------------------- | ----------------------------------------- | --------------------------------------------------------------------- |
| pldm-type2          | feature:enabled/disable, default:disabled | Set enabled to activate pldm type2 feature                            |
| sensor-polling-time | integer, default:249                      | The PLDM t2 sensor update interval.(\*note1)                          |
| libpldmresponder    | feature:enabled/disabled, default enabled | Set enabled to accept asynchronous PLDM event from device.            |
| local-eid-over-i2c  | integer, default:8                        | Set local EID for I2C binding for set EventReceiver command to device |
| local-eid-over-pcie | integer, default:10                       | Set local EID for PCIe binding for setEventReceiver command to device |

\*note1: The resolution is 250ms. It means that interval is 250ms for value in
between 0 to 249. The interval is 500ms for value in between 250 to 499.

- If platform needs to change the default setting, it can be done by a [bbappend](#example-pldm-bbappend)
  file. Assigning a value to the option into EXTRA_OEMESON variable.

#### Example pldm bbappend

- enable PLDM T2 feature by a pldm\_%.bbappend file in your platform meta-layer recipes-phosphor/pldm folder.

```
cat openbmc/meta-nvidia/meta-gh/recipes-phosphor/pldm/pldm_%.bbappend
# Enable PLDM Type2
EXTRA_OEMESON += "-Dpldm-type2=enabled"
EXTRA_OEMESON += "-Dsensor-polling-time=200"
...skip...
```

#### Priority Sensor Config JSON file

When pldmd starts, it loads /usr/local/share/pldm/pldm_t2_config.JSON for the PrioritySensorNameSpace list.
If the JSON file does not exist, the temperature, power and energy is the default value of PrioritySensorNameSpace.
The valid sensor name space is defined in [xyz.openbmc_project.Sensor.Value PDI](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Sensor/Value.interface.yaml)

##### example pldm t2 config JSON

```
cat /usr/local/share/pldm/pldm_t2_config.JSON
{
    "PrioritySensorNameSpaces": [
        "/xyz/openbmc_project/sensors/temperature/",
        "/xyz/openbmc_project/sensors/power/",
        "/xyz/openbmc_project/sensors/energy/"
    ],
}
```

### Enablement for Grace SatMC PLDM T2 Device

To ensure that the Grace PLDM T2 sensors can be populated to correct Redfish
URL, the following items need to be verified.

1. [Enabling SMBIOS for CPU inventory](#enabling-smbios-for-cpu-inventory)
2. [Enabling EntityManager for ProcessorModule inventory](#enabling-entitymanager-for-processormodule-inventory)
3. [Grace SatMC Entity Association PDRs](#grace-satmc-entity-association-pdrs)

#### Enabling SMBIOS for CPU inventory

1. System UEFI needs to push SMBIOS Type 4 CPU info record to BMC
2. The socket designation field of SMBIOS type 4 record should be in format
   {SilkScreenLabel}:{LogicalSocket}.{LogicalChip}. (e.g.,G1:0.0. the record
   is for ProcessorModule0 CPU0)
3. Smbios-mdr service decode the socket designation field and use LogicalSocket
   to find which ProcessorModule inventory D-Bus object it should be belong to.
4. Smbios-mdr service decode the socket designation field and use LogicalChip
   to for the value of InstanceNumber property of xyz.openbmc_project.Inventory.Decorator.Instance PDI

##### SMBIOS Type 4 record example

```
root@skinnyjoe:~# busctl introspect xyz.openbmc_project.Smbios.MDR_V2 /xyz/openbmc_project/inventory/system/board/ProcessorModule_0/CPU_0
NAME                                                 TYPE      SIGNATURE RESULT/VALUE                             FLAGS
org.freedesktop.DBus.Introspectable                  interface -         -                                        -
.Introspect                                          method    -         s                                        -
org.freedesktop.DBus.Peer                            interface -         -                                        -
.GetMachineId                                        method    -         s                                        -
.Ping                                                method    -         -                                        -
org.freedesktop.DBus.Properties                      interface -         -                                        -
.Get                                                 method    ss        v                                        -
.GetAll                                              method    s         a{sv}                                    -
.Set                                                 method    ssv       -                                        -
.PropertiesChanged                                   signal    sa{sv}as  -                                        -
xyz.openbmc_project.Association.Definitions          interface -         -                                        -
.Associations                                        property  a(sss)    2 "processors" "all_processors" "/xyz... emits-change writable
xyz.openbmc_project.Inventory.Connector.Slot         interface -         -                                        -
xyz.openbmc_project.Inventory.Decorator.Asset        interface -         -                                        -
.BuildDate                                           property  s         ""                                       emits-change writable
.Manufacturer                                        property  s         "NVIDIA"                                 emits-change writable
.Model                                               property  s         ""                                       emits-change writable
.Name                                                property  s         ""                                       emits-change writable
.PartNumber                                          property  s         "900-2G530-0000-000"                     emits-change writable
.SKU                                                 property  s         "0x01000030"                             emits-change writable
.SerialNumber                                        property  s         "0x000000017820B1C8080000000F010240"     emits-change writable
.SparePartNumber                                     property  s         ""                                       emits-change writable
.SubModel                                            property  s         ""                                       emits-change writable
xyz.openbmc_project.Inventory.Decorator.AssetTag     interface -         -                                        -
.AssetTag                                            property  s         "1643023000613"                          emits-change writable
xyz.openbmc_project.Inventory.Decorator.Instance     interface -         -                                        -
.InstanceNumber                                      property  t         0                                        emits-change writable
xyz.openbmc_project.Inventory.Decorator.LocationCode interface -         -                                        -
.LocationCode                                        property  s         "G1:0.0"                                 emits-change writable
xyz.openbmc_project.Inventory.Decorator.Revision     interface -         -                                        -
.Version                                             property  s         "Grace A02"                              emits-change writable
xyz.openbmc_project.Inventory.Item                   interface -         -                                        -
.Present                                             property  b         true                                     emits-change writable
.PrettyName                                          property  s         ""                                       emits-change writable
xyz.openbmc_project.Inventory.Item.Chassis           interface -         -                                        -
.Type                                                property  s         "xyz.openbmc_project.Inventory.Item.C... emits-change writable
xyz.openbmc_project.Inventory.Item.Cpu               interface -         -                                        -
.Characteristics                                     property  as        2 "xyz.openbmc_project.Inventory.Item... emits-change writable
.CoreCount                                           property  q         72                                       emits-change writable
.EffectiveFamily                                     property  q         257                                      emits-change writable
.EffectiveModel                                      property  q         0                                        emits-change writable
.Family                                              property  s         "ARMv8"                                  emits-change writable
.Id                                                  property  t         8647279169                               emits-change writable
.MaxSpeedInMhz                                       property  u         0                                        emits-change writable
.Microcode                                           property  u         0                                        emits-change writable
.Socket                                              property  s         "G1:0.0"                                 emits-change writable
.Step                                                property  q         0                                        emits-change writable
.ThreadCount                                         property  q         72                                       emits-change writable
```

#### Enabling EntityManager for ProcessorModule inventory

1. EntityManager JSON file should expose Inventory.Item.ProcessorModule PDI
   for detected ProcessorModule FRU record at D-Bus path, /xyz.openbmc*project/inventory/system/board/ProcessorModule*{Instance#}
2. The JSON file should expose correct instanceNumber for the ProcessorModule by
   xyz.openbmc_project.Inventory.Decorator.Instance PDI at the step1 D-Bus path

##### EM JSON for ProcessorModule example

```
[
    {
        "Exposes": [
            {
                "Address": "$address",
                "Bus": "$bus",
                "Name": "ProcMod_0",
                "Type": "EEPROM_24C64"
            }
        ],
        "Name": "ProcessorModule 0",
        "Probe": "xyz.openbmc_project.FruDevice({'BUS': 2, 'ADDRESS': 80, 'BOARD_PRODUCT_NAME': '.?CG1.?|.?PG530.?'})",
        "Type": "Board",
        "Parent_Chassis":  "/xyz/openbmc_project/inventory/system/chassis/Baseboard_0",
        "xyz.openbmc_project.Inventory.Decorator.Asset": {
            "Manufacturer": "$BOARD_MANUFACTURER",
            "Model": "$BOARD_PRODUCT_NAME",
            "PartNumber": "$BOARD_PART_NUMBER",
            "SerialNumber": "$BOARD_SERIAL_NUMBER"
        },
        "xyz.openbmc_project.Inventory.Decorator.I2CDevice": {
            "Bus": "$bus",
            "Address": "$address",
            "Name": "ProcMod_0"
        },
        "xyz.openbmc_project.Inventory.Item.Chassis": {
            "Type": "xyz.openbmc_project.Inventory.Item.Chassis.ChassisType.Module"
        },
        "xyz.openbmc_project.Inventory.Item.ProcessorModule": {},
        "xyz.openbmc_project.Inventory.Decorator.Instance": {
            "InstanceNumber": 0
        }
    }
]
```

#### Grace SatMC Entity Association PDRs

1. The PLDM T2 sensors should be associated to correct inventory as long as
   the entityType, entityInstanceNumber and containerID fields of sensor PDR
   are assigned properly.
2. The sensor's entity type should be 135 if it belongs to CPU.
3. The sensor's containerID is used for identifying which ProcessorModule
   it belongs to.
4. The sensor's entityInstanceNumber is used for identify which CPU in the
   same ProcessorModule it belongs to

### Enablement for ConnectX-7 PLDM T2 Device

To ensure that the ConnectX-7 T2 sensors can be populated to correct Redfish
URL, the following items need to be verified.

1. ConnectX-7 FRU and OOB interface connection
   - Its OOB interface is MCTP over SMBus/I2C
   - Its FRU EEPROM is on the same I2C bus.
2. ConnectX-7 Entity Association PDR
   - The PLDM entity and sensor should follows DSP2054 PLDM NIC Modeling
   - Its entity type of top level container is Add-In Card(68)
3. A EM JSON file with required [settings](#connectx-7-entity-manager-json-file)

#### ConnectX-7 Entity Manager JSON file

- The entity manager JSON file should have the properties in table for pldmd
  and bmcweb to consume
- The example EM JSON file for [ConnectX-7](#example-connectx-7-em-json-file)

| Property Name                                 | Value                                   | description                                                                                                                |
| --------------------------------------------- | --------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| Name                                          | D-Bus object name                       | The string will be used for the URL of Redfish API                                                                         |
| Type                                          | inventory type of the device            | The type of inventory for this device(e.g., Board)                                                                         |
| Parent_Chassis                                | The parent chassis of the PLDM terminus | Define which inventory object the PLDM device should belong to                                                             |
| Probe                                         | The D-Bus object match expression       | Specific the condition if inventory object should be exposed to D-Bus                                                      |
| xyz.openbmc_project.Inventory.Decorator.Asset | FRU fields                              | Populate properties which will be displayed in Redfish chassis API                                                         |
| xyz.openbmc_project.Inventory.Item.X          | {}                                      | Any Addition inventory should be exposed to D-Bus(e.g. xyz.openbmc_project.item.Inventory.NetworkInterface for ConnectX-7) |
| expose                                        | The configuration PDIs                  | Any addition configuration PDI should be exposed to D-Bus(e.g. I2CDeviceAssociation for ConnectX-7)                        |

##### Entity Manager Configuration PDIs

1. xyz.openbmc_project.Configuration.I2CDeviceAssociation
   For the PLDM terminus which is MCTP over SMBus, an Entity manager configuration
   JSON file should be provided to create its device inventory to D-Bus for pldmd
   to associate PLDM T2 sensor. In the expose section, the JSON file should define
   the xyz.openbmc_project.Configuration.I2CDeviceAssociation PDI
   - For pldmd to identify which inventory D-Bus path the PLDM T2 sensor should
     associate to

| **Property** | **Type** | **Description**                           |
| ------------ | -------- | ----------------------------------------- |
| Type         | string   | set to "I2CDeviceAssociation"             |
| Name         | string   | for an unique D-Bus object Path           |
| Bus          | uint8_t  | The I2C Bus# the MCTP device connected to |
| Address      | uint8_t  | The I2C Addr# the MCTP device is          |

- The EM Configuration PDI is exposed only when a ConntectX-7 FRU EEPROM is detected.
  done by the probe property of EM config [example](#example-connectx-7-em-json-file)
- It is expected that ConnectX-7 FRU EEPROM and its OOB interface is on the same I2C bus
- EM JSON exposes which I2C Bus# the FRU EEPROM is detected and the ConnectX-7
  OOB I2C Address# to the I2CDeviceAssociation PDI
- pldmd gets the MCTP EID's I2C Bus# and Addr# from xyz.openbmc_project.Inventory.Decorator.I2CDevice
  PDI exposed by mctp-i2cX-ctrl service
- All the sensors discovered from the EID will be associated to the path of inventory D-Bus
  object created by this EM JSON.

2. xyz.openbmc_project.Configuration.SensorAuxName PDI
   For the PLDM terminus of which device inventory is created by EntityManager, the
   xyz.openbmc_project.Configuration.SensorAuxName PDI can be provided in
   expose section to specify the sensor name to sensor ID in case that the PLDM
   terminus does not provide the Sensor auxiliary name PDR.

| **Property** | **Type**        | **Description**                                                                                         |
| ------------ | --------------- | ------------------------------------------------------------------------------------------------------- |
| Type         | string          | set to "SensorAuxName"                                                                                  |
| Name         | string          | for an unique D-Bus object Path                                                                         |
| SensorId     | uint16_t        | The sensor Id which needs a customized sensor name                                                      |
| AuxNames     | array of string | array of sensor name. index0 for first composited sensor, index1 for second composited sensor and so on |

- If both of PLDM auxiliary name PDR and configuration.SensorAuxName PDI are available.
  pldmd will follow the priority order in table below to name the sensor.

| **Priority** | **Source**                 | **Description**                                                                          |
| ------------ | -------------------------- | ---------------------------------------------------------------------------------------- |
| Highest      | EntityManager              | Customized by xyz.openbmc_project.Configuration.SensorAuxName PDI defined in EM jon file |
|              | Aux Name PDR from Terminus | named by terminus FW                                                                     |
| Lowest       | Not available              | PLDM*Sensor*{$SENSOR_ID}_{$TID}                                                          |

##### Example ConnectX-7 EM JSON file

```
[
    {
        "Exposes": [
            {
                "Name": "CX7_$BUS",
                "Type": "I2CDeviceAssociation",
                "Bus": "$BUS",
                "Address": "50"
            },
            {
                "Name": "CX7_$BUS_Temp",
                "Type": "SensorAuxName",
                "SensorId": 1,
                "AuxNames": ["CX7_$BUS_Temp"]
            },
            {
                "Name": "CX7_$BUS_Port_0_Temp",
                "Type": "SensorAuxName",
                "SensorId": 8,
                "AuxNames": ["CX7_$BUS_Port_0_Temp“]
            },
            {
                "Name": "CX7_$BUS_Port_0_State",
                "Type": "SensorAuxName",
                "SensorId": 4,
                "AuxNames": ["Port_0"]
            }
        ],
        "Name": "CX7_$BUS",
        "Type": "Board",
        "Parent_Chassis": "/xyz/openbmc_project/inventory/system/chassis/Baseboard_0",
        "Probe": "xyz.openbmc_project.FruDevice({'PRODUCT_PRODUCT_NAME': 'ConnectX-7 NDR 1P.*'})",
        "xyz.openbmc_project.Inventory.Decorator.Asset": {
            "Manufacturer": "$BOARD_MANUFACTURER",
            "Model": "$BOARD_PRODUCT_NAME",
            "PartNumber": "$BOARD_PART_NUMBER",
            "SerialNumber": "$BOARD_SERIAL_NUMBER"
        },
        "xyz.openbmc_project.Inventory.Item.Chassis": {
            "Type": "xyz.openbmc_project.Inventory.Item.Chassis.ChassisType.Component"
        },
        "xyz.openbmc_project.Inventory.Decorator.Location": {
            "LocationType": "xyz.openbmc_project.Inventory.Decorator.Location.LocationTypes.Embedded"
        },
        "xyz.openbmc_project.State.Decorator.Health": {
            "Health": "xyz.openbmc_project.State.Decorator.Health.HealthType.OK"
        },
        "xyz.openbmc_project.Inventory.Item.NetworkInterface": {},
        "xyz.openbmc_project.Inventory.Decorator.Instance": {
            "InstanceNumber": 1
        }
    }
]
```

### Enablement for Unsupported PLDM Modeling Device

1. Decide the method to create inventory D-Bus object for PLDM T2 sensor association.
   It can refer to the method described in [the ConnectX-7 section](#enablement-for-connectx-7-pldm-t2-device)
   if the inventory information is also from FRU EEPROM.
2. Adding code change if the unsupported sensor type in the PLDM T2 Device.
   Please find the steps below for enabling PLDM T2 sensor/effecter
   1. [PLDM T2 Numeric Sensor](#pldm-t2-numeric-sensor)
   2. [PLDM T2 State Sensor](#the-steps-needed-by-developer-for-unsupported-state-sensor)
   3. [PLDM T2 Numeric Effecter](#the-steps-needed-by-developer-for-unsupported-numeric-effecter)
   4. [PLDM T2 State Effecter](#the-steps-needed-by-developer-for-unsupported-state-effecter)

#### PLDM T2 Numeric Sensor

- The upstream bmcweb has implementation to expose D-Bus sensor to RF API for
  the D-Bus path with sensor.value and Association.Definition PDIs.
- PLDM T2 numeric sensors are exposed to the RF URL /redfish/v1/Chassis/<$ChassisID>/Sensors/<$SensorName>
  by default
  - The ChassisID is from the Association property of xyz.openbmc_project.Association.Definition PDI
  - The sensorName is from D-Bus object path where the xyz.openbmc_project.Sensor.Value PDI is.

| Redfish Property Name | Description                                                      | backend PDI                                                          |
| --------------------- | ---------------------------------------------------------------- | -------------------------------------------------------------------- |
| Name                  | The Sensor name is from Sensor Auxiliary Name PDR                | The leaf of D-Bus path where xyz.openbmc_project.Sensor.Value PDI is |
| Reading               | Reading is from the response of PLDM T2 getSensorReading command | The value property of xyz.openbmc_project.Sensor.Value               |
| ReadingRangeMax       | Range is from the Max Readable field of Numeric Sensor PDR       | The MaxValue property of xyz.openbmc_project.Sensor.Value            |
| ReadingRangeMin       | Range is from the Min Readable field of Numeric Sensor PDR       | The MinValue property of xyz.openbmc_project.Sensor.Value            |
| ReadingType           | According to the base unit field of numeric sensor PDR           | The unit property of xyz.openbmc_project.Sensor.Value                |

#### PLDMT T2 state sensor and numeric/state effecter

- All of RF API for PLDM T2 state sensor and numeric/state effecter are platform specific.
  because the upstream bmcweb has no implementation for PLDM Numeric/State Effecter

##### The steps needed by Developer for unsupported state sensor

1. Defining a new PDI yaml file to phosphor-dbus-interfaces repo.
2. Adding new hpp file to platform-mc/state_set folder for the implementation of new defined PDI
3. Modifying SateSetCreator::createSensor API to add new condition to add new stateSet class
4. Modifying bmcweb API backend hpp file where the RF API is belonged to.

- Example of enabling MemoryPageSpareChennelPresence state sensor
  1. Adding MemorySpareChannel yaml file to /phosphor-dbus-interfaces/yaml/com/nvidia/
  2. Adding memorySpareChannel.hpp file to pldm/oem/nvidia/platform-mc/state_set
     since it is a oem state sensor.
  3. Modifying StateSetCreator::createSensor API to return a StateSetMemorySpareChannel
     object if stateSetId is PLDM_STATESET_ID_PRESENCE and entityType is PLDM_ENTITY_PROC
     or PLDM_ENTITY_MEMORY_CONTROLLER
  4. Modifying /redfish-core/lib/processor.hpp since state sensor is for the RF API,
     /redfish/v1/Systems/{SYSTEM_ID}/Processors/<PROCESSOR_ID>/ProcessorMetrics
     4.1 The RF API handler function gets object path and service of state sensor
     by Association PDI.
     4.2 The D-Bus path of state sensor is from the all_states array of Association PDI.
     4.3 The D-Bus service of state sensor is retrieved by ObjectMapper GetObject method
     for the object path get with com.nvidia.MemorySpareChannel PDI

##### The steps needed by Developer for unsupported numeric effecter

1. Defining a new PDI yaml file to phosphor-dbus-interfaces repo.
2. Adding new hpp file to inherit NumericEffecterBaseUnit class and new defined PDI class
3. Modifying constructor of NumericEffecter to instantiate the new implemented class.
4. Modifying bmcweb API backend hpp file where the RF API is belonged to.

- Example of enabling Grace SatMC numeric effecter
  for CPU Power Limit control.
  1. Skip the step since the effecter using existing upstream xyz.openbmc_project.Control.Power.Cap PDI
  2. Adding numeric_effecter_power_cap.hpp for exposing xyz.openbmc_project.Control.Power.Cap PDI
  3. Add the case to instantiate NumericEffecterWattIntf class in constructor when the base unit is watt
     and entity type is PLDM_ENTITY_PROC or PLDM_ENTITY_PROC_IO_MODULE.
  4. Modifying bmcweb API backend hpp file where the RF API is belonged to.
     4.1 Adding The /redfish-core/lib/control.hpp the backend hpp file for the power control
     4.2 Adding get and patch callback function to set PowerCapEnable and PowerCap properties of
     xyz.openbmc_project.Control.Power.Cap PDI

##### The steps needed by Developer for unsupported state effecter

1. Defining a new PDI yaml file to phosphor-dbus-interfaces repo.
2. Adding new hpp file to platform-mc/state_set folder for the implementation of new defined PDI
3. Modify SateSetCreator::createEffecter API to add new condition to add new stateSet class
4. Modify bmcweb API backend hpp file where the RF API is belonged to.

- Example of enabling clear nonvolatileVariable state effecter
  1. Adding ClearNonVolatileVariable yaml file to /phosphor-dbus-interfaces/yaml/xyz/openbmc_project/Control/Boot
  2. Adding clearNonVolatileVariables.hpp file to pldm/platform-mc/state_set
  3. Modifying StateSetCreator::createEffecter API to return a ClearNonVolatileVariablesStateIntf
     object if stateSetId is PLDM_STATESET_ID_BOOT_REQUEST
  4. Modifying /redfish-core/lib/bios.hpp since state effecter is for the RF API,
     /redfish/v1/Systems/{SYSTEM_ID}/Bios/Actions/Bios.ResetBios post action to invoke the Clear D-Bus method
     of xyz.openbmc_project.Control.Boot.ClearNonVolatileVariables PDI
