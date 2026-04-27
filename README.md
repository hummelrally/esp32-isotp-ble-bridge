# ISOTP BLE Bridge

The goal of this project is to build a native Macchina A0 (or ESP32 clone) firmware which can bridge BLE to ISOTP. The A0 has become increasingly more difficult to find in stock as of 2022, an alternative is to purchase the individual dev boards and assemble your own "clone".

---

## ESP32-C3 Super Mini — Branch Notes

> **This branch (`esp32c3-supermini`) is adapted for the [ESP32-C3 Super Mini](https://docs.nordicsemi.com/bundle/ncs-3.3.0-rc2/page/zephyr/boards/others/esp32c3_supermini/doc/index.html) board.**
> > It is forked from `Switchleg1/esp32-isotp-ble-bridge` (BridgeLEG branch), which targets the AMAleg DIY Macchina A0 clone.
> >
> > ### What changed from the original
> >
> > Two files were modified relative to the upstream BridgeLEG branch:
> >
> > **`main/constants.h` — GPIO reassignments**
> >
> > | Constant | Original (ESP32 A0) | This branch (ESP32-C3 Super Mini) | Reason |
> > |---|---|---|---|
> > | `SILENT_GPIO_NUM` | GPIO 21 | **GPIO 10** | GPIO 21 does not exist on C3; GPIO 10 is a safe available output |
> > | `LED_ENABLE_GPIO_NUM` | GPIO 13 | **GPIO 3** | GPIO 13 does not exist on C3; GPIO 3 is a safe available output |
> > | `LED_GPIO_NUM` | GPIO 2 | **GPIO 8** | The C3 Super Mini has its onboard WS2812 RGB LED hard-wired to GPIO 8 |
> > | `CAN_TX_PORT` | GPIO 5 | **GPIO 5** | Unchanged — available on C3 |
> > | `CAN_RX_PORT` | GPIO 4 | **GPIO 4** | Unchanged — available on C3 |
> >
> > **`sdkconfig.defaults` — New file (replaces ESP32-specific sdkconfig)**
> >
> > Key differences from the original ESP32 config:
> > - Target set to `esp32c3` (RISC-V architecture, not Xtensa)
> > - - CPU frequency capped at 160 MHz (C3 maximum; original used 240 MHz)
> >   - - FreeRTOS configured for unicore (C3 is single-core; original ESP32 is dual-core)
> >     - - BLE controller uses `CONFIG_BT_CTRL_*` keys (C3 API) instead of `CONFIG_BTDM_*` (ESP32-only API)
> >       - - Bluedroid stack retained for API compatibility with existing `ble_server.c` code
> >         - - Bootloader offset `0x0` (C3 standard) instead of `0x1000` (ESP32 standard)
> >           - - 4MB flash (standard on C3 Super Mini)
> >            
> >             - ---
> >
> > ## Building and Installing (ESP32-C3 Super Mini)
> >
> > This project uses the ESP32 native toolchain [ESP-IDF](https://docs.espressif.com/projects/esp-idf/en/stable/esp32c3/get-started/index.html) (based on FreeRTOS).
> >
> > ```bash
> > git clone https://github.com/hummelrally/esp32-isotp-ble-bridge
> > cd esp32-isotp-ble-bridge
> > git checkout esp32c3-supermini
> >
> > # Set the target chip and regenerate sdkconfig from sdkconfig.defaults
> > idf.py set-target esp32c3
> >
> > # Build
> > idf.py build
> >
> > # Flash via USB-C (board auto-enters download mode)
> > idf.py -p /dev/ttyUSB0 flash monitor
> > ```
> >
> > > On Windows, replace `/dev/ttyUSB0` with the appropriate COM port (e.g. `COM3`).
> > >
> > > After flashing the device will appear as a BLE peripheral named **`BLE2`** (default), connectable from SimosTools or VW_Flash.
> > >
> > > ---
> > >
> > > ## Parts List
> > >
> > > | Part | Notes |
> > > |---|---|
> > > | ESP32-C3 Super Mini | Any vendor; the onboard WS2812 LED on GPIO 8 is used for status |
> > > | SN65HVD230 breakout board | **Beware fakes** — see upstream README notes. Rectangle board only; avoid square boards and anything under $3 USD |
> > > | MP1584 buck converter | Set output to 5.0–5.5 V before connecting to the board |
> > > | OBD2 connector | Can be salvaged from an ELM adapter |
> > > | Schottky or fast-recovery diode | 1A 20V minimum — 1N5819, SR100, UF4004, etc. |
> > >
> > > ---
> > >
> > > ## Wiring Diagram
> > >
> > > ### Power
> > >
> > > | Connection | Detail |
> > > |---|---|
> > > | OBD2 Pin 16 (+12V) → [diode] → Buck converter IN+ | Schottky diode cathode toward buck converter |
> > > | OBD2 Pin 4 & 5 (GND) → Buck converter IN− | Common ground |
> > > | Buck converter OUT+ (5.0–5.5V) → ESP32-C3 Super Mini `5V` pin | |
> > > | Buck converter OUT− → ESP32-C3 Super Mini `GND` | Also ties to SN65HVD230 GND |
> > >
> > > > Set the buck converter output voltage **before** connecting the ESP32. Turn the trimmer clockwise ~1/10 of a turn from minimum until the output reads 5.0–5.5 V.
> > > >
> > > > ### CAN Bus (TWAI) — SN65HVD230
> > > >
> > > > | ESP32-C3 Super Mini | SN65HVD230 Board |
> > > > |---|---|
> > > > | `GPIO 5` | `TXD` |
> > > > | `GPIO 4` | `RXD` |
> > > > | `GPIO 10` | `RS` / `S` pin (silent mode control — pulled LOW by firmware for normal operation) |
> > > > | `3.3V` | `Vcc` |
> > > > | `GND` | `GND` |
> > > >
> > > > > If your SN65HVD230 breakout board has the `RS` pin already tied to GND, you can leave GPIO 10 unconnected.
> > > > >
> > > > > ### OBD2 Connector
> > > > >
> > > > > | OBD2 Pin | Connection |
> > > > > |---|---|
> > > > > | Pin 4 | Chassis GND |
> > > > > | Pin 5 | Signal GND |
> > > > > | Pin 6 | `CANH` (SN65HVD230 CANH) |
> > > > > | Pin 14 | `CANL` (SN65HVD230 CANL) |
> > > > > | Pin 16 | Vehicle +12V → diode → buck converter input |
> > > > >
> > > > > ### Status LED
> > > > >
> > > > > The C3 Super Mini has a **WS2812 RGB LED built onto the board at GPIO 8** — no external LED wiring is needed.
> > > > >
> > > > > ### Complete ASCII Wiring Summary
> > > > >
> > > > > ```
> > > > > OBD2 Pin 16 (+12V) --[1N5819]--.-- Buck IN+
> > > > > OBD2 Pin 4/5 (GND) ------------'-- Buck IN-
> > > > >
> > > > > Buck OUT+ (5.0-5.5V) -------------- ESP32-C3 5V pin
> > > > > Buck OUT- (GND) ----------------+-- ESP32-C3 GND
> > > > >                                 '-- SN65HVD230 GND
> > > > >
> > > > > ESP32-C3 GPIO 5  ------------------ SN65HVD230 TXD
> > > > > ESP32-C3 GPIO 4  ------------------ SN65HVD230 RXD
> > > > > ESP32-C3 GPIO 10 ------------------ SN65HVD230 RS/S pin
> > > > > ESP32-C3 3.3V    ------------------ SN65HVD230 Vcc
> > > > >
> > > > > SN65HVD230 CANH  ------------------ OBD2 Pin 6
> > > > > SN65HVD230 CANL  ------------------ OBD2 Pin 14
> > > > >
> > > > > Onboard WS2812 LED (GPIO 8) -- no external wiring needed
> > > > > ```
> > > > >
> > > > > ---
> > > > >
> > > > > ## 3D-Printed Case
> > > > >
> > > > > The C3 Super Mini is a similar width to the smaller KeeYee ESP32 boards, so the **SL1.3.5** case from the [AMAleg repo](https://github.com/Switchleg1/AMAleg) is the recommended fit:
> > > > >
> > > > > - [SL1.3.5-top.STL](https://github.com/Switchleg1/AMAleg/blob/main/SL1.3.5-top.STL) — for OBD connectors with 1.5mm flanges
> > > > > - - [SL1.3.5-bottom_half.STL](https://github.com/Switchleg1/AMAleg/blob/main/SL1.3.5-bottom_half.STL) — print x2
> > > > >  
> > > > >   - Always print with the logo facing up, with support material. You may need to oversize 1% depending on material and printer calibration.
> > > > >  
> > > > >   - ---
> > > > >
> > > > > ## Supported Software
> > > > >
> > > > > 1. [SimosTools](https://play.google.com/store/apps/details?id=com.app.simostools) — Android-based ECU flashing and logging software
> > > > > 2. 2. [VW_Flash](https://github.com/bri3d/VW_Flash) — Python-based ECU flashing and logging software
> > > > >    3. 3. Some other J2534 software
> > > > >      
> > > > >       4. ## Supported Hardware
> > > > >      
> > > > >       5. - Genuine Macchina A0: https://www.macchina.cc/catalog/a0-boards/a0-under-dash
> > > > >          - - AMAleg - DIY Macchina A0 clone: https://github.com/Switchleg1/AMAleg
> > > > >            - - **ESP32-C3 Super Mini** (this branch)
