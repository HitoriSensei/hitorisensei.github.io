---
layout: post
title:  "Breaking and fixing Mercusys MR70X router with OpenWRT firmware"
date:   2024-07-08 15:20:03 +0200
categories: 
  - networking
---
# Broken (and fixed) things

* `Bad Magic Number`
* `01221: the firmware software version dismatched` (not entirely fixed, workaround)
* `No filesystem could mount root, tried:  squashfs`
* `00331: factory boot check integer flag is not 1`

# How to break it

* Flash OpenWRT using WEB UI

* Flash OEM firmware package, don’t read guides, ignore warnings (you asked for a brick)

* Say hello to softbrick:

  ```
  3: System Boot system code via Flash.
  ## Booting image at bc040000 ...
  Bad Magic Number,00E058EC
  ```



# How to fix it

## Easy, if you are lucky

* Connect router to the PC via ethernet cable

* Set PC network settings to static IP

  * IP: 192.168.1.5
  * Netmask: 255.255.255.0
  * Gateway: 192.168.1.1

* Open page `http://192.168.1.1`

  * If the page doesn’t open up, make sure that go to hard mode below

* Select OEM firmware, upload

  * If you get an error before the progress completes, you are flashing wrong file. It must be original OEM file with .bin extension

* If the recovery page is just refreshed after completion, you are not lucky. Time go get hands dirty. Get yourself a 3.3V UART dongle.

  * During flash you should probably see a log in the serial console:

    ```
    [NM_Error](nm_checkUpdateContent) 01221: the firmware software version dismatched
    ```

    Which prevents you from flashing any software via Web UI

## Hard, most probably you end up here

### What will you need:

* Tftpserver software
  * For macosx I recommend Transfer, free trial is all you need
* 3.3v UART to USB dongle with some wires to connect it to the router
* Terminal
  * On macosx or linux a built-in terminal is all you need
  * On windows, I’ve heard that Putty can be used

### Begin the operation

#### Setup UART

* If you’ve already fiddled with UART, you can skip this part and jump right to the `BEGIN FIX` part below

* Open the case, connect the UART cable **ignore the VCC PIN, it MUST remain disconnected**

  ![UART]({{ site.url }}/assets/breaking-and-fixing-mr70x.assets/UART.jpg)

  * If the ethernet port does not start with the UART connected, *disconnect the VCC cable from UART !*

* Start terminal and connect to the UART serial

  * On MacOSX:

    * Find you UART device:
      `ls /dev/tty.*`
      One of the devices should contain `usbserial`, that sould be the one. Copy it’s path.

    * Connect to the serial:

      ```
      sudo cu -s 115200 -l /dev/tty.{your uart device}
      ```

  * Power cycle the router (unplug and plug the power adapter)

  * You should see in the terminal lines like the following:

    ```
    U-Boot 1.1.3 (Dec 21 2020 - 16:49:33)
    
    Board: Ralink APSoC DRAM:  128 MB
    relocate_code Pointer at: 87f58000
    ```

    * If you instead see the garbage characters, **baud rate is set wrong**. It must be `115200`
    * If you don’t see any characters, **Rx and Tx cables are probably connected other way around, invert them**

#### Begin the fix

##### Prepare the OEM sysupdate

* Download the latest OEW firmware from https://www.mercusys.com/en/download/mr70x/v1#Firmware
* Unzip to get the `.bin` file
* Go to https://argsnd.github.io/tp-link-stock-firmware-converter/index.html
* Upload the firmware `.bin` file to get the `sysupdate.bin` file

##### Prepare the initramfs image file

* Download the OpenWRT kernel initramfs .bin from https://downloads.openwrt.org/releases/23.05.3/targets/ramips/mt7621/openwrt-23.05.3-ramips-mt7621-mercusys_mr70x-v1-initramfs-kernel.bin
* Rename it to `test.bin`

* Start your tftp server in the directory whene `test.bin` is

  * In Transfer you can just drop the `test.bin` file into the Transfer window. It should appear on the file list.

* Open the terminal with the serial consone, power-cycle the router

* As soon as you see the new output in the terminal, press `t`

* You should end up seeing the CLI:

  ```
  4: System Enter Boot Command Line Interface.
  
  U-Boot 1.1.3 (Dec 21 2020 - 16:49:33)
  MT7621 #
  ```

  * If you instead see

    ```
     KSEG1ADDR(NetTxPacket) = 0xA7FE7180
    Trying Eth0 (10/100-M)
    
     Waitting for RX_DMA_BUSY status Start... done
    
    
     ETH_STATE_ACTIVE!!
    HTTP server is starting at IP: 192.168.1.1
    HTTP server is ready!
    ```

    Then you’ve pressed `t` too late
    or your terminal doesn’t send keystrokes properly
    or TX cable isn’t properly connected.

* Type `t` again and press `enter`. You should enter tftpboot mode.

  * You should see

    ```
    *** Warning: no boot file name; using 'test.bin'
    TFTP from server 192.168.1.5; our IP address is 192.168.1.1
    Filename 'test.bin'.
    
     TIMEOUT_COUNT=10,Load address: 0x84000000
    Loading: *
    ```

    * Note the file name, if it is different than `test.bin`, you should rename the file in tftp server to match
    * Note the server IP address, if it is different than `192.168.1.5`, you must change your PC static IP address settings so they would match

  * If download doesn’t start and you see `Retry count exceeded; starting again` message, your TFTP server is not running or cannot be reached, check IP, ethernet connection and tftp server

* After the download, you should see the message:

  ```
  Bytes transferred = 5744144 (57a610 hex)
  LoadAddr=84000000 NetBootFileXferSize= 0057a610
  MT7621 #
  ```

