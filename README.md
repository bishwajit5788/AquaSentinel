# AquaSentinel 💧
### Solar-Powered LoRa Water-Level Monitoring & Pump Automation

> **Monitor smarter. Prevent overflow.**

AquaSentinel is a three-node embedded system designed to monitor a water tank and automate pump control using ultrasonic measurements and LoRa communication. The project is being developed around three cooperating nodes—Sensor, Relay, and Master—with a focus on clear documentation, protective 3D-printed enclosures, modular firmware, and safety-first testing.

**Repository:** [bishwajit5788/AquaSentinel](https://github.com/bishwajit5788/AquaSentinel)

---

## Project status

**Current stage: planning and design.** The system architecture and intended behavior are described here. Final CAD models, verified wiring diagrams, node firmware, and hardware test results will be added after design review and testing. Planned behavior must not be mistaken for already implemented or verified behavior.

## At a glance

| Feature | Planned implementation |
|---|---|
| Water-level sensor | SR04M-2 waterproof ultrasonic sensor |
| Sensor controller | ESP32 DevKit WROOM-32 (38-pin) |
| Relay and Master controllers | Two NodeMCU ESP8266 boards |
| Wireless network | Three REYAX RYLR998 LoRa modules |
| User interface | One 0.96-inch SSD1306 I²C OLED (128×64), toggle switch, and buzzer |
| Pump interface | Relay node with correctly rated external switching hardware as required |
| Sensor-node power | Solar panel and two 18650 cells; charging topology requires final verification |
| Enclosures | Custom 3D-printed protective enclosures planned for all three nodes |
| App dependency | No Blynk dependency in the planned firmware |

## Three-node LoRa communication

The three LoRa radios form a **cooperating, bidirectional application-level communication system**. They do not automatically become a mesh network just because three radios are present: the firmware must define addressing, forwarding, commands, message validation, acknowledgements, retries, and timeouts.

~~~text
             SENSOR NODE
       ESP32 + SR04M-2 + LoRa
       Measures tank water level
                  |
                  | 1. TELEMETRY
                  v
              RELAY NODE
        ESP8266 + LoRa + relay
       Forwards data; controls pump
                  |
                  | 2. FORWARDED TELEMETRY
                  v
              MASTER NODE
       ESP8266 + LoRa + OLED
       Decides desired pump state
                  |
                  | 3. PUMP COMMAND
                  v
              RELAY NODE
       Validates command and changes
       pump-control output if safe
                  |
                  | status/acknowledgement,
                  | when implemented
                  v
              MASTER NODE
~~~

The Master can send a command to the Relay Node using LoRa; the Relay Node is the actuator node. Sensor measurements travel through the Relay Node to the Master. Acknowledgements and status messages should be implemented and tested so that the interface can distinguish a requested pump state from a confirmed relay state.

### Intended automation sequence

1. The Sensor Node measures the water-surface distance and checks whether the reading is plausible.
2. It sends a telemetry packet to the Relay Node.
3. The Relay Node validates the packet and forwards the latest valid measurement to the Master Node.
4. The Master Node evaluates the water level, sensor freshness, fault status, and user switch setting against the control policy.
5. The Master Node sends a pump ON/OFF command to the Relay Node.
6. The Relay Node validates the command and applies it only if local safety conditions allow it.
7. The system reports command/relay status where acknowledgements are implemented. Missing or stale communications must trigger a defined safe behavior.

This describes the planned flow. End-to-end behavior is not yet claimed as tested.

## Node responsibilities

### 1. Sensor Node — measure and transmit

**Hardware**
- ESP32 DevKit WROOM-32 (38-pin)
- SR04M-2 waterproof ultrasonic sensor
- REYAX RYLR998 LoRa module
- 6 V solar panel, two 18650 cells, compatible charger/protection, and regulated supplies

**Responsibilities**
- Measure distance to the water surface.
- Convert distance into an estimated level percentage using verified tank calibration.
- Transmit measurements and sensor status to the Relay Node.
- Use power-efficient operation where practical.

Initial calibration values under consideration: **22 cm = 100%** and **51 cm = 0%**. Verify these against the tank, sensor dead zone, and mounting position before use.

### 2. Relay Node — forward data and control the pump interface

**Hardware**
- NodeMCU ESP8266
- REYAX RYLR998 LoRa module
- 5 V relay module or suitable driver interface

**Responsibilities**
- Receive and validate Sensor Node telemetry.
- Forward valid telemetry to the Master Node.
- Receive and validate pump commands from the Master Node.
- Control the low-voltage relay/driver interface.
- Provide command acknowledgement or output status when implemented.

The relay module is not automatically suitable for switching a real pump. Depending on pump voltage, current, inrush, and installation, a correctly rated contactor, overload protection, enclosure, and qualified electrical installation may be required.

### 3. Master Node — display, decisions, and alerts

**Hardware**
- NodeMCU ESP8266
- REYAX RYLR998 LoRa module
- One 0.96-inch SSD1306 I²C OLED (128×64)
- SPST maintained toggle switch
- Buzzer

**Responsibilities**
- Display water level, pump state, and communication/system status.
- Decide the desired pump state using defined automation rules.
- Accept user input from the physical toggle switch.
- Signal defined faults and alerts with the buzzer.

**The switch must never override a full-tank cutoff, stale-telemetry cutoff, sensor-fault handling, or independent hardware protection.** Manual control is an input to the control policy, not permission to bypass safety rules.

## Planned pin and radio reference

These are the current design references, **not a verified connection diagram**. The dedicated wiring documents will be published after checking the exact board variants and module datasheets.

| Connection | Proposed assignment |
|---|---|
| Sensor MCU | ESP32 DevKit WROOM-32, 38-pin |
| Sensor TRIG / ECHO | GPIO32 / GPIO33 |
| Sensor LoRa UART | ESP32 GPIO16 (RX), GPIO17 (TX) |
| Relay MCU | NodeMCU ESP8266 |
| Relay output signal | D6, subject to relay-module polarity and boot testing |
| Relay LoRa UART | D1 (RX), D2 (TX), using a suitable software-UART arrangement |
| Master MCU | NodeMCU ESP8266 |
| Master OLED I²C | SDA D4, SCL D3; verify boot-strap compatibility |
| Master toggle switch | D5 with INPUT_PULLUP, subject to wiring validation |
| Logical radio addresses | Sensor 187, Relay 100, Master 200 |
| UART baud assumption | 9600; confirm/configure consistently on all radios |

Connect UART TX to the receiving device's RX and RX to TX. Verify logic levels and common reference ground for non-isolated signals. ESP32/ESP8266 GPIOs are not 5 V tolerant. The RYLR998 requires a regulated 3.3 V supply; do not feed its power pin 5 V. Confirm that all modules use compatible radio bands, antennas, network IDs, and RF parameters. Never transmit without the required antenna attached.

## Automation and overflow prevention

The control logic will be designed around safety states rather than relying only on a percentage displayed on the OLED.

- **Level thresholds:** define and document when filling should start and stop, including hysteresis to prevent rapid relay cycling.
- **Full-tank cutoff:** prohibit a pump-ON command at the high-water limit.
- **Stale data:** if a valid measurement is not received within the configured timeout, enter a defined safe state.
- **Sensor faults:** reject implausible, missing, or invalid readings; never treat a sensor fault as an empty tank.
- **Radio failure:** handle missing, malformed, duplicate, or out-of-order packets; implement acknowledgements/retries and sequence/session handling.
- **Safe boot and restart:** avoid unintended pump activation while either controller restarts.
- **Independent overflow protection:** use an independent, appropriately rated high-level float switch or equivalent hardware cutoff in the pump-control circuit. Firmware and LoRa communication alone cannot guarantee overflow prevention.
- **Fail-safe output:** choose and test the relay's safe default state for the real pump installation.

These are engineering requirements for implementation and testing—not claims that every protection is already implemented.

## Solar and battery safety

The current plan is a **6 V solar panel with two 18650 cells in series (2S)**.

- A standard 2S lithium-ion pack is typically 7.4 V nominal and 8.4 V fully charged.
- **The CN3065 is a single-cell charger and must not charge a 2S pack.**
- A generic boost converter does not replace a lithium battery charger.
- Use a solar charging controller explicitly compatible with the panel's real input range and a 2S lithium-ion pack, plus suitable 2S cell protection and balancing.
- Confirm cell type, cell datasheets, permitted charge current, panel wattage/current, charger input range, and BMS wiring before assembling or charging the pack.
- Use suitable regulated rails for the ESP32, sensor, and 3.3 V LoRa module.

The final charging circuit and exact parts are still to be selected and verified. Do not charge the planned 2S pack through the CN3065.

## Protective 3D-printed enclosures

Custom enclosures are planned for the Sensor, Relay, and Master nodes. Each model will account for:
- Exact board dimensions, mounting points, connector access, and serviceability.
- Cable routing, strain relief, and appropriate cable glands.
- Sensor aperture and mounting geometry.
- Heat dissipation, condensation, water ingress, and UV exposure.
- Lid fastening and sealing features appropriate to the intended environment.

A printed enclosure is not automatically waterproof. The finished print, seams, glands, material, and mounting orientation must be evaluated before outdoor use.

## Development roadmap

Only the introductory README and MIT license are currently published. The following artifacts will be added after planning and validation:

- [ ] Final bill of materials with exact part numbers and electrical ratings
- [ ] Complete three-node communication and control-flow diagram
- [ ] Separate, verified wiring diagram for Sensor, Relay, and Master nodes
- [ ] Final solar charger, 2S battery protection, and power-converter design
- [ ] 3D enclosure models for all three nodes
- [ ] Sensor Node firmware and standalone sensor test
- [ ] Relay Node firmware and safe relay-interface test
- [ ] Master Node firmware, OLED, switch, and buzzer test
- [ ] LoRa addressing, message validation, acknowledgements, retries, and timeout tests
- [ ] Integrated automation tests for normal filling, full tank, stale data, sensor fault, and radio loss
- [ ] Test evidence, build instructions, known limitations, and release notes

## Testing approach

1. Verify power rails and current draw before connecting the radios or sensor.
2. Test each peripheral independently.
3. Test each LoRa link with harmless test messages before enabling pump control.
4. Test malformed data, sensor disconnection, radio loss, duplicate packets, and controller restarts.
5. Test automation with an indicator or dummy load—not a live pump initially.
6. Verify full-tank cutoff and independent hardware overflow protection.
7. Perform supervised end-to-end testing before real installation.

Until test evidence is published, treat AquaSentinel as a work in progress. Do not rely on it as the only safeguard against flooding, dry running, electrical faults, or pump damage.

## Contributing

Design reviews and issue reports are welcome. Include board/module variants, wiring details, logs, and reproducible steps. Distinguish hardware-observed results from issues found by inspection or simulation.

## License

AquaSentinel is released under the [MIT License](LICENSE). See the license file for the full terms.

---

<p align="center">
  <strong>AquaSentinel</strong><br>
  <em>Monitor Smarter. Prevent Overflow.</em>
</p>
