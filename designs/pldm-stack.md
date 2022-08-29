# PLDM stack on OpenBMC

Author: Deepak Kodihalli <dkodihal@linux.vnet.ibm.com> <dkodihal>

Primary assignee: Deepak Kodihalli

Created: 2019-01-22

## Problem Description
On OpenBMC, in-band IPMI is currently the primary industry-standard means of
communication between the BMC and the Host firmware. We've started hitting some
inherent limitations of IPMI on OpenPOWER servers: a limited number of sensors,
and a lack of a generic control mechanism (sensors are a generic monitoring
mechanism) are the major ones. There is a need to improve upon the communication
protocol, but at the same time inventing a custom protocol is undesirable.

This design aims to employ Platform Level Data Model (PLDM), a standard
application layer communication protocol defined by the DMTF. PLDM draws inputs
from IPMI, but it overcomes most of the latter's limitations. PLDM is also
designed to run on standard transport protocols, for e.g. MCTP (also designed by
the DMTF). MCTP provides for a common transport layer over several physical
channels, by defining hardware bindings. The solution of PLDM over MCTP also
helps overcome some of the limitations of the hardware channels that IPMI uses.

PLDM's purpose is to enable all sorts of "inside the box communication": BMC -
Host, BMC - BMC, BMC - Network Controller and BMC - Other (for e.g. sensor)
devices.

## Background and References
PLDM is designed to be an effective interface and data model that provides
efficient access to low-level platform inventory, monitoring, control, event,
and data/parameters transfer functions. For example, temperature, voltage, or
fan sensors can have a PLDM representation that can be used to monitor and
control the platform using a set of PLDM messages. PLDM defines data
representations and commands that abstract the platform management hardware.

PLDM groups commands under broader functions, and defines
separate specifications for each of these functions (also called PLDM "Types").
The currently defined Types (and corresponding specs) are : PLDM base (with
associated IDs and states specs), BIOS, FRU, Platform monitoring and control,
Firmware Update and SMBIOS. All these specifications are available at:

https://www.dmtf.org/standards/pmci

Some of the reasons PLDM sounds promising (some of these are advantages over
IPMI):

- Common in-band communication protocol.

- Already existing PLDM Type specifications that cover the most common
  communication requirements. Up to 64 PLDM Types can be defined (the last one
  is OEM). At the moment, 6 are defined. Each PLDM type can house up to 256 PLDM
  commands.

- PLDM sensors are 2 bytes in length.

- PLDM introduces the concept of effecters - a control mechanism. Both sensors
  and effecters are associated to entities (similar to IPMI, entities can be
  physical or logical), where sensors are a mechanism for monitoring and
  effecters are a mechanism for control. Effecters can be numeric or state
  based. PLDM defines commonly used entities and their IDs, but there 8K slots
  available to define OEM entities.

- A very active PLDM related working group in the DMTF.

The plan is to run PLDM over MCTP. MCTP is defined in a spec of its own, and a
proposal on the MCTP design is in discussion already. There's going to be an
intermediate PLDM over MCTP binding layer, which lets us send PLDM messages over
MCTP. This is defined in a spec of its own, and the design for this binding will
be proposed separately.

## Requirements
How different BMC applications make use of PLDM messages is outside the scope
of this requirements doc. The requirements listed here are related to the PLDM
protocol stack and the request/response model:

- Marshalling and unmarshalling of PLDM messages, defined in various PLDM Type
  specs, must be implemented. This can of course be staged based on the need of
  specific Types and functions. Since this is just encoding and decoding PLDM
  messages, this can be a library that could shared between the BMC, and other
  firmware stacks. The specifics of each PLDM Type (such as FRU table
  structures, sensor PDR structures, etc) are implemented by this lib.

- Mapping PLDM concepts to native OpenBMC concepts must be implemented. For
  e.g.: mapping PLDM sensors to phosphor-hwmon hosted D-Bus objects, mapping
  PLDM FRU data to D-Bus objects hosted by phosphor-inventory-manager, etc. The
  mapping shouldn't be restrictive to D-Bus alone (meaning it shouldn't be
  necessary to put objects on the Bus just to serve PLDM requests, a problem
  that exists with phosphor-host-ipmid today). Essentially these are platform
  specific PLDM message handlers.

- The BMC should be able to act as a PLDM responder as well as a PLDM requester.
  As a PLDM requester, the BMC can monitor/control other devices. As a PLDM
  responder, the BMC can react to PLDM messages directed to it via requesters in
  the platform.

- As a PLDM requester, the BMC must be able to discover other PLDM enabled
  components in the platform.

- As a PLDM requester, the BMC must be able to send simultaneous messages to
  different responders.

- As a PLDM requester, the BMC must be able to handle out of order responses.

- As a PLDM responder, the BMC may simultaneously respond to messages from
  different requesters, but the spec doesn't mandate this. In other words the
  responder could be single-threaded.