* Type `b` and press `enter`, you should be able to boot into the firmware

  * If you end up with

    ```
    No filesystem could mount root, tried:  squashfs
    Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(31,2)
    ```

    You’ve flashed the wrong file, you need to use `kernel` image  as `test.bin` in TFTP

##### Flashing the OEM sysupgrade

If you haven’t dont that already, begin by preparing the `sysupdate.bin` from OEM firmware, see **Prepare the OEM sysupdate** steps above.

* After the system starts (LED should stop blinking), you should be able to access `http://192.168.1.1/cgi-bin/luci/`

  * `root` / `root` are the default credentials

* Go to „Backup / Flash firmware”
  ![image-20240708135436074]({{ site.url }}/assets/breaking-and-fixing-mr70x.assets/image-20240708135436074.png)

* Flash the `sysupgrade.bin` that we’ve got by converting the OEM firmware

  * It is not the OpenWRT sysupgrade, it is the converted one from **Prepare the OEM sysupdate** steps !

  ![image-20240708125941104]({{ site.url }}/assets/breaking-and-fixing-mr70x.assets/image-20240708125941104-0436397.png)
  ![image-20240708130003530]({{ site.url }}/assets/breaking-and-fixing-mr70x.assets/image-20240708130003530.png)

  Uncheck „Keep settings”, check „Force upgrade” (ignore the warnings, this time we know what we do), and  click „Continue”![image-20240708130309322]({{ site.url }}/assets/breaking-and-fixing-mr70x.assets/image-20240708130309322.png)

* Wait for the completion. If the router reboots to OEM firmware, you are good to go, router is fixed.

  * If you end up with dreaded

    ```~
    [NM_Error](nm_api_checkInteger) 00331: factory boot check integer flag is not 1.
    ```

    keep reading.

#### Fixing the`factory boot check integer flag is not 1.`

**Important: First finish the Begin Fix steps, as we need MTD to be set up properly before OEM initramfs could be started from TFTP**

This error seems to be fixable ONLY by using OEM Web UI flash (unless you know which partition to backup and how and which bytes to edit. I couldn’t figure it out).

To do that, we are going to do the opposite of what we did just now: Use OEM initramfs to flash OpenWRT first.

* Get the OEM converted sysupdate file if you haven’t already (see Prepare the OEM sysupdate)

* Rename the `sysupdate.bin` to `test.bin` so we could boot from it using

* Boot from TFTP, follow the steps in Prepare the initramfs image file (but using `sysupdate.bin` as `test.bin` instead of kernel image)

  * If you end up with

    ```
    No filesystem could mount root, tried:  squashfs
    Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(31,2)
    ```

    You need to perform Begin Fix steps all over again, as something didn’t work as intended there.

* System should boot up to OEM firmware

* After the LED stops blinking, the Web UI should be available at `http://192.168.1.1`

  * **Important:** router could start with the configuration that was set-up before you even flashed OpenWRT, as those settings doesn’t seem to be cleared during the process.
    If the Web UI is not reachable, **press and hold the reset button** until the reouter restarts.
    Then you need to repeat the TFTP boot process again.

* Now perform flashing the OpenWRT factory image using Web UI

  * You can skip the initial setup in the upper-right corner by clicking „Skip setup” button

* Go to „Advanced” -> „System” -> „Firmware Update”
  ![image-20240708134443783]({{ site.url }}/assets/breaking-and-fixing-mr70x.assets/image-20240708134443783.png)

* Select and update the `factory` OpenWRT image file.

  * It must be the `factory` image, not the `initramfs` that we’ve used previously.
    Eg. `openwrt-23.05.3-ramips-mt7621-mercusys_mr70x-v1-squashfs-factory.bin`

  * If you see a nice

    ```
          vendorName : Mercusys
           vendorUrl : www.mercusys.com
         productName : MR70X
     productLanguage : US
           productId : 18000001
          productVer : ff010000
           specialId : 45550000
                hwId : BEA9246D14969065F575BC7D4814655B
               oemId : F9DCD93DFBF1273E4C14A4BAFBB84C8C
    ```

    In the UART console, then everything should be fine.

* Wait for the process to complete. Router should stop blinking after the system boots up.

* Go to `http://192.168.1.1`

  * If you end up with

    ```
    Not Found
    The requested URL /webpages/index.html?t=1e83e2a8 was not found on this server.
    ```

    Then you are refreshing the flash page, this will not work. You **need** to go directly to `http://192.168.1.1` page, not just refresh.

* Router is now booted from internal memory, integer flag should be properly set to `1`, we can now process with **proper** OEM firmware flash, instead of the mess that made us do all the debricking in the first place.

##### Proper OEM flash

Follow the steps in **Flashing the OEM sysupgrade**


### Caveats

* You won't be able to perform the OEM firmware update via Web UI without going to OpenWRT first.

You may have noticed a weird version number in OEM firmware:

```
0.0.0 Build 20240322 Rel. 23809(4555)
```

This is (probably?) a leftover from the OpenWRT firmware metadata, saved somewhere in the flash memory.

This breaks the OEM update system, resulting in the `01221: the firmware software version dismatched` error.

Until a method to fix the issue properly is found, you will have first flash OpenWRT, then convert new OEM firmware to sysupgrade, and flash it via OpenWRT's Web UI.


### Epilogue

If you’ve successfully unbricked your router, well done!
You’ve just saved a couple of bucks!

Instead of throwing money on a new router, you can buy me a coffee at https://ko-fi.com/hitorisensei
