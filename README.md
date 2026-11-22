ESP32RET
=======

Reverse Engineering Tool running on ESP32 based hardware. Supports both EVTV ESP32 and Macchina A0

There is a precompiled binary version of this program here:
https://www.savvycan.com/ESP32RET_Updater.zip


## PlatformIO: ESP32-S3 DevKit N16R8 with SN65HVD230

The `stable-s3-devkitc-1-n16r8` environment targets a generic ESP32-S3 DevKit with
16 MB flash and 8 MB octal PSRAM. It uses the ESP32-S3's built-in TWAI controller
and one CAN bus. The transceiver is connected to GPIO 4 (CAN RX) and GPIO 5
(CAN TX).

Wire the SN65HVD230 module as follows:

| SN65HVD230 | ESP32-S3 DevKit / CAN bus |
| --- | --- |
| VCC | 3V3 |
| GND | GND |
| RXD | GPIO 4 |
| TXD | GPIO 5 |
| RS | GND (normal/high-speed mode) |
| CANH, CANL | CAN bus CAN_H, CAN_L |

Use a 3.3 V-compatible SN65HVD230 module. Do not connect CANH/CANL directly to
the ESP32. Terminate the bus at its two physical ends (typically 120 ohms at
each end; about 60 ohms measured between CANH and CANL with the bus powered
off).

To build and upload from Windows 11:

1. Install Visual Studio Code and the PlatformIO IDE extension.
2. Open this project folder in VS Code and wait for PlatformIO to install the
   ESP32 platform and libraries.
3. Select the `stable-s3-devkitc-1-n16r8` environment in PlatformIO and run
   **Build**. The firmware is written to
   `.pio/build/stable-s3-devkitc-1-n16r8/firmware.bin`.
4. Connect the DevKit by USB and run **Upload** in PlatformIO. If the board is
   not detected, hold **BOOT**, press and release **RESET**, then release
   **BOOT** and retry. PlatformIO uploads the bootloader, partition table, and
   firmware at the required addresses.

The same actions are available from the PlatformIO terminal:

```text
pio run -e stable-s3-devkitc-1-n16r8
pio run -e stable-s3-devkitc-1-n16r8 -t upload
```

By default, the device creates a Wi-Fi access point named `ESP32RETSSID` with
password `aBigSecret`. Connect the PC to it, then in SavvyCAN add a GVRET
network/TCP connection to `192.168.4.1`, port `23`. Configure the CAN bitrate
in SavvyCAN to match the bus (the firmware default is 500 kbit/s).

The existing S3 partition table is retained for compatibility; it allocates
the first 8 MB of flash, while the firmware image is built for the 16 MB chip.
If the board has saved settings from another ESP32RET board profile, set
`SYSTYPE=4` in the serial console and restart it.

### Automated GitHub releases

The [release workflow](./.github/workflows/release-firmware.yml) runs when a
version tag is pushed.

It builds the `stable-s3-devkitc-1-n16r8` environment and publishes a release
named for the ESP32-S3 DevKit N16R8 and SN65HVD230. The release includes the
firmware as `ESP32RET-ESP32-S3-DevKit-N16R8-SN65HVD230.bin`.

```text
git tag v1.0.0
git push origin v1.0.0
```

### Requirements (legacy Arduino IDE build):

You will need the following to be able to compile the run this project:

