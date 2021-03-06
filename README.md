# Bluetooth fix for Lenovo A485 (and possibly others with RTL8822BE wifi)

## Steps

1. Download [`rtl8822befw.bin`](https://github.com/samcv/A485-RTL8822BE-firmware/raw/master/rtl8822befw.bin) and
[`rtl_bt-rtl8822b_fw.bin`](https://github.com/samcv/A485-RTL8822BE-firmware/raw/master/rtl_bt-rtl8822b_fw.bin)
2. Backup old wifi firmware ` sudo cp /lib64/firmware/rtlwifi/rtl8822befw.bin /lib64/firmware/rtlwifi/rtl8822befw.bin.bak`
3. Backup old bluetooth firmware `sudo cp /lib64/firmware/rtl_bt/rtl8822b_fw.bin /lib64/firmware/rtl_bt/rtl8822b_fw.bin.bak`
4. Copy new wifi firmware: `sudo cp rtl8822befw.bin /lib64/firmware/rtlwifi/rtl8822befw.bin`
5. copy new Bluetooth firmware: `sudo cp rtl_bt-rtl8822b_fw.bin /lib64/firmware/rtl_bt/rtl8822b_fw.bin`
6. Reboot (though putting it to sleep may work too).
* Note: firmware may be in `/lib/firmware` instead of `/lib64/firmware` on some systems

## Caveats
Bluetooth will start working the first time the laptop comes out of sleep. So if you restart your computer, you will
have to put it to sleep and wake it up. The bluetooth will then be fully functional. This is probably caused by a bug in the
kernel code which handles loading of the bluetooth firmware. Both wifi and bluetooth firmware must be loaded to the card
on startup and on resuming from sleep; for whatever reason the kernel doesn't try and load bluetooth firmware no boot but resume works fine.


## Where is this firmware from? How did you get it?

With a fair amount of effort I was able to extract the Lenovo provided firmware
for this wifi card.

How did I do this?
After much debugging trying to fix the bluetooth on Linux I decided to try another approach. I downloaded the Windows wifi driver from Lenovo
[`801373fb20311f5acc45476c7caeeb45b7fc39fb r0wrw05w.exe`](https://support.lenovo.com/us/en/downloads/ds504117)

After installing it you will find a binary blob located at `C:/DRIVERS/WLAN Driver/Source/rtwlane.sys` (sha1sum `eb63b4db367003382d3189e23f57798c1661815e`). The problem was to
find where in this large file the firmware was (where it started and where it ended).
I ended up spending a lot of time in a hex editor viewing this binary blob and the current
linux-firmware provided one. After identifying where I thought the start of the file was (at bytes `22 88 00`),
the next issue was where it ended. Not finding any real identifier at the end of the
linux-firmware version I looked to the Linux source code where the firmware is loaded.
You want `drivers/staging/rtlwifi/halmac/halmac_88xx/halmac_api_88xx.c` in function
halmac_download_firmware_88xx(). Here I was able to find the validation code that is used to validate
firmware prior to loading. Lucky for me there are 4 or so locations toward the start of the firmware
which after some operations are applied to it allow it to validate the firmware has the correct length.
By using the hex editor and finding these locations I was able to find the correct length of the firmware.

In case anybody doesn't trust the firmware I provide you can download Lenovo's firmware (check that the sha1sum is the
same) and do:

```
dd if=rtwlane.sys of=rtl8822befw.bin skip=4932176 bs=1 count=78320
```

Final result should have the following checksum:
`0017b27df0dcd9b944f40fce873487b80c3d9eb4  rtl8822befw.bin`

The Bluetooth firmware was far easier, once the Lenovo [driver download](https://pcsupport.lenovo.com/us/en/products/LAPTOPS-AND-NETBOOKS/THINKPAD-A-SERIES-LAPTOPS/THINKPAD-A485-TYPE-20MU-20MV/downloads/DS504118) is installed, the Bluetooth firmware is at `C:/DRIVERS/Bluetooth Driver/Source/rtl8822b_mp_chip_bt40_fw_asic_rom_patch_new.dll` with sha1sum a4288e0e7b3f818d50251e8115885ed35b11beef

What | offset | width | Description|Hex Data|Value
-----|--------|-------|-------------|-------|------
HALMAC_FWHDR_OFFSET_VERSION_88XX | 4 | 2 bytes | Major firmware version number|14|20
HALMAC_FWHDR_OFFSET_SUBVERSION_88XX | 6 | 1 byte | Minor firmware version number|04|4
HALMAC_FWHDR_OFFSET_SUBINDEX_88XX | 7 | 1 byte | Patch firmware version number|00|0
HALMAC_FWHDR_OFFSET_MEM_USAGE_88XX | 24 | 1 byte || 08 | 8
HALMAC_FWHDR_OFFSET_DMEM_SIZE_88XX | 36 | 4 bytes ||D8 0B 00 |3032
HALMAC_FWHDR_OFFSET_IRAM_SIZE_88XX |48 | 4 bytes| |C8 25 01|75208
HALMAC_FWHDR_OFFSET_ERAM_SIZE_88XX| 52|4 bytes| |00 00 00 | 0

Static Values:
* HALMAC_FW_CHKSUM_DUMMY_SIZE_88XX = 8
* HALMAC_FWHDR_SIZE_88XX = 64

Simplified

```
/* Simplified algorithm based on code in halmac_api_88xx.c */
dmem_pkt_size = DMEM_SIZE + HALMAC_FW_CHKSUM_DUMMY_SIZE_88XX;
iram_pkt_size = IRAM_SIZE + HALMAC_FW_CHKSUM_DUMMY_SIZE_88XX;
if ((MEM_USAGE & 4) != 0)
    eram_pkt_size = ERAM_SIZE;
if (eram_pkt_size != 0)
    eram_pkt_size += HALMAC_FW_CHKSUM_DUMMY_SIZE_88XX;
halmac_fw_size = (HALMAC_FWHDR_SIZE_88XX + dmem_pkt_size + iram_pkt_size + eram_pkt_size);
/* Now solving for halmac_fw_size by using our calculated values: */
78320          =            64           +   (3032+8)    + (75208+8)     + (0)

```

The Lenovo provided firmware shows this is version 20.4.0 based on the above values stored in those offsets. All values
are little endian by the way.

# Hardware info
USB shows up as:
`Bus 004 Device 008: ID 0bda:b023 Realtek Semiconductor Corp. `
Wifi shows up as:
`02:00.0 Network controller [0280]: Realtek Semiconductor Co., Ltd. RTL8822BE 802.11a/b/g/n/ac WiFi adapter [10ec:b822]`


# 18 October Lenovo Update:
On 18 October Lenovo published an update which also contains updated firmware,
updating it from version 20.4.0 to 27.2.0.

```
Version: 1B: 27
Subver: 02: 2
patch: 0
HALMAC_FWHDR_OFFSET_MEM_USAGE_88XX | 08 | 8
HALMAC_FWHDR_OFFSET_DMEM_SIZE_88XX | 58 2C 00 00 | 11352
HALMAC_FWHDR_OFFSET_IRAM_SIZE_88XX | 20 72 02 00 | 160288
HALMAC_FWHDR_OFFSET_ERAM_SIZE_88XX | 00 00 00 00 | 0

171720 = 64 + (11352 +8) + (160288+8) + 0
```


sha1sum of the 27.2.0 firmware is 25d2ba457bc1884e81b2579a249e960dbbc997c2
after extracting it from rtwlane.sys.
