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

The `Shell` value was the setting of interest.

### Changing the shell

For the test, the shell configuration was changed to point to:

`explorer.exe`

After the modified hive was saved and unloaded, the updated `NTUSER.DAT` was copied back to the Windows installation, replacing the original file.

When the drive was returned to the ES3 and the system was started, this change allowed the machine to load the Windows Explorer shell instead of following the previous shell behavior.

### Why this mattered

This was a significant turning point. Rather than relying solely on the original arcade startup path, I had a way to change the user's shell configuration offline and investigate the Windows environment more directly.

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

### What the investigation established

The update workflow depended on more than simply inserting a USB drive. Its scripts, registry configuration, BitLocker-protected media, and virtual disk images worked together.

Understanding those relationships made it possible to investigate the update media more effectively and informed the development of the maintenance application.

## 7. From Research to a Maintenance Application

After investigating the BIOS, memory, storage, registry, and update workflow, the next step was to make routine maintenance more accessible.

The **ES3 Maintenance Toolkit** is the practical application that came out of this work.

Instead of requiring every maintenance task to begin with manual investigation or navigation through the original arcade environment, the application provides a dedicated interface for common actions.

### Application interface

The Windows Forms application includes a maintenance interface with controls such as:

- **Command Prompt** — access to a command-line environment for troubleshooting.
- **Open Log Folder** — convenient access to the application's logs.
- **Restart ES3** — a direct way to restart the machine.
- **System overview** — displays basic information about the computer, current user, operating system, and architecture.
- **Maintenance output** — displays status and diagnostic information.

The interface is intended to make the maintenance workflow easier to navigate and provide a central starting point for supported operations.

### Read-only information and actions

The application distinguishes between system information and actions that change the machine's state.

System overview and inventory information can be presented without changing the cabinet's configuration. Other actions, such as restarting the machine, have an explicit effect and should be used deliberately.

The application does not replace the underlying Windows, storage, or firmware mechanisms. It provides a more convenient interface for working with the system.

## Conclusion

The ES3 Maintenance Toolkit grew out of a practical effort to understand a dedicated arcade PC that was difficult to access through its normal operating environment.

Recovering BIOS access with a CH314A programmer opened the door to firmware investigation. DMA-based memory acquisition and BitLocker analysis helped explain the storage-protection mechanisms. Linux provided an offline route to the physical drive, while registry-hive editing demonstrated a way to change the Windows shell behavior. Examining the update scripts and their registry references then helped clarify how the update USB workflow operated.

The maintenance application brings that investigation toward a practical result: a dedicated interface for common maintenance tasks, informed by a deeper understanding of the ES3's hardware and software.

The objective is to make the platform easier to understand, troubleshoot, preserve, and maintain.
