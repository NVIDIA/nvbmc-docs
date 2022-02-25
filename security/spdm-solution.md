# SPDM solution

Author:
Marcin Nowakowski <marcin.nowakowski@conclusive.pl>

Primary assignee:

Other contributors:
Ratan Gupta <guptar@nvidia.com>

Created: 2021-11-30

## Problem Description

In a typical system where there are multiple subsystems, each subsystem can provide the measurement service but the problem is
- How can we securely share it with other subsystems.
- How can we safeguard from man in the middle attack.
- How does the subsystems establish the trust

SPDM’s vision is to resolve the long-lasting problem of compatible secure communication solution between two endpoints of embedded systems

Use-case:
- The authentication initiator can request from the component/subsystems a set of measurements of its firmware.
- These measurements can be compared to a database of validated measurements to assess whether or not the
  correct firmware is installed.
- If a measurement is determined to be incorrect, this indicates that an update is required,
  that malware has been introduced.
- If the authentication initiator sitting outside the BMC, can get this measurements through
  Redfish protocol (out of band management).

## Background and References

*Security Protocol and Data Model (SPDM) is a standard published by the Distributed Management Task Force (DMTF) organization Platform Management Components Intercommunication (PMCI) working group.
The Security Protocol and Data Model (SPDM) Specification defines messages, data objects, and sequences for performing message exchanges between devices over a variety of transport and physical media. The description of message exchanges includes authentication of hardware identities, measurement for firmware identities and session key exchange protocols to enable confidentiality and integrity protected data communication. The SPDM enables efficient access to low-level security capabilities and operations.

