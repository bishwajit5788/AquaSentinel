# AquaSentinel 💧
### Solar-Powered LoRa Water-Level Monitoring & Pump Automation

> **Monitor smarter. Prevent overflow.**

AquaSentinel is a three-node embedded system designed to monitor a water tank remotely and automate pump control using ultrasonic distance measurements and long-range LoRa communication. The project is being developed with an emphasis on clear documentation, modular firmware, practical enclosure design, and safe testing before deployment.

**Repository:** [bishwajit5788/AquaSentinel](https://github.com/bishwajit5788/AquaSentinel)

---

## Project at a glance

| Feature | Planned implementation |
|---|---|
| Water-level sensing | SR04M-2 waterproof ultrasonic sensor |
| Sensor controller | ESP32 DevKit WROOM-32 |
| Wireless communication | REYAX RYLR998 LoRa modules |
| Distributed architecture | Sensor, Relay, and Master nodes |
| Pump switching | ESP8266 relay node and appropriately rated switching hardware |
| User interface | ESP8266 Master with one 0.96-inch SSD1306 OLED, toggle switch, and buzzer |
| Sensor-node power | Solar panel, rechargeable 18650 pack, compatible charger, protection, and regulators |
| Firmware | Separate firmware for each node; no Blynk dependency in the planned design |
| Enclosures | 3D-printed protective enclosures planned for all three nodes |

> **Development status: Planning and design.** Final enclosure CAD, verified connection diagrams, node firmware, and hardware test results have not yet been published. The architecture below describes the current plan, not a claim that the complete system is tested.

## System architecture

AquaSentinel separates sensing, forwarding/actuation, and user interaction into three nodes. This modular design is intended to make each node easier to build and test independently.

~~~text
┌────────────────────────────┐
│ SENSOR NODE                │
│ ESP32 + SR04M-2             │
│ Measures tank distance     │
│ Sends level telemetry      │
└──────────────┬─────────────┘
               │ LoRa
               ▼
┌────────────────────────────┐
│ RELAY NODE                 │
│ ESP8266 + relay interface  │
│ Forwards telemetry         │
│ Receives pump commands     │
└──────────────┬─────────────┘
               │ LoRa
               ▼
┌────────────────────────────┐
│ MASTER NODE                │
│ ESP8266 + OLED + switch    │
│ Displays level and status  │
│ Applies pump-control rules │
│ Buzzer for alerts          │
└────────────────────────────┘
~~~

The exact command flow, acknowledgements, timeout behavior, and fail-safe rules will be documented and tested before the firmware is treated as deployment-ready.

## The three nodes

### 1. Sensor Node — measurement

**Planned hardware**
- ESP32 DevKit WROOM-32 (38-pin)
- SR04M-2 waterproof ultrasonic sensor
- REYAX RYLR998 LoRa module
- Solar panel, rechargeable battery pack, charging/protection hardware, and regulated power rails

**Planned responsibilities**
- Measure the distance from the sensor to the water surface.
- Convert distance to an estimated level percentage using calibrated empty/full reference points.
- Transmit readings and sensor status to the Relay Node.
- Operate efficiently on solar power, with low-power behavior considered during implementation.

Initial calibration values under consideration are **22 cm = 100%** and **51 cm = 0%**. These values must be checked against the actual tank geometry and sensor mounting before use.

### 2. Relay Node — forwarding and pump interface

**Planned hardware**
- NodeMCU ESP8266
- REYAX RYLR998 LoRa module
- 5 V relay module or suitable driver interface

**Planned responsibilities**
- Receive and validate Sensor Node telemetry.
- Forward valid readings to the Master Node.
- Receive pump-state commands from the Master Node.
- Control the low-voltage relay interface and report status where supported.

The relay node is not a substitute for independent electrical protection. A real pump may require a correctly rated contactor, overload protection, a suitable enclosure, and installation by a qualified person.

### 3. Master Node — display and control

**Planned hardware**
- NodeMCU ESP8266
- REYAX RYLR998 LoRa module
- One 0.96-inch SSD1306 I²C OLED (128 × 64)
- SPST maintained toggle switch
- Buzzer

**Planned responsibilities**
- Display water level and communication/system status.
- Apply configured automatic pump-control rules.
- Accept physical user input through the toggle switch.
- Signal defined alerts through the buzzer.

Manual input must **never bypass the tank-full cutoff, stale-data timeout, or sensor-fault safety behavior**. Exact control thresholds and alert patterns will be documented before implementation is considered complete.

## Planned hardware and communication reference

| Item | Current design reference |
|---|---|
| Sensor MCU | ESP32 DevKit WROOM-32, 38-pin |
| Relay MCU | NodeMCU ESP8266 |
| Master MCU | NodeMCU ESP8266 |
| Radio | REYAX RYLR998 on all nodes |
| Logical radio addresses | Sensor 187, Relay 100, Master 200 |
| UART assumption | 9600 baud; confirm against module configuration |
| Sensor interface | TRIG GPIO32, ECHO GPIO33 on ESP32 |
| Sensor radio UART | ESP32 GPIO16 (RX), GPIO17 (TX) |
| Relay control signal | ESP8266 D6, subject to relay-module polarity and boot validation |
| Relay radio UART | ESP8266 D1 (RX), D2 (TX), using a suitable software UART arrangement |
| Master OLED | I²C SDA D4, SCL D3 — verify ESP8266 boot behavior with the chosen display |
| Master toggle switch | D5 with INPUT_PULLUP, subject to final wiring validation |

**These are proposed reference assignments, not a verified wiring diagram.** Check each board's pin labels and electrical limits before connecting hardware. The RYLR998 supply is 3.3 V; do not power its supply pin from 5 V. Confirm the radio variant, legal operating band, antenna, and matching radio parameters on all nodes. Never transmit without the required antenna connected.

## Solar and battery safety

The Sensor Node is planned to use a **6 V solar panel and two 18650 cells intended to be connected in series (2S)**.

- A 2S lithium-ion pack is typically 7.4 V nominal and 8.4 V fully charged when using standard 4.2 V-per-cell cells.
- **The CN3065 is a single-cell charger and must not be used to charge a 2S pack.**
- A generic boost converter is not a battery charger.
- Use a solar charging solution explicitly compatible with the panel's operating range and a 2S lithium-ion pack, plus suitable 2S cell protection and balancing.
- Confirm the exact cell specifications, permitted charge current, panel power, and charger design before assembling or charging the pack.
- Use regulated supplies appropriate for the ESP32, ESP8266, sensor, and radio. Verify all signal voltages; ESP GPIOs are not 5 V tolerant.

Charging topology and parts selection remain **unfinalized**. Do not use the planned battery arrangement until the charger, protection circuit, and wiring have been verified.

## Safety-first control principles

The implementation and testing plan will cover the following:

1. **High-water cutoff:** stop pump filling at the configured full level.
2. **Stale telemetry:** transition to a defined safe state if valid sensor updates stop.
3. **Sensor fault handling:** reject invalid readings rather than treating them as trustworthy level data.
4. **Radio loss and malformed messages:** validate packets and define timeout/recovery behavior.
5. **Safe startup:** avoid unintended pump activation while controllers boot or restart.
6. **Independent overflow protection:** use a suitably rated, independent high-level float switch or equivalent hardware cutoff in the pump-control circuit. Firmware alone is not independent overflow protection.
7. **Electrical isolation and ratings:** size switching components for the actual pump and supply; keep hazardous mains wiring out of hobby-level prototype wiring.

These are design requirements, not a statement that all protections are already implemented or tested.

## Enclosures and environmental protection

Protective 3D-printed enclosures are planned for the Sensor, Relay, and Master nodes. Each enclosure design will need to account for:

- Board dimensions, connector access, cable routing, and strain relief.
- Sensor opening and mounting geometry.
- Ventilation or thermal considerations where needed.
- Water ingress, condensation, UV exposure, and the limits of the chosen print material.
- Serviceability: lids, fasteners, and access for debugging or replacing parts.

A 3D-printed enclosure should not be assumed waterproof merely because it is closed. Environmental sealing and outdoor suitability must be evaluated for the final print, seams, cable glands, and mounting orientation.

## Repository roadmap

Only this introductory README is being published at the current planning stage. Additional files will be added after the design has been reviewed.

- [ ] Final bill of materials with exact module variants and ratings
- [ ] Verified system block diagram and separate connection diagram for each node
- [ ] Solar charging and battery-protection design review
- [ ] 3D enclosure models for Sensor, Relay, and Master nodes
- [ ] Sensor Node firmware and isolated sensor test
- [ ] Relay Node firmware and relay-interface test
- [ ] Master Node firmware, OLED, switch, and buzzer test
- [ ] LoRa integration, packet validation, timeout, and recovery tests
- [ ] End-to-end pump-control tests, including fault injection
- [ ] Build instructions, test evidence, and known limitations

## Testing philosophy

Each node will be tested independently before the complete system is integrated:

1. Verify power rails and current draw.
2. Test peripherals locally without activating a real pump.
3. Verify radio communication with test messages.
4. Test invalid readings, disconnected sensors, radio loss, and controller restarts.
5. Verify full-tank cutoff and independent overflow protection.
6. Run supervised end-to-end tests before any real installation.

Until test evidence is published, treat this project as a work in progress and do not rely on it as the only protection against flooding or pump damage.

## Contributing and feedback

Suggestions, issue reports, and design reviews are welcome. Please include the relevant board/module variant, wiring details, logs, and reproducible steps when reporting a problem. Safety-related issues should clearly state whether the behavior was observed on hardware or identified during design review.

## License

No license has been selected yet. Until a license is added, all rights remain with the copyright holder; do not assume the repository is licensed for reuse, modification, or redistribution.

---

<p align="center">
  <strong>AquaSentinel</strong><br>
  <em>Monitor Smarter. Prevent Overflow.</em>
</p>
