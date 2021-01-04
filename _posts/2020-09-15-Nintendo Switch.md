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
    <td class="tg-0lax">0x0170:</td>
    <td class="tg-0lax"></td>
    <td class="tg-0lax">Header</td>
  </tr>
  <tr>
    <td class="tg-0lax">0x0190:</td>
    <td class="tg-0lax"></td>
    <td class="tg-0lax">Body (encrypted)</td>
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
The Tegra X1 as mentioned before runs four ARM Cortex A53 for the boot start up, and then four ARM Cortex A57 cores with a Maxwell based graphics processing unit. The manual for the Tegra X1 can be found here. The Maxwell GPU has 256 cores, supports MPEG-4 HEVC, VP9, H.264, and many more formats. The SoC itself has a 20nm process the X1+ models have a 16nm process. The A57 cores clock at 1.9 GHz and the A52 cores clock at 1.53 GhZ. 
I’m not going to go into detail here, but in the SoC manual, it covers every corner, edge and detail on how the SoC works. In short, the technical manual is created to provide guidance to programmers writing code for Tegra X1 devices. It describes the register interfaces and hardware functionality, from the memory controller, address map, peripherals, chipset, IO, graphics, power management, interrupts, etc.
Below is a block diagram of the SoC.

![tegra-x1](https://github.com/YungRaj/YungRaj.github.io/raw/master/images/Nintendo-Switch/tegra_x1_soc.png)

As one can see here, there is a large set of responsibilities delegated to the SoC because it contains the whole system on just one flat sheet of silicon. On the top left you can see all of the CPU cores, their caches, etc. The wires coming out of the CPU cores are the memory buses designed for writes and reads to the main system memory, which is handled on multiple channels. 
On the left side to the right of the memory controller, are all of the peripherals from SATA, USB, MIPI, SD/MMC, HDMI, mini DisplayPort, etc. There are two TSEC chips and a Security Engine processor, which handle cryptographically secure operations on the SoC. You can think of the Security Engine as basically a hardware crypto accelerator and the TSEC as a conforming agent in implementing the TrustZone architecture for ARM64. For anyone that isn’t familiar with the TrustZone model, here is a short diagram that expresses what it’s for and why.

![switch-tz](https://github.com/YungRaj/YungRaj.github.io/raw/master/images/Nintendo-Switch/switch_tz.png)

Basically the idea is that there is a notion of a Secure World and Normal World in the ARM TrustZone. The Normal World is what you expect to be normal execution of the computer. This is known as Exception Levels 0, 1, and 2, ELx for short. EL0 is generally userspace inside an operating system, EL1 is the kernel and EL2 is the hypervisor, generally dedicated for the purpose of running virtual machines under a host. EL3 adds a 4th dimension to this, which runs in the Secure World. The Secure World has been used in the past as a watchdog to ensure that everything executing in the normal world is as expected. In the examples shown before on the Nintendo Switch, the Horizon OS is EL0 and EL1, contained in Package2. Generally another chip on the ARM SoC handles the Secure Monitor, which is interfaced with by the Application Processor using TrustZone RAM in its own MMIO space, and this happens to be the TSEC Falcon microprocessor. We’ll discuss that later. This TZRAM memory is generally locked down and encrypted so that no attacker can modify it directly with physical memory access or other special page table handling mechanisms.
All of these devices and processors are connected to the AHB, also known as the Advanced High Performance Bus, which is a bus protocol designed by ARM that conforms to the Advanced Microcontroller Bus Architecture. In the top middle you got the IRAM and the IROM, which are basically containing initial boot code in short.

And then the right side is all the other peripherals that haven’t been mentioned like I2C, SPI, Quad-SPI, and UART on the FAPB peripheral bus. The Power Management Controller, Real Time Clock, Pulse Width Modulator, which handles reduction of power in electrical signals, Fuses, which contain non-programmable data that deals with firmware and software downgrade and upgrade restrictions at least on the Switch, use cases are different for each device that contains this SOC, the Thermal Sensor, and everything else.

In an extremely well written whitepaper, the way the SoC in the Switch handles power management events in the PMC leaves room for an attack against the TrustZone by modifying the PMC registers when the machine wakes, replacing the TZRAM with attacker controlled data, and with another bug found in the Security Engine we can replace the keys used to decrypt TZRAM to verify their own TZRAM.

Examples of many kinds of security compromises that go against the SoC are glitch attacks, fault injection attacks, power management and state changes leaving security critical components wide open, hardware level attacks such as misconfiguring the memory mapped IO config, file systems buffer overflows, device driver hijacking, DMA attacks, low level boot resources being compromised, and much more.

In short, all of these things are essential to know if you want to explore the security, performance and integrity of a system like a Nintendo Switch. I can’t go that much further in detail because this would extend far beyond anything you’d ever imagined in terms of technical breakdown.

## The Falcon TSEC Processor
TSEC (Tegra Security Co-processor) is a special unit inside the Tegra X1 powered by a Falcon microprocessor designed by NVIDIA to handle security critical operations inside the Nintendo Switch. On the Nintendo Switch the Memory Mapped IO region is as follows.

0x54500000 to 0x54501000: THI (Tegra Host Interface) </br>
0x54501000 to 0x54501400: FALCON (Falcon microcontroller) </br>
0x54501400 to 0x54501600: SCP (Secure coprocessor) </br>
0x54501600 to 0x54501680: TFBIF (Tegra Framebuffer Interface) </br>
0x54501680 to 0x54501700: CG (Clock Gate) </br>
0x54501700 to 0x54501800: BAR0 (HOST1X device DMA) </br>
0x54501800 to 0x54501900: TEGRA (Miscellaneous interfaces) </br>

A larger more detailed overview of the memory mapped regions and the registers used on the TSEC Falcon is found here. Also note that this information can also be found on the NVIDIA Tegra X1 SoC manual that I mentioned before.
The TSEC has three modes in a security context. Non-Secure, which restricts the micoprograms executing from reading most registers and memory, Light-Secure, which mainly used in the context of debugging and development and Heavy Secure, which enables full access to the cryptographic hardware, and protected/secret registers and memory. The Falcon has a limited microcontroller environment leaving out the heavy asymmetric cryptographic operations like RSA out which both require expensive resources of memory to implement in software and hardware. Falcon separates instructions for data and instruction memory through IMEM and DMEM and together measures data in the KB range.

In the article I mentioned before, they achieved code execution in the Heavy Secure mode of the TSEC Falcon processor by smashing the stack when KeygenLdr loads and reads a blob of data that isn’t authenticated. This blob of data is attacker controlled and gets copied into TSEC memory. After a series of more complicated steps that are refinely described in the paper, they were able to achieve ROP code execution after the Keygen has been decrypted but with its pages marked as secret.

Sept is a payload designed by the best Nintendo hackers in the world but is proprietary and loaded as an obfuscated payload in the custom firmwares and bootloaders designed for a hacked unpatched Switch. It was designed to bypass checks implemented in SecureBoot and does so by loading the original TSEC firmware unmodified. The TSEC firmware is designed to perform an AES CMAC of the Package1 binary before returning execution to the bootloader stored in it. Sept forges the CMAC of a custom Package1 binary, passing secure boot authentication mechanisms in SecureBoot. It derives keys in place using results of registers, and scrambles the TSEC and TSEC root key, making it impossible to use the original Secure Monitor firmware as the keys to perform key generation are non-existent at that point. Below is an example of loading custom firmware to the TSEC inside of the source code of ReiNX, a custom firmware for the Nintendo Switch.

![tsec-fw-load](https://github.com/YungRaj/YungRaj.github.io/raw/master/images/Nintendo-Switch/tsec_fw_load.png)


## Implementations Similar to the TSEC Processor
# Secure Enclave
On the Apple frontier, the SecureEnclave is the dedicated processor that serves as the watchdog described in the ARM TrustZone. The iOS bootloader is also finally responsible for initializing the Secure Enclave Processor firmware.
The purpose of the Secure Enclave is to handle keys and other info such as biometrics that is sensitive enough to not be handled by the Application Processor (AP). It is isolated with a hardware filter so the AP cannot access it. It shares RAM with the AP, but it's portion of the RAM (known as TZ0) is encrypted. The secure enclave itself is a flashable 4MB AKF processor core called the Secure Enclave Processor (SEP). The technology used is similar to the ARM TrustZone but contains proprietary code for Apple KF cores in general and SEP specifically. It is also responsible for generating the UID key on A9 or newer chips that protects user data at rest. Here is a short brief overview in graph form that shows basic operation of the SEP.

 ![secure-enclave](https://github.com/YungRaj/YungRaj.github.io/raw/master/images/Nintendo-Switch/secure_enclave.png)

During Boot the Apple processor relies on the SEP to handle the Secure World, and thus needs to be configured properly by the AP to be initialized properly. This is done by setting the TZ0 and TZ1 registers, which are set before giving the SEP the clear to boot. For each register there are three 4 byte registers base/end/lock for each trust zone to be protected. The TZ0 TZ1 registers are protected by the Apple Memory Cache Controller so that they don’t get mapped into the operating system to be modified by an attack or read/written to. Once written to, they cannot be modified. The TZ0 base/end registers are generally at 

![tz0_tz1](https://github.com/YungRaj/YungRaj.github.io/raw/master/images/Nintendo-Switch/tz0_tz1.png)

These are the expected values of the TZ0 TZ1 registers.

![tz_map](https://github.com/YungRaj/YungRaj.github.io/raw/master/images/Nintendo-Switch/tz_map.png)

When the boot_tz0() is triggered, the Trust Zone registers are ensured that they are locked. The TZ memory is generally encrypted as well, so we must set the memory ranges for the encryption to take place as well as set the AES keys and finally activate encryption of that memory.
The mailbox is the communication channel between the SEP and AP, which allows the AP and SEP to reliably exchange information and trigger the security critical tasks that it is supposed to perform. The AP is mapped into memory mapped IO range in the SEP at 0x20DA0400 and the SEP is mapped into the AP at PA 0x20DA00B80 or VA 0xCDA00B80

Basically there is a message loop that the SEP executes to wait for opcodes and data to come into the mailbox mechanism to perform tasks. The first stage of boot handles opcodes 1-10 and 14-16 and the second stage will handle opcodes 1-4, 6-10 and 15-16.

![tz_commands](https://github.com/YungRaj/YungRaj.github.io/raw/master/images/Nintendo-Switch/tz_commands.png)

In general, the SEP has been shown to be extremely resistant to attacks against it to ensure that even when an Apple device is hijacked and compromised down to the kernel, that the SEP serves to protect the extremely vital and important information that is stored on it. It is not perfect, but because Apple went from the T2 security chip on the X86 version of the MacBooks to the M1 Mac containing a full SoC with the functionality of the T2 packed all into it including the SEP part, the frontiers of ARM and what’s going on here is important to consider as an engineer and architecturally conscious individual to better protect and preserve the ethics and moral considerations of technology.

## Purpose of this Paper
Basically over the last 15 years, there have been chip wars between various processor manufacturers on the grounds of performance, energy consumption, minimizing size, and finally security. Because NVIDIA recently bought ARM Holdings co and the fact that they are a key proprietor in the development of GPUs, it feels right to briefly research and speak on the innovations they’ve come up with in the past recently and what may be to come, since they will now take the pressure of setting the stage for new innovations and technologies. X86 has been poorly losing the battle against ARM when it comes to the future of semiconductors and manufacturing sheets of silicon on small < 10 nm processes which meet the demands of performance, energy consumption and security. For mobile devices, computers, and even game consoles now, it seems like NVIDIA and ARM are going to be leading the pack in the new developments that are occurring. This also means that there is going to be competition and cooperation along the way, in which case companies like Apple are following suit to all the things occurring and jumping on the wave of ARM silicon to be shipped in their new Macs. Apple seems to be also competing by developing their silicon in house and branding it as their own, though to be manufactured elsewhere. The M1 chip, although just being released, seems to be a great new invention to add on top of additional previous technologies like the Secure Enclave, T2 Security, and the firmware bootloading mechanisms like EFI, SecureROM and iBoot.

## Sources
Corporation, Nvidia. Tegra Boot Flow, http.download.nvidia.com/tegra-public-appnotes/tegra-boot-flow.html.
“Nintendo SwitchBrew Wiki.” Nintendo Switch Brew, switchbrew.org/wiki/.
SciresM. “SciresM/hactool” GitHub, github.com/SciresM/hactool.
NintendoSwitchPkg. “imbushuo/NintendoSwitchPkg.” GitHub, github.com/imbushuo/NintendoSwitchPkg.
Methodically Defeating Nintendo Switch Security. https://arxiv.org/pdf/1905.07643.pdf
Wiibrew Contributors. Signing bug https://wiibrew. org/wiki/Signing bug
Console Hacking 2010, PS3 Epic Fail, Dec. 2010. https://media.ccc.de/v/27c3-4087-en-console hacking 2010

Viva la Vita Vida, Hacking the most secure handheld console, https://media.ccc.de/v/c4.openchaos.2018.06. glitching- the- switch

3DBrew Contributors. Browserhax. https://www. 3dbrew.org/wiki/Browserhax
Console Hacking 2013, WiiU, Dec. 2013 https://media.ccc.de/v/30C3 - 5290 - en - saal 2 - 201312272030

3dbrew.org/wiki/Category:File formats

System calls. https:switchbrew.org/wiki/SVC

IPC Marshalling: https://switchbrew.org/wiki/
3dbrew Contributors. 3ds system flaws - firm sysmodules.https://www.3dbrew.org/wiki/3DS System Flaws

3ds system flaws https://www.3dbrew.org/wiki/3DS System Flaws

ReSwitched Contributors. Cagetheunicorn https://github.com/reswitched/cagetheunicorn

https://github.com/reswitched/mephisto

https://github.com/windknown/presentations/blob/master/Attack_Secure_Boot_of_SEP.pdf