SPDM specification is described here:
 Security Protocol and Data Model (SPDM) Specification (DSP0274)](https://www.dmtf.org/sites/default/files/standards/documents/DSP0274_1.1.1.pdf

Existing Reference implementation for SPDM:
 libspdm - a sample implementation that follows the DMTF SPDM specification](https://github.com/DMTF/libspdm)

### Glossary
* **RTT** - This is the worst case round trip transport timing.
* **ST1** - This shall be the maximum amount of time the Responder has to provide a response to
   requests that do not require cryptographic processing, such as GET_CAPABILITIES,
   GET_VERSION or NEGOTIATE_ALGORITHMS
   Value should be 100ms.
* **T1** - This shall be the minimum amount of time the Requester shall wait before issuing a
   retry for requests that do not require cryptographic processing.
   Value should be RTT + ST1.
* **CT** - This is the cryptographic timeout in microseconds. CTExponent is reported in the CAPABILITIES message
   Value should be 2^CTExponent
* **T2** - This shall be the minimum amount of time the Requester shall wait before issuing a retry for requests that require cryptographic processing.
* **Requester** - Original transmitter, or source, of an SPDM request message.
   It is also the ultimate receiver, or destination, of an SPDM response message.
* **Responder** - Ultimate receiver, or destination, of an SPDM request message.
   It is also the original transmitter, or source of an SPDM response message.

## Requirements

1. SPDM solution supports the following versions
	- Version 1.0 : Device Authentication and Measurement.
	- Version 1.1 : Device session key establishment and introduces secure communication.

2. BMC will act as an SPDM Requester, hence need to follow the requirements captured from the SPDM spec(DSP0274).

	- Concurrent SPDM message processing (SPDM Requestor)
    	- A Requester shall not have multiple outstanding requests to the same Responder, with the exception of GET_VERSION addressed in GET_VERSION request message and VERSION response message.
    	- If the Requester has sent a request to a Responder and wants to send a subsequent request to the same Responder, then the Requester shall wait to send the subsequent request until after the Requester completes one of the following actions:
     		• Receives the response from the Responder for the outstanding request.
     		• Times out waiting for a response.
     		• Receives an indication, from the transport layer, that transmission of the request message failed.
    	- A Requester may send simultaneous request messages to different Responders

	- Timing requirements (SPDM Requester)
    	- If the Requester does not receive a response within T1 or T2 time accordingly, the Requester may retry a request message.
    	- Retry of a request message shall be a complete retransmission of the original SPDM request message.

3. Fresh measurement will be taken on first boot or every boot of the BMC. This feature can be turned off in the configuration.

4. The BMC will be SPDM v1.1 compliant and the following messages need to be implemented.
   - GET_CAPABILITIES / CAPABILITIES,
   - NEGOTIATE_ALGORITHMS / ALGORITHMS,
   - GET_DIGEST / DIGEST,
   - GET_CERTIFICATE / CERTIFICATE,
   - CHALLENGE / CHALLENGE_AUTH,
   - GET_MEASUREMENTS / MEASUREMENTS

5. Supported requester capabilities:
   - CERT_CAP
   - CHAL_CAP
   - MEAS_CAP
   - MEAS_FRESH_CAP

6. The BMC will not perform attestation by itself but will collect measurements on behalf of the Redfish client.

## Proposed Design

### High Level Design Diagram

#### SPDM Endpoint Discovery

    +--------------+         +-----------------+       +-----------------+     +------------------+      +-----------------+
    |SPDM D-Bus Obj|         | SPDM Requester  |       |   MCTP Daemon   |     |   MCTP Endpoint  |      |  SPDM Responder  |
    |              |         |  SPDM Daemon    |       |     (HMC)       |     |                  |      |                  |
    +------+-------+         +--------+--------+       +--------+--------+     +---------+--------+      +---------+--------+
           |                          |                         |                        |                         |
           |                          |                         |                        |                         |
           |                          |                         |                        |                         |
           |                            Discover MCTP endpoints |                        |                         |
           |                          +------------------------->                        |                         |
           |                          |  MCTP MessageType =5,   |                        |                         |
           |                          |RootPath=/xyz/openbmc_   |                        |                         |
           |                          |project/mctp             |                        |                         |
           |                          |Interface=xyz.openbmc_   |                        |                         |
           |                          |project.MCTP.Endpoint    |     MCTP Message       |                         |
           |                          |                         +----------------------->+                         |
           |                          |                         |                        |                         |
           |                          |                         |     MCTP Message       |                         |
           |                          |                         +<-----------------------+                         |
           |                          |                         |                        |                         |
           |                          |     MCTP Endpoints      |                        |                         |
           |                          <-------------------------+                        |                         |
           |                          |                         |                        |                         |
           |                   For Each Endpoint                |                        |                         |
           |  Create D-Bus Object     |                         |                        |                         |
           +<-------------------------+                         |                        |                         |
           |                          |                         |                        |                         |
           |                          |                         |  SPDM Initial Handshake|                         |
           |                          +---------------------------------------------------------------------------->
           |                          <----------------------------------------------------------------------------+
           |                          |                         |                        |
           |                          |                         | SPDM GET Measurements  |                         |
           |                          +---------------------------------------------------------------------------->
           |                          <----------------------------------------------------------------------------+
           |                          |                         |    Measurements        |                         |
           |    Update the D-Bus obj  |                         |                        |                         |
           <--------------------------+                         |                        |                         |
           |                          |                         |                        |                         |
           |                          |                         |                        |                         |
           |                          |                         |                        |                         |
           |                          |                         |                        |                         |
           +                          +                         +                        +                         +


- The SPDM daemon uses MCTP discovery mechanism to check available MCTP devices, supporting MCTP message type = 5.
- The MCTP discovery mechanism is provided by ctrld, implemented in mctp library.
- SPDM daemon creates the D-bus object for all the connected SPDM endpoints.
- SPDM daemon initiate the handshake and get the measurement data from the connected devices and updates the D-Bus objects.
- Once all MCTP capable devices are discovered, SPDM daemon will do the handshake with all of them.


#### Redfish Client Invoking the signMeasurementFunction

                                                     +---------------------+
                                                     | SPDM Daemon         |
    +--------------+        +---------------+        | +-----------------+ |   +------------------+  +-----------------+
    |Redfish Client|        |    bmcweb     |        | |    D-Bus Obj    | |   |        MCTP      |  | SPDM Responder  |
    |              |        |Redfish Server |        | |                 | |   |                  |  |                 |
    +------+-------+        +--------+------+        | +-----------------+ |   +---------+--------+  +--------+--------+
           |                         |               +---------------------+             |                    |
           |Get Signed Measurement   |                          |                        |                    |
           +------------------------->                          |                        |                    |
           |                         | Get Sign Measurement     |                        |                    |
           |                         +-------------------------->                        |                    |
           |                         |(Dbus Method)             |              SPDM GetVersion                |
           |                         |                          +--------------------------------------------->
           |                         |                          |                  Version                    |
           |                         |                          <---------------------------------------------+
           |                         |                          |              Get Capabilities               |
           |                         |                          +--------------------------------------------->
           |                         |                          |                Capabilities                 |
           |                         |                          <---------------------------------------------+
           |                         |                          |              Negotiate Algorithms           |
           |                         |                          +---------------------------------------------+
           |                         |                          |                  Algorithms                 |
           |                         |                          <---------------------------------------------+
           |                         |                          |              GetDigestS                     |
           |                         |                          +---------------------------------------------+
           |                         |                          |               Digests                       |
           |                         |                          +------------------------+--------------------+
           |                         |                          |              Get Certificate                |
           |                         |                          +--------------------------------------------->
           |                         |                          |                Certificate                  |
           |                         |                          <---------------------------------------------+
           |                         |                          |              Get Measurements               |
           |                         |                          +--------------------------------------------->
           |                         |                          |                Measurements                 |
           |                         |                          <------------------------+--------------------+
           |                         |                      Update the D-bus Object      |                    |
           |                         |  Get Sign Measurement    |                        |                    |
           |                         ^--------------------------+                        |                    |
           |                      If success                    |                        |                    |
           |                         +     Get D-Bus Object     |                        |                    |
           |                         +-------------------------->                        |                    |
           |                         <--------------------------+                        |                    |
           |                      Prepare the bmcweb response   |                        |                    |
           |  Return the response    |                          |                        |                    |
           +<------------------------+                          |                        |                    |
           +                         +                          +                        +                    +

- In the above flow diagram, Redfish client asks the BMC to get the fresh measurement of SPDM responder.
- SPDM daemon will start the SPDM message exchange.
- Get the measurements and update the corresponding D-Bus object.

### D-Bus parameters expected from the D-Bus Object:

- Array of Measurements
- Certificate
- SPDM Version
- HashingAlgorithm
- SigningAlgorithm
- Status of connection between SPDM requester and a responder
- Associations interface to MCTP endpoint, capable of SPDM responder functionality 

The above D-Bus object will not be persistent (i.e) fresh measurement could be taken after BMC reboot or service restart.

Additionally, the D-Bus object will have a method to get fresh measurements, with the following parameters:
- SlotId
- Nonce
- Measurements index

#### How to get the inventory path of the EROT

- Pldm creates the inventory of the EROT
- During creation of the inventory object pldm implements the UUID interface
   as well as the Item.ComponentIntegrity interface(Yet to be defined)
- D-Bus client will query for inventory objects that implement Item.ComponentIntegrity
  interface by the pldm service.
- For each object which implements the ComponentIntegrity interface, gets the MCTP UUID
  from that object and match with the MCTP D-bus object UUID which is associated with the
  SPDM measurement D-Bus object.

#### How to get the inventory path of the components protected by the EROT

- Application which creates the EROT inventory object, will implement the component-
  integrity interface and its associations.
- D-Bus client will query for all the inventory objects implementing the Item.ComponentIntegrity interface.
- Get the associated objects from the objects implementing the Item.ComponentIntegrity interface.

### Structure of the SPDM Module

SPDM module will consists of three parts:
- SPDM library - prepared as a separate library to support SPDM daemon, unitary tests and any external responder or requester implementation
- SPDM daemon - main daemon to manage SPDM queries
- SPDM unitary tests - unitary tests for SPDM library implementation

The solution could be prepared as a single SPDM daemon with D-Bus interface and without a dedicated library. However, layering the design into separate daemon and library may be useful to reuse the source code for other responders. Also, it should make unitary tests more robust.

### Dependencies
- SPDM library requires MCTP control module
- SPDM library requires also security libraries
- SPDM daemon requires accessing D-Bus

### Implementation assumptions
- Build system: meson
- Language for all source code files: `C++` (`C++` is chosen to distinguish implementation from open spdm reference project and also to speed up development, especially using standard classes)
- SPDM solution is meant to work in Linux environment, and it will not support Windows environment.
- We will not use shared (or global) variables. The whole implementation should be "thread safe" and use std::mutex and std::lock_guard if required.
- The daemon should work in Linux user space - we do not predict kernel modules in the implementation.

### SPDM library - *libspdmcpp*
The library should provide the following external API
- Serialization and deserialization of the required SPDM messages.
- Sending and receiving messages over MCTP with SPDM payload in secured and non-secured versions.
- Any security or certificates management required by SPDM requester or responder.

Security algorithms support: SHA-2, RSA-SSA/ECDSA, FFDHE/ECDHE, AES_GCM/ChaCha20Poly1305, HMAC
The library internally uses as security library: [Mbed TLS](https://tls.mbed.org/kb/development/hw_acc_guidelines)
Open SSL could be used also, but we need only one crypto library. Mbed TLS seems to be a better choice.

##### Supported communication layers:

- Stage 1, SPDM over MCTP
[DSP0275 Security Protocol and Data Model (SPDM) over MCTP Binding Specification](https://www.dmtf.org/sites/default/files/standards/documents/DSP0275_1.0.0.pdf)
[DSP0276 Secured Messages using SPDM over MCTP Binding Specification](https://www.dmtf.org/sites/default/files/standards/documents/DSP0276_1.0.0.pdf)
- Optional stage 2, SPDM over the PCI DOE
[PCI Data Object Exchange (DOE)](https://members.pcisig.com/wg/PCI-SIG/document/14143)
[PCI Component Measurement and Authentication (CMA)](https://members.pcisig.com/wg/PCI-SIG/document/14236)
[PCI Integrity and Data Encryption (IDE)](https://members.pcisig.com/wg/PCI-SIG/document/15149)

Secure communication:
- [DSP0277 Secured Messages using SPDM Specification](https://www.dmtf.org/sites/default/files/standards/documents/DSP0277_1.0.0.pdf)

### SPDM daemon - *spdmd*
SPDM daemon could work as a single threaded process and could be implemented using a similar approach as it is done for example for pldmd (but without any references to this implementation).

The main thread functionality is to perform requester tasks.
After realizing requester discovery the daemon will provide information about confirmed certified modules through D-Bus (measurement data of each connected endpoint which supports SPDM).
Also, in the future, it will provide means for secure SPDM transmission.
We do not predict more than one SPDM daemon in the system.

SPDM daemon should use libspdmcpp API and Linux system API. It does not call security libs or mctp APIs directly. Also most of the memory allocation should be done in libspdmcpp and not in the spdmd.
This approach should move testing of proper memory usage to unitary tests, making the implementation of the daemon more robust.

This assumption does not apply for D-Bus connection, as it should be provided directly from spdmd.
It will not be necessary to add D-Bus to libspdmcpp. This is because D-Bus is specific
for daemon and it is not a part of the official SPDM specification.

#### Configuration

We assume that spdm daemon will be configured using a dedicated configuration file.
Additionally, it will support run-time parameters. This approach should provide better means for development, testing and verification.
However, it should be also possible to set default spdmd values using yocto meta-layers, so configuration file usage could be omitted.

Note that the below specification may be adjusted during the implementation phase.

#### Parameters for spdmd
- **--version** (-v)
Returns spdm daemon version and compilation date.
- **--help** (-h)
Provides params syntax, also for a configuration file
- **--verbose** [0-4]
Sets debug level:
0-critical errors and expected output,
1-all errors and authentication results,
2-warnings,
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
With *start* value runs the SPDM daemon as a service. After that the command will be finished,
SPDM daemon should be started in a separate thread.
With *stop* value stops the SPDM daemon thread.
- **--authenticate** (-a) [RoT device ID]
Authenticate RoT device ID and display result.
It should be used without --service param and the SPDM daemon should be already started.
- **--measurements** (-m) [RoT device ID]
Get measurements from a RoT device ID and display result.
It should be used without --service param and the SPDM daemon should be already started.
- **--certificates** (-c) [RoT device ID]
Get certificates from a RoT device ID and display result.
It should be used without --service param and the SPDM daemon should be already started.

#### Configuration file for spdmd
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
Enables support for sessions - by default true, if implemented.
- **cached_measurements**: [true|false]
[Id device 1],
[Id device 2];
Enables initial getting of measurements for the listed RoT devices. The initial communication should be performed without an initiating D-Bus command.
- **cached_measurements_delay**: [seconds];
The initial communication should be performed after running the daemon with a delay configured by this param.
Default value: 60.
- **cached_measurements_renew**: [seconds];
Renew getting measurements for already authenticated devices after the provided time period.
Value of 0 seconds makes no renew available.
Default value: 0.

#### SPDM over MCTP connection
After spdmd discovers SPDM responders, getting measurements could be initiated in two ways:
1. Redfish client requests for authentication, certificates or measurements from an SPDM responder capable device.
2. The SPDM daemon initiates authentication after initial delay, getting certificates or measurements basing on provided parameters or the configuration file. After that the daemon stores cached values, available for immediate answering on Redfish client queries.

All cached values like authorization status, certificates and measurements are kept in the SPDM daemon and are exposed in D-Bus objects.

#### D-Bus communication layer
D-Bus communication layer should provide dedicated means to get all information required by Redfish schema, specific for SPDM.

SPDM daemon will use D-Bus API provided by [sdbusplus repo](https://github.com/openbmc/sdbusplus.git).

D-Bus requests params
Id - could be a mash-up of the eRoT’s id plus protected entity’s Id (Ex: GPU’s Id)
"Id" : "<ComponentIntegrity’s Id>"
for example: "Chassis/{ChassisId}" or "ComputerSystems/{SystemId}/Processors/{ProcessorId}"

Request type:
1. get measurements data,
2. get SPDM version,
3. (?optional) get authentication status,
4. get certificates,
5. (optional) get session Id
ComponentCommunication is optional for now as we don’t need to encrypt the SPDM channel between HMC and eRoT. However, SessionId property can be useful once we have a way of letting the Redfish client ask HMC to re-negotiate SPDM version and/or algorithm with the eRoT. It can indicate if the session changed after a re-negotiation.

D-Bus objects should not be persistent.
The maximum number of supported MCTP/SPDM endpoints will be 255.

Once the initial implementation is prepared, an exact description of D-Bus communication commands will be provided in this document.

##### Error handling
Any error occurred during communication or due to timeout will cause failure in authorization verification.

##### Getting authorization, certificates or measurement delay handling
Using caching approach for getting authorizations, certificates or measurement should significantly decrease required timing to answer Redfish client queries.

## Licensing
In the initial phase we assume [Apache v2](https://www.apache.org/licenses/LICENSE-2.0) type of the license for the repository and all sources.

## Testing
Unitary tests: each function available in libspdmcpp API has to have own unitary tests.
Requirements: 80% code coverage
Manual verification:
- use spdm-emu tool for manual verification of requester functionality, implemented in spdmd.
- use D-Bus mockup requester to check D-Bus implementation in spdmd.