- [Arduino IDE](https://www.arduino.cc/en/Main/Software) Tested on 1.8.13
- [Arduino-ESP32](https://github.com/espressif/arduino-esp32) - Allows for programming the ESP32 with the Arduino IDE
- [esp32_can](https://github.com/collin80/esp32_can) - A CAN library that supports the built-in CAN
- [esp32_mcp2517fd](https://github.com/collin80/esp32_mcp2517fd) - CAN library supporting MCP2517FD chips
- [can_common](https://github.com/collin80/can_common) - Common structures and functionality for CAN libraries

PLEASE NOTE: The Macchina A0 uses a WRover ESP32 module which includes PSRAM. But, do NOT use the WRover
board in the Arduino IDE nor try to enable PSRAM. Doing so causes a fatal crash bug.

The EVTV board has no PSRAM anyway.

This program is larger than the default partitioning scheme. You will need to use
a larger scheme. The recommended way to do this: Tools -> Partition Scheme -> Minimal SPIFFS

By default a wifi hotspot will be created by this firmware. The SSID is either ESP32RETSSID (for EVTV board) or
A0RETSSID (For Macchina A0). The default WPA2 password is "aBigSecret" (Minus the quote marks) You can configure
different settings from the serial port created when connected to USB. The serial port is 1 Megabit speed.

All libraries belong in %USERPROFILE%\Documents\Arduino\hardware\esp32\libraries (Windows) or ~/Arduino/hardware/esp32/libraries (Linux/Mac).

The canbus is supposed to be terminated on both ends of the bus. This should not be a problem as this firmware will be used to reverse engineer existing buses. However, do note that CAN buses should have a resistance from CAN_H to CAN_L of 60 ohms. This is affected by placing a 120 ohm resistor on both sides of the bus. If the bus resistance is not fairly close to 60 ohms then you may run into trouble.

### The firmware is a work in progress. What works:
- CAN0 / CAN1 reading and writing
- Preferences are saved and loaded
- Text console is active (configuration and CAN capture display)
- Can connect as a GVRET device with SavvyCAN
- LAWICEL support (somewhat tested. Still experimental)
- Bluetooth works to create an ELM327 compatible interface (tested with Torque app)

### What does not work:
- Digital and Analog I/O

## Flashing ESP32 with esptool and RFC2217 (Windows PowerShell)

1. Connect the ESP32 to a USB‑OTG port.
2. Hold the **BOOT** button, briefly press **RST**, then release **BOOT** to enter programming mode.
3. Open **PowerShell** and navigate to the directory where `esptool` is located:

    ```powershell
    mkdir "$env:USERPROFILE\esptool"
    cd "$env:USERPROFILE\esptool"
    ```

    Download and place the latest release of [esptool](https://github.com/espressif/esptool/releases) here.

4. List available COM ports:

    ```powershell
    Get-CimInstance Win32_SerialPort | Select-Object DeviceID, Description
    ```

    Sample output (your ports may differ):
    ```text
    DeviceID Description
    -------- -----------
    COM3 Intel(R) Active Management Technology - SOL
    COM5 Standardowy port szeregowy przez link Bluetooth
    COM4 Standardowy port szeregowy przez link Bluetooth
    COM8 Urządzenie szeregowe USB
    ```

5. Verify the ESP responds (replace `COM8` with the correct port):

    ```powershell
    .\esptool.exe --port COM8 chip_id
    ```

    Expected output (example):
    ```text
    esptool.py v4.14.0
    Serial port COM8
    Connecting...
    Detecting chip type... ESP32-S3
    Chip is ESP32-S3 (QFN56) (revision v0.2)
    Features: WiFi, BLE, Embedded PSRAM 8MB (AP_3v3)
    Crystal is 40MHz
    USB mode: USB-Serial/JTAG
    MAC: 00:00:00:00:00:00
    Uploading stub...
    Running stub...
    Stub running...
    Warning: ESP32-S3 has no Chip ID. Reading MAC instead.
    MAC: 00:00:00:00:00:00
    Hard resetting via RTS pin...
    ```

6. Start the RFC‑2217 server (replace `COM8` if needed):

    ```powershell
    .\esp_rfc2217_server.exe -v -p 4000 COM8
    ```

    Sample output:
    ```text
    INFO: RFC 2217 TCP/IP to Serial redirector - type Ctrl-C / BREAK to quit
    INFO: Serving serial port: COM8
    INFO: TCP/IP port: 4000
    INFO: Waiting for connection ... use the 'rfc2217://192.168.1.90:4000?ign_set_control' as a PORT
    ```

7. From **WSL2** (or any Linux environment) connect through the RFC‑2217 proxy:

    ```bash
    ~/.platformio/penv/bin/python -m esptool --port "rfc2217://127.0.0.1:4000?ign_set_control" --before no-reset chip-id
    ```

    Expected output (example):
    ```text
    esptool v5.1.0
    Note: Pre-connection option "no-reset" was selected. Connection may fail if the chip is not in bootloader or flasher stub mode.
    Connected to ESP32-S3 on rfc2217://127.0.0.1:4000?ign_set_control:
    Chip type: ESP32-S3 (QFN56) (revision v0.2)
    Features: Wi‑Fi, BT 5 (LE), Dual Core + LP Core, 240MHz, Embedded PSRAM 8MB (AP_3v3)
    Crystal frequency: 40MHz
    USB mode: USB-Serial/JTAG
    MAC: 00:00:00:00:00:00
    Stub flasher running.
    Warning: ESP32‑S3 has no chip ID. Reading MAC address instead.
    MAC: 00:00:00:00:00:00
    Hard resetting via RTS pin...
    ```

8. Return to programming mode on the ESP32 and, if required, restart the RFC‑2217 server on Windows.

9. Open the PlatformIO project in **VS Code (WSL2)**, select **ESP32-S3-DevKit-N16R8** and press **PlatformIO: Upload**.

10. After the upload completes, press the **RST** button on the board.

## Connecting with SavvyCAN

1. Open **SavvyCAN** (tested with `V208`) and navigate to **Connection** -> **Open Connection Window** (or click **Add New Device Connection** inside **Connection Settings**).
2. In the **New Connection** dialog, configure the following options:
   - **Connection Type:** `Serial Connection (GVRET)`
   - **Serial Port:** Select your ESP32's COM port (e.g., `COM8`)
3. Click **Create New Connection**. Your device should appear in the **Connected Devices** list:

   | # | Type | Subtype | Port | Buses | Status |
   | :- | :--- | :------ | :--- | :---- | :----- |
   | 1 | GVRET | gvret | COM8 | 1 | Connected |

4. Select the connected device and configure **Bus Details** (Bus `0`):
   - **Speed:** `125000` *(adjust to match your target CAN bus baud rate, e.g., `125000`, `250000`, `500000`)*
   - **Listen Only:** Checked *(recommended when passively sniffing traffic to avoid sending ACKs or error frames)*
   - **Enable Bus:** Checked
5. Click **Save Bus Settings** to start capturing CAN frames.


![SavvyCAN Connection](SavvyCAN%20Connection.png)

## License:

This software is MIT licensed:

Copyright (c) 2014-2020 Collin Kidder, Michael Neuweiler

Permission is hereby granted, free of charge, to any person obtaining
a copy of this software and associated documentation files (the
"Software"), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to
permit persons to whom the Software is furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be included
in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