- It should be possible to plug-in OEM PLDM types/functions into the PLDM stack.

- As a PLDM sensor monitoring daemon, the BMC must be able to enumerate and
  monitor the static or self-described(with PDRs) PLDM sensors in satellite
  Management Controller, on board device or PCIe add-on card.

## Proposed Design
This document covers the architectural, interface, and design details. It
provides recommendations for implementations, but implementation details are
outside the scope of this document.

The design aims at having a single PLDM daemon serve both the requester and
responder functions, and having transport specific endpoints to communicate
on different channels.

The design enables concurrency aspects of the requester and responder functions,
but the goal is to employ asynchronous IO and event loops, instead of multiple
threads, wherever possible.

The following are high level structural elements of the design:

### PLDM encode/decode libraries

This library would take a PLDM message, decode it and extract the different
fields of the message. Conversely, given a PLDM Type, command code, and the
command's data fields, it would make a PLDM message. The thought is to design
this as a common library, that can be used by the BMC and other firmware stacks,
because it's the encode/decode and protocol piece (and not the handling of a
message).

### PLDM provider libraries

These libraries would implement the platform specific handling of incoming PLDM
requests (basically helping with the PLDM responder implementation, see next
bullet point), so for instance they would query D-Bus objects (or even something
like a JSON file) to fetch platform specific information to respond to the PLDM
message. They would link with the encode/decode lib.

It should be possible to plug-in a provider library, that lets someone add
functionality for new PLDM (standard as well as OEM) Types. The libraries would
implement a "register" API to plug-in handlers for specific PLDM messages.
Something like:

template <typename Handler, typename... args>
auto register(uint8_t type, uint8_t command, Handler handler);

This allows for providing a strongly-typed C++ handler registration scheme. It
would also be possible to validate the parameters passed to the handler at
compile time.

### Request/Response Model

The PLDM daemon links with the encode/decode and provider libs. The daemon
would have to implement the following functions:

#### Receiver/Responder
The receiver wakes up on getting notified of incoming PLDM messages (via D-Bus
signal or callback from the transport layer) from a remote PLDM device. If the
message type is "Request" it would route them to a PLDM provider library. Via
the library, asynchronous D-Bus calls (using sdbusplus-asio) would be made, so
that the receiver can register a handler for the D-Bus response, instead of
having to wait for the D-Bus response. This way it can go back to listening for
incoming PLDM messages.

In the D-Bus response handler, the receiver will send out the PLDM response
message via the transport's send message API. If the transport's send message
API blocks for a considerably long duration, then it would have to be run in a
thread of it's own.

If the incoming PLDM message is of type "Response", then the receiver emits a
D-Bus signal pointing to the response message. Any time the message is too
large to fit in a D-Bus payload, the message is written to a file, and a
read-only file descriptor pointing to that file is contained in the D-Bus
signal.

#### Requester
Designing the BMC as a PLDM requester is interesting. We haven't had this with
IPMI, because the BMC was typically an IPMI server. PLDM requester functions
will be spread across multiple OpenBMC applications (instead of a single big
requester app) - based on the responder they're talking to and the high level
function they implement. For example, there could be an app that lets the BMC
upgrade firmware for other devices using PLDM - this would be a generic app
in the sense that the same set of commands might have to be run irrespective
of the device on the other side. There could also be an app that does fan
control on a remote device, based on sensors from that device and algorithms
specific to that device.

##### Proposed requester design

A requester app/flow comprises of the following :

- Linkage with a PLDM encode/decode library, to be able to pack PLDM requests
  and unpack PLDM responses.

- A D-Bus API to generate a unique PLDM instance id. The id needs to be unique
  across all outgoing PLDM messages (from potentially different processes).
  This needs to be on D-Bus because the id needs to be unique across PLDM
  requester app processes.

- A requester client API that provides blocking and non-blocking functions to
  transfer a PLDM request message and to receive the corresponding response
  message, over MCTP (the blocking send() will return a PLDM response).
  This will be a thin wrapper over the socket API provided by the mctp demux
  daemon. This will provide APIs for common tasks so that the same may not
  be re-implemented in each PLDM requester app. This set of API will be built
  into the encode/decode library (so libpldm would house encode/decode APIs, and
  based on a compile time flag, the requester APIs as well). A PLDM requester
  app can choose to not use the client requester APIs, and instead can directly
  talk to the MCTP demux daemon.

##### Proposed requester design - flow diagrams

a) With blocking API

