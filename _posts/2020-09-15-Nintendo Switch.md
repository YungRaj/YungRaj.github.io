---
layout: post
title:  "Nintendo Switch"
date:   2020-09-15 12:00:00
categories: blog
---

## Introduction
The ARM architecture, formerly Advanced RISC Microarchitecture and now Acorn RISC Machine, is a family of processors that use reduced instruction set computing and are designed for various purposes, typically in the arena of System on a Chip (SoC) and System on a Module (SoM) which incorporate memory, radio, other interfaces, etc. You can think of the ARM SoC’s as extremely special and complex microcontrollers optimized for both performance and low power consumption. SoC’s are typically seen in mobile phones, IOT devices etc but now are migrating towards game consoles and even standard consumer based computers. ARM Holdings, the company that distributes authority to manufacture these processors, was recently bought out by Nvidia. In the mainstream market there are limited examples of Nvidia shipping out ARM processors, but an example of a game console that did is the Nintendo Switch, being sold to customers with a Nvidia Tegra X1 SoC. Typically game consoles of all products that are available are expected to have security that is only 2nd to military grade engineering feats. In its implementation, an extreme challenge in the ARM architecture as a whole is posed, because not only are the power consumption and performance standards are high, but the security posture must be robust enough to be resilient against anything thrown at it, from an interaction-less attack from a remote user to a physical attack like an evil maid. The Nintendo Switch addresses all of these issues with its relatively robust implementation of firmware, bootloaders and operating systems.


## The Nintendo Switch
The Switch is an extremely secure device architecturally, because from the bottom up considerations on behalf of security, cryptography, and modularity are made to ensure integrity and protection of safekeeping components of the machine. However, there have been numerous efforts made to break this integrity and on earlier revisions of the Nintendo Switch before 2018, named Erista, the initial code that runs on the hardware called the BootROM was exploited. This exploit not only can’t be patched but an attack can funnel down every single boot behavior that occurs afterwards. The exploit is run by injecting a payload over the USB protocol to the device in a special boot mode called RCM (Recovery Mode) and the reuptake of the payload passed through causes a buffer overflow that results in code execution. However, it takes a special process to enter this special recovery mode because it is intended for use by Nintendo internal engineers. The process involves shorting out the pins 1 and 10 on the right joystick of the console using either a paperclip or a special jig that you can order on Amazon that slides right in, pressing the volume up button, and then the power button for at least 4 seconds. Successful entry into RCM mode is shown by the screen never turning on or showing the Nintendo logo and the device being recognized by the operating system that you’re using to inject the payload. Once in RCM, you can inject a payload such as a bootloader, which in our case is called hekate. After that you can force RCM to be entered on boot every time without a key combination and use the device as you please. You can run the homebrew application, run modded or pirated games with the correct patches and install custom firmware and bootloaders needed to boot other OS’es like Android or Ubuntu.

Note: the exploit for the Nintendo Switch only works on the Nintendo Switch V1. It is code named Erista. Around 2018, Nintendo decided to patch the exploit and prevent the capability to enter RCM mode using the joystick pin shorting, and released the Nintendo Switch V2 that is codenamed Mariko. The Mariko units, after later discovery, have extra measures that I will explain later to prevent tampering, key dumping, and package1 unpacking.


To figure out if your Switch is vulnerable refer to the below chart.

```
Unpatched

XAW10000000000 to XAW10074000000
XAW40000000000 to XAW40011000000
XAW70000000000 to XAW70017800000
XAJ10000000000 to XAJ10020000000
XAJ40000000000 to XAJ40046000000
XAJ70000000000 to XAJ70040000000


Patched

XAW10120000000 and up
XAW40012000000 and up
XAW70030000000 and up
XAJ10030000000 and up
XAJ40060000000 and up
XAJ70050000000 and up
XKW10000000000 and up
XKJ10000000000 and up
XJW01000000000 and up
XWW01000000000 and up

```


## File Formats
There are numerous file formats used in the Nintendo Switch system to protect against tampering and ensure integrity of the system.

