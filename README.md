# AES Accelerator IoT Camera Demo

Target Platform:

- Hardware: MiniZed Development Board
- Workstation OS: Ubuntu 16.04 LTS
- Essential Tools: PetaLinux Tools, Vivado Design Suite

This project is built upon two other repos.

1. App & Modules [aes128_driver](https://github.com/happyx94/aes128_driver)

2. HDL [AXIS_AES128](https://github.com/happyx94/AXIS_AES128)

There are several essential components with dependency on one another. Please follow the following sections *in the listed order* to build the project.

---

## Section I: Create HDF and Generate Bitstream

### Prerequisites

- Have Vivado 2017.4 and git installed on your machine
- Have the MiniZed board definition file (BDF) in <your-vivado-install_path>/data/boards (* This can be downloaded from Avnet website. Tutorials are also available there.

### Build Instructions

1. On the workstation PC, download the minized_petalinux project by issuing

```shell
mkdir ~/Vivadoprojects
cd ~/Vivadoprojects
git clone git://github.com/Avnet/hdl.git
```

2. Launch Vivado. In the TCL prompt window, issue

```shell
cd ~/Vivadoprojects/hdl/Scripts
source ./make_minized_petalinux.tcl
```

3. Open the project `/hdl/Projects/minized_petalinux/`

4. Clone the [AXIS_AES128](https://github.com/happyx94/AXIS_AES128) repo to wherever you want on your workstation PC. 

5. In Vivado, click *Open Block Design*. In the *IP Catalog* window, right click and select *Add Repository*. Select the AXIS_AES128 repo you just cloned.

6. Add the following to the block design.

- axis_aes128
- axilite_aes_cntl
- AXI Direct Memory Access
- Two instances of AXIS_DATA_FIFO

7. Connect the signals accordingly. 

| From                        | To                         |
| --------------------------- |:--------------------------:|
| M_AXIS_MM2S of the axi_dma  | S_AXIS of axis_data_fifo_0 |
| M_AXIS of axis_data_fifo_0  | S00_AXIS of axis_aes128    |
| M00_AXIS of axis_aes128     | S_AXIS of axis_data_fifo_1 |
| M_AXIS of axis_data_fifo_1  | S_AXIS_S2MM of axi_dma     |
| aes_key of axilite_aes_cntl | cipher_key of axis_aes128  |
| set_IV of axilite_aes_cntl  | set_IV of axis_aes128      |

![AXI-DMA Schematic](/images/axi_dma_schematic.png)

8. Let auto-connection tool handle the rest of the connections.

9. Double click on axi_dma.

- Disable Scatter Gather Engine
- Set Width of Buffer Length Register to 20 bits.
- Set the Memory Map Data Width and Stream Data Width to 128 bits.

![AXI-DMA Configuration](/images/axi_dma_tutorial.png)

10. Double click on the axi_data_fifo (do this step for both data fifos)

- Set TDATA width to 16 bytes

![AXI Data FIFO Configuration](/images/data_fifo_tutorial.png)

11. Double clock on the processing_system (PS). Under Clock Configuration -> PL Fabric Clocks, set 

- Set FCLK_CLK_0 to 71 MHz
- Set FCLK_CLK_1 to 34.782608 MHz

![PS Configuration](/images/ps_tutorial.png)

12. Double click on bluetooth_uart

- Set External CLK Freqency to 34.782608 MHz

![Bluetooth Configuration](/images/bluetooth_tutorial.png)

13. On the toolbar, click *Validate Block Design*. (And pray for no errors :) )

14. Click *Regenerate Layout* on the toolbar. You should have something similar to the following schematic. Save the block design.

![PL Schematic](/images/overview.png)

15. On top of the toolbar of block design window, click on the *Address Editor* tab. Record the Offset Address of

- processing_system7_0
  - -> Data
    - -> axi_dma_0
    - -> axilite_aes_cntl_0

16. Go to Tools -> Settings -> Project Settings -> Implementation

- Change the Strategy under Options to Performance_ExtraTimingOpt

17. Run Synthesis. Run Implementation. Generate Bitstream.

18. File -> Export -> Export Hardware (check Include Bitstream). A small window will popup and you can specify the export location or leave it as default.

    **\*Make sure you know where the HDF and BIT files are exported to**
---

## Section II: Create Bootable Image and Program the Flash

### Prerequisites

- Running Ubuntu 16.04
- Installed PetaLinux Tools 2017.3 (Link Follow the user guide to so do)
- Have the minized_petalinux.hdf and minized_petalinux.bit files from the previous section.

### Build Instructions

1. Download minized_qspi.bsp 2017.2 from minized.org (under MiniZed->Tutorial).

2. Source $(PETALINUX)/settings.sh to start PetaLinux enviornment if you haven't

3. Create a folder for PetaLinux projects if you don't have one yet

```shell
mkdir ~/projects
cd ~/projects
```

4. Create a petalinux project by typing

```shell
petalinux-create -t project -n minized_qspi -s <path-to-minized_qspi.bsp>
```

5. Replace the following files with the ones you have exported in Section I.

- minized_qspi/hardware/MINIZED/minized_petalinux.sdk/minized_petalinux_hw.hdf
- minized_qspi/hardware/MINIZED/minized_petalinux.sdk/minized_petalinux_hw/minized_petalinux_hw.bit

6. Issue

```shell
cd ~/projects/minized_qspi
petalinux-config --get-hw-description=./hardware/MINIZED/minized_petalinux.sdk/
```

The configuration screen should pop up. Just simply save and exit.

7. Build the project by issuing

```shell
petalinux-build
```

8. Copy the following three files from minized_qspi.zip file to ~/projects/minized_qspi

- boot_gen.sh
- bootgen.bif

    Also copy `./image/linux/zynq_fsbl.elf` to the same directory. If it is not there, download the python server tutorial on minized.org and just use the `zynq_fsbl.elf` in the zip file.

9. Generate bootable binary. Under ~/projects/minized_qspi, issue

```shell
./boot_gen.sh
```

10. Start xsct in sudo mode by typing

```shell
sudo <your-vivado-install_path>/Xilinx/SDK/2017.4/bin/xsct
```

11. Connect MiniZed to your workstation PC

12. In XSCT, issue

```shell
exec program_flash -f BOOT.BIN -bin zynq_fsbl.elf -flash_type qspi_single
```

You should expect a message saying program flash operation succeeded.

---

## Section III: Create Customized OS with PetaLinux

### Prerequisites

- Running Ubuntu 16.04
- Installed PetaLinux Tools 2017.3 (Link Follow the user guide to so do)
- You have created the minized_qspi project and programmed the flash accordingly
- A USB stick

### Build Instructions

1. Download minized.bsp 2017.2 from minized.org (under MiniZed->Tutorial).

```shell
petalinux-create -t project -n minized -s <path-to-the-minized.bsp>
```

2. Source $(PETALINUX)/settings.sh to start PetaLinux environment if you haven't

```shell
source $(PETALINUX)/settings.sh
```

3. Create a new petalinux project by typing the following.

```shell
cd ~/projects
petalinux-create -t project -n minized_qspi -s <path-to-the-minized.bsp>
```

4. Replace `minized/hardware/MINIZED/minized_petalinux.sdk/minized_petalinux_hw.hdf` with the HDF you have exported in Section I.

5. Issue

```shell
cd ~/projects/minized
petalinux-config --get-hw-description=./hardware/MINIZED/minized_petalinux.sdk/
```

Optional: When the settings screen pop up, change the rootfs type from initram to initrd if you think your image.ub will exceed 64MB.

6. Issue

```shell
cd ./project-spec/meta-user/recipes-core/images
```

7. Replace or modifiy `petalinux-user-image.bbappend` with [/petalinux_configs/petalinux-user-image.bbappend](/petalinux_configs/petalinux-user-image.bbappend).

8. Configure the kernel. Two ways:

    Simply replace `./project-spec/configs/config with petalinux_configs/config` with [/petalinux_configs/configs](/petalinux_configs/configs)

    *OR*

    Issue
    ```shell
    petalinux-config -c kernel
    ```
    In the pop up configuration screen,
    - **EXCLUDE** all Xilinx AXI-DMA drivers.
    - Include the UVC driver

    Save and exit.

9. Configure the rootfs. Also two ways:

    Replace `./project-spec/configs/rootfs_config` with [/petalinux_configs/rootfs_config](/petalinux_configs/rootfs_config)

    *OR*

    Issue
    ```shell
    petalinux-config -c rootfs
    ```
    In the pop up configuration screen,
    - Include all Gstreamer plugins
    - Include the av encoder/decoder package

    Save and exit.
---

## Section IV: Add Necessary Drivers, Packages, and Applications

### Prerequisites

- Have Section I, II, and III done without any errors

### Build Instructions

1. Follow the steps in [aes128_driver](https://github.com/happyx94/aes128_driver) to include the necessary applications and modules.

2. Copy `./images/linux/image.up` to your USB stick. Also copy the tool scripts as well as the wifi config files to the usb stick.

3. Boot your minized. Interrupt the autobooting. In uboot shell, type

```shell
run boot_qspi
```

4. Plug-in an extra power cable and the USB stick. Mount the device to `/mnt/usb` if it is not mounted automatically.

5. Copy the image, [scripts](/demo_scripts), [config files](/wifi_conf) to the eMMC.

```shell
cd /mnt/usb
cp image.ub ../emmc/
cp <all-demo-files-and-wifi-conf-file> ../emmc
sync
reboot
```

---

## Section V: Setup the Client Side and Run the Demo

1. Setup the receiver programs on your client side computer. Run the following to see your ip address

```shell
ifconf
```

2. Download **ALL** source files for the AES128 program in [aes128_driver](https://github.com/happyx94/aes128_driver)/aes128 and /common

3. Compile the AES128 program on your client PC by issuing

```shell
cc -o aes128 aes128.c dma_driver.c sw_aes.c
```

4. Install Gstreamer and Netcat tools on your client PC if you haven't.

5. Download the receiver side script /demo_scripts/receiver.sh to the same folder as your aes128 program. Make sure you have the same *key* file in the directory.

6. Boot minized. Connect an extra power cable and the USB camera. Connect MiniZed to the Wi-Fi by issuing

```shell
wifi.sh
```

7. Make sure the client PC and the MiniZed are in the same network and there is no firewall blocking the connections between them (e.g. you get a response on one if you `ping` the other).

8. On the **Client PC**, in the directory where you put the client scripts, issue

```shell
./receiver 5000 8192
```

9. On the **MiniZed board**, run

```shell
/mnt/emmc/easy_demo.sh <your-computers-ip>
```

10. You should see the video streaming on the client PC now (hopefully).

---

## Section VI: Additional Server Programs and Android Client

See the following repos:

- [Server](https://github.com/edwardWang95/SeniorDesignServerClean)
- [Android Client](https://github.com/edwardWang95/SeniorDesignAndroidClient)
