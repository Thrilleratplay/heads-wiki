#Requirements
## Required equipment:

* Supported motherboard or laptop (see below)
* [Spi Programmer](https://trmm.net/SPI_flash): ch341a programmer or raspberry pi or bus pirate (ch341a is recommended for new users and can be found almost [anywhere](https://www.amazon.com/s?k=ch341a+programmer))
* Wires and a clip to connect your programmer of choice to the board’s SPI flash chip(s)
* If you plan to use a ch341a programmer you will need another computer to flash from (Try to use a recommended operating system: Qubes or Debian 9 or Fedora 30)
* USB flash drive

## Suggested equipment
* security keys
  * For HOTP or TOTP authentication:
    * [Librem Key](https://puri.sm/products/librem-key/)
    * [Nitrokey](https://www.nitrokey.com/) (
  * TOTP authentication only:
    * Yubikey
    * other security keys.

## Supported devices
|Device| Board name| Notes|
|--|--|--|
|Asus KGPE-D16|`kgpe-d16`||
|Dell R630|`r630`||
|Intel S2600wf|`s2600wf`||
|Lenovo Thinkpad T420|`t420`||
|Lenovo Thinkpad T430|`t430-flash`|flashed image|
|Lenovo Thinkpad T430|`t430`||
|Lenovo Thinkpad X220|`x220`||
|Lenovo Thinkpad X230|`x230-flash`|flashed image|
|Lenovo Thinkpad X230|`x230-hotp-verification`|with hotp verification|
|Lenovo Thinkpad X230|`x230`| |
|Open Compute Project	Leopard node|`leopard`||
|Open Compute Project	TiogaPass node|`tioga`||
|Open Compute Project	Winterfell node|`winterfell`||
|Purism Librem 13 v2|`librem_13v2`||
|Purism Librem 13 v4|`librem_13v4`||
|Purism Librem 15 v3|`librem_15v3`||
|Purism Librem 15 v4|`librem_15v4`||
|Purism Librem Mini|`librem_mini`||

## Emulated devices
|Device| Board name| Notes|
|--|--|--|
|QEMU development image|`qemu-coreboot-fbwhiptail`||
|QEMU development image|`qemu-coreboot`||
|QEMU development image|`qemu-linuxboot`||