## NCAs: Nintendo Compressed Archives
NCAs are compressed archives like zip files that contain any kind of data, from game data, to operating systems packages, firmware packages, etc. Generally they also encapsulate NSO files of various kinds within them, which are the main executable format. The raw data in the NCA files are encrypted using numerous keysets that are unique per console and initialized by components such as the BootROM and the Falcon microprocessor, which will be discussed later. 

The entire raw NCA's are encrypted except the logo section if it ever gets included. For the plaintext version of the NCA's that are produced, according to the Nintendo SwitchBrew website, encryption is implemented as such
“The first 0xC00 bytes are encrypted with AES-XTS with sector size 0x200 with a non-standard "tweak" (endianness is reversed, see here), this encrypted data is an 0x400 NCA header + an 0x200 header for each section in the section table.
For pre-1.0.0 "NCA2" NCAs, the first 0x400 byte are encrypted the same way as in NCA3. However, each section header is individually encrypted as though it were sector 0, instead of the appropriate sector as in NCA3.”

The header of the NCA files at offset 0x100 contain signatures on the region of the file from offsets 0x200 to 0x400 using two RSA-2048 signatures. The end of the header also includes an array of SHA256 hashes over each FsHeader for each FS file contained within.
Within the NCA files, there are different FS files that have different use cases:

**EXEFS - Executable file system (code)**
**ROMFS - Read Only file system (data)**
**PFS0 - P??? File system (data)**

Some of the use cases of these files are unknown because when decrypted and decompressed, we get a final output file directory that doesn’t exactly resemble anything we can inspect further into, at least to my knowledge.
There is a tool written by a hacked named SciresM that parses these files, decrypts them, decompresses them, prints their metadata and processes the possible files that may be contained inside at the end. In specific, there is a file named package1 and package2 that is extremely vital to the boot process that is found at the end of unpacking these archives.

## Package1
Once you unpackage the firmware images that are encrypted and compressed using the NCA archive format using hactool and the python script I wrote to organize them, you’ll find package1 in the title id’s **(0100000000000819, 010000000000081A, 010000000000081B and 010000000000081C)**. They are also found in the eMMC storage partitions of boot0 and boot1. Package1 contains the first Switch bootloader to run on the Tegra X1 Nvidia Boot processor called the **BPMP-Lite**, running an **ARM7TDMI (ARM7 is typically used for MCU’s)**.

The **ARM7TDMI (ARM7 + 16 bit Thumb + JTAG Debug + fast Multiplier + enhanced ICE)** processor implements the ARMv4 instructions set and was widely used by embedded system designs on a smaller scale. ARM7TDMI was designed in the Nokia 6110, the first ARM powered GSM phone.
Package1 also contains the encrypted PK11 blob that contains the Switch Bootloader and ARM TrustZone code. This is important to reverse engineer as security conscious and critical startup code are numerously performed here. Here is a test run of hactool on a package1 blob on a 10.0.4 firmware for a Mariko package1.

