# tpm-luks

## What is this about?
Have you ever wondered how does Windows Bitlocker-protected computers start without entering password during boot? While this is only one of the possible Bitlocker configurations (and certainly not the most secure one), it is very user, friendlthe and provides certain level of security. (if the storage root key and computer state in PCRs match the expectation defineds during the sealing process) or unsealing fails (if a system state has been tampered with). When the unsealing process fails, your 

Windows uses the so-called Trusted Platform Module (TPM) to provide Bitlocker encryption without entering password on boot.
As the Bitlocker component is configured when Windows is up and running, when the TPM is present in the system, it is initialized and desired state of boot is precomputed by the operating system. Bitlocker secret is then bound (sealed) to this precomputed state and stored on the boot drive (a TPM storage root key is used in this process; TPM itself is the component which does the sealing).

During boot, each component which is part of the boot process gets measured (hashed) and the measurement is then extended into TPM Platform Control Registers (PCRs). This includes your BIOS/UEFI code, some data (such as current state of your partitions in MBR or GPT), operating system boot loader, kernel and its start-up parameters (such as "is safe mode enabled?"). Based on these measurements, the Windows secret is either unsealed by the TPM (if the storage root key and computer state in PCRs match the expectations defined during the sealing process) or unsealing fails (if a system state has been tampered with). When the unsealing process fails, Windows prompt user for the Bitlocker recovery password.

