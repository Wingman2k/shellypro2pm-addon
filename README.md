> **⚠️ Safety & Legal Disclaimer**
> * **Mains Voltage Hazard:** The Shelly Pro 2PM connects directly to **110–240 V AC mains voltage**. Always disconnect all mains power and verify zero voltage before opening the enclosure, disassembling internal sub-boards, or connecting a USB-TTL programmer.
> * **No Warranty / Use at Your Own Risk:** This documentation and the accompanying ESPHome configuration are based on independent reverse engineering and are provided "as is" without any warranty. Flashing custom firmware voids the manufacturer's warranty and may cause permanent hardware damage if wired incorrectly.
> * **Trademarks:** "Shelly" is a registered trademark of Shelly Europe Ltd. This project is an independent community effort and is not affiliated with or endorsed by Shelly Europe Ltd.


# Shelly Pro Add-On (STM32) – Hardware & UART Protocol Documentation

## 1. Tested Environment, Prior Work & Attribution

* **Tested Host Hardware:** Shelly Pro 2PM (`ESP32-D0WDQ6`, 8 MB Flash, dual `ADE7953` energy meters, `SN74HC595B` shift register, `LAN8720A` Ethernet)
* **Tested Host Firmware:**
  ```json
  {
    "name": "Pro2PMProAddon",
    "platform": "esp32",
    "version": "2.2.0",
    "build_id": "20261109-150431/2.2.0-g7968dca",
    "build_timestamp": "2026-11-09T15:04:31Z"
  }
  ```
