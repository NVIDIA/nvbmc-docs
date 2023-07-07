# NCSI over MCTP Feature

Author: Qian Sun qians@nvidia.com Ben Peled bpeled@nvidia.com

Please refer to the [mctp.md](MCTP Overview) document for general MCTP design
description, background and requirements.

This document describes a user space implementation of NCSI over MCTP
infrastructure, providing a utiliy for NCSI over MCTP communication within an
OpenBMC-based platform.

## Background and References

- [Network Controller Sideband Interface (NC-SI)
  Specification](https://www.dmtf.org/sites/default/files/standards/documents/DSP0222_1.2.0WIP80.pdf)
- [Management Component Transport Protocol (MCTP) Base
  Specification](https://www.dmtf.org/sites/default/files/standards/documents/DSP0236_1.3.1.pdf)
- [NC-SI over MCTP Binding
  Specification](https://www.dmtf.org/sites/default/files/standards/documents/DSP0261_1.0.0.pdf)

## Prerequisites

- MCTP protocol transmission layer supported.
- NIC Devices which support NCSI over MCTP feature. (Tested NIC Devices:
  Bluefield 3)

## Requirements

- Support the creation of NC-SI command frame format.
- Support the creation of NC-SI command response frame format.
- Support sending and receiving NC-SI commands over MCTP layer.
- Support the configuration modification of the NIC through NC-SI OEM commands, eg:
  - Control the DPU mode of operation (DPU, Restricted, NIC, enhanced NIC).
  - Control the Rshim state enabled/disabled.
  - DPU BMC get the strap values of the NIC.

## Proposed Design

The DPU BMC is equipped with a dedicated SMBus connection to the NIC, which
serves as a hardware interface facilitating communication between the BMC and
the NIC via MCTPoSMBus. Within the system, it is necessary to provide support
for the NCSI commands, which allow the DPU BMC to control the configuration of
the NIC. The application of userspcae can get MCTP device information form
MCTP-Ctrl D-bus Interface and provide support of the NCSI over MCTP feature to
control and get the information from NIC device.

***The architecture of the system components and the data flow***

```
┌───────────────────────────────────┐
│             DPU BMC               │
│                                   │
│   ┌───────────────────────────┐   │
│   │   NCSI over MCTP utility  │   │
│   │                           │   │
│   └──────────┬───▲────────────┘   │
│              │   │                │
│   ┌──────────▼───┴────────────┐   │
│   │         MCTP-ctrl         │   │
│   │                           │   │
│   └──────────┬───▲────────────┘   │
│              │   │                │
│   ┌──────────▼───┴────────────┐   │
│   │          SMBus            │   │
│   │                           │   │
│   └───────────────────────────┘   │
│                                   │
│                                   │
└──────────────┬────▲───────────────┘
               │    │
             ┌─▼────┴─┐
             │  NIC   │
             └────────┘
```

There are few new components be implemented in utility as follow list.

- NCSI-over-MCTP binding implementations, which interact with the hardware
  channel(s).
  1. Provide the function which send packaged the NCSI format command to MCTPd
     and receive the NCSI format command from MCTPd via socket.
  2. Package NCSI format command into MCTP format command with specific MCTP EID
     and type.

- User space implementations, which provide NCSI related parameters to user.
  1. This new application named `ncsi-mctp` will be created to configure NCSI
     command and will be located in the `phosphor-networkd` repository.
  2. This may be invoked from a systemd service to configure NIC behavior.
  3. Package the NCSI command with NCSI control packet header and payload, which
     from user inputs.
  4. Unpack the NCSI command into json format, payload is printed as raw uint32
     size hex strings.

***NCSI Command Format***

```
-----------------------------------------------
|NCSI Command | Bits                          |
|---------------------------------------------|
|Bytes        |31..24 |23..16 |15..08 |07..00 |
|---------------------------------------------|
|00..15       |  NCSI Control packet Header   |
|---------------------------------------------|
|16..N        |     NCSI Command Packet       |
-----------------------------------------------
```

## General usage

The ncsi-mctp will provides the use cases as following. It can send NC-SI
standard commands, or vendor specific OEM commands (eg: Mellanox Specific NC-SI
OEM Commands).

### User Interface
#### Request

```
ncsi-mctp --eid=<eid> --package=<package> --channel=<channel> --cmd=<cmd> --payload=<hex data> --verbose
```

Eg:
  - Send Clear Package Initial State Command:
    ```
    ncsi-mctp --eid=100 --package=3 --channel=0 --cmd=0
    ```
  - Send Select Package Command:
    ```
    ncsi-mctp --eid=100 --package=3 --channel=31 --cmd=0 --payload=00000000
    ```
  - Send Get PF Mac Address Command, which is a Mellanox OEM Command:
    ```
    ncsi-mctp --eid=100 --package=3 --channel=0 --cmd=80 --payload=0000811900000000 --verbose
    ```
  - Send Get Smart NIC Mode Command, which is a Mellanox OEM Command:
    ```
    ncsi-mctp --eid=100 --package=3 --channel=31 --cmd=80 --payload=0000811900133300 --verbose
    ```
  - Send Get Host Access Command, which is a Mellanox OEM Command:
    ```
    ncsi-mctp --eid=100 --package=3 --channel=31 --cmd=80 --payload=0000811900131900 --verbose
    ```

#### Response

The response of this utility will be in json format. The properties are
mentioned in the following structure.
```
{
    "Error": {
        "Code": <Requester error code>,
        "Message": <Explanation for this error code>,
        "Data": <Additional data to analyse>,
    },
    "Code": <NCSI error code>,
    "Reason": <NCSI error reason>,
    "Manufacture_ID": <Manufacture ID in OEM command>,
    "OEM_Payload": [
        <Response Payload in OEM command>
    ]
}
```

Eg:
  - Response without error:
    ```
    {
        "Code": "0x0000",
        "Reason": "0x0000"
        "Manufacture_ID": "MLX",
        "OEM_Payload": [
          "0x00000000",
          "0x4fae6d94",
          "0x0000481a"
        ]
    }
    ```
  - Response with error:
    ```
    {
        "Error": {
            "Code": -3,
            "Message": "Failed to send and receive ncsi messages",
            "Data": -11
        },
        "Code": "0x0001",
        "Reason": "0x0001",
        "Manufacture_ID": "MLX",
        "OEM_Payload": [
            "0x00000000"
        ]
    }
    ```

### Options

| Name                   | Argument        | Description              |
| ---------------------- | ----------------| -------------------------|
| **eid**                | Dec             | EID for NCSI Device      |
| **package**            | Dec             | Package ID               |
| **channel**            | Dec             | Internal Channel ID      |
| **cmd**                | Dec             | Command ID               |
| **Payload**            | Hex Data        | Payload of NCSI command  |
| **help**               | No-argument     | Print the menu           |
| **verbose**            | No-argument     | Vobosity flag            |

### Requester Error Code

| Name                               | Description                                                                | Value |
| ---------------------------------- | -------------------------------------------------------------------------- | ----- |
|NCSI_REQUESTER_SUCCESS              |Returned for a successful command completion                                |0      |
|NCSI_REQUESTER_OPEN_FAIL            |Returned to report that utiliy failed to open the socket of NCSI over MCTP  |-1     |
|NCSI_REQUESTER_NOT_NCSI_MSG         |Returned to report that the response is NCSI message                        |-2     |
|NCSI_REQUESTER_RESP_MSG_ERROR       |Returned to report that the command has NCSI standard error code            |-3     |
|NCSI_REQUESTER_RECV_TIMEOUT         |No response returned. Default receiving timeout is 5 seconds                |-4     |
|NCSI_REQUESTER_RESP_MSG_TOO_SMALL   |Returned to report that received message is less than NCSI response header  |-5     |
|NCSI_REQUESTER_INSTANCE_ID_MISMATCH |Returned to report that the received instance id is mismatched              |-6     |
|NCSI_REQUESTER_SEND_FAIL            |Returned to report that error happened when sending NCSI message            |-7     |
|NCSI_REQUESTER_RECV_FAIL            |Returned to report that error happened when receiving NCSI message          |-8     |
|NCSI_REQUESTER_INVALID_RECV_LEN     |Returned to report that the size of received message is not vaiid           |-9     |

## Testing

For NCSI Over MCTP implementation, testing models would depend on the structure
we adopt in the design section.
