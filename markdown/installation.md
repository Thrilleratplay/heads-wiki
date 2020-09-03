# Installing/Flashing

![Flashing Heads on an x230 at HOPE](markdown/images/Flashing_Heads_on_an_x230_at_HOPE.jpg)

### Installing Heads
These instructions are only for the Lenovo Thinkpad x230 and require physical access to the hardware. There are risks in installation that might brick your system and cause loss of data. You will need another computer to perform the flashing and building steps. If you want to experiment, consider [Emulating Heads](/Emulating-Heads) with qemu before installing it on your machine.

There are five major steps:
* Flashing the boot ROM
* Taking ownership of the TPM
* Installing Qubes
* Sealing disk encryption keys
* Signing Qubes installation

![Underside of the x230](markdown/images/Underside_of_the_x230.jpg)

Unplug the system and remove the battery while you're disassembling the machine! You'll need to remove the palm rest to get access to the SPI flash chips, which will require removing the keyboard. There are seven screws marked with keyboard and palm rest symbols.

![Keyboard tilted up](markdown/images/Keyboard_tilted_up.jpg)

The keyboard tilts up on a ribbon cable. You can keep the cable installed, unless you want to swap the keyboard for the nice x220 model.

![Ribbon cable](markdown/images/Ribbon_cable.jpg)

The palm rest trackpad ribbon cable needs to be disconnected. Flip up the retainer and pull the cable out. It shouldn't require much force. Once the palmrest is removed you can replace the keyboard screws and operate the machine without the palm rest. Since the thinkpad has the trackpoint, even mouse applications will still work fine.

![Flash chips](markdown/images/Flash_chips.jpg)

