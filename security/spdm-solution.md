# SPDM solution

Author:
Marcin Nowakowski <marcin.nowakowski@conclusive.pl>

Primary assignee:

Other contributors:

Created: 2021-11-30

## Problem Description

Open BMC does not provide SPDM implementation.

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
The implementation cannot use open SPDM github project, but may use similar approach.

## Proposed Design

### Overview
SPDM module will consists of three parts:
- SPDM library - prepared as a separate library to support SPDM deamon,
unitary tests and any external responder or requester
- SPDM deamon - main deamon to manage SPDM queries
- SPDM unitary tests - unitary tests for SPDM library implementation

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

### SPDM deamon - *spdmcppd*
SPDM deamon could work as a single threaded process and could be implemented using a similar
approach as it is done for pldmd. The main thread functionality is to perform requester tasks.
After realizing requester discovery the deamon will provide information about confirmed certified modules through dbus.

SPDM deamon should use libspdmcpp API and Linux system API. It does not call security libs or mtcp
APIs directly. Also any memory allocation should be done in libspdmcpp and not in the spdmcppd.
This does not apply for the below exceptions:
- Dbus connection should be provided directly from spdmcppd.
It will not be necessary to add dbus to libspdmcpp.

We assume that spdm deamon will be configured using a dedicated configuration file.
Additionally, it will support few run-time parameters.

#### Parameters for spdmcppd
TBD

#### Configuration file for spdmcppd
TBD

## Licensing
In initial phase we assume [Apache v2](https://www.apache.org/licenses/LICENSE-2.0) type of the license for the repository and all sources.

## Testing
Unitary tests: each function available in libspdmcpp API has to have own unitary tests.
Requirements: 80% code coverage
Manual verification: use spdm-emu tool for manual verification of responders and requesters.