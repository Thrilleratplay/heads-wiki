# Hardening


### TPM Disk encryption keys
The keys are currently derived only from the user passphrase, which is expanded via the LUKS expansion algorithm to increase the time to brute force it. For extra protection it is possible to store the keys in the TPM so that they will only be released if the PCRs match.

If you want to use the TPM to seal a secret used to unlock your LUKS volumes:

1. Enter recovery mode
2. Ensure that your the boot devices is mounted: `mount -o ro /dev/sda1 /boot` or whatever is appropriate
3. Insert your GPG card
4. Run `kexec-save-key -p /boot/ ...` with the followed by options appropriate to your OS.  The key will be installed in all devices in the LVM volume group as well as any other devices specified after the `-l` option.

Examples for the `kexec-save-key` parameters:

| Installation Type | Command |
| ---- | ---- |
| Previous Heads installation | `kexec-save-key -p /boot/ -l qubes_dom0` |
| Default Qubes / Default Fedora 25 | `kexec-save-key -p /boot/ /dev/sda2` |
| Default Ubuntu 16.04 / Debian 9 (\*) | `kexec-save-key -p /boot/ /dev/sda5` |

5. Reboot and you will be prompted for your boot password when that device is used to boot in the future.


NOTE: should the new LUKS headers be measured and the key re-sealed with those parameters? This is what the Qubes AEM setup uses and is probably a good idea (although we've already attested to the state of the firmware).

This is where things get messy right now. The key file can not persist on disk anywhere, since it would allow an adversary to decrypt the drive. Instead it is necessary to unseal/decrypt the key from the TPM and then bundle the key file into a RAM copy of Qubes' dom0 initrd on each boot. The initramfs format allows concatenated cpio files, so it is easy for the Heads firmware to inject files into the Qubes startup script.



### Generating your PGP key
If you're using a new Yubikey, you'll need to generate your key files. If you
already have the public key stubs for your Yubikey, please proceed
to the next section.  There is some more info in the [GPG guide](http://osresearch.net/GPG))

Insert your Yubikey into the x230, then invoke GPG's the "Card Edit"
function with it targetting the local directory:

```
gpg --homedir=/media/gnupg/ --card-edit
```

Go into "Admin" mode and generate a new key inside the Yubikey:

```
admin
generate
```

Since this key can be replaced by replacing the ROM, it is not necessary
to make a backup unless you want to.
This will prompt you for the admin pin (`12345678` by default) and then
the existing pin (`123456`).  Follow the other prompts and eventually
you should have a key in `/media/gnupg/`.

Create a single file containing the public key for this Yubikey (the secret key lives only in the Yubikey).

```
gpg --homedir=/media/gnupg/ --export -a > /media/gnupg/public.key
```

### Adding your PGP key
Heads uses your own GPG key to sign updates and as a result it needs the
key stored in the ROM image before flashing the full Heads ROM.

Add your key to the Heads ROM using the following command:

```
cbfs -o /media/x230.rom -a "heads/initrd/.gnupg/keys/public.key" -f /media/gnupg/public.key
```

Any name can be used as long as the it is preceded by `heads/initrd/.gnupg/keys/`.

After these files are added to the `/media/x230.rom`, you should flash the full ROM:

```
flashrom-x230.sh /media/x230.com
```

Once `flashrom` is complete, reboot (using the `reboot` command)
and now you should now be back in the Heads runtime. It should
display a message that is is unable to unseal TOTP.

Because the reproducible flash has an empty MRC cache, you need to
reboot one more time so that the PCR values as they would be going
forward.

## Configuring the TPM
There aren't very many good details on how to setup TPMs, so this section could use some work.

### Taking ownership
If you've acquired the machine from elsewhere, you'll need to establish physical presence, perform a force clear and take ownership with your own password. Should the storage root key (SRK) be set to something other than the well-known password?

```
tpm-reset
```

There is something weird with enabling, presence and disabling. Sometimes reboot fixes the state.

### tpmtotp

![TPMTOTP QR code](markdown/images/TPMTOTP_QR_code.jpg)

Once you own the TPM, run `seal-totp` to generate a random secret, seal it with the current TPM PCR values and store the sealed value in the TPM's NVRAM. This will generate a QR code that you can scan with your google authenticator application and use to validate that the boot block, rom stage and Linux payload are un-altered.

![TPMTOTP output](markdown/images/TPMTOTP_output.jpg)

On the next boot, or if you run `unseal-totp`, the script will extract the sealed blob from the NVRAM and the TPM will validate that the PCR values are as expected before it unseals it. If this works, the current TOTP will be computed and you can compare this one-time-password against the value that your phone generates.

This does not eliminate all firmware attacks (such as evil maid ones that replace the SPI flash chip), but when combined with the WP# pin and BP bits should eliminate a software only attack.


## Keys and passwords in Heads

There are "too many secrets" involved in booting a Heads system.  Luckily most of them are stored in hardware and only a few need to be memorized by the users.  This page attempts to document their usage and the risks if an attacker can compromise the different keys.

* Management Engine signing key
* Bootguard ACM fuses (hash of OEM public key)
* TPM Owner password (used to initialize counters, NVRAM spaces, etc)
* TPM Endorsement Key
* TPM counter key
* TPMTOTP secret (shared with phone authenticator)
* TPM disk encryption key (stored in the TPM, sealed with PCRs and encrypted with disk unlock key)
* Disk unlock key (entered by the user on every boot to unseal the TPM disk encryption key)
* Disk recovery key (created when the system is first installed, used rarely, if the TPM fails or if the PCRs change)
* LUKS disk encryption key (two copies, one encrypted with TPM disk encryption key, one encrypted with disk recovery key)
* User GPG private key (stored in a hardware token, not available to normal system)
* User GPG public key (stored in the ROM, used to validate xen, Linux dom0, etc)
* User login password
* Root password

### Management Engine and Bootguard ACM fuses
![Bootguard fuses](markdown/images/Bootguard_fuses.jpg)

The very first key used in the system is Intel's public key that signs the Management Engine firmware partition table in the SPI flash.  This key is stored in the on-die ROM of the ME and the ME will not start up if this signature does not match.  An attacker who controls this key (which is highly unlikely) can subvert the Bootguard checks as well as the measured boot process.

The [Bootguard fuses](https://trmm.net/Bootguard) fuses provide protection against most "evil maid" attacks against the firmware.  The hash of the ACM signing key is set in write-once fuses in the CPU chipset and during the CPU bringup phase the ME and the CPU microcode cooperate in some undocumented way to validate the "Startup ACM" in the SPI flash.  Since this key is fused into hardware, an evil maid attack would need to replace the CPU to install malicious firmware into the SPI flash.  The x230 Thinkpads do not support bootguard and only the Librem laptops ship with unfused keys.

An attacker who controls this key can flash new firmware via hardware means (and possibly remotely via software, unless other steps are taken).

### TPM Owner password
As part of setting up Heads the TPM is "owned" by the user and the owner password is set.  This clears all existing NVRAM and spaces (but does not reset counters?).

Are there any consequences of an attacker controlling this key?

### TPMTOTP shared secret
![TPMTOTP in use](markdown/images/TPMTOTP_in_use.jpg)

Since humans have trouble doing RSA public key cryptography in their brains, Heads uses [TPM TOTP](https://trmm.net/Tpmtotp) to let the system attest to the user that the firmware is unmodified.  During system setup a random 20-byte value is generated and shared (via QR code) to the user's phone as well as sealed with the correct TPM PCR values into the TPM NVRAM.  On subsequent boots the TPM will unseal the secret if and only if the PCRs match, and the computer then generates a one-time password based on the current clock time, which the user can compare to the value displayed on their phone.  A new secret must be generated each time the firmware is updated since this will change the PCRs.

If an attacker can control this shared secret (such as by directly sending PCR values into the TPM) they can install malicious firmware in the SPI flash and generate valid TOTP codes.

### TPM counter key
The TPM's increment-only counters can be used to prevent roll-back attacks on the signed kernel and initramfs configurations.  An attacker who controls this key can increment the counter, causing a denial of service attack against the system, but it does not provide any access to encrypted data nor any way to roll back to an old version.

### TPM disk encryption key
The TPM NVRAM also stores one of the disk encryption keys, which is encrypted with the user's disk unlock password and sealed with the TPM PCR values for the firmware and the LUKS headers on the disk.  On every boot the user types in their password and the TPM will unseal and decrypt the disk encryption key if and only if the firmware is unmodified and the password matches.  Since the TPMTOTP one-time code matched, the user can have confidence that the firmware is unmodified before they enter their password.  If the system is booted in recovery mode, the PCRs will not match and this key is not accessible to the user.

The Heads firmware inserts this key into the Qubes `initramfs.cpio` as `/secret.key`, which is listed in the `/etc/crypttab` file as the decryption key for the various partitions.  The dracut/systemd startup scripts will read the `/etc/crypttab` file and use it to decrypt the drives without further user intervention.

The sealed blob is not secret since it is both encrypted and sealed, but if an attacker can extract this unsealed and decrypted key they can decrypt the data on the disk.  If they extract the TPM they can set the PCRs to the correct values and attempt to brute force the unlock code, although the TPM should provide some rate limiting. TODO: Can the TPM also flush the keys if too many attempts are made?

### Disk recovery key
During initial system setup the disk is encrypted with a user chosen passphrase.  This key is only entered if the TPM PCRs have been changed, such as following a firmware update, or if the disk has been moved to a new machine and the user needs to back up the code.  If the system is booted into recovery mode, the TPM PCRs will not match the TPM sealed encryption key, so the user will need to enter the recovery key to decrypt the drives.

If an attacker gains control of this recovery key they can decrypt the disk to get access to the data, but not necessarily the system configuration if dm-verity is configured.  They can also add additional disk decryption keys, although this will be detected by the TPM measurement of the LUKS headers.

### LUKS disk encryption key
The TPM disk encryption and user's disk recovery keys are not the actual encryption keys for the disk; they are passed through [PBKDF2](https://en.wikipedia.org/wiki/PBKDF2) and the result is used to encrypt the actual disk encryption key, which is stored in the LUKS header on the disk.  Under normal circumstances this key is not visible to the user and is only an implementation detail.

Access to this key has the same risks as the disk recovery key.

### Owner's GPG key
![Yubikey](markdown/images/Yubikey.jpg)

The owner of the machine generates a GPG key pair as part of installing Heads.  The public key is inserted into the ROM image that is flashed and the owner signs the `/boot/boot.sh` script as well as the Xen hypervisor, the dom0 Linux kernel and initramfs, the TPM version counter of the system, and the dm-verity root hash if configured.  Ideally the private key does not live on the machine, but instead is in a Yubikey or other hardware token.

TODO: Can this be used in the disk decryption process?

An attacker who controls this private key can replace executables in `/boot` and if they also control the disk encryption key they can tamper with files in a dm-verity protected root filesystem.

### User login password
The user's login password is used to control access to the system once it has booted.

An attacker who controls this key can access the system and the decrypted disks if they gain physical access to the system while it is running or asleep.  This provides access to the data on the drive, but not necessarily the ability to modify a dm-verity protected root filesystem.

### Root password
The root password is not enabled by default on Qubes, so it is functionally equivalent to the login password.  Other operating systems might differ.

## TPM PCRs
![TPM](markdown/images/TPM.jpg)

The actual assignment needs to be updated in the code; there are outstanding issues (
[MRC cache](https://github.com/osresearch/heads/issues/150),
[SMM reloc](https://github.com/osresearch/heads/issues/13)
) that need to be resolved as well.  Until then this is a rough draft of how Heads uses the TPM PCRs.

0: Nothing for the moment

1: Nothing for the moment

2: Boot block, ROM stage, RAM stage, Heads linux kernel and initrd

3: Nothing for the moment

4: Boot mode (0 during `/init`, then `recovery` or `normal-boot`)

5: Heads Linux kernel modules

6: Drive LUKS headers

7: Heads user-specific config files



## GPG
### Generating a new card


NOTE: gpg 2.1.21 is required to create 4096 bits keys, which is not provided with heads for the moment.

```
diamond:~/clean/heads: gpg --card-edit --homedir ./initrd/.gnupg

gpg: detected reader `Yubico Yubikey NEO OTP+CCID 00 00'
Application ID ...: D2760001240102000006053147090000
Version ..........: 2.0
Manufacturer .....: unknown
Serial number ....: 05314709
Name of cardholder: [not set]
Language prefs ...: [not set]
Sex ..............: unspecified
URL of public key : [not set]
Login data .......: [not set]
Signature PIN ....: forced
Key attributes ...: 2048R 2048R 2048R
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 3 3
Signature counter : 0
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]
General key info..: [none]

gpg/card> admin
Admin commands are allowed

gpg/card> generate
Make off-card backup of encryption key? (Y/n) n

Please note that the factory settings of the PINs are
   PIN = `123456'     Admin PIN = `12345678'
You should change them using the command --change-pin

gpg: 3 Admin PIN attempts remaining before card is permanently locked

Please enter the Admin PIN

Please enter the PIN
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 1y
Key expires at Thu 12 Apr 2018 09:36:15 AM EDT
Is this correct? (y/N) y

You need a user ID to identify your key; the software constructs the user ID
from the Real Name, Comment and Email Address in this form:
    "Heinrich Heine (Der Dichter) <heinrichh@duesseldorf.de>"

Real name: Heads Firmware
Email address: heads@osresearch.net
Comment:
You selected this USER-ID:
    "Heads Firmware <heads@osresearch.net>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? o
gpg: generating new key
gpg: please wait while key is being generated ...
gpg: key generation completed (7 seconds)
gpg: signatures created so far: 0
gpg: generating new key
gpg: please wait while key is being generated ...
gpg: key generation completed (18 seconds)
gpg: signatures created so far: 1
gpg: signatures created so far: 2
gpg: generating new key
gpg: please wait while key is being generated ...
gpg: key generation completed (13 seconds)
gpg: signatures created so far: 3
gpg: signatures created so far: 4
gpg: key 9F7861CD marked as ultimately trusted
public and secret key created and signed.

gpg: checking the trustdb
gpg: 3 marginal(s) needed, 1 complete(s) needed, classic trust model
gpg: depth: 0  valid:   3  signed:   7  trust: 0-, 0q, 0n, 0m, 0f, 3u
gpg: depth: 1  valid:   7  signed:   2  trust: 5-, 0q, 0n, 0m, 2f, 0u
gpg: next trustdb check due at 2017-10-06
pub   2048R/9F7861CD 2017-04-12 [expires: 2018-04-12]
      Key fingerprint = 7679 00D2 E314 B920 91C2  ED88 7E28 A403 9F78 61CD
uid                  Heads Firmware <heads@osresearch.net>
sub   2048R/B4DB844D 2017-04-12 [expires: 2018-04-12]
sub   2048R/7E276412 2017-04-12 [expires: 2018-04-12]


gpg/card>
diamond:~/clean/heads: gpg --change-pin
gpg: detected reader `Yubico Yubikey NEO OTP+CCID 00 00'
gpg: OpenPGP card no. D2760001240102000006053147090000 detected

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? 3
gpg: 3 Admin PIN attempts remaining before card is permanently locked

Please enter the Admin PIN

New Admin PIN

New Admin PIN
PIN changed.     

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? 1

Please enter the PIN

New PIN

New PIN
PIN changed.     

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? q
```

### Fully resetting a card

If you mess up the admin pin it is necessary to do a full reset of
the PGP applet.  These instructions worked for me:
https://developers.yubico.com/ykneo-openpgp/ResetApplet.html
I had to install `scdaemon` and start `gpg-agent` to make them
work, but eventually that `gpg-connect-agent -r yubikey.reset`
script did the right thing:

```
/hex
scd serialno
scd apdu 00 20 00 81 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 81 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 81 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 81 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 83 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 83 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 83 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 83 08 40 40 40 40 40 40 40 40
scd apdu 00 e6 00 00
scd apdu 00 44 00 00
```
