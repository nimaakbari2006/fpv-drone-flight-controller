Custom FPV Drone — Power Distribution Board & Flight Controller

Two custom PCBs designed from scratch in KiCad for a 5-inch FPV drone, plus complete component selection and compatibility validation for the full build. This project covers the full stack: switching power supply design, a Betaflight-compatible flight controller, and systems-level integration of the entire aircraft.

Overview

The goal was to replace the stock electronics of a 5-inch FPV drone with custom-designed hardware, as an electrical engineering portfolio project. Two boards were designed:

Power Distribution Board (PDB) — takes 4S LiPo battery input and generates regulated 5V and 3.3V rails while distributing raw battery power to the ESC.
Flight Controller (FC) — an STM32F405-based, Betaflight-compatible flight controller with a full peripheral set.

Both boards were designed to the standard 30.5 × 30.5 mm FPV stack mounting pattern to integrate with the airframe and ESC.

Power Distribution Board

A dual-output switching power supply feeding the drone's electronics.

Key features:

4S LiPo (up to ~16.8V) input with XT60 connection
Two synchronous buck converters (MP2307) generating 5V and 3.3V
Input protection: SM6T22A TVS diode against voltage spikes
Raw battery pass-through (VBATOUT) to the 4-in-1 ESC via wide copper
Feedback dividers, compensation networks, soft-start, and proper input/output decoupling on each channel

Design decisions:

Chose the MP2307 synchronous buck over the older LM2596 after finding that the LM2596's through-hole inductors exceeded the frame's stack-height clearance — a real mechanical constraint that drove the part selection toward a smaller SMD synchronous design.
Replaced the large through-hole bulk capacitor with an SMD polymer cap to fit the board within the mounting footprint.

(See /PDB/ for schematic, layout, 3D render, and Gerbers.)

Flight Controller

A full-featured, Betaflight-compatible flight controller.

Core:

STM32F405RGT6 MCU (LQFP-64)
ICM-42688-P gyro/accelerometer over SPI1
8 MHz crystal, full power decoupling, VCAP, reset & boot circuitry
USB-C for configuration and firmware flashing (with CC pulldowns and power-OR Schottky)
Onboard 3.3V LDO so the board runs from USB alone on the bench

Peripherals:

AT7456E OSD for on-screen display over the analog video feed (SPI3)
W25Q128 SPI flash for blackbox flight logging
4 motor outputs (DShot-capable timer pins) + ESC telemetry & current sense
ELRS receiver UART, VTX (SmartAudio) control, GPS UART
Analog camera input and VTX video output through the OSD
Battery voltage sensing, buzzer driver, addressable LED output
SWD debug header

Design decisions:

Followed the official Betaflight F405 pin mapping so the board matches an existing firmware target rather than requiring a custom one (gyro on SPI1, OSD/flash sharing SPI3, careful UART assignment to avoid the DFU/USART conflict).
4-layer stackup with a dedicated internal ground plane beneath the IMU for clean gyro readings, and controlled routing of the USB differential pair.

(See /FlightController/ for schematic, layout, 3D render, and Gerbers.)

Circuit Design & Component Selection

The design methodology was datasheet-driven: each IC's manufacturer datasheet specifies a recommended application circuit, and following those recommended values — for decoupling, compensation, loop stability, and support components — is the correct engineering practice rather than guessing or reinventing them. Where the design required application-specific values that the datasheet does not give directly (chiefly the regulator output voltages and the battery-sense scaling), those were calculated from the relevant equations. This section documents both.

Values sourced from datasheets and reference circuits

The majority of passive values came directly from the manufacturers' recommended application circuits:

MCU decoupling — the STM32F405 datasheet specifies a 100 nF ceramic at each VDD pin plus a shared bulk capacitor; the 2.2 µF low-ESR capacitors on the VCAP core-regulator pins are a hard datasheet requirement (the internal core regulator will not start without them).
Analog supply filtering — the 600 Ω @ 100 MHz ferrite bead plus 1 µF / 100 nF VDDA filter follows ST's recommended arrangement to keep digital switching noise off the ADC reference.
Buck converter support — the MP2307's loop-compensation R–C network, 10 nF soft-start capacitor, 10 nF bootstrap capacitor, and input/output capacitor values were taken from the MP2307 datasheet's typical application circuit for the chosen inductor and output-cap combination.
IMU — the ICM-42688-P's supply decoupling (2.2 µF + 100 nF) follows TDK's reference circuit, and its reserved pins were terminated exactly as the datasheet pin-description table specifies.
OSD — the on-screen-display support components and crystal frequency were taken from the OSD device's datasheet and a known-good reference design.

