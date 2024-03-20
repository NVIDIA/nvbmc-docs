# NvBMC:  GB200NVL BMC Platform Enablement Guide

## Prerequisite

* Buildable machine configuration and meta layer defined
* BMC Image boots on AST2600 with 64MB Code SPI0 and 64MB Data SPI2
* BMC UART and SSH functional
* GPIO line names defined in kernel DTS

For purposes of description, this guide will focus on one NVIDIA BMC platform called **NVIDIA GB200-NVL**.
* meta-layer:    `meta-nvidia/meta-prime/meta-gb200nvl`
* machine conf:  `gb200nvl-bmc`

## Key Functions

1. Enable Standby power and boot the FPGA
2. Enable and disable RUN power
3. Enable RUN power status monitor

## bmc-post-boot-cfg - Enable Standby Power and Boot FPGA

| | |
|--|--|
| Recipe | `meta-nvidia/meta-prime/meta-gb200nvl/meta-bmc/recipes-nvidia/bmc-post-boot-cfg/bmc-post-boot-cfg.bbappend` |
| Description | This is the basic platform enablement recipe and is a systemd required service dependency upon most other NVIDIA services. |

See the `bmc-ready.sh` script executed by this service.
1. Enable Standby Power to the FPGA and board components
2. Wait for Standby Power Good
3. Bind drivers for I2C GPIO exanders (many expanders are powered by standby power and can't be initialized at time 0)
4. Enable PCIe discovery of FPGA
5. Write SGPIO-BMC-EN-O=1 to correctly set mux to send SGPIO signals to FPGA
6. Set BMC-EROT-FPGA-SPI-MUX-SEL-O = 1 to enable FPGA to access its EROT
7. Release FPGA EROT from reset
8. Release HMC EROT from reset
9. Release HMC from reset
10. Wait for HMC ready
11. Bind any EEPROM devices that are standby powered
12. Set BMC-READY signal to FPGA (informs FPGA that BMC GPIO signals are trustworthy)

## powerctrl - RUN Power Control

| | |
|--|--|
| Recipe | `meta-nvidia/meta-prime/meta-gb200nvl/meta-bmc/recipes-nvidia/powerctrl/powerctrl.bb` |
| Description | This service is the lowest level RUN power control script.  It includes functions for power on, power off, graceful shutdown, and host CPU reset.  These functions are wrapped in systemd services that are set up as required by phosphor-state-manager in such as way that host and chassis power transition requests invoke the proper script function to actuate the hardware.  |

### Power Transitions

|	Redfish ResetType	|	ipmitool	|	IPMI raw command equivalent	|	obmcutil	|	State Transition Name	|	phosphor-state-manager	|	State Transition systemd target	|	NVIDIA Platform Power Control systemd service	|	powerctrl.sh script cmdline	|	Expected Behavior	|
|	---	|	---	|	---	|	---	|	---	|	---	|	---	|	---	|	---	|	---	|
|	ForceOn + On	|	ipmitool chassis power on/ipmitool power on	|	ipmitool raw 0x00 0x02 0x01	|	poweron	|	xyz.openbmc_project.State.Host.Transition.On	|	HOST_STATE_POWERON_TGT	|	obmc-host-start@0.target	|	host-poweron@0.service	|	powerctrl.sh power_on	|	Power On if off, no effect if on	|
|	ForceOff	|	ipmitool chassis power off/ipmitool power off	|	ipmitool raw 0x00 0x02 0x00	|	chassisoff	|	xyz.openbmc_project.State.Chassis.Transition.Off	|	CHASSIS_STATE_HARD_POWEROFF_TGT	|	obmc-chassis-hard-poweroff@0.target	|	host-poweroff@0.service	|	powerctrl.sh power_off	|	Power off if on, no effect if off	|
|	ForceRestart	|	ipmitool chassis power reset/ipmitool power reset	|	ipmitool raw 0x00 0x02 0x03	|	--none--	|	xyz.openbmc_project.State.Host.Transition.ForceWarmReboot	|	HOST_STATE_FORCE_WARM_REBOOT	|	obmc-host-force-warm-reboot@0.target	|	host-reset@0.service	|	powerctrl.sh reset	|	Ungraceful reset of CPU, no power off/on - restart boot	|
|	GracefulShutdown	|	ipmitool chassis power soft/ipmitool power soft	|	ipmitool raw 0x00 0x02 0x05	|	poweroff	|	xyz.openbmc_project.State.Host.Transition.Off	|	HOST_STATE_SOFT_POWEROFF_TGT	|	obmc-host-shutdown@0.target	|	graceful-host-poweroff@0.service	|	powerctrl.sh grace_off	|	OS Shutdown if booted, power off at end of shutdown, remains off	|
|	GracefulRestart	|		|		|	--none--	|	xyz.openbmc_project.State.Host.Transition.GracefulWarmReboot	|	HOST_STATE_WARM_REBOOT	|	obmc-host-warm-reboot@0.target	|	host-reset@0.service	|	powerctrl.sh reset	|	Ungraceful reset of CPU, no power off/on - restart boot	|
|	PowerCycle	|	ipmitool chassis power cycle/ipmitool power cycle	|	ipmitool raw 0x00 0x02 0x02	|	--none--	|	xyz.openbmc_project.State.Host.Transition.Reboot	|	HOST_STATE_REBOOT_TGT	|	obmc-host-reboot@0.target	|	graceful-host-poweroff@0.service	|	powerctrl.sh grace_off	|	OS Shutdown if booted, power off at end of shutdown, powers back on after a few seconds	|
|	Nmi	|		|		|		|		|		|		|		|		|	not supported	|

## nvidia-power-monitor - RUN Power Status Monitor

| | |
|--|--|
| Recipe | `meta-nvidia/meta-prime/meta-bmc-common/recipes-nvidia/powerstatus/nvidia-power-monitor.bb` |
| Description | This runs a service that monitors the power good signal and ensures phosphor-state-manager host and chassis power states reflect hardware status. It also performs additional activities on power off such as re-asserting reset to the host CPUs and removing the Redfish Host Interface user account(s).|

The `nvidia-power-monitor` starts after `bmc-post-boot-cfg` is active (standby power is enabled and the FPGA is initialized.)  It is a script that sets the initial RUN POWER GOOD state as the Chassis on/off state in `phosphor-state-manager` and then enters a while loop and blocks on RUN POWER GOOD GPIO signal changes.  For each change it activates `phosphor-state-manager` transitions to keep state in sync with the hardware.  It also performs any power off housekeeping tasks.

## Other NVIDIA BMC Recipes
Refer to sample code for additional detail.  
If customization is required, consider adding a .bbappend under a lower meta-layer.

### bmc-internal-network-config
|Paths|
|---|
|meta-nvidia/meta-prime/meta-bmc-common/recipes-nvidia/bmc-internal-network-config/bmc-internal-network-config.bb|

The BMC has two internal USB network connections.  This recipe configures both.
1. USB to host - this is the Redfish Host Interface which also services as the Virtual Media USB Device endpoint for the host.
2. USB to HMC - this is the BMC/HMC internal network connection.

### bmc-systemd-conf
|Paths|
|---|
|meta-nvidia/meta-prime/meta-bmc-common/recipes-nvidia/bmc-systemd-conf/bmc-systemd-conf.bb|

Modifies `systemd-networkd-wait-online.service` start behavior.  Do not wait on the host USB network connection to be ready before `systemd-networkd-wait-online.service` starts.  This connection will be configured during UEFI POST.

### nvidia-emmc_partition
|Paths|
|---|
|meta-nvidia/recipes-nvidia/emmc-partition/nvidia-emmc-partition.bb|
|meta-nvidia/meta-prime/meta-gb200nvl/recipes-nvidia/emmc-partition/nvidia-emmc-partition.bbappend|

Initializes, partitions, and mounts the EMMC device to the BMC or HMC.

### nvidia-fpga-ready-monitor
|Paths|
|---|
|meta-nvidia/meta-prime/recipes-nvidia/fpga-ready-monitor/nvidia-fpga-ready-monitor.bb|

Monitors the FPGA_READY GPIO signal and activates systemd services upon changes in state.

### nvidia-journal-conf
|Paths|
|---|
|meta-nvidia/meta-prime/recipes-nvidia/systemd-conf/nvidia-journal-conf.bb|

Configures the systemd journal for NVIDIA systems.

### nvidia-mac-update
|Paths|
|---|
|meta-nvidia/recipes-nvidia/mac-update/nvidia-mac-update.bb|
|meta-nvidia/meta-prime/meta-bmc-common/recipes-nvidia/mac-update/nvidia-mac-update.bbappend|

Parses a MAC address from the BMC FRU EEPROM and applies it to the BMC network configuration.  This is the means by which content in the FRU EEPROM controls the BMC's persistent MAC address.

### nvidia-mc-aspeed-lib
|Paths|
|---|
|meta-nvidia/meta-prime/recipes-nvidia/nvidia-mc-aspeed-lib/nvidia-mc-aspeed-lib.bb|

Contains GPIO pin definitions for user mode operations in the form of a sourceable shell script.


### nvidia-mc-lib
|Paths|
|---|
|meta-nvidia/meta-prime/recipes-nvidia/nvidia-mc-lib/nvidia-mc-lib.bb|
|meta-nvidia/meta-prime/meta-gb200nvl/meta-bmc/recipes-nvidia/nvidia-mc-lib/nvidia-mc-lib.bbappend|

Contains a number of utility and initialization scripts for the BMC.

### sbiosbootaccess
|Paths|
|---|
|meta-nvidia/meta-prime/meta-bmc-common/recipes-nvidia/sbiosbootaccess/sbiosbootaccess.bb|

Monitors the system CPU_BOOT_DONE GPIO signal and changes access to the Redfish Host Interface.  Also removes the Redfish Host Interface UEFI user account at the end of POST or on Power Off.

CPU_BOOT_DONE is 0 if RUN power is OFF, the host CPU is reset, or if the host is in POST.  The value changes to 1 at the end of UEFI upon boot attempt.

This recipe performs actions that need to happen when this state changes.

### set-hmc-time
|Paths|
|---|
|meta-nvidia/meta-prime/meta-bmc-common/recipes-nvidia/set-hmc-time/set-hmc-time.bb|

Updates the HMC time using a Redfish PATCH operation.  This can happen on demand by starting the `set-hmc-time.service` and also happens once per hour.

## NVIDIA BMC Recipe Overrides (.bbappend)

### nvidia-code-mgmt
|Paths|
|---|
|meta-nvidia/meta-prime/recipes-nvidia/nvidia-code-mgmt/nvidia-code-mgmt.bbappend|

Performs firmware management operations including inventory and update on non PLDM devices.

### nvidia-ipmi-oem
|Paths|
|---|
|meta-nvidia/meta-prime/meta-bmc-common/recipes-nvidia/nvidia-ipmi-oem/nvidia-ipmi-oem-git.bbappend|

Enables the Grace version of the IPMI OEM commands.

### nvidia-power-manager
|Paths|
|---|
|meta-nvidia/meta-prime/meta-gb200nvl/recipes-nvidia/nvidia-power-manager/nvidia-power-manager-%.bbappend|
|meta-nvidia/meta-prime/meta-gb200nvl/meta-bmc/recipes-nvidia/nvidia-power-manager/nvidia-power-manager-%.bbappend|

Enables power reporting and limiting for NVIDIA devices such as the GPU and module.

## NVIDIA GB200 NVL U-Boot and Kernel Customizations

### U-Boot
|Paths|
|---|
|meta-nvidia/meta-prime/meta-gb200nvl/meta-bmc/recipes-bsp/u-boot/u-boot-aspeed-sdk-%.bbappend|

Contains the U-Boot DTS file for the BMC.

### Kernel
|Paths|
|---|
|meta-nvidia/meta-prime/meta-gb200nvl/meta-bmc/recipes-kernel/linux/linux-aspeed-%.bbappend|

Contains the kernel DTS file, configuration, and GPIO line names for the BMC.