In Linux, the support of TPM has existed for years in kernel, but there is very limited support in bootloaders and toolchain to allow similar set-up for LUKS-encrypted root filesystem drives. There are the following projects regarding Measured boot and LUKS-TPM in Linux:
* [TrustedGrub](https://projects.sirrix.com/trac/trustedgrub/) - fails to compile on up-to-date distros
* [TrustedGrub2](https://github.com/Rohde-Schwarz-Cybersecurity/TrustedGRUB2) - has to be compiled manually, prints errors on boot, lacks support for UEFI and does not measure kernel commandline separately (only as part of commands executed from grub.cfg), halts the system on TPM absence
* [Grub2 with TPM support by Matthew Garrett](https://github.com/mjg59/grub) - has support for TPM1.2 and TPM 2.0, UEFI, but does not fully fit the Ubuntu environment (PCR[10] is used for IMA in Ubuntu, therefore cannot be used for tpm-luks without complex work-arounds; mixes PCRs for different uses and makes PCR precomputation difficult)
* [tpm-luks by Kent Yoder](https://github.com/shpedoikal/tpm-luks) - expects boot via TrustedGRUB, lack support of Grub2, supports initramfs generation for old Fedoras only
* [shim with TPM support by Matthew Garrett](https://github.com/mjg59/shim/commits/tpm) - introduces support for TPM into the shim bootloader.
 
While there were multitude options to choose from, none of them matched the feature set needed for Ubuntu support, namely:
* be predictable, so that the PCR state can be precomputed programatically
* be compilable by Launchpad on latest Ubuntu distributions
* support Grub2 as much as possible
* on initrd update and/or kernel installation, automagically precompute and store LUKS secret into new slot of NVRAM to facilitate password-less boot with the newly installed components
* provide support for UEFI-based boot when SHIM is used as well as when SHIM is not used (Grub is loaded by UEFI directly)
 
As a result, the tools to facilitate the feature set above are included in this repository, together with the [Grub2-tpm for Ubuntu](https://github.com/zajdee/ubuntu1704-grub-tpm) repository. These repositories are accompanied by a [LUKS-TPM Launchpad PPA](https://launchpad.net/~radek-zajic/+archive/ubuntu/measuredluks), where the precompiled packages are available for easy installation.

## How does it work
After an initial configuration, a random 32 byte (256 bit) key is generated by `tpm-luks-init`. System is measured by `tpm-luks-gen-tgrub-pcr-values` and PCR values are precomputed (an extra tool called pcrsum has been created to enhance UEFI based measurements). The key is stored in TPM together with the precomputed PCR values. The key is then added to a free LUKS key slot.

Each time a new **active** initramdisk is generated by system, a `/etc/initramfs/post-update.d/tpm-luks-update` hook is executed, which (after some additional checks) executes `tpm-luks-update`. `tpm-luks-update` then precomputes PCR values for the new system state and migrates the current LUKS key to a new NVRAM index in TPM. This allows for easy kernel updates without manual intervention.
**Note:** active initramdisk is the initramdisk to which a default Grub entry points at the moment of executing `tpm-luks-update`.

If you modify your bootloader (e.g. reinstall grub or change its configuration), tamper with your kernel image, cmdline or initramdisk manually, you must run `tpm-luks-update` manually as well to perform the PCR precomputation and NVRAM data migration. Otherwise you lose your TPM-stored key after a reboot (unless you can reboot back to a state which is expected by the TPM and recover the key manually).

If you lose your TPM-stored key, it is easy to kill the key from LUKS header (`cryptsetup luksKillSlot X` where `X` is the position of your TPM-stored key) and then re-run `tpm-luks-init`.

## Constraints
* You cannot use this toolset with Secure boot, unless you sign your bootloader with your own keys. Steps to do that are not covered in this guide.
* Only TPM 1.2 is supported as of now, sorry. (This is due to the lack of TPM 2.0 systems availability ahead of the initial release.)
* Ubuntu 16.10 or 17.04 is required. Support for 16.04 LTS is planned.
* Only amd64 releases have been tested, there is no guarrantee that the tools will work on i386.
* Both BIOS and UEFI-based boot are supported and tested.
* Support for "Bitlocker-like" TPM+PIN is currently not present, although it **might** work the same if you do NOT configure a NVRAM password in `/etc/tpm-luks.conf`.
* If you boot into a system state, which does not release **any** LUKS key from TPM, and then run `tpm-luks-update` (either directly or by updating kernel or initramdisk), **all** TPM keys are released and you will have to re-run `tpm-luks-init`! This is a current limitation of `tpm-luks-update` and might be fixed in next release.
* Some [Lenovo laptops](https://support.lenovo.com/cz/cs/product_security/len_2015_082) and Intel NUCs have been shipped without the TPM nvLocked flag set. **You must enable this flag manually for the key to be properly protected by your TPM.**

## A short rant about security
The system is modereately secure, based on these assumptions:
* you will measure at least PCR[4] and PCR[9] to be sure your bootloader, kernel, initrd and cmdline has not been changed
* you will add panic=60 to your kernel cmdline to prevent dropping to initrd shell and leaking the key
The following risks are mitigated (key will not be released by TPM) in such case:
* Bootloader replacement (PCR[4], PCR[9])
* Kernel and initrd replacement (PCR[9])
* Kernel cmdline modifications (PCR[9])

Additional possible measurements are:
* BIOS and BOOT ROM code integrity (PCR[0], PCR[2])
* BIOS and BOOT ROM data integrity (PCR[1], PCR[3])
* MBR partition layout OR GPT partition layout integrity (if you add a partition, the measurements change) (PCR[5])
* Secure boot and/or UEFI variables related measurements (PCR[7])
* Modules loaded by Grub during boot (PCR[8])
* Commands executed by Grub from grub.cfg (PCR[11])
As long as these components have not been tampered with and the measurements will produce correct hashes, your LUKS key will be released from TPM during boot and your volume unlocked.

The following risks are **NOT mitigated** in the proposed set-up below:
* Evil maid scenario (you will still be prompted for your LUKS password, should the TPM fail to release your secret; this can be misused by the evil maid by providing tampered initramdisk or kernel)
* DMA and coldboot-based attack
* LUKS password leak by eavesdropping the LPC bus, if your TPM's LPC bus is eavesdroppable (by an extremely skilled electrical engineer), although this attack has not been really proven feasible by anyone so far (see also [Winter](https://online.tugraz.at/tug_online/voe_main2.getvolltext?pCurrPk=59565) and [Hunt](https://securingtomorrow.mcafee.com/business/security-connected/tpm-undressed/))

**Be aware that if an attacker gets hands on your computer with just LUKS password, he can try coldboot attack once. If he gets the TPM-unlocked PC, he might be trying over and over again.** It is a matter of further discussions if and how to countermeasure these types of attacks, for example by using an extra PIN to protect LUKS password stored in TPM. Please also note that tpm_nvread can be (after default tpm-tools installation) executed by any user, which means any user can read your LUKS password.

This attack also does not protect you against misconfigured console (or GUI, e.g. not disabling a Guest session is generally a very bad idea), neither does it protect you against misconfigured firewall and/or network applications (such as network shares without a password). If an attacker gets local access, you lose.

Also, if you do not trust your motherboard vendor, your TPM vendor or your system at all, do not use these tools.

## How to use this toolset

### Install ubuntu with encrypted rootfs, update and upgrade

`sudo apt update && sudo apt dist-upgrade`

### Add PPA with grub-tpm and tpm-luks prebuilt
A prebuilt repository of Ubuntu DEB packages is available.

`sudo apt-add-repository ppa:radek-zajic/measuredluks`

### Update again
This will install grub with support for measured boot.

`sudo apt update && sudo apt dist-upgrade`

### Install tpm-luks
This step installs toolset from this repository.

`sudo apt install tpm-luks`

### Configure tpm-luks
Into `/etc/tpm-luks.conf` add the following line:

`LUKS_INITRD_ENABLE=1`

Then configure `DEVICE="/dev/sdXY"` e.g. /dev/sdf3 where sdXY is your encrypted rootfs

Next, passwords must be configured.

1. if multibooting with other tpm-luks secured system: 
 * edit `/etc/tpm-luks.conf` and copy&paste `OWNERPASS` & `NVPASS` from the other tpm-luks secured system
2. if multibooting with Windows using Bitlocker or Drive encryption
 * sorry, **unless you get the owner password from your Windows installation, this combination does not work!**
3. if this is your only TPM-secured system on that computer
 * Go to BIOS, clear your TPM and reboot
 * Go to BIOS again, enable your TPM and boot into tpm-luks secured system
 * in `/etc/tpm-luks.conf` set your `OWNERPASS` and `NVPASS` passwords 
    * do not use the same pw for both as NVPASS gets copied into initramdisk!
    * for your convenience, you can use the password generator at https://www.random.org/passwords/?num=1&len=8&format=html&rnd=new
4. optional (for UEFI boot): if you are not using secure boot, you can adjust boot order after installation, e.g.
```
    efibootmgr -v  # lists boot entries
    efibootmgr -o 9,8,0  # change boot order to entries 9, 8, 0
```

You can also adjust the measurements executed by modyfing these variables in `/etc/tpm-luks.conf`:
* `PCRS_UEFI` for UEFI-based boot
* `PCRS_BIOS` for BIOS-based (MBR) boot

Do NOT use PCR[10] for measurements, please.

### Edit grub defaults in `/etc/default/grub`
Remove the `splash` keyword from `GRUB_KERNEL_CMDLINE`
 * This is necessary to properly measure kernel cmdline, no variables are allowed. The splash keywords adds a `$vt_handoff` variable, which breaks the precomputation script.

Add `panic=60` to your kernel cmdline, otherwise your LUKS-TPM set-up is exploitable and LUKS password can be easily stolen!

Optional: if you want to start multiple entries from grub, please change savedefault parameters
```
     GRUB_DEFAULT=saved
     GRUB_SAVEDEFAULT=true
```

### Regenerate grub config file and reinstall grub
```
    sudo update-grub
    sudo grub-install /dev/sdX # where X is your boot drive letter
```

### Reboot into desired state
This is to ensure PCRs will be precomputed properly (useful mainly if you are using PCR[8] or PCR[11] for sealing, which cannot be precomputed as of now).

### Update initramfs
This step adds necessary code for TPM unlocking to your initramfs.

`sudo update-initramfs -k all -u`

### Initialize tpm-luks
In this step, the scripts add a new LUKS key to your LUKS-protected drive, then store this key into TPM NVRAM. Passwords and PCR states assured by steps above are used to seal this password to *only* allow access to the NVRAM, if NVRAM password is known and PCR state matches. In other cases, access to the NVRAM is disallowed.

`sudo tpm-luks-init`

### Test by rebooting your machine
Your PC should boot without entering LUKS password now!

### Further harden your system:
  * disable Guest log-in
 http://ubuntuhandbook.org/index.php/2016/04/remove-guest-session-ubuntu-16-04/
  * set permissions for tpm_nvread to root-only so that users can't read your NVRAM password

`sudo chmod 0700 /usr/sbin/tpm_nvread`