```
+---------------+               +----------------+            +----------------+               +-----------------+
|BMC requester/ |               |PLDM requester  |            |PLDM responder  |               |PLDM Daemon      |
|client app     |               |lib (part of    |            |                |               |                 |
|               |               |libpldm)        |            |                |               |                 |
+-------+-------+               +-------+--------+            +--------+-------+               +---------+-------+
        |                               |                              |                                 |
        |App starts                     |                              |                                 |
        |                               |                              |                                 |
        +------------------------------->setup connection with         |                                 |
        |init(non_block=false)          |MCTP daemon                   |                                 |
        |                               |                              |                                 |
        +<-------+return_code+----------+                              |                                 |
        |                               |                              |                                 |
        |                               |                              |                                 |
        |                               |                              |                                 |
        +------------------------------>+                              |                                 |
        |encode_pldm_cmd(cmd code, args)|                              |                                 |
        |                               |                              |                                 |
        +<----+returns pldm_msg+--------+                              |                                 |
        |                               |                              |                                 |
        |                               |                              |                                 |
        |----------------------------------------------------------------------------------------------->|
        |DBus.getPLDMInstanceId()       |                              |                                 |
        |                               |                              |                                 |
        |<-------------------------returns PLDM instance id----------------------------------------------|
        |                               |                              |                                 |
        +------------------------------>+                              |                                 |
        |send_msg(mctp_eids, pldm_msg)  +----------------------------->+                                 |
        |                               |write msg to MCTP socket      |                                 |
        |                               +----------------------------->+                                 |
        |                               |call blocking recv() on socket|                                 |
        |                               |                              |                                 |
        |                               +<-+returns pldm_response+-----+                                 |
        |                               |                              |                                 |
        |                               +----+                         |                                 |
        |                               |    | verify eids, instance id|                                 |
        |                               +<---+                         |                                 |
        |                               |                              |                                 |
        +<--+returns pldm_response+-----+                              |                                 |
        |                               |                              |                                 |
        |                               |                              |                                 |
        |                               |                              |                                 |
        +------------------------------>+                              |                                 |
        |decode_pldm_cmd(pldm_resp,     |                              |                                 |
        |                output args)   |                              |                                 |
        |                               |                              |                                 |
        +------------------------------>+                              |                                 |
        |close_connection()             |                              |                                 |
        +                               +                              +                                 +
```


b) With non-blocking API

```
+---------------+               +----------------+            +----------------+             +---------------+
|BMC requester/ |               |PLDM requester  |            |PLDM responder  |             |PLDM daemon    |
|client app     |               |lib (part of    |            |                |             |               |
|               |               |libpldm)        |            |                |             |               |
+-------+-------+               +-------+--------+            +--------+-------+             +--------+------+
        |                               |                              |                              |
        |App starts                     |                              |                              |
        |                               |                              |                              |
        +------------------------------->setup connection with         |                              |
        |init(non_block=true            |MCTP daemon                   |                              |
        |     int* o_mctp_fd)           |                              |                              |
        |                               |                              |                              |
        +<-------+return_code+----------+                              |                              |
        |                               |                              |                              |
        |                               |                              |                              |
        |                               |                              |                              |
        +------------------------------>+                              |                              |
        |encode_pldm_cmd(cmd code, args)|                              |                              |
        |                               |                              |                              |
        +<----+returns pldm_msg+--------+                              |                              |
        |                               |                              |                              |
        |-------------------------------------------------------------------------------------------->|
        |DBus.getPLDMInstanceId()       |                              |                              |
        |                               |                              |                              |
        |<-------------------------returns PLDM instance id-------------------------------------------|
        |                               |                              |                              |
        |                               |                              |                              |
        +------------------------------>+                              |                              |
        |send_msg(eids, pldm_msg,       +----------------------------->+                              |
        |         non_block=true)       |write msg to MCTP socket      |                              |
        |                               +<---+return_code+-------------+                              |
        +<-+returns rc, doesn't block+--+                              |                              |
        |                               |                              |                              |
        +------+                        |                              |                              |
        |      |Add EPOLLIN on mctp_fd  |                              |                              |
        |      |to self.event_loop      |                              |                              |
        +<-----+                        |                              |                              |
        |                               +                              |                              |
        +<----------------------+PLDM response msg written to mctp_fd+-+                              |
        |                               +                              |                              |
        +------+EPOLLIN on mctp_fd      |                              |                              |
        |      |received                |                              |                              |
        |      |                        |                              |                              |
        +<-----+                        |                              |                              |
        |                               |                              |                              |
        +------------------------------>+                              |                              |
        |decode_pldm_cmd(pldm_response) |                              |                              |
        |                               |                              |                              |
        +------------------------------>+                              |                              |
        |close_connection()             |                              |                              |
        +                               +                              +                              +
```

