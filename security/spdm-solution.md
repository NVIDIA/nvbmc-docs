# SPDM solution

Author:
Marcin Nowakowski <marcin.nowakowski@conclusive.pl>

Primary assignee:

Other contributors:

Created: 2021-11-30

## Problem Description

SPDM is a significant securtiy extension for Open BMC as it does not provide SPDM implementation.
It will be usefull for ROT hardware which tries to get the attestation data
from all the devices to authenticate the firmware on the connected devices.

### User scenarios
1. BMC authenticates all the SPDM supported devices.
Basic on the authentication results BMC may decide to use a device or shut it down.
Also, BMC may decide to shut down devices, which do not support SPDM.
2. BMC gets measurements data from the SPDM supported devices.

## Background and References

SPDM specification is described here:
[[1] Security Protocol and Data Model (SPDM) Specification (DSP0274)](https://www.dmtf.org/sites/default/files/standards/documents/DSP0274_1.1.1.pdf)

*The Security Protocol and Data Model (SPDM) Specification defines messages, data objects,
and sequences for performing message exchanges between devices over a variety of transport
and physical media. The description of message exchanges includes authentication of hardware
identities, measurement for firmware identities and session key exchange protocols to enable
confidentiality and integrity protected data communication. The SPDM enables efficient access
to low-level security capabilities and operations.
Other mechanisms, including non-PMCI- and DMTF- defined mechanisms, can use the SPDM.*

## Requirements

The currently available open spdm reference design is not ready to be used in
the current Open BMC solution. Additionally, its license requirements are
limiting open and free usage of the design.

[[2] libspdm - a sample implementation that follows the DMTF SPDM specification](https://github.com/DMTF/libspdm)

SPDM requires to be implemented up to version 1.1.
Version 1.0 requires implementation of certificates verification.
Version 1.1 introduces secure communication.
In this project the SPDM communication must be implemented over MCTP.
Getting measurements is mandatory and must be supported.
Mutual authentication should be supported.
Communication over the PCI DOE is optional.
Sessions are alo optional to be supported in this implementation.
Responder functionality in SPDM deamon is not required in this project.
The implementation cannot use the mentioned open SPDM github project, but may use a similar approach.

## Proposed Design

### Overview
SPDM module will consists of three parts:
- SPDM library - prepared as a separate library to support SPDM deamon,
unitary tests and any external responder or requester implementation
- SPDM deamon - main deamon to manage SPDM queries
- SPDM unitary tests - unitary tests for SPDM library implementation

The solution could be preapred as a single SPDM deamon with dbus interface and
without a dedicated library. However, layering the design into deamon and library
may be usefull to reuse the source code for other responders. Also, it should make
unitary tests more robust.

### Dependencies
- SPDM library requires MCTP control module
- SPDM library requires also security libraries
- SPDM deamon requires accessing dbus

### Implementation assumptions
- Build system: meson
- Language for all source code files: `C++` (`C++` is chosen to distinguish
implementation from open spdm reference project and also to speed up development,
especially using standard classes)
- SPDM solution is meant to work in Linux environment, and it will not support Windows environment.
- We will not use shared (or global) variables. The whole implementation should be "thread safe"
and use std::mutex and std::lock_guard if required.
- The deamon should work in Linux user space - we do not predict kernel modules in the implementation.

### SPDM library - *libspdmcpp*
The library should provide the following external API
- Serialization and deserialization of SPDM messages.
- Sending and receiving messages over MCTP with SPDM payload in secured and non-secured versions.
- Any security or certificates management required by SPDM requester or responder.

Security algorithms support: SHA-2, RSA-SSA/ECDSA, FFDHE/ECDHE, AES_GCM/ChaCha20Poly1305, HMAC
Security library: [Mbed TLS](https://tls.mbed.org/kb/development/hw_acc_guidelines)
Open SSL could be used also, but we need only one crypto library. Mbed TLS seems to be a better choice.

Supported communication layers:
- Stage 1, SPDM over MCTP
[DSP0275 Security Protocol and Data Model (SPDM) over MCTP Binding Specification](https://www.dmtf.org/sites/default/files/standards/documents/DSP0275_1.0.0.pdf)
[DSP0276 Secured Messages using SPDM over MCTP Binding Specification](https://www.dmtf.org/sites/default/files/standards/documents/DSP0276_1.0.0.pdf)
- Optional stage 2, SPDM over the PCI DOE
[PCI Data Object Exchange (DOE)](https://members.pcisig.com/wg/PCI-SIG/document/14143)
[PCI Component Measurement and Authentication (CMA)](https://members.pcisig.com/wg/PCI-SIG/document/14236)
[PCI Integrity and Data Encryption (IDE)](https://members.pcisig.com/wg/PCI-SIG/document/15149)

Secure communication:
- [DSP0277 Secured Messages using SPDM Specification](https://www.dmtf.org/sites/default/files/standards/documents/DSP0277_1.0.0.pdf)


### SPDM deamon - *spdmcppd*
SPDM deamon could work as a single threaded process and could be implemented using a similar
approach as it is done for example for pldmd (but without any references to this implementation).
The main thread functionality is to perform requester tasks.
After realizing requester discovery the deamon will provide information about confirmed certified modules
through dbus (measurement data of each connected endpoint which supports SPDM).
Also, in the future, it will provide means for secure SPDM transmission.

We do not predict more than one SPDM deamon in the system.

SPDM deamon should use libspdmcpp API and Linux system API. It does not call security libs or mctp
APIs directly. Also most of the memory allocation should be done in libspdmcpp and not in the spdmcppd.
This approach should move testing of proper memory usage to unitary tests, making the implementation
of the deamon more robust.
This assumption does not apply for Dbus connection, as it should be provided directly from spdmcppd.
It will not be necessary to add dbus to libspdmcpp. This is because dbus is specific
for deamon and it is not a part of the official SPDM specification.

We assume that spdm deamon will be configured using a dedicated configuration file.
Additionally, it will support run-time parameters.

Note that the below specification may be adjusted during the implementation phase.

#### Parameters for spdmcppd
- **--version** (-v)
Returns spdm deamon version and compilation date.
- **--help** (-h)
Provides params syntax, also for a configuration file
- **--verbose** [0-4]
Sets debug level:
0-critical errors and expected output,
1-all errors and authentication results,
2-warnigns,
3-debug,
4-trace
- **--conf** ""
Points a configuration file.
Default file checked: "./spdmd.config"
If a config file is not found then default configuration params values will be used.
- **--logfile** ""
Points a log file for output debug messages.
By default standard output is used.
- **--service** (-s) [start|stop]
With *start* value runs the SPDM deamon as a service. After that the command will be finished,
SPDM deamon should be started in a spearate thread.
With *stop* value stops the SPDM deamon thread.
- **--authenticate** (-a) [RoT device ID]
Authenticate RoT device ID and display result.
It should be used without --service param and the SPDM deamon should be already started.
- **--measurements** (-m) [RoT device ID]
Get measurements from a RoT device ID and display result.
It should be used without --service param and the SPDM deamon should be already started.
- **--certificates** (-c) [RoT device ID]
Get certificates from a RoT device ID and display result.
It should be used without --service param and the SPDM deamon should be already started.

#### Configuration file for spdmcppd
In a case of providing the same settings through a parameter or a configuration file, the first one will be used.

Configuration file is a standard human readable text file for providing configuration options.
Configuration parameters must use the following syntax:
param_name: value;
Colon and semicolon are required.
Text values should be provided with quote signs.
Lines starting with a hash '#' will be treated as a comment.

- **debug_level**: [0-4];
Debug levels - the same as for verbose parameter.
- **logifle**: [log file path];
- **mutual_authentication**: [true|false];
Enables or disables mutual authentication feature - by default true, if implemented.
- **sessions_support**: [true|false];
Enables support for seesions - by default true, if implemented.
- **cached_authorization**: [true|false]
[Id device 1],
[Id device 2];
Enables initial authorization for the listed RoT devices. The initial authorization
should be performed without an initiating dbus command.
- **cached_authorization_delay**: [seconds];
The initial authorization should be performed after running the deamon with a delay configured by this param.
Default value: 60.
- **cached_authorization_renew**: [seconds];
Renew autorization for already authenticated devices after the provided time period.
Value of 0 seconds makes no renew available.
Default value: 0.

#### Dbus communication layer
Dbus communication layer should provide a dedicated means to get all information required
by Redfish schema, specific for SPDM.

SPDM deamon will use dbus API provided by [sdbusplus repo](https://github.com/openbmc/sdbusplus.git).

Dbus requests params
Id - could be a mashup of the eRoT’s id plus protected entity’s Id (Ex: GPU’s Id)
"Id" : "<ComponentIntegrity’s Id>"
for example: "Chassis/{ChassisId}" or "ComputerSystems/{SystemId}/Processors/{ProcessorId}"

Request type:
1. get measurements data,
2. get SPDM version,
3. get authentication status,
4. get certificates,
5. (optional) get session Id
ComponentCommunication is optional for now as we don’t need to encrypt the SPDM channel between HMC and eRoT. However, SessionId property can be useful once we have a way of letting the Redfish client ask HMC to re-negotiate SPDM version and/or algorithm with the eRoT. It can indicate if the session changed after a re-negotiation.

Below, an exact description of dbus communication commands will be provided.

## Licensing
In the initial phase we assume [Apache v2](https://www.apache.org/licenses/LICENSE-2.0) type of the license for the repository and all sources.

## Testing
Unitary tests: each function available in libspdmcpp API has to have own unitary tests.
Requirements: 80% code coverage
Manual verification:
- use spdm-emu tool for manual verification of requester functionality,
implemented in spdmcppd.
- use dbus mockup requester to check dbus implementation in spdmcppd.