Using the datasheet's recommended circuit is deliberate: manufacturers characterise these values for stability and noise performance, and deviating from them without reason is how power supplies end up oscillating or ICs end up unstable.

Values calculated for this design

A few values are application-specific and were derived rather than copied:

Buck converter output voltages (feedback dividers). The MP2307 servos its feedback pin to an internal 0.925 V reference; the output is set by a divider:

Vout = 0.925 V × (1 + Rtop / Rbot)

With Rbot fixed at 10 kΩ, Rtop was solved for each rail:

5V rail: Rtop ≈ 44.05 kΩ → nearest E96 value 44.2 kΩ (gives 5.01 V)
3.3V rail: Rtop ≈ 25.68 kΩ → nearest E96 value 25.5 kΩ (gives 3.28 V)

1% (E96) resistors were used so output accuracy is set by the reference and divider, not resistor drift.

Battery voltage sensing. A 100 kΩ / 10 kΩ divider scales the raw battery voltage down to the STM32's 3.3 V ADC range: at 4S full charge (16.8 V) the ADC sees ~1.5 V, at 6S (25.2 V) ~2.3 V — both safely under the 3.3 V limit with headroom. This matches Betaflight's default voltage-sensing scale.

Crystal load capacitors. Sized from the crystal's specified load capacitance and estimated board stray capacitance using Cload_cap ≈ 2 × (CL − Cstray), giving ~22 pF per oscillator node.

Values set by standards
5.1 kΩ pull-downs on the USB-C CC pins — the specific value required by the USB-C specification for a device to advertise itself and draw bus power.
75 Ω termination on the analog video lines — matches the standard 75 Ω video signal impedance.
Values from standard digital-design practice

Pull-ups and pull-downs were chosen high enough to draw negligible current but low enough to hold a defined logic level against pin leakage — 10 kΩ for the NRST pull-up, BOOT0/BOOT1 pull-downs, and the SPI flash/OSD chip-select and hold lines; 100 kΩ for the buzzer MOSFET gate pull-down so it stays off while its GPIO is floating during boot.

Full Drone Build & Component Selection

Beyond the PCBs, the project involved specifying and validating an entire FPV drone, ensuring every component is electrically and mechanically compatible.

Key selection decisions:

Motors: iFlight XING 2207 1800KV — the correct kV range for a 4S 5-inch freestyle build (higher-kV variants are 2–4S racing parts).
Video system: a fully analog 5.8GHz chain — RunCam Robin 3 → JHEMCU VTX20-600 → FlyFishRC Osprey antenna → goggles — with RHCP polarization matched from drone to goggles.
Control link: ELRS 2.4GHz (RadioMaster Pocket + Nano RX), both on FCC firmware for region compatibility.
ESC: 55A rating, sized comfortably above the motors' ~35A peak draw.

Interface & connector compatibility (each verified against the actual component pinouts):

FC ↔ ESC: 8-pin JST-SH (motor signals, telemetry, current, power)
FC ↔ Camera: 6-pin JST-SH, wired to the RunCam Robin 3's pinout (power, GND, video, camera control)
FC ↔ VTX: 4-pin (video, GND, battery, SmartAudio)
FC ↔ GPS / Receiver: JST-SH UART connectors
Video RF: MMCX (VTX to antenna), RHCP polarization matched throughout

(See /BOM/FPV_Drone_BOM.xlsx for the complete bill of materials with compatibility notes.)

Tools & Skills

KiCad · schematic capture · PCB layout · 4-layer stackup design · switching power supply design · SPI/UART bus architecture · USB routing · DRC/ERC verification · library management (symbol/footprint import) · datasheet-driven design · systems integration · component selection & compatibility analysis

Status

Both boards are fully designed and verified in KiCad (ERC and DRC clean). Manufacturing files (Gerbers, BOM, placement) generated for fabrication and assembly. Full drone build specified and components sourced.

Physical assembly and flight testing to follow — this README will be updated with assembly photos and results.
