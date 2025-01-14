# Cross-Platform Spotify Car Thing (superbird) hacking toolkit

This toolkit re-implements most of the functionality from [frederic's scripts](https://github.com/frederic/superbird-bulkcmd).
The key difference here is that this tool uses [`pyamlboot`](https://github.com/superna9999/pyamlboot) instead of the proprietary `update` binary from Amlogic, 
which allows it to work on many more platforms!

Everything in [`images/`](images/) came directly from [frederic's repo](https://github.com/frederic/superbird-bulkcmd).

The purpose of this tool is to provide useful, working examples for how to use `pyamlboot` to perform development-related tasks on the Spotify Car Thing.

Contributions are welcome. This code is unlicensed: you can do whatever you want with it.
 `pyamlboot` is Apache-2.0, `libusb` is LGPL-2.1

A [Changelog can be found here](Changelog.md)

## Warranty and Liability

None. You definitely can mess up your device in ways that are difficult to recover. I cannot promise a bug in this script will not brick your device.
By using this tool, you accept responsibility for the outcome. 

I highly recommend connecting to the UART console, [frederic's repo](https://github.com/frederic/superbird-bulkcmd) has some good pictures showing where the pads are.

Make backups.

## One Big Caveat
This tool tries to replace the proprietary `update` binary from Amlogic, and it covers enough functionality to be useful for superbird.
However, dumping partitions is MUCH slower.

The original tool from Amlogic manages to read directly from the mmc, without having to first read it into memory, 
so it is a lot faster at about `12MB/s` or about 7 minutes to dump all partitions.
Unfortunately, we cannot currently replicate this method using `pyamlboot`.

Instead, to dump partitions we first have to tell the device to read a chunk (128KB) into memory, and then we can read it from memory out to a file, one chunk at a time.
The copy rate for reading is about `545KB/s`, and in my testing on Ubuntu x86_64 it takes about 110 minutes to dump all partitions!

The same thing must be done in reverse to restore a partition, but writing is much faster, and we can use larger chunks (512KB), 
so copy rate for writing is about `5.1MB/s`, and it takes about 11 minutes to write all partitions. If the `data` and `settings` partitions are omitted,
it takes about 4 minutes to write.


## Supported Platforms

The only requirements to run this are:
1. python3
2. libusb
3. pyamlboot from [github master branch](https://github.com/superna9999/pyamlboot)

You need to install pyamlboot from [github master branch](https://github.com/superna9999/pyamlboot) because the current pypy package is too old,
and is missing `bulkcmd` functionality.

### macOS
Tested on `aarch64` and `x86_64`

On macOS, you must install `python3` and `libusb` from homebrew, and execute using that version of python
```bash
brew install python3 libusb
/opt/homebrew/bin/python3 -m pip install git+https://github.com/superna9999/pyamlboot
/opt/homebrew/bin/python3 superbird_tool.py --find_device
```
`root` is not needed on macOS

### Linux
Tested on `aarch64` and `x86_64`

On Linux, you just need to install pyamlboot.
However, `root` is needed on Linux, unless you fiddle with udev rules, which means the pip package also needs to be installed as `root`
```bash
sudo python3 -m pip install git+https://github.com/superna9999/pyamlboot
sudo ./superbird_tool.py --find_device
```

### Windows

Tested on `x86_64`, but it seems really difficult to get this working consistently on Windows. I recommend Linux or macOS.

On Windows, setup is a little more involved. First download and install [python for windows](https://www.python.org/downloads/windows/) (tested with 3.10 and 3.11).
Next you need to install a couple extra packages:
```bash
python -m pip install git+https://github.com/superna9999/pyamlboot
python -m pip install pyusb
python -m pip install libusb
```

You also need to add to add to your PATH: 

```bat
rem NOTE: Python310 for python 3.10.x
%LOCALAPPDATA%\Programs\Python\Python310\Lib\site-packages\libusb\_platform\_windows\x64
```
This [stackoverflow post](https://stackoverflow.com/questions/44272416/how-to-add-a-folder-to-path-environment-variable-in-windows-10-with-screensho)
has a good visual explanation of how to edit PATH in Windows 10

After doing the above, you'll need to install the correct driver. Start off by downloading [Zadig](https://zadig.akeo.ie/).
- Once you have it downloaded, open it then put your Car Thing into USB Burn Mode by holding the preset 1 & 4 buttons while plugging it in. The screen should stay black.
- In Zadig you should see a `GX-CHIP` device appear. When it shows up, click `Edit` then configure Zadig so the options are like this:
![image](https://github.com/user-attachments/assets/69da7f52-876e-4172-b4fe-dd2732c3c6ee)
- Click `Install Driver` and wait for it to finish.

Finally, you should be able to run the tool
```bash
python superbird_tool.py --find_device
```

Confirm things actually work by connecting device in USB Mode (hold buttons 1 & 4 while connecting), and then entering USB Burn Mode:
```bash
python superbird_tool.py --burn_mode
```

## Usage

```
General:
  -h, --help            Show this help message and exit
  --find_device         Find superbird device and show its current boot mode
  --burn_mode           Enter USB Burn Mode (if currently in USB Mode)
  --continue_boot       Continue booting normally (if currently in USB Burn Mode)

Booting:
  --boot_adb_kernel BOOT_SLOT
                        Boot a kernel with adb enabled on chosen slot (A or B)(not persistent)
  --disable_avb2 BOOT_SLOT
                        Disable A/B booting, lock to chosen slot(A or B)
  --enable_burn_mode    Enable USB Burn Mode at every boot (when connected to USB host)
  --enable_burn_mode_button
                        Enable USB Burn Mode if preset button 4 is held while booting (when connected to USB host)
  --disable_burn_mode   Disable USB Burn Mode
  --disable_charger_check
                        Disable check for valid charger at boot
  --enable_charger_check
                        Enable check for valid charger at boot

Restoring:
  --restore_device INPUT_FOLDER
                        Restore all partitions from a folder
  --restore_partition PARTITION_NAME INPUT_FILE
                        Restore a partition from a dump file
  --dont_reset          Don't factory reset when restoring device. Use in combination with restore commands.
  --slow_burn           Use a slower burning speed. Use this if restoring crashes mid-flash.
  --slower_burn         Use an even slower burning speed. Use this if --slow_burn doesn't work.

Dumping:
  --dump_device OUTPUT_FOLDER
                        Dump all partitions to a folder
  --dump_partition PARTITION_NAME OUTPUT_FILE
                        Dump a partition to a file

U-Boot Enviroment:
  --get_env ENV_TXT     Dump device env partition, and convert it to env.txt format
  --send_env ENV_TXT    Import contents of given env.txt file (without wiping)
  --send_full_env ENV_TXT
                        Wipe env, then import contents of given env.txt file
  --restore_stock_env   Wipe env, then restore default env values from stock_env.txt
  --convert_env_dump ENV_DUMP OUTPUT_TXT
                        Convert a local dump of env partition into text format
Advanced:
  --bulkcmd COMMAND     Run a uboot command on the device
  --enable_uart_shell   Enable UART shell

```

## Boot Modes
There are four possible boot modes

### USB Mode
This is what you get if you hold buttons 1 & 4 while plugging in the device.

The UART console will print: 
```
G12A:BL:0253b8:61aa2d;FEAT:F0F821B0:12020;POC:D;RCY:0;USB:0;
```

In this mode, the device shows up on USB as: `1b8e:c003 Amlogic, Inc. GX-CHIP`

### USB Burn Mode
This is a special uboot image, which we can interact with via usb.

The UART console output will typicaly end with:
```
U-Boot 2015.01 (Jan 21 2022 - 08:55:34 - v1.0-57-gec3ec936c2)

DRAM:  512 MiB
Relocation Offset is: 16e42000
InUsbBurn
[MSG]sof
Set Addr 11
Get DT cfg
Get DT cfg
set CFG
``` 
Which indicates it is ready to receive commands

In this mode, the device shows up on USB as: `1b8e:c003 Amlogic, Inc.`

### Normal Bootup
If USB Burn mode is not enabled at every boot, or if you use `--continue_boot`, the device will boot up normally and launch the Spotify app.

In this mode, the device does not show up on USB.

### Normal Bootup with USB Gadget
If you use `--boot_adb_kernel`, a modified kernel and image will be uploaded to the device (non-persistent), which enables USB Gadget.

The USB Gadget can be configured to provide `adb` (like an Android phone), among other possible functionality including `rndis` for usb networking.

In this mode, the device shows up on USB as: `18d1:4e40 Google Inc. Nexus 7 (fastboot)`

Please do NOT try to use fastboot with superbird device, there is potential to brick it.

## Persistent USB Gadget with USB Networking

We have provided a script to set up ADB and USB Networking. You can find it, along with documentation in the [scripts/usb-gadget](scripts/usb-gadget) folder.

This is a heavily modified version of what [frederic provided](https://github.com/frederic/superbird-bulkcmd/blob/main/scripts/enable-adb.sh.client)

## Example Usage

As an example (on Linux), here are steps to enable persistent adb and usbnet, disable a/b booting, and disable charger check, on a fresh device.

```
# starting from a fresh device

# plug in with buttons 1 & 4 held
sudo ./superbird_tool.py --find_device  # check that it is in usb mode
sudo ./superbird_tool.py --burn_mode
sudo ./superbird_tool.py --enable_burn_mode_button
sudo ./superbird_tool.py --disable_avb2 a # disable A/B, lock to A
sudo ./superbird_tool.py --disable_charger_check

# unplug and replug while holding button 4

sudo ./superbird_tool.py --find_device   # check that it is in usb burn mode
sudo ./superbird_tool.py --boot_adb_kernel a

# device boots to spotify logo, but app does not launch

adb devices  # check that your device shows up in adb

# setup persistent USB Gadget (adb and usbnet)
cd scripts/usb-gadget
./push_usbgadget.sh

# unplug and replug without holding any buttons
#   it should boot normally (app should launch), now with adb and usbnet enabled

ip addr  # you should see usb0 listed
```

## Known Issues
* Sometimes flashing can fail mid flash, especially while flashing bigger partitions like `system`. If this happens, try opening `superbird_device.py` in a text editor, go to line 161, and change the `MULTIPlLIER` parameter from `8` to `4`. If problems persist, change it to `1`. This will be a command flag in the future.
* Multiple people have reported issues with trying to use superbird-tool on AMD systems, specifically 5000 series systems. Sometimes a BIOS update can fix this issue but you may just need to use another computer.
* The option `--enable_uart_shell` is really only meant to be run on a fresh device. It will rewrite `initargs` env var, removing any other changes you made like using a particular system partition every boot.
* The option `--disable_avb2` will ALSO enable the uart shell; consider using that instead.
* If you boot from USB mode into burn mode (using `--burn_mode`), `--boot_adb_kernel` won't work. This is due to u-boot not setting up some parts of the hardware.
* In some cases you might get a Timeout Error. This happens sometimes if a previous command failed, and you just need to power cycle the device (actually unplug and plug it back in), and try again. 
  * ALSO, avoid connecting the device through a USB hub. In my testing, I had many more timeout issues when using a hub.
  * You might need to power cycle and try again multiple times

## Making Standalone Binaries

I have provided a (very barebones) script to generate a standalone `superbird_tool` binary using `nuitka`.

You need to install `nuitka`, `zstandard` and `ordered-set` packages from pip to use it.

On Linux, you also need to install `patchelf` from your system package manager. ex: `sudo apt-get install -y patchelf`

I have not tested this much yet, just a neat idea for now.

Compilied binaries include `images/` so they should work fine standalone.

# Disclaimer
"Spotify", "Car Thing" and the Spotify logo are registered trademarks or trademarks of Spotify AB. Thing Labs is not affiliated with, endorsed by, or sponsored by Spotify AB. All other trademarks, service marks, and trade names are the property of their respective owners.
