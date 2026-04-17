# ESP32 Security & Firmware Lockdown Specification

This document outlines the implementation of a "Phone-Style" security architecture for the ESP32. It ensures the device only accepts authorized firmware, ignores standard flashing tools, and provides visual feedback for security events.

## 1. Code Sovereignty (Secure Boot V2)
To lock the hardware so it only accepts your firmware, you must implement **Secure Boot V2**.

*   **The Key:** Generate a private **RSA-3072 key**. During the first production flash, the ESP32 burns a hash of your public key into its **eFuses** (One-Time Programmable memory).
*   **The Lock:** The chip's internal **BootROM** checks the digital signature of any code before execution. If the signature doesn't match your private key, the ESP32 refuses to boot.

## 2. Anti-Tamper & Visual Warnings
To prevent unauthorized access and standard hacking attempts, the hardware must be configured to block external tools.

### Disable Standard Flashing
In the `sdkconfig`, set the **UART ROM Download Mode** to `Permanently Disabled` or `Secure Mode`.
*   **Result:** This makes the chip ignore `esptool.py`. Holding the "Boot" button will no longer allow the device to be re-flashed via standard methods.

### Bootloader Hooks
Program custom logic into the [Bootloader Hooks](https://espressif.com) to handle pre-boot events:
*   **Initialize Display/LED:** Trigger visual hardware before the main OS loads.
*   **Security Feedback:** If the system detects a failed boot or signature mismatch, the bootloader can display a screen stating: `SECURITY WARNING: UNAUTHORIZED FIRMWARE` instead of crashing.

## 3. Proprietary Java Update System (Serial OTA)
Since standard flashing is disabled, firmware updates are handled via a custom serial protocol between a Java Application and the ESP32.

### The Bridge (ESP32 Side)
The running firmware maintains a background task listening to the **UART (Serial)** port for a specific handshake command.

### The Java App
Using a library like `JSerialComm`, the Java application:
1.  Initiates a handshake.
2.  Sends the signed firmware binary in small chunks.
3.  Monitors the transfer progress.

### Validation & Swap
1.  **Transfer:** ESP32 writes incoming chunks to the `ota_0` or `ota_1` partition.
2.  **Verify:** After the full transfer, the ESP32 verifies the **Secure Boot signature**.
3.  **Commit:** If the signature is valid, the device updates the boot partition and reboots. If invalid, the update is rejected and a warning is displayed.

## Implementation Matrix


| Feature | Developer Mode | Production (Locked) Mode |
| :--- | :--- | :--- |
| **Flashing Method** | `esptool.py` / JTAG | **Custom Java App Only** |
| **Code Validation** | None | **Hardware-Level (RSA-3072)** |
| **Serial Port** | Open Debugging | **Locked / Serial OTA Protocol** |
| **Boot Failure** | Reboot Loop | **Visual Security Warning** |

> [!CAUTION]
> **IRREVERSIBLE PROCESS:** Once eFuses are burned to disable UART Download Mode, the process cannot be undone. You must ensure the Java-based Serial OTA system is fully functional before deployment.