There are two SPI flash chips hiding under the black plastic, labelled "SPI1" and "SPI2". The top one is 4MB and contains the BIOS and reset vector. The bottom one is 8MB and has the [Intel Management Engine (ME)](https://www.flashrom.org/ME) firmware, plus the flash descriptor.


Using a chip clip and a [SPI programmer](https://trmm.net/SPI_flash), dump the existing ROMs to files. Dump them again and compare the different dumps to be sure that were no errors. Maybe dump them both a third time, just to be safe.

![Flashing x230 SPI flash](markdown/images/Flashing_x230_SPI_flash.jpg)

Ok, now comes the time to write the 4MB `build/x230-flash/x230-flash.rom` file to SPI2 chip. With my programmer and minicom, I hit i to verify that the flash chip signature is correctly read a few times, and then send `u0 400000`↵ to initiate the upload. I then drop to a shell with Control-A J and finally send the file with `pv x230.rom > /dev/ttyACM0`↵. A minute later, I resume minicom and hit i again to check that the chip is still responding.

Move the clip to the SPI1 chip. Read out the chip using `flashrom -r`, keep
a copy as backup and run `ifdtool -u` on it to enable writing to the flash
from software later. Also, [clean the ME firmware](Clean-the-ME-firmware).
This will wipe out the official Intel firmware, leaving only a stub of it to
bring up the Sandybridge CPU before shutting down the ME. As far as I can
tell there are no ill effects. Flash back your modified 8MB image. This
time you’ll send the command u0 800000↵. (you can also use the
[Skulls project](https://github.com/merge/skulls/tree/master/x230)'s
`external_install_bottom.sh -m` script to do all this work on the SPI1 chip
automatically).

Finally, remove the programmer, connect the power supply and try to reboot.

If all goes well, you should see the keyboard LED flash, and within a second the Heads recovery splash screen will appear. It currently drops you immediately into the shell, to allow you to flash the full 12MB `x230.bin` Heads rom (or `build/x230/coreboot.rom` if you've built it locally). If it doesn't work, well, sorry about that. Please let me know what the symptoms are or what happened during the flashing.

Congratulations! You now have a Coreboot + Heads Linux machine. Adding your own signing key, installing Qubes and configuring tpmtotp are the next steps.

On insert a USB drive containing the 12MB Heads rom and mount it using:

```
mount-usb
```

This will load the USB kernel modules and mount your drive at `/media`.

## External Flashing:

First remove the battery or cable powering your device. The Thinkpad x230 has two SPI flash chips that hold the BIOS, ME, etc. and are located under the palm rest. To access these chips, first remove the indicated screws on the back of the laptop.

Removing these screws will allow you to remove the keyboard and palm rest.

The keyboard is connected to the motherboard by a ribbon cable which easily detaches from the motherboard. (The keyboard only needs to be removed so that the palm rest can be removed. After removing the palm rest, you can put the keyboard back.)

The palm rest is also connected to the motherboard, but there is a little latch holding its ribbon cable. After undoing that latch, the palm rest should be fairly easy to remove.

Pull up the black plastic on the bottom right of the palm rest to reveal the two SPI flash chips.

The top chip is 4MB and contains the BIOS and reset vector. The bottom chip is 8MB and has the [Intel Management Engine (ME)](https://www.flashrom.org/ME) firmware, plus the flash descriptor.

Try to read the name on the top SPI flash chip. Then, connect the clip and ch341a programmer to the top SPI flash chip. Use flashrom to check the chip that you are connected to:

```
sudo flashrom -p ch341a_spi
```

Find the chip and read from it twice (For me the SPI flash chip is `YYY`):

```
sudo flashrom -r ~/top.bin --programmer ch341a_spi -c YYY && sudo flashrom -v ~/top.bin --programmer ch341a_spi -c YYY
```

If the files differ then try reconnecting your programmer to the SPI flash chip and make sure your flashrom software is up to date.

If they are the same then write `x230-flash.rom` to the SPI flash chip:

```
sudo flashrom -p ch341a_spi -c “YYY” -w ~/heads/build/x230-flash/x230-flash.rom
```

Try to read the name on the bottom SPI flash chip. Then, connect the clip and ch341a programmer to the bottom SPI flash chip. Use flashrom to check the chip that you are connected to:

```
sudo flashrom -p ch341a_spi
```

Find the chip and read from the chip twice (For me the SPI flash chip is `ZZZ`):

```
sudo flashrom -r ~/bottom.bin --programmer ch341a_spi -c ZZZ && sudo flashrom -v ~/bottom.bin --programmer ch341a_spi -c ZZZ
```

### Cleaning the ME:

If your reads match then you are good to move foreward with cleaning the ME firmware.

Clone the me_cleaner repo:

```
git clone https://github.com/corna/me_cleaner.git
```

Use me_cleaner to verify that the read from the SPI flash chip contains ME firmware:

```
python me_cleaner.py -c ~/bottom.bin
```

The output should be something like:

```
Full image detected
The ME/TXE region goes from 0x3000 to 0x4ff000
Found FPT header at 0x3010
Found 23 partition(s)
Found FTPR header: FTPR partition spans from 0x183000 to 0x24d000
ME/TXE firmware version 8.1.30.1350
Checking FTPR RSA signature... VALID
```

If the output is good then unlock the descriptor and ME regions with ifdtool:

```
~/heads/build/coreboot-4.8.1/util/ifdtool/ifdtool -u ~/bottom.bin
```

The resulting file will be called `bottom.bin.new` and should be located in the same directory as `bottom.bin`.

Remove all of the ME firmware that is not necessary to boot from the unlocked rom file:

```
python me_cleaner.py -r -t -d -S -O clean_flash.bin bottom.bin.new --extract-me extracted_me.rom
```

Flash `clean_flash.bin` to the SPI flash chip:

```
flashrom -p ch341a_spi -c “ZZZ” -w clean_flash.bin
```

### What is the Intel Management Engine
The Intel ME is a coprocessor, running inside your Intel CPU, which is supposed to function as a out-of-band management system for your computer.  
It's based on a ARC/i386 core depending of the version, can be powered even when the main CPU is off and has more privileges than the CPU itself.  
More info: [Intel Active Management Technology](https://en.wikipedia.org/wiki/Intel_Active_Management_Technology)

### How to disable/deactive most of it
The ME firmware sits on the second SPI flash chip of the x230 (the 8MB one). We cannot remove it completely, otherwise the machine will shut itself off after 30 minutes. We can, however, reduce it to the bare minimum necessary to keep it running, but without any malicious code in it (or so we hope, depending of what the ROMP and BUP modules really do...).

The initial step is to upgrade the proprietary BIOS to the last upgradeable version one for each platform.
As an example, for the x230, the latest upgradeable version would be [version 2.76](https://download.lenovo.com/pccbbs/mobiles/g2uj32us.iso) without [EC signature verification](https://support.lenovo.com/us/en/solutions/len-27764). Newer firmware version [won't permit to swap a x220 keyboard on the x230](https://github.com/hamishcoleman/thinkpad-ec/pull/130).  

Prepare a USB bootable disk by following [el torito instructions](https://askubuntu.com/questions/651281/write-bootable-bios-update-iso-to-usb-stick), then boot that prepared USB disk and upgrade the prioprietary firmware to latest available version following on screen instructions. Be sure to have a fully charged battery, be connected to power source prior of attempting to upgrade, else you will have to wait for the battery to be changed.

Once the proprietary firmware is updated to the latest available user ownable version, take a flash dump of the bottom SPI chip and verify that its backup is valid.

Flashrom can be downloaded on most linux distribution on the external laptop that will be used to flash the cleaned rom. (`sudo dnf install flashrom` or `sudo apt-get install flashrom`.

With a ch341a programmer, the command would look like the following:
`sudo flashrom -r ~/down.rom --programmer ch341a_spi && sudo flashrom -v ~/down.rom --programmer ch341a_spi`

If they match, clone this repo:  
`https://github.com/corna/me_cleaner.git`  
and then run:  
`python ~/me_cleaner/me_cleaner.py -c flash.bin`  
to check if it's a valid ME firmware.  
The output should be like:  

```
Full image detected
The ME/TXE region goes from 0x3000 to 0x4ff000
Found FPT header at 0x3010
Found 23 partition(s)
Found FTPR header: FTPR partition spans from 0x183000 to 0x24d000
ME/TXE firmware version 8.1.30.1350
Checking FTPR RSA signature... VALID
```

If it's not, replace the reprogrammer clip and try again :)  

Next, unlock the descriptor and ME regions with ifdtool. We consider here that you already build Heads through `make BOARD=x230`:
`~/heads/build/coreboot-4.8.1/util/ifdtool/ifdtool -u down.rom`
This produced a new unlocked rom under `down.rom.new`

Next, let's strip all the nasty bits:  

`python ~/me_cleaner/me_cleaner.py -r -t -d -S -O clean_flash.bin down.rom.new --extract-me extracted_me.rom`

Output:  

```
Full image detected
The ME/TXE region goes from 0x3000 to 0x4ff000
Found FPT header at 0x3010
Found 23 partition(s)
Found FTPR header: FTPR partition spans from 0x183000 to 0x24d000
ME/TXE firmware version 8.1.30.1350
Removing extra partitions...
Removing extra partition entries in FPT...
Removing EFFS presence flag...
Removing ME/TXE R/W access to the other flash regions...
Correcting checksum (0x7b)...
Reading FTPR modules list...
 UPDATE           (LZMA   , 0x1cf4f2 - 0x1cf6b0): removed
 ROMP             (Huffman, fragmented data    ): NOT removed, essential
 BUP              (Huffman, fragmented data    ): NOT removed, essential
 KERNEL           (Huffman, fragmented data    ): removed
 POLICY           (Huffman, fragmented data    ): removed
 HOSTCOMM         (LZMA   , 0x1cf6b0 - 0x1d648b): removed
 RSA              (LZMA   , 0x1d648b - 0x1db6e0): removed
 CLS              (LZMA   , 0x1db6e0 - 0x1e0e71): removed
 TDT              (LZMA   , 0x1e0e71 - 0x1e7556): removed
 FTCS             (Huffman, fragmented data    ): removed
 ClsPriv          (LZMA   , 0x1e7556 - 0x1e7937): removed
 SESSMGR          (LZMA   , 0x1e7937 - 0x1f6240): removed
Relocating FTPR from 0xd00 - 0xcad00 to 0xd00 - 0xcad00...
 Adjusting FPT entry...
 Adjusting LUT start offset...
 Adjusting Huffman start offset...
 Adjusting chunks offsets...
 Moving data...
The ME minimum size should be 98304 bytes (0x18000 bytes)
The ME region can be reduced up to:
 00003000:0001afff me
Setting the AltMeDisable bit in PCHSTRP10 to disable Intel ME...
Removing ME/TXE R/W access to the other flash regions...
Extracting and truncating the ME image to "extracted_me.rom"...
Checking the FTPR RSA signature of the extracted ME image... VALID
Checking the FTPR RSA signature... VALID
Done! Good luck!
```

After that, you got your new, cleaned up version of the ME firmware inside clean_flash.bin  
Flash it back on the SPI:

`sudo flashrom -w clean_flash.bin --programmer ch341a_spi`

You're now good to go :)
