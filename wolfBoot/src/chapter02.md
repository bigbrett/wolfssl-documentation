# Compiling wolfBoot

WolfBoot is portable across different types of embedded systems. The platform-specific code is contained in a single file under the `hal` directory, and implements the hardware-specific functions.

To enable specific compile options, use environment variables while calling make, e.g.

`make CORTEX_M0=1`

As an alternative, you can provide a .config file in the root directory of wolfBoot. Command line options have priority on `.config` options, as long as .config options are defined using the `?=` operator, e.g.:

`WOLFBOOT_PARTITION_BOOT_ADDRESS?=0x14000`

## Generate a new configuration

A new `.config` file with a set of default parameters can be generated by running `make config`. The build script will ask to enter a default value for each configuration parameter. Enter confirm the current value, indicated in between `[]`.

Once a .config file is in place, it will change the default compile-time options when running `make` without parameters.

.config can be modified with a text editor to alter the default options later on.

## Platform selection

If supported natively, the target platform can be specified using the `TARGET` variable. Make will automatically select the correct compile option, and include the corresponding HAL for the selected target. 

For a list of the platforms currently supported, see the chapter on [HAL](chapter04.md#hardware-abstraction-layer).

To add a new platform, simply create the corresponding HAL driver and linker script file  in the `hal` directory.

Default option if none specified: `TARGET=stm32f4`

Some platforms will require extra options, specific for the architecture. By default, wolfBoot is compiled for ARM Cortex-M3/4/7. To compile for Cortex-M0, use:

`CORTEX_M0=1`

### Flash partitions

The file `include/target.h` is generated according to the configured flash geometry, partitions size and offset of the target system. The following values must be set to provide the desired flash configuration, either via the command line, or using the .config file: 

 - `WOLFBOOT_SECTOR_SIZE` 

This variable determines the size of the physical sector on the flash memory. If areas with different block sizes are used for the two partitions (e.g. update partition on an external flash), this variable should indicate the size of the biggest sector shared between the two partitions.

WolfBoot uses this value as minimum unit when swapping the firmware images in place. For this reason, this value is also used to set the size of the SWAP partition. 

 - `WOLFBOOT_PARTITION_BOOT_ADDRESS`

This is the start address of the boot partition, aligned to the beginning of a new flash sector. The application code starts after a further offset, equal to the partition header size (256B  for Ed25519 and ECC signature headers).

 - `WOLFBOOT_PARTITION_UPDATE_ADDRESS`

This is the start address of the update partition. If an external memory is used via the  `EXT_FLASH` option, this variable contains the offset of the update partition from the beginning of the external memory addressable space.

 - `WOLFBOOT_PARTITION_SWAP_ADDRESS`

The address for the swap spaced used by wolfBoot to swap the two firmware images in place, in order to perform a reversable update. The size of the SWAP partition is exactly one sector on the flash. If an external memory is used, the variable contains the offset of the SWAP area from the beginning of its addressable space.

 - `WOLFBOOT_PARTITION_SIZE`

The size of the BOOT and UPDATE partition. The size is the same for both partitions.

## Bootloader features

A number of characteristics can be turned on/off during wolfBoot compilation. Bootloader size, performance and activated features are affected by compile-time flags.

### Change DSA algorithm

By default, wolfBoot is compiled to use Ed25519 DSA. The implementation of ed25519 is smaller, while giving a good compromise in terms of boot-up time.

Better performance can be achieved using ECDSA with curve p-256. To activate ECC256 support, use

`SIGN=ECC256`

when invoking `make`.

RSA is also supported, with different key length. To activate RSA2048 or RSA4096, use:

`SIGN=RSA2048`

or

`SIGN=RSA4096`

respectively.

Ed448 is also supported via `SIGN=ED448`.

The default option, if no value is provided for the `SIGN` variable, is

`SIGN=ED25519`

Changing the DSA algorithm will also result in compiling a different set of tools for key generation and firmware signature.

Find the corresponding key generation and firmware signing tools in the `tools` directory.

It's possible to disable authentication of the firmware image by explicitly using:

`SIGN=NONE`

in the Makefile commandline. This will compile a minimal bootloader with no support for public-key authenticated  secure boot.

### Incremental updates

wolfBoot support incremental updates. To enable this feature, compile with `DELTA_UPDATES=1`.

An additional file is generated when the sign tool is invoked with the `--delta` option, containing only the differences between the old firmware to replace, currently running on the target, and the new version.

For more information and examples, see the [firmware update](chapter06.md#firmware-update) section.

### Enable debug symbols

To debug the bootloader, simply compile with `DEBUG=1`. The size of the bootloade will increase consistently, so ensure that you have enough space at the beginning of the flash before  `WOLFBOOT_PARTITION_BOOT_ADDRESS`.

### Disable interrupt vector relocation

On some platforms, it might be convenient to avoid the interrupt vector relocation before boot-up. This is required when a component on the system already manages the interrupt relocation at a different  stage, or on these platform that do not support interrupt vector relocation.

To disable interrupt vector table relocation, compile with `VTOR=0`. By default, wolfBoot will relocate the interrupt vector by setting the offset in the vector relocation offset register (VTOR).

### Limit stack usage

By default, wolfBoot does not require any memory allocation. It does this by performing all the operations using the stack. Although the stack space used by the algorithms can be predicted at compile time, the amount of stack space be relatively big, depending on the algorithm selected.

Some targets offer limited amount of RAM to use as stack space, either in general, or in a configuration dedicated for the bootloader stage.

In these cases, it might be useful to activate `WOLFBOOT_SMALL_STACK=1`. With this option, a fixed-size pool is created at compile time to assist the allocation of the object needed by the cryptography implementation. When compiled with `WOLFBOOT_SMALL_STACK=1`, wolfBoot reduces the stack usage considerably, and simulates dynamic memory allocations by assigning dedicated, statically allocated, pre-sized memory areas.

### Disable Backup of current running firmware

Optionally, it is possible to disable the backup copy of the current running firmware upon the installation of the update. This implies that no fall-back mechanism is protecting the target from a faulty firmware installation, but may be useful in some cases where it is not possible to write on the update partition from the bootloader. The associated compile-time option is

`DISABLE_BACKUP=1`

### Enable workaround for 'write once' flash memories

On some microcontrollers, the internal flash memory does not allow subsequent writes (adding zeroes) to a sector, after the entire sector has been erased. WolfBoot relies on the mechanism of adding zeroes to the 'flags' fields at the end of both partitions to provide a fail-safe swap mechanism.

To enable the workaround for 'write once' internal flash, compile with

`NVM_FLASH_WRITEONCE=1`

**warning** When this option is enabled, the fail-safe swap is not guaranteed, i.e. the microcontroller cannot be safely powered down or restarted during a swap operation.

### Allow version roll-back

WolfBoot will not allow updates to a firmware with a version number smaller than the current one. To allow  downgrades, compile with `ALLOW_DOWNGRADE=1`. 

Warning: this option will disable version checking before the updates, thus exposing the system to potential forced downgrade attacks.

### Enable optional support for external flash memory

WolfBoot can be compiled with the makefile option `EXT_FLASH=1`. When the external flash support is enabled, update and swap partitions can be associated to an external memory, and will use alternative HAL function for read/write/erase access.  To associate the update or the swap partition to an external memory, define `PART_UPDATE_EXT` and/or  `PART_SWAP_EXT`, respectively. By default, the makefile assumes that if an external memory is present, both `PART_UPDATE_EXT` and `PART_SWAP_EXT` are defined.

If the `NO_XIP=1` makefile option is present, `PART_BOOT_EXT` is assumed too, as no execute-in-place is available on the system. This is typically the case of MMU system (e.g. Cortex-A) where the operating system image(s) are position-independent ELF images stored in a non-executable non-volatile memory, and must be copied in RAM to boot after verification.

When external memory is used, the HAL API must be extended to define methods to access the custom memory. Refer to the [HAL](chapter04.md#hardware-abstraction-layer) chapter for the description of the `ext_flash_*` API.

#### SPI devices

In combination with the `EXT_FLASH=1` configuration parameter, it is possible to use a platform-specific SPI drivers, e.g. to access an external SPI flash memory. By compiling wolfBoot with the makefile option `SPI_FLASH=1`, the external memory is directly mapped to the additional SPI layer, so the user does not have to define the `ext_flash_*` functions.

SPI functions, instead, must be defined. Example SPI drivers are available for multiple platforms in the `hal/spi` directory.

#### UART bridge towards neighbor systems

Another alternative available to map external devices consists in enabling a UART bridge towards a neighbor system. The neighbor system must expose a service through the UART interface that is compatible with the wolfBoot protocol.

In the same way as for SPI devices, the `ext_flash_*` API is automatically defined by wolfBoot when the option `UART_FLASH=1` is used.

For more details, see the section [Remote External flash memory support via UART](chapter06.md#remote-external-flash-memory-support-via-uart)

#### Encryption support for external partitions

When update and swap partitions are mapped to an external device using `EXT_FLASH=1`, either in combination with `SPI_FLASH`, `UART_FLASH`, or any custom external mapping, it is possible to enable ChaCha20, Aes128 or Aes256 encryption when accessing those partition from the bootloader. The update images must be pre-encrypted at the source using the key tools, and wolfBoot should be instructed to use a temporary ChaCha20 symmetric key to access the content of the updates.

For more details about this optional feature, please refer to the [Encrypted external partitions](chapter06.md#encrypted-external-partitions) section.


### Executing flash access code from RAM

On some platform, flash access code requires to be executed from RAM, to avoid conflict e.g. when writing to the same device where wolfBoot is executing, or when changing the configuration of the flash itself.

To move all the code accessing the internal flash for writing, into a section in RAM, use the compile time option `RAM_CODE=1` (on some hardware configurations this is required for the bootloader to access the flash for writing).

### Enable Dual-bank hardware-assisted swapping

When supported by the target platform, hardware-assisted dual-bank swapping can be used to perform updates. To enable this functionality, use `DUALBANK_SWAP=1`. Currently, only STM32F76x and F77x support this feature.


### Store UPDATE partition flags in a sector in the BOOT partition

By default, wolfBoot keeps track of the status of the update procedure to the single sectors in a specific area at the end of each partition, dedicated to store and retrieve a set of flags associated to the partition itself.

In some cases it might be helpful to store the status flags related to the UPDATE partition and its sectors in the internal flash, alongside with the same set of flags used for the BOOT partition. By compiling wolfBoot with the `FLAGS_HOME=1` makefile option, the flags associated to the UPDATE partition are stored in the BOOT partition itself.

While on one hand this option slightly reduces the space available in the BOOT partition to store the firmware image, it keeps all the flags in the BOOT partition.

### Invert logic of flags
By default, most NVMs set the content of erased pages to `0xFF` (all ones).  Some FLASH memory models use inverted logic for erased page, setting the content to `0x00` (all zeroes) after erase. For these special cases, the option `FLAGS_INVERT = 1` can be used to modify the logic of the partition/sector flags used in wolfBoot.

Note: if you are using an external FLASH (e.g. SPI) in combination with a flash with inverted logic, ensure that you store all the flags in one partition, by using the `FLAGS_HOME=1` option described above.



### Using Mac OS/X

If you see 0xC3 0xBF (C3BF) repeated in your factory.bin then your OS is using Unicode characters.

The "tr" command for assembling the 0xFF padding between `"bootloader" ... 0xFF ... "application" = factory.bin`, which requires the "C" locale.

Set this in your terminal
```
LANG=
LC_COLLATE="C"
LC_CTYPE="C"
LC_MESSAGES="C"
LC_MONETARY="C"
LC_NUMERIC="C"
LC_TIME="C"
LC_ALL=
```

Then run the normal `make` steps.

### Enabling mitigations against glitches and fault injections

One type of attacks against secure boot mechanisms consists in skipping the execution of authentication and validation steps by injecting faults into the CPU through forced voltage or clock anomalies, or electromagnetic interferences at close range.

Extra protection from specific attacks aimed to skip CPU instructions can be enabled using `ARMOR=1`. This feature is currently only available for ARM Cortex-M targets.
