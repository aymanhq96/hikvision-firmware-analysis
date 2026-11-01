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

### Phase 5: RTOS Static String Hunting & Attack Surface Mapping
Since the monolithic eCos RTOS lacks a traditional Linux file system (no `/etc/shadow`), static string analysis was performed on the raw binary payload to map the attack surface and identify hidden APIs.

**Key Findings:**
1.  **Hidden CGI Endpoints:** Discovered undocumented API endpoints at `/cgi-bin/Config.cgi` and `/cgi-bin2/Config.cgi`. Internal format strings (`CONFIG.CGI: Set %s = %s`) indicate these endpoints handle configuration state changes, presenting a high-value target for authentication bypass or command injection testing.
2.  **UPnP & SOAP Vulnerability Surface:** Identified extensive UPnP service initialization (`UpnpRegisterRootDevice`) relying on SOAP XML envelopes (`http://schemas.xmlsoap.org/soap/envelope/`). In embedded RTOS environments, SOAP parsers are historically prone to Buffer Overflows and XXE injections.
3.  **Hardcoded Diagnostic Commands:** Revealed internal CLI syntax for network configuration (e.g., `wifi_connect_bssid 0/1 [SSID] BSSID...`), indicating leftover factory diagnostic tools present in the production firmware.

<img width="570" height="122" alt="image" src="https://github.com/user-attachments/assets/7e0b59b0-73ee-4bd1-b21e-5228621b6904" />
<img width="565" height="168" alt="image" src="https://github.com/user-attachments/assets/e0c22dbd-e795-40d1-9f1d-862d7a1cecf6" />
<img width="784" height="739" alt="image" src="https://github.com/user-attachments/assets/91c91c2f-530f-4516-beb7-048512450dd3" />

### Phase 6: Redundancy & OTA Update Mechanism Identification
Further analysis of the secondary compressed block (`D020.xz` at decimal offset 53280) revealed a clean `.7zXZ` header, lacking the plaintext bootloader obfuscation present in the primary block.

**Action:** 
Decompressed the block directly using standard utilities (`unxz`) and performed a component scan on the raw binary using `binwalk`.

**Finding:**
The extracted payload was a structural match to the primary eCos RTOS payload, containing the exact same ESP32/ESP32-C3 firmware segments, AES S-Box tables, and x509 certificates. 

**Conclusion:**
The presence of two identical monolithic OS blocks indicates a **Dual-Bank (A/B Partition) memory design**. This is an industry-standard IoT hardware resilience mechanism designed to facilitate Fail-Safe Over-The-Air (OTA) updates, ensuring device recovery and preventing "bricking" in the event of memory corruption during a firmware flash. 

> **Screenshot: Dual-Bank RTOS block analysis.**
<img width="935" height="832" alt="image" src="https://github.com/user-attachments/assets/57beb2d6-0a39-44c3-9ccc-091d037a90e2" />



**Conclusion:**
The device obfuscates its firmware to prevent casual analysis, but statically exposes critical, legacy attack surfaces (CGI/UPnP) within its monolithic RTOS architecture.


## Next Steps / Threat Modeling
Due to the monolithic nature of the eCos RTOS, further vulnerability research requires string extraction to map hidden CGI endpoints and hardcoded credentials, followed by loading the raw binary into **Ghidra** for ARM disassembly and dynamic flow analysis.