![hactool output](https://github.com/YungRaj/YungRaj.github.io/raw/master/images/Nintendo-Switch/hactool-output.png)

As you can see on the Mariko units there’s a Mariko OEM header that contains the signature, salt, hashes and other metadata of the OEM bootloader. You will not see this on the Erista package1, because it does not contain this header.

The package1 metadata is the same however, which contains the hashes of each section and then finally the pkg1 blob that contains the encrypted contents of the files shown above. You need special keys to get the package1 file payload, and on Erista you need special keys to decrypt the firmware, which includes the package1 binary. On Erista, this is what the Header looks like.

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;margin:0px auto;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-0lax{text-align:left;vertical-align:top}
</style>
<table class="tg">
<thead>
  <tr>
    <th class="tg-0lax">Offset</th>
    <th class="tg-0lax">Size</th>
    <th class="tg-0lax">Description</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-0lax">0x0</td>
    <td class="tg-0lax">0x4</td>
    <td class="tg-0lax">Package1ldr hash (first four bytes of SHA256(package1ldr))</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x4</td>
    <td class="tg-0lax">0x4</td>
    <td class="tg-0lax">Secure Monitor hash (first four bytes of SHA256(secure_monitor))</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x8</td>
    <td class="tg-0lax">0x4</td>
    <td class="tg-0lax">NX Bootloader hash (first four bytes of SHA256(nx_bootloader))</td>
  </tr>
  <tr>
    <td class="tg-0lax">0xC</td>
    <td class="tg-0lax">0x4</td>
    <td class="tg-0lax">Build ID</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x10</td>
    <td class="tg-0lax">0xE</td>
    <td class="tg-0lax">Build Timestamp (yyyyMMddHHmmss)</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x1E</td>
    <td class="tg-0lax">0x1</td>
    <td class="tg-0lax">[7.0.0+] Key Generation</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x1F</td>
    <td class="tg-0lax">0x1</td>
    <td class="tg-0lax">Version</td>
  </tr>
</tbody>
</table>

The keys were not that difficult to obtain on a hacked Switch, so on Firmwares 6.2+, Nintendo added an extra key that is used by the TSEC processor, also known as Falcon, that I will explain later that encrypts/decrypts the final contents of the pk11 blob inside package1. We are working on an exploit to the TSEC processor that obtains this key so that on Firmwares 6.2+ we can obtain the Secure Monitor/NX Bootloader, etc. On Mariko units, the extra key that gets created by the TSEC chip is replaced by two new keys that decrypt the Mariko units, the Mariko KEK and BEK. More information on this will come later as members of the Nintendo homebrew community are withholding information on the use cases of these keys.
You should expect to see an output of hactool actually dumping the final payloads, but this is not the case as we are missing those special keys for Erista coming from the TSEC and on Mariko the KEK and BEK keys.

**Package1ldr**

The code for this stage of boot is stored in plaintext once a package1 is decrypted. The Boot Configuration Table, which I'll describe later, code executes this payload in IRAM at address 0x40010020 or 0x40010020 on firmware's 4.2+.

**PK11 (Package1)**

This blob is encrypted inside the package1 file and is decrypted by the package1ldr code. The encryption scheme used is AES-CTR mode and after the CTR and total size of the image is prepended to the encrypted blob.

When decrypted, the blob is encapsulated in the following header.

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;margin:0px auto;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-0pky{border-color:inherit;text-align:left;vertical-align:top}
</style>
<table class="tg">
<thead>
  <tr>
    <th class="tg-0pky">Offset</th>
    <th class="tg-0pky">Size</th>
    <th class="tg-0pky">Description</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-0pky">0x0</td>
    <td class="tg-0pky">4</td>
    <td class="tg-0pky">Magic "PK11"</td>
  </tr>
  <tr>
    <td class="tg-0pky">0x4</td>
    <td class="tg-0pky">4</td>
    <td class="tg-0pky">Section 0 size</td>
  </tr>
  <tr>
    <td class="tg-0pky">0x8</td>
    <td class="tg-0pky">4</td>
    <td class="tg-0pky">Section 0 offset</td>
  </tr>
  <tr>
    <td class="tg-0pky">0xC</td>
    <td class="tg-0pky">4</td>
    <td class="tg-0pky">Unknown</td>
  </tr>
  <tr>
    <td class="tg-0pky">0x10</td>
    <td class="tg-0pky">4</td>
    <td class="tg-0pky">Section 1 size</td>
  </tr>
  <tr>
    <td class="tg-0pky">0x14</td>
    <td class="tg-0pky">4</td>
    <td class="tg-0pky">Section 1 offset</td>
  </tr>
  <tr>
    <td class="tg-0pky">0x18</td>
    <td class="tg-0pky">4</td>
    <td class="tg-0pky">Section 2 size</td>
  </tr>
  <tr>
    <td class="tg-0pky">0x1C</td>
    <td class="tg-0pky">4</td>
    <td class="tg-0pky">Section 2 offset</td>
  </tr>
</tbody>
</table>
**Section 0**
This section tentatively contains the warmboot binary.

**Section 1**
This section tentatively contains the NX bootloader, which is run after the initial bootloader in package1.

**Section 2**
This section tentatively contains the Secure Monitor binary.

Mariko units have a different signed and encrypted format to make exploring the contents of package1 much harder, so reverse engineering is made more difficult.

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;margin:0px auto;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-0lax{text-align:left;vertical-align:top}
</style>
<table class="tg">
<thead>
  <tr>
    <th class="tg-0lax">Offset</th>
    <th class="tg-0lax">Size</th>
    <th class="tg-0lax">Description</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-0lax">0x0</td>
    <td class="tg-0lax">0x110</td>
    <td class="tg-0lax">Cryptographic signature</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x0000: CryptoHash (empty)</td>
    <td class="tg-0lax"></td>
    <td class="tg-0lax"></td>
  </tr>
  <tr>
    <td class="tg-0lax">0x0010: RsaPssSig</td>
    <td class="tg-0lax"></td>
    <td class="tg-0lax"></td>
  </tr>
  <tr>
    <td class="tg-0lax">0x110</td>
    <td class="tg-0lax">0x20</td>
    <td class="tg-0lax">Random block</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x130</td>
    <td class="tg-0lax">0x20</td>
    <td class="tg-0lax">SHA256 hash over package1 data</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x150</td>
    <td class="tg-0lax">0x4</td>
    <td class="tg-0lax">Version</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x154</td>
    <td class="tg-0lax">0x4</td>
    <td class="tg-0lax">Length</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x158</td>
    <td class="tg-0lax">0x4</td>
    <td class="tg-0lax">LoadAddress</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x15C</td>
    <td class="tg-0lax">0x4</td>
    <td class="tg-0lax">EntryPoint</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x160</td>
    <td class="tg-0lax">0x10</td>
    <td class="tg-0lax">Reserved</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x170</td>
    <td class="tg-0lax">Variable</td>
    <td class="tg-0lax">Package1 data</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x0170: Header</td>
    <td class="tg-0lax"></td>
    <td class="tg-0lax"></td>
  </tr>
  <tr>
    <td class="tg-0lax">0x0190: Body (encrypted)</td>
    <td class="tg-0lax"></td>
    <td class="tg-0lax"></td>
  </tr>
</tbody>
</table>

## Package2
You'll find package2 in the title id’s **(0100000000000819, 010000000000081A, 010000000000081B and 010000000000081C)** and installed on the eMMC storage's **BCPKG2** partitions. Package2 contains the Switch kernel for the Horizon operating system and the built-in sysmodules.

Package2 is distributed encrypted, so therefore extra encryption is not applied when installed to the flash system.

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;margin:0px auto;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-0pky{border-color:inherit;text-align:left;vertical-align:top}
</style>
<table class="tg">
<thead>
  <tr>
    <th class="tg-0pky">Offset</th>
    <th class="tg-0pky">Size</th>
    <th class="tg-0pky">Description</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-0pky">0x0</td>
    <td class="tg-0pky">0x100</td>
    <td class="tg-0pky">RSA-2048 signature (PKCS#1 v2.1 RSASSA-PSS-VERIFY with SHA256)</td>
  </tr>
  <tr>
    <td class="tg-0pky">0x100</td>
    <td class="tg-0pky">0x100</td>
    <td class="tg-0pky">Encrypted header</td>
  </tr>
  <tr>
    <td class="tg-0pky">0x200</td>
    <td class="tg-0pky">Variable</td>
    <td class="tg-0pky">Encrypted body</td>
  </tr>
</tbody>
</table> 

Package2's contents are encrypted with AES-CTR mode with a key only known by the TrustZone, where the first 0x10 bytes are the encrypted header's CTR. The encrypted body is split into four sections, where a CTR is stored inside each of the decrypted headers. This is what the package2's header looks like decrypted.

When decrypted, package2's header is as follows.

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;margin:0px auto;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-0lax{text-align:left;vertical-align:top}
</style>
<table class="tg">
<thead>
  <tr>
    <th class="tg-0lax">Offset</th>
    <th class="tg-0lax">Size</th>
    <th class="tg-0lax">Description</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-0lax">0x0</td>
    <td class="tg-0lax">0x10</td>
    <td class="tg-0lax">Header's CTR, official code copies the pre-decryption CTR over the decrypted result. Also used as metadata</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x10</td>
    <td class="tg-0lax">0x10</td>
    <td class="tg-0lax">Section 0 CTR</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x20</td>
    <td class="tg-0lax">0x10</td>
    <td class="tg-0lax">Section 1 CTR</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x30</td>
    <td class="tg-0lax">0x10</td>
    <td class="tg-0lax">Section 2 CTR</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x40</td>
    <td class="tg-0lax">0x10</td>
    <td class="tg-0lax">Section 3 CTR</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x50</td>
    <td class="tg-0lax">0x4</td>
    <td class="tg-0lax">Magic "PK21"</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x54</td>
    <td class="tg-0lax">0x4</td>
    <td class="tg-0lax">Base offset</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x58</td>
    <td class="tg-0lax">0x4</td>
    <td class="tg-0lax">Always 0</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x5C</td>
    <td class="tg-0lax">0x1</td>
    <td class="tg-0lax">Package2 version. Must be &gt;= {minimum valid package2 version} constant in TZ.</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x5D</td>
    <td class="tg-0lax">0x1</td>
    <td class="tg-0lax">Bootloader version. Must be &lt;= {current bootloader version} constant in TZ.</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x5E</td>
    <td class="tg-0lax">0x2</td>
    <td class="tg-0lax">Padding</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x60</td>
    <td class="tg-0lax">0x4</td>
    <td class="tg-0lax">Section 0 size</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x64</td>
    <td class="tg-0lax">0x4</td>
    <td class="tg-0lax">Section 1 size</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x68</td>
    <td class="tg-0lax">0x4</td>
    <td class="tg-0lax">Section 2 size</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x6C</td>
    <td class="tg-0lax">0x4</td>
    <td class="tg-0lax">Section 3 size</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x70</td>
    <td class="tg-0lax">0x4</td>
    <td class="tg-0lax">Section 0 offset</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x74</td>
    <td class="tg-0lax">0x4</td>
    <td class="tg-0lax">Section 1 offset</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x78</td>
    <td class="tg-0lax">0x4</td>
    <td class="tg-0lax">Section 2 offset</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x7C</td>
    <td class="tg-0lax">0x4</td>
    <td class="tg-0lax">Section 3 offset</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x80</td>
    <td class="tg-0lax">0x20</td>
    <td class="tg-0lax">SHA-256 hash over encrypted section 0</td>
  </tr>
  <tr>
    <td class="tg-0lax">0xA0</td>
    <td class="tg-0lax">0x20</td>
    <td class="tg-0lax">SHA-256 hash over encrypted section 1</td>
  </tr>
  <tr>
    <td class="tg-0lax">0xC0</td>
    <td class="tg-0lax">0x20</td>
    <td class="tg-0lax">SHA-256 hash over encrypted section 2</td>
  </tr>
  <tr>
    <td class="tg-0lax">0xE0</td>
    <td class="tg-0lax">0x20</td>
    <td class="tg-0lax">SHA-256 hash over encrypted section 3</td>
  </tr>
</tbody>
</table>
**Section 0**
This section contains the plaintext Switch kernel binary

**Section 1**
This section contains the built in sysmodules encapsulated in a custom format with the ini and kip, where ini stores the initial kernel process list and kip stores the kernel initial process.


## The Tegra X1 SoC


## Resources

Corporation, Nvidia. Tegra Boot Flow, http.download.nvidia.com/tegra-public-appnotes/tegra-boot-flow.html.

“Nintendo SwitchBrew Wiki.” Nintendo Switch Brew, switchbrew.org/wiki/. 

SciresM. “SciresM/hactool” GitHub, github.com/SciresM/hactool.

NintendoSwitchPkg. “imbushuo/NintendoSwitchPkg.” GitHub, github.com/imbushuo/NintendoSwitchPkg.

