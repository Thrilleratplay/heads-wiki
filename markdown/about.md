![Heads booting on an x230](markdown/images/Heads_booting_on_an_x230.jpg)


# Overview/FAQ

## What is Heads?
Heads is an open source custom firmware and OS configuration for laptops
and servers that aims to provide slightly better physical security and
protection for data on the system.

Heads is not just another Linux distribution -- it combines physical
hardening of specific hardware platforms and flash security features with
custom coreboot firmware and a Linux boot loader in ROM.  This moves
the root of trust into the write-protected region of the SPI flash and
prevents further software modifications to the bootup code (and on
platforms that support it, [Bootguard](https://trmm.net/Bootguard) can
protect against many hardware attacks as well).  Controlling the
first instruction the CPU executes allows Heads to measure every step of
the boot firmware and configuration into the TPM, which makes it possible
to attest to the user or a remote system that the machine has not been
tampered with. While modern Intel CPUs require binary blobs to boot,
these non-Free components are included in the measurements and are at least
guaranteed to be unchanging.  Once the system is in a known good state, the
TPM is used as a hardware key storage to decrypt the drive.

![TPM TOTP in action](markdown/images/TPM_TOTP_in_action.jpg)

Additionally, the hypervisor, kernel and initrd images are signed by
keys controlled by the user, and the OS uses a signed, immutable root
filesystem so that any software exploits that attempt to gain persistence
will be detected.  While all of these firmware and software changes don't
secure the system against every possible attack vector, they address
several classes of attacks against the boot process and physical hardware
that have been neglected in traditional installations, hopefully raising
the difficulty beyond what most attackers are willing to spend.

**Installing Heads requires opening the machine and extensive warranty voiding.**

 ![Flashing an x230 bootrom](markdown/images/Flashing_an_x230_bootrom.jpg)


## Frequently Asked Questions

, so this document tries to answer some of the questions about it.
### Why replace UEFI with coreboot?

While Intel's edk2 tree that is the base of UEFI firmware is open source, the firmware that vendors install on their machines is proprietary and closed source. Updates for bugs fixes or security vulnerabilities are at the vendor's convenience; user specific enhancements are likely not possible; and the code is not auditable.

UEFI is much more complex than the BIOS that it replaced. It consists of millions of lines of code and is an entire operating system, with network device drivers, graphics, USB, TCP, https, etc, etc, etc. All of these features represents increased "surface area" for attacks, as well as unnecessary complexity in the boot process.

coreboot is open source and focuses on just the code necessary to bring the system up from reset. This minimal code base has a much smaller surface area and is possible to audit. Additionally, self-help is possible if custom features are required or if a security vulnerability needs to be patched.

### What's wrong with UEFI Secure Boot?
Can't audit it, signing keys are controlled by vendors, doesn't handle hand off in all cases, depends on possible leaked keys.

### Why use Linux instead of vboot2?
vboot2 is part of the coreboot tree and is used by Google in the Chromebook system to provide boot time security by verifying the hashes on the coreboot payload. This works well for the specialized Chrome OS on the Chromebook, but is not as flexible as a measured boot solution.
By moving the verification into the boot scripts we're able to have a much flexible verification system and use more common tools like PGP to sign firmware stages.

### What about Trusted GRUB?
The mainline grub doesn't have support for TPM and signed kernels, but there is a Trusted grub fork that does. Due to philosophical differences the code might not be merged into the mainline. And due to problems with secure boot (which Trusted Grub builds on), many distributions have signed insecure kernels that bypass all of the protections secure boot promised.
Additionally, grub is closer to UEFI in that it must have device drivers for all the different boot devices, as well as filesystems. This duplicates the code that exists in the Linux kernel and has its own attack surface.

Using coreboot and Linux as a boot loader allows us to restrict the signature validation to keys that we control. We also have one code base for the device drivers in the Linux-as-a-boot-loader as well as Linux in the operating system.

### What is the concern with the Intel Management Engine?
"Rootkit in your chipset", "x86 considered harmful", etc

### How about the other embedded devices in the system?
goodbios, funtenna, etc.

### Should we be concerned about the binary blobs?
Maybe. x230 has very few (MRC) since it has native vga init.

### Why use ancient Thinkpads instead of modern Macbooks?
The x230 Thinkpad has coreboot support, TPM, nice keyboards and are very cheap to experiment on. If you're willing to spend a bit more, the Chell Chromebooks (commericaly available as the HP Chromebook 13 G1) has Skylake.

### How likely are physical presence attacks vs remote software attacks?
Who knows.

### Defense in depth vs single layers
Yes.

### is it worth doing the hardware modifications?
Depends on your threat model.

### Should I validate the TPMTOTP on every boot?
Probably. I want to make it also do it at S3.

### suspend vs shutdown?
S3 is subject to cold boot attacks, although they are harder to pull off on a Heads system since the boot devices are constrained.

However, without tpmtotp in s3 it is hard to know if the system is in a safe state when the xscreensaver lock screen comes up. Is it a fake to deceive you and steal your login password? Maybe! It wouldn't get your disk password, which is perhaps an improvement.

### Disk key in TPM or user passphrase?
Depends on your threat model. With the disk key in the TPM an attacker would need to have the entire machine (or a backdoor in the TPM) to get the key and their attempts to unlock it can be rate limited by the TPM hardware.
However, this ties the disk to that one machine (without having to recover and type in the master key), which might be an unacceptable risk for some users.

### Why is it called Heads?
*The flip side of [Tails](https://tails.boum.org/).*

Unlike [Tails](https://tails.boum.org/), which aims to be a stateless OS that
leaves no trace on the computer of its  presence, Heads is intended for the
case where you need to store data and state on the computer.

### How does Heads improve security?
* [Heads threat model](https://trmm.net/Heads_threat_model) goes into more detail about what classes of threats Heads attempts to counter.
![33c3](markdown/images/33c3.jpg)
* [Presentation at 33c3](https://trmm.net/Heads_33c3)

### What is Heads based on?
Heads is influenced by several years of firmware vulnerability research:
* ([Thunderstrike](https://trmm.net/Thunderstrike) and [Thunderstrike 2](https://trmm.net/Thunderstrike_2))
* ("[Hardening hardware and choosing a #goodBIOS](https://media.ccc.de/v/30C3_-_5529_-_en_-_saal_2_-_201312271830_-_hardening_hardware_and_choosing_a_goodbios_-_peter_stuge#t=2372)" by Peter Stuge,
* "[Beyond anti evil maid](https://media.ccc.de/v/32c3-7343-beyond_anti_evil_maid)" by Matthew Garret,
* "[Towards (reasonably) trustworthy x86 laptops](http://www.theregister.co.uk/2015/12/31/rutkowska_talks_on_intel_x86_security_issues/)"
by Joanna Rutkowska,
* "[LightEater malware seek GPG keys in Tails](http://www.theregister.co.uk/2015/03/19/cansecwest_talk_bioses_hack/)"
by Kallenberg and Kovah, etc.).


## Community
### Slack Open Source Firmware
* [Register to Open Source Firmware](https://slack.osfw.dev/)
* [Head's channel](https://osfw.slack.com/archives/C92MNSRC1)