### MCTP endpoint discovery
'pldmd'(PLDM daemon) utilizes the [MCTP D-Bus interfaces](https://github.com/openbmc/phosphor-dbus-interfaces/tree/master/yaml/xyz/openbmc_project/MCTP)
to enumerate all MCTP endpoints in the system. The MCTP D-Bus interface
implements the SupportedMessageTypes to have which Message type supported by
each endpoint. 'pldmd' watches the interfacesAdded D-Bus signal from MCTP D-Bus
interface to get the latest EID table. 'pldmd' can also take the EID table from
JSON file for the MCTP service which does not implement MCTP D-Bus interfaces.

### Terminus management
'pldmd' will maintain a terminus table to manage the PLDM terminus in system.
When 'pldmd' received updated EID table from MCTP D-Bus interface, 'pldmd'
should check if the EID support PLDM message type(0x01) and then adds the EID
which is not in the terminus table yet. 'pldmd' should also clean up the
removed endpoint from terminus table by checking if the EID in terminus table
is also in the updated EID table from MCTP D-Bus interface. When a new terminus
is added in terminus table, 'pldmd' should send getTID, setTID to assign a
Terminus ID(TID) to the EID and provide APIs for other services to lookup the
mapping from EID to TID and vice versa. After done that, 'pldmd' should send
GetPLDMType and GetPLDMVersions commands to the terminus to record what PLDM
type message/version the terminus supports in terminus table and provide APIs
for other services to query. If there is any change in terminus table, 'pldmd'
should invoke callback function of other services to process the event of
terminus adding/removing.

### Sensor Monitoring
To find out all sensors from PLDM terminus, 'pldmd' should retrieve all the
Numeric Sensor PDRs by PDR Repository commands(GetPDRRepositoryInfo, GetPDR)
for the necessary parameters(e.g., sensorID, unit, etc.). 'pldmd' can use
libpldm encode/decode APIs(encode_get_pdr_repository_info_req,
decode_get_pdr_repository_info_resp, encode_get_pdr_req, decode_get_pdr_resp)
to build the commands message and then send it to PLDM terminus.

Regarding to the static device described in section 8.3.1 of DSP0248 1.2.1,
the device uses PLDM for access only and doesn't support PDRs. The PDRs for the
device needs to be encoded by Platform specific PDR JSON file by the platform
developer. 'pldmd' will generate these sensor PDRs encoded by JSON files and
parse them as the same as the PDRs fetched by PLDM terminus.

#### Numeric Sensors 
'pldmd' should expose the found PLDM numeric sensor to D-Bus object path. The object
path consists of the name from Sensor Aux Name PDR and TID# with the format,
"/xyz/openbmc_project/sensors/<sensor_type>/<$SensorAuxName_SensorID#_TID#>".
If the Sensor Aux Name is absent, then using the format,
"/xyz/openbmc_project/sensors/<sensor_type>/PLDM_Device_<SensorID#_TID#>"
instead. For exposing sensor status to D-Bus, 'pldmd' should implement
following D-Bus interfaces to the D-Bus object path of PLDM sensor.
- [xyz.openbmc_project.Sensor.Value](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Sensor/Value.interface.yaml),
the interface exposes the sensor reading unit, value, Max/Min Value.

- [xyz.openbmc_project.State.Decorator.OperationalStatus](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/State/Decorator/OperationalStatus.interface.yaml),
the interface exposes the sensor status which is functional or not.

- [xyz.openbmc_project.Association.Definitions](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Association/Definitions.interface.yaml)  
'pldmd' builds a containment hierarchy from Entity Association PDR, the
hierarchy contains all entities, and each entity looks for its D-Bus inventory
by looking for relevant D-Bus interfaces, and the matched InstanceNumber in
D-Bus interface Decorator.Instance. Below example provides the mapping between
PLDM Entity [DSP0249
1.0.0](https://www.dmtf.org/sites/default/files/standards/documents/DSP0249_1.0.0.pdf)
and the chosen dbus interface.

| PLDM Entity                         | D-Bus interface                                    |
|-------------------------------------|----------------------------------------------------|
| Overall System(Container ID 0x0000) | xyz.openbmc_project.Inventory.Item.System          |
| Processor/IO module(Entity ID 81)   | xyz.openbmc_project.Inventory.Item.ProcessorModule |
| Processor(Entity ID 135)            | xyz.openbmc_project.Inventory.Item.Cpu             |

When there are multiple D-Bus inventories match the D-Bus interface and
InstanceNumber, 'pldmd' looks for the D-Bus path under its container's inventory
path. For example a PLDM entity may match both: `/foo1/bar0` and `/foo2/bar0`,
in this case pldmd would get the entity's parent from association PDRs and that
would match foo1 or foo2; if a match occurs at this stage then we're good, else
continue. Match should stop at top-most entity which should match Item.System.

Every PLDM sensor adds the DBus association with its DBus inventory, if the
entity does not find a DBus inventory, look for the closest container entity
inventory from the hierarchy.
```
{'chassis','all_sensors', inventory}
```

e.g., There are existing D-Bus inventories below:
```
/xyz/openbmc_project/inventory/system/board/BaseBoard - Item.System
├── Procssor_Module0 - Item.ProcessorModule - InstanceNumber 0
│   └── Processor0 - Item.Cpu - InstanceNumber 0
└── Procssor_Module1 - Item.ProcessorModule - InstanceNumber 1
    ├── Processor0 - Item.Cpu - InstanceNumber 0
    └── Processor1 - Item.Cpu - InstanceNumber 1
```

Example PLDM entities find their corresponding inventories.
- A: EntityId 81, EntityInstance 0, it associates to Procssor_Module0
- B: EntityId 81, EntityInstance 1, it associates to Procssor_Module1
- C: EntityId 135, EntityInstance 0, container is A, it associates to Processor0
  under Procssor_Module0
- D: EntityId 135, EntityInstance 0, container is B, it associates to Processor0
  under Procssor_Module1
- E: EntityId 135, EntityInstance 1, container is B, it associates to Processor1
  under Procssor_Module1
- F: EntityId 143(Memory controller), container is A, no corresponding
  inventories found, it associates to the closest container inventory
  Procssor_Module0

For NVidia project, the Entity Manager(EM) should expose the baseboard inventory
and ProcessorModule inventories, EM knows the instanceNumber of the module by
predefined I2C bus number and address in configuration JSON files. smbios-mdr
should expose the processor inventories from SMBIOS table, the Socket
Designation of SMBIOS table type 4 has the location information in the format
`SilkScreenLabel:LogicalSocket.LogicalChip`, smbios-mdr find the container
ProcessorModule inventory with the instanceNumber equal to `LogicalSocket`, then
create the Processor inventory with the instanceNumber `LogicalChip` under the
container's inventory.
```
/xyz/openbmc_project/inventory/system/board/CG4_BaseBoard (EM)
├── CG1_Module0 (EM)
│   └── cpu0 (smbios-mdr. Socket Designation: label:0.0)
└── C2_Module1 (EM)
    ├── cpu0 (smbios-mdr. Socket Designation: label:1.0)
    └── cpu1 (smbios-mdr. Socket Designation: label:1.1)
```
'pldmd' creates D-Bus associations for its sensors.
```
Example associations of sensors with Module entity, instanceNumber 0:
{'chassis','all_sensors', '/xyz/openbmc_project/inventory/system/board/CG4_BaseBoard/CG1_Module0'}
Example association of sensor with Grace processor entity, instanceNumber 0, and parent is CG1_Module0:
{'chassis','all_sensors', '/xyz/openbmc_project/inventory/system/board/CG4_BaseBoard/CG1_Module0/cpu0'}
Example association of sensor with VREGs entity, and parent is CG1_Module0:
{'chassis','all_sensors', '/xyz/openbmc_project/inventory/system/board/CG4_BaseBoard/CG1_Module0'}
```

But the current EM can only create inventories on the same level, it lost the
hierarchical relation between BaseBoard and Modules. It has not caused the issue
because we only have one BaseBoard, all instance numbers under the Processor/IO
module(Entity ID 81) are unique, and the modules do not need the containing
relation to find the correct inventory.
```
/xyz/openbmc_project/inventory/system/board
├── CG4_BaseBoard
├── CG1_Module0
│   └── cpu0
└── C2_Module1
    ├── cpu0
    └── cpu1
```

After doing the discovery of PLDM sensors, 'pldmd' should initialize all found
sensors by necessary commands(e.g., SetNumericSensorEnable, SetSensorThresholds
,SetSensorHysteresis, and InitNumericSensor) and then start to update the
sensor status to D-bus objects by polling or async event method depending on
the capability of PLDM terminus.

'pldmd' should update the value property of Sensor.Value D-Bus interface after
getting the response of GetSensorReading command successfully. If 'pldmd'
failed to get the response from PLDM terminus or the completion code returned
by PLDM terminus is not PLDM_SUCCESS, the Functional property of
State.Decorator.OperationalStatus D-Bus interface should be updated to false.

#### State Sensors 
'pldmd' should expose the found PLDM state sensors to D-Bus object path. The object
path consists of the name from Sensor Aux Name PDR and TID# with the format,
"/xyz/openbmc_project/State/<$SensorAuxName_SensorID#_TID#>" when Aux Name PDR contains single Aux name.
If the Sensor Aux Name is absent, OR contains more than one Aux name (composite sensor with 
sensor count more than 1) then using the format,
"/xyz/openbmc_project/State/PLDM_Device_<SensorID#_TID#>"
instead.
The dbus interface expossed by the sensor will be dependent upon the state set ids 
present in the state sensor PDR. For each of the state id present a relavant dbus 
interface will be implmemented. 
Below example provides the mapping between state set id [DSP0249 1.0.0](https://www.dmtf.org/sites/default/files/standards/documents/DSP0249_1.0.0.pdf) and the chosen dbus interface. 
| State Set ID               | Interface                                                  |       
| ---------------------------|:----------------------------------------------------------:| 
| Performance (14)           |  xyz.openbmc_project.State.ProcessorPerformance            | 
| Power Supply State (256)   |  xyz.openbmc_project.State.Decorator.PowerSystemInputs     |   

'pldmd' will also create the association between the state sensor object and the related 'chassis' entity 
found via the entity assocition PDR. 
```
{"chassis", "all_states", inventory_path}
```

#### Polling v.s. Async method
'pldmd' maintains a list to poll all PLDM sensors and expose the status to
D-Bus. 'pldmd' has a polling timer to update PLDM sensors periodically. The
timeout of polling timer is 1 second. The PLDM sensor in list has a counter
which is initialized to the value of updateIntervals defined in PDR. Upon
polling timer timeout, the sensor's counter is decreased by 1 second. When
sensor's counter reaches zero, 'pldmd' will send GetSensorReading command to
the PLDM sensor and then reset sensor's counter back to initial value. 'pldmd'
should have APIs to be paused and resumed by other task(e.g. pausing sensor
polling during firmware updating to maximum bandwidth).

To enable async event method for a sensor to update its status to 'pldmd',
'pldmd' needs to implement the responder of PlatformEventMessage command
described in 13.1 PLDM Event Message of [DSP0248 1.2.1](https://www.dmtf.org/sites/default/files/standards/documents/DSP0248_1.2.1.pdf). 'pldmd' checks the
response of EventMessageSupported command from PLDM terminus to identify if it
can generate events. A PLDM sensor can work in event aync method if the
updateIntervals of all sensors in the same PLDM terminus are longer than final
polling time. Before 'pldmd' starts to receive async event from PLDM terminus,
'pldmd' should remove the sensor from poll list and then send necessary
commands(e.g., EventMessageBufferSize and SetEventReceiver) to PLDM terminus
for the initialization.

### Numeric Effecters
'pldmd' should expose the found PLDM effecters to D-Bus object path. The object
path consists of the name from Effecter Auxiliary Names PDR and TID# with the
format, "/xyz/openbmc_project/control/<$EffecterAuxName_TID#>". If the Effecter
Aux Name is absent, then using the format,
"/xyz/openbmc_project/control/PLDM_Effecter_<TID#>_<EffecterID#>" instead.
'pldmd' creates different D-Bus interface for different baseUnit in Numeric
Effecter PDR.

#### Watts
For numeric effecter with baseUnit Watts, 'pldmd' should implement the D-Bus
interface, xyz.openbmc_project.Control.Power.Cap, This interface display the
present value of the effecter in PowerCap property, and Send
SetNumericEffecterValue command to terminus if the PowerCap was set.

'pldmd' will also create the association between the effecter object and the
related 'chassis' entity found via the entity assocition PDR.
```
{'chassis', 'power_controls', inventory_path}
```

'pldmd' also creates `xyz.openbmc_project.State.Decorator.OperationalStatus`
D-Bus interface, the properties values are according to the
effecterOperationalState in response of GetNumericEffecterValue Command.

| effecterOperationalState | Functional | State              |
|--------------------------|------------|--------------------|
| enabled-updatePending    | true       | Deferring          |
| enabled-noUpdatePending  | true       | Enabled            |
| disabled                 | false      | Disabled           |
| unavailable              | false      | UnavailableOffline |
| statusUnknown            | false      | UnavailableOffline |
| failed                   | false      | UnavailableOffline |
| initializing             | false      | Starting           |
| shuttingDown             | false      | UnavailableOffline |
| inTest                   | false      | UnavailableOffline |

For NVidia Grace platform, SatMC only have 2 numeric effecters, `Set volatile
TDP limit` and `Set non-volatile TDP limit`. Their baseUnit are both Watts,
'pldmd' should create association like this.
```
{'chassis', 'power_controls', '/xyz/openbmc_project/inventory/system/board/CG4_BaseBoard/CG1_Module0/cpu0'}
```

The bmcweb query the 'power_controls' association for all Chassis, then bmcweb
create the "/redfish/v1/Chassis/{ChassisId}/Controls/{ControlId}" for every
objects it found.

#### OEM PDR
If 'pldmd' read any OEM PDR with vendorIANA equal to 0x1647(NVIDIA), and
vendorSpecificData[3] equal to 0x01(Effecter Lifetime), 'pldmd' parse the OEM
PDR as below table, and it adds the interface
xyz.openbmc_project.State.Decorator.Lifetime to the effecter D-Bus object.

|       Type  |     OEM PDR Description  |                                                      OEM Effecter Lifetime PDR Description                                                  |
|-------------|:------------------------:|:-------------------------------------------------------------------------------------------------------------------------------------------:|
|     -       |   commonHeader           |   commonHeader                                                                                                                              |
|     uint32  |   vendorIANA             |   vendorIANA – 0x1647 for NVIDIA                                                                                                            |
|     uint16  |   OEMRecordID            |   OEMRecordID – as same as EffecterID for this OEM Effecter Lifetime PDR                                                                    |
|     uint16  |   dataLength             |   dataLength                                                                                                                                |
|     byte    |   vendorSpecificData[1]  |   Lower byte of the PLDMTerminusHandle                                                                                                      |
|     byte    |   vendorSpecificData[2]  |   Upper byte of the PLDMTerminusHandle                                                                                                      |
|     Byte    |   vendorSpecificData[3]  |   OEM PDR type – 0x01 for this OEM Effecter Lifetime PDR                                                                                    |
|     byte    |   vendorSpecificData[4]  |   OEM Effecter Lifetime ID  0x00 – volatile (valid until next reset)  0x01 – non-volatile (permanent unless a variable storage is cleared)  |
|     byte    |   vendorSpecificData[5]  |   Lower byte of the associated Effecter ID                                                                                                  |
|     byte    |   vendorSpecificData[6]  |   Upper byte of the associated Effecter ID                                                                                                  |

### Event handling
The pldmd might be responder to handle PlatformEventMessage from terminus
asynchronously or be requester to send PollForPlatformEventMessage actively.
Before pldmd ready to receive event from terminus, it should execute the
steps below to initialize each terminus.

#### Terminus Initialization
1. Send EventMessageSupported command to get value of synchronyConfiguration.
If terminus doesn’t support asynchronous messaging. Put the terminus in the
polling list.
2. For each sensor if its terminus supports asynchronous messaging
    * Send SetNumericSensorEnable command to turn on eventMessageEnable if it
    is a numeric sensor.
    * Send SetSateSensorEnables command to turn on eventMessageEnable if it is
    a state sensor.
3. Send EventMessageBufferSize to report receiverMaxBufferSize and get
terminusMaxBufferSize. pldmd set the buffer size to min(receiverMaxBufferSize,
terminusMaxBufferSize)
4. Send SetEventReceiver command to enable terminus Async or Polling mode and
set up the receiver address(MCTP/EID).

After terminus initialized, the pldmd should response PlatformEventMessage
from terminus and process received event data according to its event class.
For terminus in polling list added in step1, pldmd should perform the steps
of handling pldmMessagePollEvent Periodically.

#### Event Classes
* sensorEvent
    * NumericSensorState
        1. Update sensor's D-Bus Value.interface, Warning.interface and
        Critical.interface according to the values(eventState, previousState,
        presentReading) in PLDM event.
        2. Log the event message by D-Bus xyz.openbmc_project.Logging.Create
        interface,Create Method with parameters below.
            - Message: one of message registries below based on the eventState
            and previousState fields in PLDM event.
                * OpenBMC.0.2.SensorThresholdCriticalHighGoing[High|Low]
                * OpenBMC.0.2.SensorThresholdCriticalLowGoing[High|Low]
                * OpenBMC.0.2.SensorThresholdWarningHighGoing[High|Low]
                * OpenBMC.0.2.SensorThresholdWarningLowGoing[High|Low]
            - Severity: xyz.openbmc_project.Logging.Entry.Level based on
            eventState field in PLDM event.
            - AdditionalData: a dict including key-value pairs below.
                * REDFISH_MESSAGE_ID=[equal to Message parameter]
                * REDFISH_MESSAGE_ARGS=SensorName,presentReading,thresholdValue
**note:** SensorName is from D-Bus objPath, presentReading is from PLDM event,
thresholdValue is from Numeric Sensor PDR.
    * StateSensorState
        1. Update sensor's D-Bus IMPDEF interface according to the values in
        PLDM event.
        2. Log the event message by D-Bus xyz.openbmc_project.Logging.Create interface, Create Method with parameters below.
            - Message: IMPDEF Redfish registry MessageId.
            - Severity: xyz.openbmc_project.Logging.Entry.Level based on eventState field in PLDM event.
            - AdditionalData: a dict including key-value pairs below.
                * REDFISH_MESSAGE_ID=[equal to Message parameter]
                * REDFISH_MESSAGE_ARGS=IMPDEF                
**note:** An implementation can define Redfish message registries for all set ids
in [DSP0249 1.0.0](https://www.dmtf.org/sites/default/files/standards/documents/DSP0249_1.1.0.pdf)

* pldmMessagePollEvent
    1. pldmd sends pollForPlatformEventMessage(GetFirstPart, 0x0000, 0x00).
    2. Wait till response received and then cache event data in response.
    3. Check if TransferOpFlag=END|START_AND_END, goto 5.
    4. Send pollForPlatformEventMessage(GetNextPart, nextDataTransferHndl,
    0xffff) and goto 2.
    5. Assemble cached event data into completed event message.
    6. Process event message according to its event class.
    7. Send pollForPlatformEventMessage(AckOnly, 0x0000, eventId).
    8. Wait till response received and then goto 1 if eventId in response
    equals 0xffff.
* cperEvent
    1. Save the cper data to /var/cper/cper-XXXXXXX (filename is generated by
    mkstemp()).
    2. Generate Dump record by D-Bus xyz.openbmc_project.Dump.Create interface,
    CreateDump Method at /xyz/openbmc_project/dump/FaultLog objPath with parameters
    below.
        - AdditionalData: a dict including key-value pairs below.
            * CPER_TYPE=cper or cperSection
            * CPER_PATH=/var/cper/cper-XXXXXXX

**note:** The CPER dump entry is logged to D-Bus Object with [FaultLog.interface](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Dump/Entry/FaultLog.interface.yaml)
and the FaultLog Dump manager which implements createDump method should move
the CPER binary file from CPER_PATH to the dump folder managed by Dump.Manager
and perform the action when the total number or size of CPER dumps exceeded the
predefined limitation.

##### Alternative to the proposed requester design

a) Define D-Bus interfaces to send and receive PLDM messages :

```
method sendPLDM(uint8 mctp_eid, uint8 msg[])

signal recvPLDM(uint8 mctp_eid, uint8 pldm_instance_id, uint8 msg[])
```

PLDM requester apps can then invoke the above applications. While this
simplifies things for the user, it has two disadvantages :
- the app implementing such an interface could be a single point of failure,
  plus sending messages concurrently would be a challenge.
- the message payload could be large (several pages), and copying the same for
  D-Bus transfers might be undesirable.

### Multiple transport channels
The PLDM daemon might have to talk to remote PLDM devices via different
channels. While a level of abstraction might be provided by MCTP, the PLDM
daemon would have to implement a D-Bus interface to target a specific
transport channel, so that requester apps on the BMC can send messages over
that transport. Also, it should be possible to plug-in platform specific D-Bus
objects that implement an interface to target a platform specific transport.

### Processing PLDM FRU information sent down by the host firmware

Note: while this is specific to the host BMC communication, most of this might
apply to processing PLDM FRU information received from a device connected to the
BMC as well.

The requirement is for the BMC to consume PLDM FRU information received from the
host firmware and then have the same exposed via Redfish. An example can be the
host firmware sending down processor and core information via PLDM FRU commands,
and the BMC making this information available via the Processor and
ProcessorCollection schemas.

This design is built around the pldmd and entity-manager applications on the
BMC:

- The pldmd asks the host firmware's PLDM stack for the host's FRU record table,
  by sending it the PLDM GetFRURecordTable command. The pldmd should send this
  command if the host indicates support for the PLDM FRU spec. The pldmd
  receives a PLDM FRU record table from the host firmware (
  www.dmtf.org/sites/default/files/standards/documents/DSP0257_1.0.0.pdf). The
  daemon parses the FRU record table and hosts raw PLDM FRU information on
  D-Bus. It will house the PLDM FRU properties for a certain FRU under an
  xyz.openbmc_project.Inventory.Source.PLDM.FRU D-Bus interface, and house the
  PLDM entity info extracted from the FRU record set PDR under an
  xyz.openbmc_project.Source.PLDM.Entity interface.

- Configurations can be written for entity-manager to probe an interface like
  xyz.openbmc_project.Inventory.Source.PLDM.FRU, and create FRU inventory D-Bus
  objects. Inventory interfaces from the xyz.openbmc_project. Inventory
  namespace can be applied on these objects, by converting PLDM FRU property
  values into xyz.openbmc_project.Invnetory.Decorator.Asset property values,
  such as Part Number and Serial Number, in the entity manager configuration
  file. Bmcweb can find these FRU inventory objects based on D-Bus interfaces,
  as it does today.

## Alternatives Considered
Continue using IPMI, but start making more use of OEM extensions to
suit the requirements of new platforms. However, given that the IPMI
standard is no longer under active development, we would likely end up
with a large amount of platform-specific customisations. This also does
not solve the hardware channel issues in a standard manner.
On OpenPOWER hardware at least, we've started to hit some of the limitations of
IPMI (for example, we have need for >255 sensors).

## Impacts
Development would be required to implement the PLDM protocol, the
request/response model, and platform specific handling. Low level design is
required to implement the protocol specifics of each of the PLDM Types. Such low
level design is not included in this proposal.

Design and development needs to involve the firmware stacks of management
controllers and management devices of a platform management subsystem.

## Testing
Testing can be done without having to depend on the underlying transport layer.

The responder function can be tested by mocking a requester and the transport
layer: this would essentially test the protocol handling and platform specific
handling. The requester function can be tested by mocking a responder: this
would test the instance id handling and the send/receive functions.

APIs from the shared libraries can be tested via fuzzing.

The APIs to parse PDRs from PLDM terminus can be tested by a mocking responder.
A sample JSON file is provided to test the APIs for mocking PDRs for static
PLDM sensors.