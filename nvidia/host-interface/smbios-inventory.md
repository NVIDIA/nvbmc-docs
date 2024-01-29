# SMBIOS Inventory 
Author: Rohit PAI 
Created: Jan 29th, 2024

## Features 
- BMC supports [phosphor-ipmi-blobs][7]  to allow BIOS to send the SMBIOS table to BMC.
- BIOS follows [phosphor-ipmi-blobs][7]  to send the SMBIOS table to BMC.
- BMC parses the SMBIOS table and exposes useful information to D-BUS. SMBIOS inventory information is available in Redfish. 

## Standard Specs 
1. [System Management BIOS (SMBIOS) Reference Specification](https://www.dmtf.org/sites/default/files/standards/documents/DSP0134_3.7.0.pdf)

## High Level Architecture 
There are host side inventories in the system which the BMC cannot directly read but they exist in the form of smbios table. BMC can show them on web service and redfish if BIOS can send the smbios table to the BMC, and BMC has the capability to parse it.

The [smbios-mdr][1] has [IPMI blob][2] handler library to allow user or BIOS to send the [SMBIOS][3] table to BMC, then the smbios-mdr generates inventories from the SMBIOS table. Besides IPMI blob commands, BMC should also provide the [MDRv1 commands][5] to provide an alternative to transferring the SMBIOS table.

## Low level design 

### SMBIOS MDRv2 App
The main application in this repo is smbiosmdrv2app, capable of parsing a binary SMBIOS table and publishing the system information on D-Bus, to be consumed by other OpenBMC applications.

The SMBIOS table is usually sent to the BMC by the host firmware (BIOS). The system designer can theoretically choose any transport and mechanism for sending the SMBIOS data, but currently on Nvidia platforms only ipmi blob transfer way is be implemented and tested. 


### phosphor-ipmi-blobs
[phosphor-ipmi-blobs](https://github.com/openbmc/phosphor-ipmi-blobs) is an alternative implementation of a generic IPMI blob transfer API. Compared to MDRv2, it is simpler and easier to use, but also transfers the data in-band with the IPMI commands and therefore slower than using a shared memory region (which may or may not be a concern).

phosphor-ipmi-blobs provides a blob manager shared library for ipmid which implements the IPMI commands. In turn, it loads blob handler libraries that each implement support for specific blobs. Here in smbios-mdr we provide such a blob handler for the /smbios blob. It works by writing the data into /var/lib/smbios/smbios2 (the local persistent cache for the SMBIOS table) and calling the AgentSynchronizeData D-Bus method to trigger smbios-mdr to reload and parse the table from that file.

### SMBIOS Transfer 
BMC implemented a set of ipmi-blob handlers for SMBIOS
- The Network Function is 0x2e (OEM/Group). Command code is 128 (0x80)
- The IANA Enterprise Number is 49871 (0x00c2cf, OpenBMC)
- Subcommands are defined in phosphor-ipmi-blobs. The blob ID is  “/smbios”
- The SMBIOS is stored at /var/lib/smbios/smbios2 with 10 bytes header
- This command can be used in all interfaces, including RMCP+ and SSIF.
- At end of ipmi-blob subcommand, BmcBlobCommit, it will call a DBus method, [AgentSynchronizeData](https://github.com/openbmc/phosphor-dbus-interfaces/blob/d1484a1499bc241316853934e6e8b735166deee2/yaml/xyz/openbmc_project/Smbios/MDR_V2.interface.yaml#L127) to notify [smbios-mdr](https://github.com/openbmc/smbios-mdr/).
- smbios-mdr reject SMBIOS data if the version is not in `supportedSMBIOSVersions`.
- smbios-mdr service decodes the SMBIOS and populate DBus inventory objects.

### CPU DBus objects
- Create CPU objects for every SMBIOS Type 4 data
- The object name is `CPU_{instance_id}`
- CPU must have DBus interface "xyz.openbmc_project.Inventory.Item.Cpu" to be identify as CPU
- CPU must have DBus interface "xyz.openbmc_project.Inventory.Item.Chassis" to be identify as Redfish Chassis
- DBus Path `<Module path>/CPU_{instance_id}`
- PLDM creates the `contains` and `contained` relationship with Module by DBus Path
- Nvidia platform has `Socket Designation` value `<Silk Screen Label>:<LogicalSocket>.<LogicalChip> `
- LogicalSocket: Module index. LogicalChip: CPU index in the Module
- Module is DBus object populated by entity-manager. Module has DBus interface "xyz.openbmc_project.Inventory.Item.ProcessorModule"
- Module has Dbus interface "xyz.openbmc_project.Inventory.Decorator.Instance" to indicate the index number of this module in system smbios-mdr can use this `Socket Designation` to find the DBus Module contains this CPU. The DBus path of CPU will be `<Module path>/CPU_{instance_id}`


### Memory DBus objects
- Create Memory objects for every SMBIOS Type 17 data
- The object name is ProcessorModule_{instance_id}_Memory_{instance_id}
- Memory must have DBus interface "xyz.openbmc_project.Inventory.Item.Dimm" to be identify as Memory
- Finding the ProcessorModule instance id: Looking for SMBIOS type 2 (Baseboard or Module) which Board Type is Processor Module, Processor/Memory Module, or Processor/IO Module. If the handle of Memory is in the Contained Object Handles, the type 2 module contains this memory. - Use the index of the module as instance_id

### PCIe Slot DBus objects
- Create PCIe Slot objects for every SMBIOS Type 9 data
- PCIe Slot must have DBus interface "xyz.openbmc_project.Inventory.Item.PCIeSlot" to be identify as PCIeSlot
- DBus Path `<system path>/pcieslot{instance_id}`
- smbios-mdr search the system path with Dbus interface "xyz.openbmc_project.Inventory.Item.System". Create the PCIe Slot object with path `<system path>/pcieslot{instance_id}`

### TPM DBus object
- Create TPM object if there is SMBIOS Type 43 data
- TPM must have DBus interface "xyz.openbmc_project.Inventory.Item.Tpm" to be identify as TPM
- DBus Path `<system path>/tpm`

### Firmware DBus objects
- Create Firmware objects for SMBIOS Type 45 data if it does not exist.
- Firmware must have DBus interface "xyz.openbmc_project.Software.Version" to be identify as Firmware. DBus Path `/xyz/openbmc_project/software/<Firmware Name>`
- Firmware ID is in type 45 data, system could have multiple device has the same Firmware ID, such as multiple the same NICs in the system - If type 45 data associated components contains Processor(Type 4), SystemSlot(Type 9), or Onboard Devices Extended(Type 41), the Firmware Name is <Firmware ID>_<component designation>
- If type 45 data associated components contains System Power Supply(Type 39), the Firmware Name is <Firmware ID>_<component location>
Else the Firmware Name is <Firmware ID>


## RF APIs 
CPU Information 
```
/redfish/v1/Chassis 
/redfish/v1/Systems/{ComputerSystemId}/Processors/
```
PCIe Slots 
```
/redfish/v1/Chassis/{ChassisId}/PCIeSlots 
```
TPM
```
/redfish/v1/Chassis/{ChassisId}/TrustedComponents/{TrustedComponentId}
``` 
Memory Objects 
```
/redfish/v1/Systems/{ComputerSystemId}/Memory/{MemoryId}
```
Firmware Inventory 
```
/redfish/v1/UpdateService/FirmwareInventory/{FirmwareInventoryId}
```

## Platform Enablement 

### Yocto Recipes 
Platforms need to bundle smbios-mdr, phosphor-ipmi-blobs in their image for SMBIOS related features. 
```
Example 
OBMC_IMAGE_EXTRA_INSTALL:append = " phosphor-ipmi-blobs \
                                    smbios-mdr \
```

### Compilation Macros Usage 
1. smbios-mdr 

| Macro | Description | Default | Grace Platform Reference Value |
| ----------- | ----------- |------------|-------------|
| EXPOSE_INVENTORY  | Process Type 45 and report fw version | On | Off : Inventory is populated by other services like PLDM |
| DIMM_DBUS | Expose DIMM D-Bus Interface | On | On |
| DIMM_ONLY_LOCATOR | Only use the DIMM number, and not the bank | Off | Off |
| CPU_INFO | Add Cpuinfo Service specific to intel platforms | On | Off |
| NVIDIA | Enable NVIDIA features including parsing socket info for CPU and Dbus path customization for CPU objects | Off | On |
| IPMI_BLOB | Add IPMI Blobs | On | On |
| FIRMWARE_COMPONENT_NAME_BMC | BMC firmware component name in Type 45, it is used to avoid duplicated version object | Empty String | BMC Firmware |
| FIRMWARE_COMPONENT_NAME_BIOS | BIOS firmware component name in Type 45, it is used to avoid duplicated version object | Empty String | System ROM |
| FIRMWARE_COMPONENT_NAME_TPM | TPM firmware component name in Type 45, it is used to avoid duplicated version object | Empty String | TPM Firmware |
| PLATFORM_PREFIX | Platform prefix, it prepend to DBus object name, such as ${PLATFORM_PREFIX}_CPU_0 | Empty String | Empty String |

2. phosphor-ipmi-blobs

The repo is used as it is from OpenBmc upstream and there are no Nvidia specific customizations.  

## References 
1. [smbios-mdr](https://github.com/openbmc/smbios-mdr)
2. [phosphor-ipmi-blobs](https://github.com/openbmc/phosphor-ipmi-blobs)

[1]:https://github.com/openbmc/smbios-mdr
[2]:https://github.com/openbmc/phosphor-ipmi-blobs
[3]:https://www.dmtf.org/sites/default/files/standards/documents/DSP0134_3.4.0.pdf
[4]:https://github.com/openbmc/docs/blob/master/architecture/object-mapper.md
[5]:https://github.com/openbmc/intel-ipmi-oem/blob/master/include/oemcommands.hpp#L100-L105
[6]:https://www.intel.com/content/dam/www/public/us/en/documents/guides/bios-bmc-technical-guide-v2-1.pdf
[MDR_V2]:https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Smbios/MDR_V2.interface.yaml
[7]:https://github.com/openbmc/phosphor-ipmi-blobs