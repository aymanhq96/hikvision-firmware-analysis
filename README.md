# hikvision-firmware-analysis
Analyzing Hikvision firmware with binwalk
<img width="1918" height="1029" alt="image" src="https://github.com/user-attachments/assets/e57c5769-0e8b-46fa-944a-945e4aaa4d66" />

<img width="1189" height="790" alt="Captura de pantalla 2026-11-01 172635" src="https://github.com/user-attachments/assets/14e9e20a-3e34-4dd3-b720-81005fca4a01" />

<img width="1152" height="670" alt="image" src="https://github.com/user-attachments/assets/c8685107-4f0f-41e7-bb7a-95c971473546" />

<img width="651" height="208" alt="image" src="https://github.com/user-attachments/assets/5f973fb2-6f4f-48fc-8b0d-d8fd5e3cb513" />

<img width="774" height="809" alt="image" src="https://github.com/user-attachments/assets/4ef90cc6-903d-4024-b261-bfa37fcc3f0c" />

# IoT Firmware Audit: Overcoming Manufacturer Obfuscation

## Objective
Perform a static security analysis and reverse engineering of a commercial IP camera firmware (`SD_CarDV.bin`). The objective was to extract the core filesystem, identify the underlying architecture, and map potential attack surfaces.

## Executive Summary
This audit demonstrates the evasion of proprietary firmware obfuscation techniques. Standard automated extraction tools (`binwalk`) failed due to intentional packing methods. Through manual data carving and hex analysis, the true payload was isolated. The core firmware was identified not as a standard Linux filesystem, but as a monolithic **eCos RTOS** running a dual-MCU architecture, heavily relying on **ESP32 and ESP32-C3** co-processors.

## Methodology & Findings

### Phase 1: Initial Static Triage
Initial inspection using `binwalk` identified multiple multimedia assets, web documents (HTML/XML), and what appeared to be `xz` compressed data blocks. 
*   **Command:** `binwalk SD_CarDV.bin`
*   **Finding:** The presence of HTML/XML indicated an embedded web server, but no standard filesystem (e.g., SquashFS) was immediately visible.

### Phase 2: Extraction Failure & Hex Analysis
Automated extraction using `binwalk -e` successfully dumped the initial `.xz` blocks. However, decompression using `unxz` resulted in a `Compressed data is corrupt` exception.
*   **Investigation:** A hex dump (`xxd`) of the target block (`B608.xz`) revealed a custom obfuscation technique. The manufacturer prepended plaintext bootloader execution logs directly after the `.7zXZ` magic bytes, causing standard decompressors to crash.
*   **Evidence:** `[SD_FAT_Init ERR!]`, `Keyin for ICE load`, and `CRC32 Check fail` strings were found masking the true payload.

> **Screenshot: Bootloader strings bypassing standard compression headers.**
<img width="704" height="166" alt="image" src="https://github.com/user-attachments/assets/a9399eee-785e-49e6-9445-0627a82c57c3" />


### Phase 3: Manual Data Carving (`dd`)
To bypass the anti-analysis mechanism, I analyzed the corrupted block to find the exact offset of the secondary, legitimate `xz` header.
*   **Action:** Identified the true compressed payload started at decimal offset `6680`.
*   **Carving:** Used `dd` to surgically amputate the obfuscated bootloader strings:
    ```bash
    dd if=B608.xz bs=1 skip=6680 of=payload_real.xz
    ```
*   **Result:** Successfully isolated a clean, 3.9 MB `xz` compressed payload, defeating the manufacturer's packing technique.

> **Screenshot: Successful manual data carving using dd.**
<img width="710" height="71" alt="image" src="https://github.com/user-attachments/assets/4c9b7410-a269-4e27-a116-55b42a9c12b7" />


### Phase 4: Monolithic RTOS & ESP32 Discovery
Decompressing the carved payload yielded a raw binary data file. A deep scan of this core binary completely shifted the architectural understanding of the device.
*   **OS Identification:** The system is not running an embedded Linux kernel, but rather a monolithic Real-Time Operating System (**eCos RTOS**).
*   **Hardware Architecture:** Identified embedded firmware images for **ESP32 and ESP32-C3** microcontrollers (`ESP Image segment count... flash mode: QUIO`). This confirms a dual-MCU design where the main processor delegates Wi-Fi/network capabilities to an ESP32 edge chip.
*   **Cryptographic Primitives:** Discovered hardcoded `AES S-Box` tables, `Base64 standard index tables`, and `x509 v3 DER Certificates`, highlighting the cryptographic implementations used for secure communications (HTTPS/MQTT).

> **Screenshot: Identification of RTOS and ESP32 components.**
<img width="760" height="177" alt="image" src="https://github.com/user-attachments/assets/d152e692-a752-4a22-a19f-c0089623c376" />
<img width="943" height="825" alt="image" src="https://github.com/user-attachments/assets/a577cb59-f071-412b-9693-9fa6a6d7614c" />




## Next Steps / Threat Modeling
Due to the monolithic nature of the eCos RTOS, further vulnerability research requires string extraction to map hidden CGI endpoints and hardcoded credentials, followed by loading the raw binary into **Ghidra** for ARM disassembly and dynamic flow analysis.







