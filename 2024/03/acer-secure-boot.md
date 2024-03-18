# Acer screwed up: How to (not) Secure Boot

This is picking up from the laptop review on last post. Upon installing Arch Linux, I forgot my install media plugged into the machine, which in this case was a Ventoy stick full of ISOs. After using sbctl to enroll the custom Secure Boot keys into the system's firmware:

```
tabby@calico ~> sudo sbctl status
Installed:      ✓ sbctl is installed
Owner GUID:     9c2ef588-efb7-4f84-aede-cb0033482a6e
Setup Mode:     ✓ Disabled
Secure Boot:    ✗ Disabled
Vendor Keys:    none
```

I rebooted to double check the machine worked, as I'm well familiar with buggy UEFI implementations blowing up when someone actually bothers using Secure Boot with their own keys. Then, something peculiar happened: The machine booted into Ventoy. To explain why this is weird, here's a primer on Secure Boot.

## Securing the x86 PC platform

PCs have suffered from the scourge of rootkits for a long time. Back in the XP days basically any machine could be owned from bootmgr upwards since most everything ran with Administrator permissions, no questions asked. With Vista and 7 things improved somewhat thanks to User Account Control, however people are conditioned to just hit "yes" on any dialog they're given and so, the problem continued. Microsoft decided to finally set things straight with Windows 8, by not only requiring UEFI boot for new machines, but also the new feature called Secure Boot.

Secure Boot leverages how the UEFI boot system is no longer confined by reading the first sector of your hard drive or whatever janky hell the Legacy PC platform uses to boot. It's now advanced enough where it just has a filesystem driver (if I recall the standard *requires* FAT support, but other filesystems can be supported by a vendor) and it uses that driver to open up a regular partition on a disk and look for a file, usually with a ".efi" extension, and chainload into that.

Secure Boot adds cryptographic verification to the boot process, through the use of public and private keys.

Every machine that ships with Windows 8 or newer requires Microsoft's digital signature to be installed to the Secure Boot database. Oversimplifying, Secure Boot has both an "allowed signatures" database and a "forbidden signatures" database, where leaked or otherwise insecure keys are added and the system checks and specifically blocks.

When the UEFI BIOS chainloads an EFI binary, be it an operating system chainloader or even an Option ROM, it will check its digital signature against the Secure Boot databases. If Secure Boot is enabled and is not in Setup Mode (more on Setup Mode soon), the system will check the signature of the file that's about to be loaded and if it's not in its database, or is in the forbidden signatures database, the system will refuse to load it.

However, Microsoft (to my knowledge) requires OEMs to allow you to not only disable Secure Boot, but also be able to *enroll your own keys* into the system, giving you full control over your computer's chain of trust, allowing it to only load binaries you trust and you signed yourself with your own keys.

## The problem

Now, to get into Acer's screwup, we need to explain Microsoft's keychain. Microsoft has three Secure Boot keys that are enrolled on every PC shipping Windows:

- The **Microsoft Windows Production PCA 2011** and **Windows UEFI CA 2023** certificates are what signs the Windows bootloader itself and allows you to boot Windows.
- The **Microsoft Corporation UEFI CA 2011** and **Microsoft UEFI CA 2023** certificates (also known as the Microsoft 3rd Party UEFI CA certificates) are used by Microsoft to sign third party binaries, like Option ROMs, UEFI drivers, and things like shim-signed that allow Linux distros to boot with the standard Microsoft keys.
- The **Microsoft Corporation KEK CA 2011** and **Microsoft Corporation KEK 2K CA 2023** certificates "enable revocation of bad images by updating the dbx and potentially for updating db to prepare for newer Windows signed images", however Windows will boot without this in your Secure Boot database.

If you have something like a laptop with integrated graphics, where the built-in GPU is "implicitly trusted" due to it being literally part of the CPU itself, you can often get away with not a single Microsoft key, and have full control of your system's boot signing checks. This just so happens to be the same setup my laptop has. So with this goal in mind, I set my machine into Setup Mode.

*Setup Mode is when you clear the Secure Boot databases, often with an option in the BIOS, which makes the system be able to boot any binary even with Secure Boot enabled, and the new running OS is able to enroll new keys into the database, which will automatically disable Setup Mode and set Secure Boot to User Mode, where the firmware will now do signature checks again.*

So, I opened the machine's BIOS, punched in my password and cleared the Secure Boot keys, properly kicking the system into Setup Mode, and booted up back into Arch. Using sbctl, I've enrolled my keys into Secure Boot, and double checked they were in place:

```
tabby@calico ~> sudo sbctl enroll-keys
Enrolling keys to EFI variables...✓
Enrolled keys to the EFI variables!
tabby@calico ~> sudo sbctl status
Installed:      ✓ sbctl is installed
Owner GUID:     9c2ef588-efb7-4f84-aede-cb0033482a6e
Setup Mode:     ✓ Disabled
Secure Boot:    ✗ Disabled
Vendor Keys:    none
```

And then, what happened in this intro came to be. The system booted into Ventoy. But why? I didn't sign the Ventoy EFI binary!

Booting back into Arch I ran sbctl again, and lo and behold:

```
tabby@calico ~> sbctl status
Installed:      ✓ sbctl is installed
Owner GUID:     9c2ef588-efb7-4f84-aede-cb0033482a6e
Setup Mode:     ✓ Disabled
Secure Boot:    ✓ Enabled
Vendor Keys:    microsoft
```

> `Vendor Keys:    microsoft`

