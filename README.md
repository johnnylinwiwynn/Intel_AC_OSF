# Steps for building Intel 2S Archer City CRB coreboot + LinuxBoot

## Build Linux payload:  
`git clone https://github.com/linuxboot/osf-builder`  
At the time of the writing, it's based on tip: `78bcde1 Merge pull request #16 from johnnylinwiwynn/pr`  
`cd examples/qemu; make kernel`  
The built image will be at build/qemu-x86_64/kernel/arch/x86/boot/bzImage  
You can add vpdbootmanager utility by modifying examples/qemu/Makefile:  
`UROOT_ADDITIONAL_CMDS := \
        github.com/u-root/u-root/cmds/boot/boot github.com/u-root/u-root/tools/vpdbootmanager`  
## coreboot  

Reference the steps for installing the needed tools and git clone the repo from https://doc.coreboot.org/tutorial/part1.html  

At the time of the writing, it's based on upstream tip: `bd054832d2 drivers/spi: Remove SPI_FRAM_RAMTRON from makefile`  
Cherry-pick the below 2 changes for multi-socket support  
https://review.coreboot.org/c/coreboot/+/61164/3  
https://review.coreboot.org/c/coreboot/+/74297/2  
Add CONFIG_XAPIC_ONLY=y to configs/builder/config.intel.crb.ac  
Or enable CONFIG_X86_X2APIC in your Linux payload if you need to enable X2APIC.  

Put the binary files according to the paths in configs/builder/config.intel.crb.ac  
`mkdir -p site-local/archercity`  
CONFIG_PAYLOAD_FILE="site-local/archercity/linuxboot_bzImage"  
This is the Linux payload.

### Intel binaries that should be obtained under Intel NDA  
CONFIG_FSP_T_FILE="site-local/archercity/Server_T.fd"  
CONFIG_FSP_M_FILE="site-local/archercity/Server_M.fd"  
CONFIG_FSP_S_FILE="site-local/archercity/Server_S.fd"  
These are FSP API mode binaries and should be built from 2022 WW43 Intel Eagle Stream BKC,  
please contact your Intel representatives for the detailed steps, you need to revert one commit for coreboot + LinuxBoot.  

CONFIG_IFD_BIN_PATH="site-local/archercity/descriptor.bin"  
CONFIG_ME_BIN_PATH="site-local/archercity/me.bin"  
descriptor.bin is Intel flash descriptor and me.bin is the SPS/IGN binary.  
We can extract them from an Archer City CRB firmware image via ifdtool.  
Build ifdtool first:  
`cd util/ifdtool; make`  
Extract them from an Intel 2022 WW43 Archer City image BIOS.bin:  
`./ifdtool -x BIOS.bin`  
flashregion_0_flashdescriptor.bin is the descriptor.bin  
flashregion_2_intel_me.bin is the me.bin  

CONFIG_CPU_UCODE_BINARIES="site-local/archercity/mbf806f8.mcb"  
mbf806f8.mcb is the microcode binary for your CPU stepping.  

### Build crossgcc-i386  
`make crossgcc-i386 -j$(nproc)`  
This may take a while but only needs to be done once.  

### Build coreboot image  
`make defconfig KBUILD_DEFCONFIG=configs/builder/config.intel.crb.ac`  
`make olddefconfig`  
`make clean`  
`make -j$(nproc)`  
It will git clone the needed git submodules and that should only be done for the first time.  
If there is no need to update git submodules, you can skip it and build by  
`make UPDATED_SUBMODULES=1 -j$(nproc)`  
The built coreboot image will be at build/coreboot.rom.  

## Add VPD variables to the image
You can build vpd utility from source  
https://chromium.googlesource.com/chromiumos/platform/vpd  
or use the binary from  
https://github.com/linuxboot/osf-builder/tree/main/tools  
### Format RO_VPD and add key-value pairs  
`vpd -f build/coreboot.rom -O -i RO_VPD -s fsp_log_enable=0`  
`vpd -f build/coreboot.rom -i RO_VPD -s firmware_version=v1.0.0`  
`vpd -f build/coreboot.rom -i RO_VPD -s coreboot_log_level=7`  
### Format RW_VPD as an empty region  
`vpd -f build/coreboot.rom  -O -i  RW_VPD`  

You are all set, flash coreboot.rom and have fun!
