# NamcoES3MaintenanceApp
Suite of tools useful for finding issues with the Namco ES3 arcade board

# Namco ES3 Maintenance Toolkit

A maintenance utility for the Namco ES3 arcade platform, developed through hands-on investigation of its hardware, Windows 7 Embedded environment, boot process, BitLocker-protected storage, and update workflow.

What began as an attempt to understand and regain access to an ES3 system evolved into a dedicated Windows maintenance application that makes common diagnostic and recovery tasks easier to access.

## Contents

- [Project Overview](#project-overview)
- [The ES3 Platform](#the-es3-platform)
- [1. BIOS Access and Password Recovery](#1-bios-access-and-password-recovery)
- [2. DMA Memory Acquisition](#2-dma-memory-acquisition)
- [3. Understanding BitLocker](#3-understanding-bitlocker)
- [4. Accessing the Physical Drive from Linux](#4-accessing-the-physical-drive-from-linux)
- [5. Offline Windows Registry Editing](#5-offline-windows-registry-editing)
- [6. Investigating Update USB Drives](#6-investigating-update-usb-drives)
- [7. From Research to a Maintenance Application](#7-from-research-to-a-maintenance-application)

## Project Overview

The Namco ES3 is a Windows-based arcade platform that combines conventional PC hardware with a specialized software environment, protected storage, and a game-specific startup workflow.

Working with the platform can be difficult when the original maintenance procedures or credentials are unavailable. The system's normal startup behavior also limits access to the underlying Windows desktop.

The investigation covered several areas:

- Accessing and examining the motherboard's BIOS firmware.
- Recovering the BIOS setup password.
- Acquiring system memory through a DMA interface.
- Investigating BitLocker-related material in memory.
- Accessing protected storage from Linux.
- Inspecting and modifying Windows registry hives offline.
- Understanding how the system unlocks BitLocker-protected update media.
- Building a dedicated maintenance application.

The maintenance application is the practical result of this work: a more convenient interface for common maintenance tasks.

## The ES3 Platform

The system investigated was based on the following hardware and software configuration:

| Component | Observed configuration |
|---|---|
| Platform | Namco ES3 |
| Operating system | Windows 7 Embedded |
| Motherboard | Supermicro C7B7 |
| Processor | Intel Core i5-3550 |
| Memory | 8 GB DDR3 |
| Graphics | NVIDIA GeForce GTX 760 with custom firmware |
| Primary storage | 500 GB hard drive |
| Storage security | BitLocker and TPM-related mechanisms |
| Arcade interface | Namco JVS interface |
| Observed serial devices | JVS on COM3; game data card reader on COM4 |

This configuration is specific to the unit examined. Other ES3 revisions or game installations may differ.

The initial challenge was that the cabinet behaved like a dedicated arcade appliance rather than a general-purpose Windows PC. Accessing its underlying configuration required investigating the hardware, firmware, memory, and storage separately.

## 1. BIOS Access and Password Recovery

### Reading the BIOS chip

The first major obstacle was the BIOS setup password, which restricted access to the motherboard's configuration settings.

I used a **CH314A BIOS chip reader/writer** to read the motherboard's firmware directly from the BIOS chip. This produced a firmware image that could be examined independently of the running operating system.

I then used UEFITool NE to inspect the firmware structure and locate the relevant BIOS setup data.

### Recovering the password

Examining the firmware dump revealed the BIOS password information in the setup data. This allowed me to recover the password without the original service documentation.

**Recovered BIOS password:**

`2PAK5bnD4M>G`

With the recovered password, I could access BIOS Setup and continue examining the system's hardware configuration and boot behavior.

The CH314A was used to read the chip; UEFITool was used to examine the resulting firmware image. These were separate parts of the process.

## 2. DMA Memory Acquisition

After investigating firmware access, the next challenge was understanding what the running system held in memory.

The ES3 used BitLocker-protected storage, so examining the disk alone did not necessarily reveal the underlying filesystem contents. This led to investigating the relationship between the running Windows environment, its encryption state, and the material present in physical memory.

### Memory acquisition

An Immortal DMA Scout PCIe card was used for DMA-based memory acquisition. PCILeech was used to acquire a memory dump, and MemProcFS was used to explore the resulting data.

The purpose was to investigate the running system's memory for evidence related to its storage-encryption state.

### Investigating BitLocker-related memory

The acquired memory was examined for BitLocker-related artifacts, including information associated with Full Volume Encryption Keys (FVEKs).

This investigation helped establish a connection between the running system's memory and the encrypted volumes on the ES3, informing the subsequent work on identifying and accessing those volumes.

## 3. Understanding BitLocker

The ES3's storage protection was not simply a matter of finding a password prompt. The investigation involved understanding how the volumes were protected and what information was available from the running system.

The observed installation used Windows 7-era BitLocker, including AES-128 encryption with Elephant Diffuser on the examined volume.

The investigation distinguished between several related concepts:

- **BitLocker recovery information:** material used to recover access to an encrypted volume.
- **FVEK:** the Full Volume Encryption Key used in the volume-encryption process.
- **TPM-related protection:** a mechanism that can help bind unlocking behavior to the machine and its boot state.
- **Encrypted partitions:** individual volumes that must be correctly identified and accessed.

These are not interchangeable. A BIOS password does not decrypt a BitLocker volume, and a BitLocker recovery password is not the same as an FVEK.

### Examining the drives

Windows tools such as `manage-bde` were used to inspect BitLocker status and understand which volumes were protected.

The investigation found that access and decryption behavior differed between volumes. Secondary volumes could be handled differently from the Windows system volume, whose protection was tied to the machine's security configuration.

The important outcome was learning to identify the relevant volume and its encryption state before attempting recovery or modification.

## 4. Accessing the Physical Drive from Linux

Once the storage and encryption mechanisms were better understood, Linux became a useful environment for examining the ES3 drive outside its normal Windows startup process.

Ubuntu and Dislocker were used as part of the investigation.

### The offline workflow

The general process was:

1. Shut down the ES3 and access its physical drive.
2. Connect the drive to a separate computer running Linux.
3. Identify the relevant partitions and their encryption status.
4. Use the available BitLocker recovery material to access the appropriate encrypted volume.
5. Mount the decrypted NTFS filesystem for inspection.
6. Examine the Windows installation and its configuration files without starting the original arcade environment.

Correctly identifying the partition was important. A physical disk can contain multiple partitions, and the key material associated with one volume should not be assumed to apply to another.

## 5. Offline Windows Registry Editing

One of the most useful discoveries was that the Windows shell configuration could be inspected and modified while the system drive was mounted offline.

The relevant file was the user registry hive:

`C:\Users\arcade\NTUSER.DAT`

### Investigating the shell configuration

Windows uses shell configuration to determine which interface is launched for a user session. On the ES3, the startup environment was configured around the arcade software rather than a conventional desktop workflow.

I examined the offline `NTUSER.DAT` hive using Registry Editor, loading it under a temporary key in `HKEY_LOCAL_MACHINE`.

The relevant configuration was under:

`Software\Microsoft\Windows NT\CurrentVersion\Winlogon`

### Obtaining the "shell"

For the test, the shell configuration was added, and pointed to:

`explorer.exe`

After the modified hive was saved and unloaded, the updated `NTUSER.DAT` was copied back to the Windows installation partition, replacing the original file.

When the drive was returned to the ES3 and the system was started, this change allowed the machine to load the Windows Explorer shell instead of booting the game directly.

### Why this mattered

This was a significant turning point. Rather than booting directly into the game, I had a way to change the user's shell configuration offline leading to be able to investigate the Windows environment more directly.

It also demonstrated the value of offline registry editing as a recovery technique when a system is accessible at the filesystem level but difficult to maintain through its normal interface.

**Caution:** Changing the shell can affect the cabinet's normal startup behavior. Preserve the original registry hive and record the original value before modifying it.

## 6. Investigating Update USB Drives

The next part of the investigation focused on the update USB drives used by the ES3 game-update workflow.

These drives contained update-related files, including virtual disk images and PowerShell scripts. The update process also involved BitLocker-protected storage.

The goal was to understand how the ES3's own software obtained the information needed to unlock the update media.

### Examining the update scripts

The PowerShell scripts referenced registry locations under:

- `HKLM:\SOFTWARE\NBGI\ESx\Key0`
- `HKLM:\SOFTWARE\NBGI\ESx\Key1`

These references provided a starting point for tracing how the update environment handled its key-related configuration.

By examining the scripts and their interaction with the registry, I investigated how the system obtained the passphrase used by the update workflow.

This was distinct from the earlier BIOS password recovery and from the FVEKs investigated during the DMA work. Each belonged to a different part of the system's security design.

By adding a print statement after the BitLocker keys were obtained, I was able to determine the BitLocker passphrases for the following games;

- `Tekken 7/Fated Retribution/Round 2: xVn4ZCvoH29Ck9Aw`
- `Time Crisis 5: FHVH4oQ5LezzRyB4`

### What the investigation established

The update workflow depended on more than simply inserting a USB drive. Its scripts, registry configuration, BitLocker-protected media, and virtual disk images worked together.

Understanding those relationships made it possible to investigate the update media more effectively and informed the development of the maintenance application.

## 7. The ES3Maintenance Application

`ES3Maintenance` is a C# Windows Forms utility designed to make common maintenance and diagnostic tasks on the Namco ES3 easier to access from one interface.

Rather than requiring every task to be performed manually through Windows tools or command-line scripts, the application brings together system status, drive inspection, troubleshooting shortcuts, and maintenance actions.

### Application Features

#### Drive / BitLocker Status

Runs Windows' built-in `manage-bde -status` command to display BitLocker information for available volumes.

This helps identify:
- Which volumes are protected by BitLocker.
- Whether a volume is currently locked or unlocked.
- The encryption and protection status reported by Windows.

This function is intended for **inspection only**. It does not decrypt drives, retrieve encryption keys, or change BitLocker settings.

#### VHD Inventory

Scans the expected directory for virtual hard disk (`.vhd`) files and lists the files it finds.

This is useful for identifying the virtual disks stored on the ES3, including disk images associated with its software environments. It provides a quick inventory without requiring the user to browse the filesystem manually.

Stored on the Windows partition on Tekken 7 drives lives a Namco-created tool to test, create, delete, or mount VHD image files, called ESx_Tool

A copy of these files are included in release PoC 0.2 (and above), and should be placed in the root of the update USB drive.

It'll first attempt to find, and load, the ESx_Tool inside the C:/scipts directory, if not, we'll load it from the root of the update USB drive.

#### Open Explorer

Opens Windows File Explorer so the user can browse files and directories, inspect maintenance files, and locate relevant system resources.

#### Command Prompt

Opens a command-line window for maintenance tasks that require direct use of Windows commands or other command-line utilities.

This provides access to the underlying operating system tools without requiring the user to locate and launch the command interpreter separately.

Commands entered here run with the permissions of the account or process context used to launch them.

#### System Information

Provides a shortcut to Windows system information, making it easier to inspect the machine's hardware and operating-system configuration while troubleshooting the ES3.

This is useful when documenting the hardware platform or checking system details before performing maintenance.

#### Network Diagnostics

Runs `ipconfig /all` to display detailed network configuration information, including available network adapters, IP addresses, gateways, and DNS settings.

This is particularly useful when investigating network connectivity, adapter configuration, and communication between the ES3 and external equipment.

The output is diagnostic information; the function does not itself change the network configuration.

#### Maintenance Logging

Writes application activity to:

`%ProgramData%\ES3Maintenance\maintenance.log`

The log provides a persistent record of supported application activity, helping with troubleshooting and documenting what happened during a maintenance session.

The log records only the activity the application is configured to write; it should not be treated as a complete Windows system audit.

#### Restart ES3

Provides a convenient way to restart the machine from the application.

Because restarting interrupts active processes and can cause unsaved work to be lost, the application asks for confirmation before initiating the restart.

This is the main feature in the interface that intentionally causes a system-level change.

#### ...Secret music cheat code easter egg? 👀

Have fun! 😄

### Design Philosophy

The application is intended to make routine diagnostics and maintenance more accessible while keeping potentially disruptive actions distinct from inspection tools.

| Function | Purpose | Changes system state? |
|---|---|---|
| Drive / BitLocker Status | Inspect volume protection status | No |
| VHD Inventory | List available VHD files | No |
| Open Explorer | Browse files | Not by itself |
| Command Prompt | Run Windows commands | Depends on commands entered |
| System Information | Inspect system configuration | No |
| Network Diagnostics | Inspect network settings | No |
| Maintenance Logging | Record application activity | Writes a log file |
| Restart ES3 | Restart the machine | Yes |

### Relationship to the Research

The application grew out of hands-on investigation of the ES3 platform, including BIOS access, memory acquisition, BitLocker analysis, offline drive access, and Windows registry troubleshooting.

Those investigations helped establish the maintenance workflow and the importance of having convenient access to system diagnostics. The application brings selected day-to-day tools into a single interface; it does **not** automatically perform the entire research process.

In particular, the current diagnostic interface should not be described as automatically recovering BIOS passwords, dumping RAM, decrypting BitLocker volumes, extracting encryption keys, or modifying game VHDs unless those capabilities are implemented separately.

The goal is a practical, maintainable utility for inspecting and servicing the Namco ES3 platform without having to remember every command or manually navigate to every Windows utility.

## Conclusion

The ES3 Maintenance Toolkit grew out of a practical effort to understand a dedicated arcade PC that was difficult to access through its normal operating environment.

Recovering BIOS access with a CH314A programmer opened the door to firmware investigation. DMA-based memory acquisition and BitLocker analysis helped explain the storage-protection mechanisms. Linux provided an offline route to the physical drive, while registry-hive editing demonstrated a way to change the Windows shell behavior. Examining the update scripts and their registry references then helped clarify how the update USB workflow operated.

The maintenance application brings that investigation toward a practical result: a dedicated interface for common maintenance tasks, informed by a deeper understanding of the ES3's hardware and software.

The objective is to make the platform easier to understand, troubleshoot, preserve, and maintain.

## How to set up your USB drive, and use the tool (At your own risk!)

!!!As of now, the only way to utilize this tool is by having the real update USB drive!!! (Update drives for Tekken 7 FR show up all the time on Yahoo Japan Auctions)

Note: Update drives are compatible with other games, you just need to change the BitLocker passphrases to the other game's. 

Unlocking a Time Crisis 5 update drive, and updating the passphrase with Tekken 7's will work.

1. Download the latest release (PoC 0.2 or greater)
2. Unzip the entire contents onto a safe place on your PC
3. Take your game's USB update drive, plug it in, and use one of the passwords above and unlock it
4. MAKE A BACKUP OF THE CONTENTS OF THE DRIVE BEFORE OVERWRITING/DELETING FILES
5. Copy the contents of what you unzipped earlier and drop them onto the root of your USB drive
6. Unplug the USB from your PC, and plug it into a powered-off ES3
7. Turn on the ES3
8. Let it continue through the normal booting process
9. Once it sees the USB drive, it'll attempt to update the game. Instead of trying to find and copy the update VHD, it'll launch the ES3 Maintenance App