* **Tested Add-On Co-Processor Firmware:** STM32 `FW v2.04` (Build stamp `26/10` / `0x1A 0x0A`)
* **Prior Work & Credits:**
  * **Base Shelly Pro 2PM Pinout & Hardware Reference:** Based on the community documentation at [ESPHome Devices – Shelly Pro 2 PM](https://esphome.io) (covering `ADE7953` SPI/reset behavior, `SN74HC595B` shift-register relay/LED mapping, `LAN8720A` Ethernet pins, and the `1.27 mm` programming header).
  * **Add-On Protocol Reverse Engineering:** Protocol framing, command IDs, checksum calculation, and hardware timings were reverse-engineered through live UART/GPIO sniffing, static firmware disassembly using **Ghidra** with **ghidra-mcp**, and iterative analysis assisted by **Gemini** as AI agent.
* **Author's Note (For Fellow Tinkerers):** This entire reverse-engineering effort—from initial concept to a fully working ESPHome installation—was completed within a single day with very little prior experience in protocol reverse engineering or Ghidra. Having a background as a qualified **Industrial Electronics Technician** certainly helped with the fine-pitch soldering and having basic test equipment on hand, so don't hesitate to tackle similar hardware integrations yourself!

---

## 2. Hardware Interface & Critical Flashing / Boot Notes

Unlike the *Shelly Plus Add-On* (which uses passive `ISO7221A` digital isolators and bit-banged 1-Wire), the **Shelly Pro Add-On** features an onboard **STM32 microcontroller** that handles all 1-Wire timing, ADC sampling, digital I/O, and relay switching internally over a 7-pin interface header.

### 7-Pin Header Pinout (Shelly Pro 2PM & Internal Add-On Interface)

* **Connector Pitch Note:** While the Shelly Pro 2PM programming header uses a **`1.27 mm`** pitch, the internal add-on interface board uses an intermediate **`2.00 mm`** pitch (between standard `2.54 mm` Dupont headers and `1.27 mm` micro-headers).

| Pin | ESP32 GPIO | Level / Type | Function & Behavior |
| :--- | :--- | :--- | :--- |
| **GND** | `GND` | `0 V` | Common ground. |
| **EN** | `GPIO0` | `3.3 V PWM` | **Enable / Watchdog Clock:** Continuous PWM signal at **`9,600 Hz`**, **`30%` duty cycle** (`32 µs HIGH` / `73 µs LOW`, `105 µs` period). Shared with ESP32 boot-mode strapping (`GPIO0` to `GND` on power-up) and `ADE7953 #1` SPI `CS` (`allow_other_uses: true`). |
| **RST** | `CHIP_PU` (`EN`) | `3.3 V Reset` | **ESP32 Hardware Reset Pin:** Connected to the ESP32's own `CHIP_PU` reset line (not connected to the STM32). Used exclusively by the programming interface to reset the ESP32. |
| **3,3V** | — | `3.3 V DC` | Logic supply voltage. |
| **RX** | `GPIO3` (`U0RXD`) | `3.3 V TTL` | UART RX (STM32 → ESP32), `57,600 Baud`, `8N1`, non-inverted. |
| **TX** | `GPIO1` (`U0TXD`) | `3.3 V TTL` | UART TX (ESP32 → STM32), `57,600 Baud`, `8N1`, non-inverted. |
| **12V** | — | `12 V DC` | Main power supply for the add-on relay coil and onboard 3.3 V regulator. |

### Critical Flashing & Booting Constraints
* **Must Be Flashed on the Bare Main PCB (All Sub-Boards Disconnected):** Both the external Shelly Pro Add-On **and** the Shelly Pro 2PM's internal sub-assemblies (relay assembly, power supply PCB, and terminal contact boards) must be disconnected before flashing via a USB-TTL serial adapter (`1.27 mm` pitch header on the Shelly Pro; `2.00 mm` pitch header on the internal add-on interface board). Because `GPIO1` (`TX`), `GPIO3` (`RX`), and `GPIO0` (`EN` / `ADE7953 #1` `CS`) are shared across these boards, leaving the add-on or internal sub-PCBs attached causes UART0 bus contention, interferes with `GPIO0` boot-mode strapping, and loads down the flashing signals.
* **Requires Full Reassembly & Mains Power to Boot Normally:** Standard USB-TTL 3.3 V adapters cannot supply enough current for the ESP32 + `LAN8720A` Ethernet PHY + internal sub-boards. After initial serial flashing on the bare main PCB, reassemble the device, attach the add-on, and power via mains for normal operation—using **OTA** for all subsequent firmware updates.
* **Shared `GPIO0` & `GPIO2` Notes (Shelly Pro 2PM):**
  * `GPIO0` serves both as `ADE7953 #1` SPI `CS` and as the `9,600 Hz` `EN` PWM output for the add-on (`allow_other_uses: true` required).
  * `GPIO2` is the active-low reset pin for both `ADE7953` energy meter chips (must be driven `HIGH` via an inverted `gpio` output) and is **not** connected to the add-on.

---

## 3. Communication Architecture (Master-Slave Polling)

* **Baud Rate & Framing:** `57,600 Baud`, `8N1` (`17.36 µs` bit width), non-inverted TTL.
* **Master-Slave Principle:** The STM32 never transmits sensor data unsolicited (except its `0xB0` boot frame immediately upon cold power-up). The ESP32 master must actively poll each channel in a round-robin loop.
* **Startup Sequence:** Because the ESP32 boots slower than the STM32 on cold power-up, the initial `0xB0` boot frame is missed unless the ESP32 explicitly requests it using `0x30` (`AA 55 02 30 00 CE`), followed by 1-Wire bus scans (`0x71` / `0x72`) on `CH0` and `CH1`.

---

## 4. Packet Framing & Checksum

Every UART frame on both TX and RX uses the exact same binary structure:

```text
+------+------+-----+-----+---------------+----+

| 0xAA | 0x55 | LEN | CMD |   PAYLOAD     | CS |
+------+------+-----+-----+---------------+----+
```

* **Header (`2 Bytes`):** Fixed `0xAA 0x55`.
* **LEN (`1 Byte`):** Number of bytes in `CMD + PAYLOAD` (excludes `0xAA 0x55` and `CS`).
* **CMD (`1 Byte`):**
  * **Master → STM32:** Bit 7 = `0` (`0x30`, `0x50`–`0x52`, `0x70`–`0x75`).
  * **STM32 → Master:** Bit 7 = `1` (`CMD + 0x80`, i.e., `0xB0`, `0xD0`–`0xD2`, `0xF0`–`0xF5`).
* **CS (`1 Byte`):** 8-bit one's complement checksum over all bytes starting from `0xAA`:
  $$\text{CS} = \text{0xFF} - \left( (\text{0xAA} + \text{0x55} + \text{LEN} + \text{CMD} + \sum \text{PAYLOAD}) \ \& \ \text{0xFF} \right)$$
  
## 5. Complete Command & Response Reference

| Function | Channel | Request (ESP32 → STM32) | Response (STM32 → ESP32) | Example Response Frame |
| :--- | :--- | :--- | :--- | :--- |
| **FW & Capabilities** | System | `AA 55 02 30 00 CE` | `AA 55 0E B0 [9B CAP] [4B FW] [CS]` | `AA 55 0E B0 00 00 01 01 00 00 01 00 00 02 04 1A 0A 15` |
| **Analog Input** | `0x00` / `0x01` | `AA 55 02 70 [CH] [CS]` | `AA 55 07 F0 [ST] [CH] [V3 V2 V1 V0] [CS]` | `AA 55 07 F0 00 00 00 00 00 07 02` (`Analog 0`)<br>`AA 55 07 F0 00 01 00 00 00 07 01` (`Analog 1`) |
| **1-Wire Scan 1** | `0x00` / `0x01` | `AA 55 02 71 [CH] [CS]` | `AA 55 03 F1 00 [CH] [CS]` | `AA 55 03 F1 00 00 0C` |
| **1-Wire Scan 2** | `0x00` / `0x01` | `AA 55 02 72 [CH] [CS]` | `AA 55 0C F2 00 [CH] [8B ROM-ID] [CS]`<br>`AA 55 04 F2 00 [CH] [COUNT] [CS]` | `AA 55 0C F2 00 00 00 28 69 FE DB 00 00 00 DA BE`<br>`AA 55 04 F2 00 00 01 09` |
| **DS18B20 Temp** | `0x00` / `0x01` | `AA 55 0A 73 [CH] [8B ROM] [CS]` | `AA 55 0D F3 [ST] [CH] [8B ROM] [TH] [TL] [CS]` | `AA 55 0D F3 00 00 28 69 FE DB 00 00 00 DA 00 B7 05` |
| **DHT22 Sensor** | `0x00` / `0x01` | `AA 55 02 74 [CH] [CS]` | `AA 55 03 F4 [ST] [CH] [CS]` *(Error)*<br>`AA 55 07 F4 00 [CH] [TH] [TL] [HH] [HL] [CS]` | `AA 55 03 F4 13 01 F5` |
| **Digital I/O** | `0x00` / `0x01` | `AA 55 02 75 [CH] [CS]` | `AA 55 04 F5 00 [CH] [VAL] [CS]` | `AA 55 04 F5 00 00 01 06` (`I/O0` = ON)<br>`AA 55 04 F5 00 01 01 05` (`I/O1` = ON) |
| **Relay ON** | Relay 0 | `AA 55 02 50 00 AE` | `AA 55 04 D0 00 00 01 2B` | `AA 55 04 D0 00 00 01 2B` |
| **Relay OFF** | Relay 0 | `AA 55 02 51 00 AD` | `AA 55 04 D1 00 00 00 2B` | `AA 55 04 D1 00 00 00 2B` |
| **Relay Status** | Relay 0 | `AA 55 02 52 00 AC` | `AA 55 04 D2 00 00 [VAL] [CS]` | `AA 55 04 D2 00 00 00 2A` (`0x00`=OFF, `0x01`=ON) |

---

## 6. Payload Decoding Details

* **Status Codes (`[ST]`):**
  * `0x00` = Valid reading / Command OK.
  * `0x12` = DS18B20 conversion in progress (first ~750 ms after power-up; temperature bytes read `0x00 0x00`).
  * `0x13` = DHT22 not connected or read timeout.
* **Firmware & Capabilities (`0xB0`):**
  * Bytes `1–9` (`00.00.01.01.00.00.01.00.00`): Hardware capability bitmask/counts.
  * Bytes `10–13` (`02.04.1A.0A`): Firmware version `v2.04`, build stamp `26/10`.
* **DS18B20 (`0xF3`, up to 5 per bus):** Signed 16-bit integer (`int16_t`, big-endian) `/ 10.0` in `°C`.
* **DHT22 (`0xF4`):** Temperature `int16_t` `/ 10.0` in `°C`; Humidity `uint16_t` `/ 10.0` in `%`.
* **Analog Input (`0xF0`):** Unsigned 32-bit integer (`uint32_t`, big-endian). Voltage (`V`) and percentage (`%`) scaling is handled on the ESP32.

---

## 7. Known Uncertainties & Open Items

While all primary functions (DS18B20, Digital I/O, Relay control/feedback, Firmware ID, and Analog channel polling) are verified and operational, the following minor details remain unverified or partially inferred:

1. **Exact Analog Input Scaling (`0xF0`):** The 32-bit raw value (`0x00000007` = `7` when floating/unconnected) was not calibrated against a known precision reference voltage (e.g., `3.30 V` or `10.00 V`). Whether the raw value is directly in **millivolts (`mV`)** or raw **12-bit ADC counts (`0–4095`)** scaled by a hardware voltage divider needs verification with a known input voltage.
2. **Negative Temperatures on DHT22 (`0xF4`):** Standard DHT22 sensors sometimes encode negative temperatures using the MSB sign bit (`0x8000`) rather than two's complement (`int16_t`). Because `0xF4` was only observed in an unconnected state (`0x13` error) and positive room temperatures, sub-zero DHT22 encoding on the STM32 is inferred as `int16_t` (matching `0xF3`) and unverified below `0 °C`.
3. **Individual Byte Mapping of `0xB0` Capabilities (`Bytes 1–9`):** While `00.00.01.01.00.00.01.00.00` clearly represents the board's hardware profile, the exact meaning of each individual byte position (e.g., which byte corresponds to relay count vs. 1-Wire bus count) is inferred rather than confirmed against other Shelly Pro Add-On hardware variants.
4. **Hot-Plugging 1-Wire Sensors Without Power Cycle:** Because the `RST` pin on the 7-pin header connects to the ESP32's `CHIP_PU` rather than the STM32's reset pin, it is unverified whether the STM32 detects newly hot-plugged DS18B20 sensors via `0x71`/`0x72` without a full power cycle of the add-on board.
5. **Exact Hardware Purpose of `9.6 kHz` PWM on `EN` (`GPIO0`):** While the stock firmware outputs `9.6 kHz` at `30%` duty cycle on `GPIO0` (shared with `ADE7953 #1` `CS`), it is unconfirmed whether `EN` acts as a hardware charge-pump/watchdog enable for the relay coil or as a synchronization clock for the STM32.
6. **Undocumented / Unused Command Opcodes:** Other command bytes in the `0x30`–`0x7F` range (e.g., `0x76` or bootloader/OTA update commands for the STM32 itself) may exist in the STM32 firmware but were not triggered during normal operation.