Odd. I specifcally checked the Microsoft keys didn't get slipped into the Secure Boot database. So, I promptly hopped back into the BIOS, cleared keys again, booted into Arch, checked the system was in Setup Mode:

```
tabby@calico ~> sbctl status
Installed:      ✓ sbctl is installed
Owner GUID:     9c2ef588-efb7-4f84-aede-cb0033482a6e
Setup Mode:     ✗ Enabled
Secure Boot:    ✗ Disabled
Vendor Keys:    none
```

Re-enrolled the keys, checked they weren't there again:

```
tabby@calico ~> sudo sbctl enroll-keys
Enrolling keys to EFI variables...✓ 
Enrolled keys to the EFI variables!
tabby@calico ~> sudo sbctl status
Installed:      ✓ sbctl is installed
Owner GUID:     9c2ef588-efb7-4f84-aede-cb0033482a6e
Setup Mode:     ✓ Disabled
Secure Boot:    ✗ Disabled
Vendor Keys:    none
```

Notice the `Vendor Keys: none` line. Rebooted, ran sbctl status again, and sure enough:

```
tabby@calico ~> sbctl status
Installed:      ✓ sbctl is installed
Owner GUID:     9c2ef588-efb7-4f84-aede-cb0033482a6e
Setup Mode:     ✓ Disabled
Secure Boot:    ✓ Enabled
Vendor Keys:    microsoft
```

Something weird is afoot here. With the goal of diagnosing further, I used a different method of checking for keys. The Linux kernel logs will print out the Secure Boot status as the kernel loads, showing what keys are in its database.

In setup mode, it shows the keyrings initializing but no certificates installed:

```
tabby@calico ~> sudo dmesg | grep "integrity:"
[    0.971365] integrity: Platform Keyring initialized
[    0.971366] integrity: Machine keyring initialized
```

Now here comes some Linux magic. To make sure it wasn't sbctl doing some weird stuff and lying about the state of the EFI variables, I enrolled the keys again, then kexec'd a fresh kernel (kexec is a Linux feature which allows the kernel to load a new copy of itself without actually rebooting the machine). Checking dmesg again, we get:

```
tabby@calico ~> sudo dmesg | grep "integrity:"
[    0.957792] integrity: Platform Keyring initialized
[    0.957795] integrity: Machine keyring initialized
[    1.032103] integrity: Loading X.509 certificate: UEFI:db
[    1.032534] integrity: Loaded X.509 cert 'Database Key: 6aadf3c9402913e1b3243ad5fe41fac5'
```

Now only my key is there, as it should be! And to put in the final nail in the coffin, after a full, normal reboot, from UEFI up:

```
tabby@calico ~> sudo dmesg | grep "integrity:"
[    0.944364] integrity: Platform Keyring initialized
[    0.944366] integrity: Machine keyring initialized
[    1.007742] integrity: Loading X.509 certificate: UEFI:db
[    1.008025] integrity: Loaded X.509 cert 'Database Key: 6aadf3c9402913e1b3243ad5fe41fac5'
[    1.008026] integrity: Loading X.509 certificate: UEFI:db
[    1.008039] integrity: Loaded X.509 cert 'Microsoft Corporation UEFI CA 2011: 13adbf4309bd82709c8cd54f316ed522988a1bd4'
```

> `Loaded X.509 cert 'Microsoft Corporation UEFI CA 2011`

And there it is. This laptop's firmware is *injecting Microsoft's keys* into the Secure Boot database, whether you like it or not. This has obvious security implications, and here's some of them:

## The consequences

Say you're average Joe Q. User and you have no idea what a BIOS is, let alone a Secure Boot or cryptographic keys. You just know the computer turns on, it has a desktop, it has icons and programs, and you get your work done. So needless to say, odds are your laptop's firmware will never be updated (especially because as far as I can tell, Acer does not provide firmware updates, neither through Windows Update or LVFS).

What if, say, Microsoft's signing keys get leaked. It has happened before for some auxiliary keys, so it's not unreasonable. Now this laptop is permanently injecting some arbitrary Microsoft certificate into your Secure Boot keys, keeping the door wide open for binaries signed with that bad key.

What if, you're a business relying on Secure Boot to control what your workstations can and can't boot, by using custom signing keys. Especially in this laptop, where **the BIOS does NOT ask for the administrator password to boot from an external device.** With the laptop permanently booting any Microsoft signed binary, an attacker can boot whatever they want! Even with a firmware that theoretically checks every chainloaded binary like this one should, utilities like Ventoy have been shown to work around this. I booted the Microsoft-signed Ventoy and chainloaded into an Arch Linux ISO, which is not signed at all.

And say you're running a setup like mine, running a barebones Linux distro like Arch, without automatic dbx updates because hey, my custom Secure Boot keys wouldn't allow anything I don't trust to boot anyway, right? Well... not anymore. Consider a rootkit installs itself onto your Linux box by adding itself as a highly customized version of GRUB loading off of a Microsoft-signed shimx64.efi. Suddenly, your supposedly "custom Secure Boot key" machine just got pwned.

## Disclosure

I attempted to reach out to Acer two times over email, through their official channel (vulnerability@acer.com), over **4 months ago**, with all the details of this exploit.

No reply, both times.

A vulnerability in Secure Boot should *not* be taken lightly, especially one that erodes the user's trust in the machine like this. After failing to contact them directly, I did the only thing I can do: make this public.

I'm in the process of getting a CVE for this issue, and will be updating this page accordingly.

I'll be answering any questions in the comments! Thank you for reading.
