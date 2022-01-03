# Read MAC from EEPROM writeup

Author: Hongwei Zhang (hongweiz@ami.com)

Primary assignee: Hongwei Zhang

Other contributors:
Pravinash Jeyapaul (pravinashj@ami.com)
Sanjay Ahuja (SanjayA@ami.com)

Created: December 15, 2021

## Problem Description

There are several ways for U-Boot to read, configure and store the Ethernet
 MAC address in non-volatile memory. This document describes getting MAC from 
EEPROM in U-Boot and configure it on aspeed BMC SoC based platform.

## EEPROM in  AST2600EVB and Driver

U-Boot has built-in driver for I2C EEPROM/flash storage. It can be configured 
through Kconfig. AST2600EVB has Atmel EEPROM, generic i2c eeprom driver is 
compatible with Atmel   (drivers/misc/i2c_eeprom.c. )

A sample DTS node for Atmel EEPROM:


<span style="font-family:courier new;">

            &i2c7 {
                    status = "okay";
                    #pinctrl-names = "default";
                    #pinctrl-0 = <&pinctrl_i2c8_default>;
                    eeprom@50 {
                            compatible = "atmel,24c02";
                            reg = <0x50>;
                            #pagesize = <16>;
                            macoffset = <0>;
                    };
            }

</span>


## Read and Configure the MAC from EEPROM

For the MAC from EEPROM feature to be configurable, we define 
CONFIG_MACADDR_IN_EEPROM in board/aspeed/evb_ast2600/Kconfig, implemented 
the function call to read the MAC from EEPROM driver in separate file. We can 
also add the offset of MAC address in EEPROM into the dts node, so it can be 
used in reading function.

At the early stage of U-BOOT’s initialization, if the feature is configured 
then need to check if the MAC address from EEPROM is match with already 
configured address from ENV, if they are not, then update the MAC address 
in ENV with EEPROM MAC address.

## Proposed Design


## Alternatives Considered

## Impacts

## Testing
- Unit tests to validate the MAC address read from EEPROM.

