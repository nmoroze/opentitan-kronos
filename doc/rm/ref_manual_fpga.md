# FPGA Reference Manual

This manual provides additional usage details about the FPGA.
Specifically, it provides instructions on SW development flows, testing and bitfile release procedures.

{{% toc 3 }}

## Usage Options

There are two ways to use OpenTitan on the FPGA.
The first is to build the design from scratch using Vivado tools.
The second is to program the FPGA with a released bitfile using storage devices.

For the first option, please refer to the [getting_started_fpga](../ug/fpga/getting_started_fpga.md) guide.
For the second option, please refer to the [quick_start_fpga](../ug/fpga/quick_start_fpga.md) guide.

## FPGA SW Development Flow

The FPGA is meant for both boot ROM and general software development.
The flow for each is different, as the boot ROM is meant to be fairly static while general software can change very frequently.

### Boot ROM development

The FPGA bitstream is built after compiling whatever code is sitting in `sw/device/boot_rom`.
This binary is used to initialize internal FPGA memory and is part of the bitstream directly.

To update this content without rebuilding the FPGA, a flow is required to splice a new boot ROM binary into the bitstream.
There are two prerequisites in order for this flow to work:
* The boot ROM during the build process must be correctly inferred by the tool.
  * See [prim_rom](https://github.com/lowRISC/opentitan/blob/master/hw/ip/prim_xilinx/rtl/prim_xilinx_rom.sv).
* The boot ROM's physical location must be fixed.
  * See [placement.xdc](https://github.com/lowRISC/opentitan/blob/master/hw/top_earlgrey/data/placement.xdc).

With these steps in place, a script can be invoked to take a new binary and push its contents into an existing bitfile.
For details, please see the [`splice_nexysvideo.sh` script](https://github.com/lowRISC/opentitan/blob/master/util/fpga/splice_nexysvideo.sh).

See example below:

```console
$ cd $REPO_TOP
$ ./util/fpga/splice_nexysvideo.sh
$ fusesoc --cores-root . pgm lowrisc:systems:top_earlgrey_nexysvideo
```

The script assumes that there is an existing bitfile `build/lowrisc_systems_top_earlgrey_nexysvideo_0.1/synth-vivado/lowrisc_systems_top_earlgrey_nexysvideo_0.1.bit` (this is created after following the steps in [getting_started_fpga](../ug/fpga/getting_started_fpga.md)).

The script rebuilds the contents in `sw/devices/boot_rom` and then creates a new bitfile of the same name at the same location.
The original input bitfile is moved to `build/lowrisc_systems_top_earlgrey_nexysvideo_0.1/synth-vivado/lowrisc_systems_top_earlgrey_nexysvideo_0.1.bit.orig`.

The fusesoc command can then be directly invoked to flash the FPGA.

### General Software Development

After building, the FPGA bitstream contains only the boot ROM.
Using this boot ROM, the FPGA is able to load additional software to the emulated flash, such as software in the `sw/device/benchmark`, `sw/device/examples` and `sw/device/tests` directories.
To load additional software, a custom load tool named [spiflash](../../sw/host/spiflash/README.md) is required.

Once the tool is built, also build the binary you wish to load.
For the purpose of this demonstration, we will use `sw/device/examples/hello_world`, but it applies to any software image that is able to fit in the emulated flash space.
The example below builds the `hello_world` image and loads it onto the FPGA.
The loading output is also shown.

```console
$ cd ${REPO_TOP}
$ make -C sw/device SW_DIR=examples/hello_world SW_BUILD_DIR=out clean all
$ make -C sw/host/spiflash clean all
$ ./sw/host/spiflash/spiflash --input=sw/device/out/sw.bin

Running SPI flash update.
Image divided into 6 frames.
frame: 0x00000000 to offset: 0x00000000
frame: 0x00000001 to offset: 0x000003d8
frame: 0x00000002 to offset: 0x000007b0
frame: 0x00000003 to offset: 0x00000b88
frame: 0x00000004 to offset: 0x00000f60
frame: 0x80000005 to offset: 0x00001338
```

For more details on the exact operation of the loading flow and how the boot ROM processes incoming data, please refer to the [boot ROM readme](../../sw/device/boot_rom/README.md).

## FPGA Testing and Release Procedure

As mentioned in [quick_start](../ug/fpga/quick_start_fpga.md), golden bitfiles will be released.
Before release, a check process will be performed on the release git-tag to ensure the FPGA bitstream is functional.
The [checks to be performed are](link script later):
    * FPGA is built without issues.
      * There are no timing violations.
      * There are no resource issues.
      * There is no ROM inference or initialization issue (critical warning synth 8-4445)
      * The ROM splicing script is able to correctly locate the ROM and perform updates
    * Existing sanity tests pass without failures.
      * All current tests under `sw/device/tests`.


In the future, this process will be enhanced to check for the following:
    * RISCV compliance test-suite.
    * Additional self contained tests.


Once the above checks all pass, the tested  git commit is tagged through the [CalVer convention](https://calver.org).
The bitfile is the associated with the tag in the github release flow.

See sample usage below.

```console
TO BE FILLED IN
```