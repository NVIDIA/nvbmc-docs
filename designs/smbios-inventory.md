# Example design - this is the design title

Author: Jinder Hung

Other contributors: 

Created: June 15, 2022

## Problem Description

There are some inventories in the system which BMC cannot directly read them, but they exist in smbios table. BMC can show them on web service and redfish if BIOS send the smbios table to BMC, and BMC parse it.

## Background and References

The [smbios-mdr][1] has [IPMI blob][2] handler library to allow user or BIOS to send the [SMBIOS][3] table to BMC, then the smbios-mdr generates inventories from the SMBIOS table. Besides IPMI blob commands, BMC should also provide the [MDRv1 commands][5] to provide an alternative to transferring the SMBIOS table.

## Requirements

BMC supports [MDRv1 commands][5] to allow BIOS to send the SMBIOS table to BMC.
BIOS follows [MDRv1 commands][5] to send the SMBIOS] table to BMC.
BMC parses the SMBIOS table and exposes useful information to D-BUS.

## Proposed Design

Here's one of existed flow in [smbios-mdr][1].
1. BIOS uses MDRv1 commands to lock, write, and complete SMBIOS data region.
2. While the client complete the SMBIOS data region, BMC triggers the Smbios.MDR_V2 service to generate inventories dbus objects by calling dbus method, [AgentSynchronizeData][MDR_V2].
3. Redfish service uses [object-mapper][4] to lookup inventories from specific dbus interface.

[SMBIOS to DBus/Redfish map](https://docs.google.com/spreadsheets/d/1vA98tHAIm-8kv2hQexs3TIgOAHnW1-REwG2IY36OQ3E/edit?usp=sharing)

## Alternatives Considered

BIOS might use IPMI blob commands to send the SMBIOS table to BMC.

## Impacts

The [MDRv1 docuemnt][6] and [MDRv1 code][5] has different command codes, we follow the (TBD). Users might refer to different codes.

### Organizational

None

## Testing

1. Dump a smbios binary data from a system, then transfer it into BMC.
2. View the results from redfish.

[1]:https://github.com/openbmc/smbios-mdr
[2]:https://github.com/openbmc/phosphor-ipmi-blobs
[3]:https://www.dmtf.org/sites/default/files/standards/documents/DSP0134_3.4.0.pdf
[4]:https://github.com/openbmc/docs/blob/master/architecture/object-mapper.md
[5]:https://github.com/openbmc/intel-ipmi-oem/blob/master/include/oemcommands.hpp#L100-L105
[6]:https://www.intel.com/content/dam/www/public/us/en/documents/guides/bios-bmc-technical-guide-v2-1.pdf
[MDR_V2]:https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Smbios/MDR_V2.interface.yaml

