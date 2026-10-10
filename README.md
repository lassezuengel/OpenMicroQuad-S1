# OpenMicroQuad 1S

An experimental, custom-designed micro-quadcopter built around two stacked PCBs: a flight controller (FCU) with an integrated ExpressLRS (ELRS) radio transceiver and Bosch inertial sensor, and an AM32-based 4-in-1 electronic speed controller (ESC) that plugs onto the FCU from above. Both PCBs were designed in Altium Designer for a single-cell LiPo/LiHV battery.

This repository documents the **1S design**, including its hardware files and limitations.

<p align="center">
  <a href="assets/E-assembly.png"><img src="assets/E-assembly.png" width="600" alt="3D render of the assembled PCB stack, with the ESC above the FCU"></a>
  <br>
  <strong>Board Stack</strong>
  <br>
  <sub>The 4-in-1 ESC plugs onto the flight controller from above.</sub>
<table>
  <tr>
    <td align="center" width="50%">
      <a href="assets/FCU.png"><img src="assets/FCU.png" width="400" alt="3D render of the FCU board with its RP2040, SX1280 radio, and BMI270 sensor"></a>
      <br>
      <strong>Flight controller</strong>
      <br>
      <sub>RP2040, SX1280 ELRS radio, and Bosch BMI270.</sub>
    </td>
    <td align="center" width="50%">
      <a href="assets/ESC.png"><img src="assets/ESC.png" width="400" alt="3D render of the four-channel ESC board and its motor connectors"></a>
      <br>
      <strong>4-in-1 ESC</strong>
      <br>
      <sub>Four AM32 channels, four motors.</sub>
    </td>
  </tr>
</table>
</p>


## Hardware overview

| Subsystem | Implementation |
| --- | --- |
| Flight controller | RP2040 with external W25Q16 QSPI flash |
| Radio | Semtech SX1280IMLTRT transceiver for the 2.4 GHz ELRS link, connected to the RP2040 over SPI; U.FL antenna connector |
| Inertial sensor | Bosch BMI270 accelerometer and gyroscope, connected to the RP2040 over a separate SPI bus |
| ESC | Four independent AM32-based channels, each with an STM32G071GBU6 and MP6531A gate driver |
| Power stage | 24 CSD15571Q2 N-channel MOSFETs, forming three half-bridges per motor |
| Supply | 1S LiPo/LiHV; ESC TPS61023 boost regulator for the 5 V driver supply and MIC5323 LDO for 3.3 V logic |
| Motors | 1S, ~22,000 KV Air75-class motors |
| Construction | ESC plugged onto the FCU through four six-pin headers, with a 3D-printed frame |

## Two-board architecture

The **FCU board (`RP2040-FPV`)** handles flight control, radio communication, and motion sensing. The SX1280 radio and BMI270 sensor connect directly to the RP2040 over separate SPI buses. The board also includes USB-C, battery-voltage sensing, an RGB status LED, BOOTSEL/reset/user buttons, and an SWD debugging connector.

The **upper ESC board (`ESC-FPV`)** drives four three-phase brushless motors. Each channel has its own STM32 running AM32, an MP6531A gate driver and three half-bridges made up of six MOSFETs. The FCU sends one DShot signal to each channel; each motor has a separate three-pin phase connector.

```mermaid
flowchart LR
    subgraph FCU["FCU board: RP2040-FPV"]
        Radio["SX1280 ELRS radio"] <-->|SPI| MCU["RP2040 flight controller"]
        IMU["BMI270 accelerometer / gyro"] <-->|SPI| MCU
    end
    subgraph ESC["Upper board: ESC-FPV"]
        Channels["4 AM32 channels: STM32G071 + MP6531A + 6 MOSFETs each"]
    end
    MCU -->|4 DShot signals| Channels
    Channels --> Motors["4 three-phase BLDC motors"]
    Battery["1S battery"] --> FCU
    Battery --> ESC
```

The boards share **VBAT and GND**, while their regulated supplies remain separate. The FCU selects battery or USB power and generates its own 3.3 V supply. The ESC boosts battery voltage to 5 V for its gate drivers, then regulates that to 3.3 V for its MCUs; the motor bridges run directly from VBAT. With only USB connected, the FCU is powered and the ESC is off, so _FCU firmware should avoid driving DShot into the unpowered ESC_. UART, I2C, and an ADC signal are also brought to the stack headers but are currently unused on the ESC.

## Design files

| Board | Altium project | Schematic and PCB PDF | Additional outputs |
| --- | --- | --- | --- |
| FCU | [RP2040-FPV.PrjPcb][fcu-project] | [RP2040-FPV.pdf][fcu-pdf] | No separate manufacturing archive or BOM included |
| ESC | [ESC-FPV.PrjPcb][esc-project] | [ESC-FPV.pdf][esc-pdf] | [Project Outputs for ESC-FPV.zip][esc-outputs]: Gerber X2 files, drill layers/reports, CSV BOM, pick-and-place CSV, and archived DRC reports |

The FCU project is organized as follows:

| File | Purpose |
| --- | --- |
| `MCU.SchDoc` | RP2040, QSPI flash, clock, and signal assignments |
| `RF.SchDoc` | SX1280 radio, oscillator, RF matching/filter network, and antenna connection |
| `Interfacing.SchDoc` | BMI270, stack headers, debug connector, buttons, RGB LED, and battery sensing |
| `Power.SchDoc` | Battery/USB power selection, USB-C interface, and 3.3 V regulation |
| `PCB.PcbDoc` | FCU board layout |

The ESC project is organized as follows:

| File | Purpose |
| --- | --- |
| `Main.SchDoc` | Top-level hierarchy joining four motor-driver channels, power, and interfacing |
| `MotorDriverChannel.SchDoc` | Reusable single-motor controller, gate driver, MOSFET bridge, and back-EMF sensing |
| `Power.SchDoc` | Battery input, 5 V boost converter, and 3.3 V regulation |
| `Interfacing.SchDoc` | FCU stack headers, per-channel SWD programming test points, and repeated motor connections |
| `MotorConnection.SchDoc` | Three-phase connector, repeated once per motor |
| `PCB.PcbDoc` | Combined four-channel ESC board layout |


## Firmware

In order to fly the OpenMicroQuad 1S, you need firmware for both the FCU and ESC boards. The FCU runs a version of Betaflight extended to support the RP2040 platform, while each ESC (STM32) runs on modified AM32 firmware.

### Betaflight firmware

If you don't want to build the Betaflight firmware yourself, see the `/bin` directory for prebuilt firmware for the RP2040-FPV board. The firmware is configured for a 1S battery, SX1280 ELRS radio, and BMI270 sensor. Possibly outdated, but should work fine for basic flight.

#### Building Betaflight for the RP2040-FPV board

See the fork [here](https://github.com/lassezuengel/betaflight). You can build the firmware for the RP2040-FPV board using the `RP2040FPV` target in Betaflight. The firmware is configured for a 1S battery, SX1280 ELRS radio, and BMI270 sensor.

```shell
git clone --recurse-submodules https://github.com/lassezuengel/betaflight
cd betaflight

git submodule update --init --recursive

make SDK=pico_sdk platform-sdk-hydrate
make SDK=pico_sdk platform-tools-install
make configs
```

After that, you can build the firmware for the RP2040-FPV board:

```shell
make -j4 RP2040FPV
```
´
  The target `.elf` file will be located in `obj/betaflight_<version>_RP2040_RP2040FPV.uf2`.

#### Flashing the FCU
´´´´
1. Disconnect power/USB or hold reset
2. Hold the board’s BOOTSEL button or pull the RP2040 QSPI_SS boot-select signal low.
3. Connect USB while continuing to hold BOOTSEL, release reset.
4. A USB drive named RPI-RP2 should appear, you can now release BOOTSEL.
5. Copy the generated `.uf2` file onto that drive.
6. The drive disconnects automatically and the FC reboots into Betaflight.

Use the `default/v3` image when your radio is running ExpressLRS 3.x. Use the `v4` build when it is running ExpressLRS 4.x.

### AM32 bootloader

Use the official AM32 bootloader to flash the `STM32G071` MCUs on the ESC-FPV board. The bootloader is available from [AM32's GitHub](https://github.com/AM32/AM32-Bootloader). You will need a bootloader compatible with our `G071` MCU that uses pin `PB4` as the communication pin. Consider using `AM32_G071_BOOTLOADER_PB4_V19.hex`.

For flashing the bootloader to each STM32, you can use any old ST-Link V2 programmer or a SWD programmer of your choice. The ESC-FPV board has test points for SWD programming, which are accessible from the top side of the board. Power the ESC-FPV board with a 1S battery or a bench PSU, and connect the SWD programmer to the test points (as well as `GND` and `+3V3`). Then use the STM32CubeProgrammer or any other SWD programmer software to flash the bootloader. Repeat all this for each of the four STM32s on the ESC-FPV board.

<p align="center">
  <a href="assets/E-assembly.png"><img src="assets/G071_bootloader_flashing.png" width="600" alt="Test points / contact pads for flashing the bootloader to STM32s"></a>
  <br>
</p>

We recommend using pogo pins to connect to the test points. Of course, you can also (temporarily) solder wires to the test points if needed. You only have to do this once (per STM32) to flash the bootloader. After that, you can use the AM32 online configurator to flash the firmware to all four STM32s at once.

### AM32 firmware

Again, if you don't want to build the AM32 firmware yourself, see the `/bin` directory for prebuilt firmware for the ESC-FPV board. The firmware is configured for a 1S battery and four AM32 channels.

#### Building AM32 firmware for the ESC-FPV board

If you want to build the AM32 firmware for the ESC-FPV board yourself, you can use our fork of AM32.

Clone our AM32 fork:

```shell
git clone --branch main --recurse-submodules https://github.com/lassezuengel/AM32.git
cd AM32
git submodule update --init --recursive
```

Update your system and install the required dependencies:

```shell
sudo apt-get update
sudo apt-get install -y git make wget tar python3
```

Install the ARM SDK (if you haven't already):

```shell
make arm_sdk_install
```

Finally, build the firmware for the ESC-FPV board:

```shell
make clean
make ESC_FPV_G071
```

#### Flashing the ESC

The output can be found in `obj/AM32_ESC_FPV_G071_2.21.bin`. If you have already installed the AM32 bootloader for each of the four STM32G071 MCUs, you can flash the firmware using the [AM32 online configurator](https://am32.ca). Firmware upgrading is done via DShot commands using the FCU, so you can just connect the stack to your computer via USB, power the ESC board using a battery or PSU, and use the AM32 configurator to flash the firmware to all four ESC channels.

## Measurements

<p align="center">
  <!-- The 4:3 photo and 3:4 thermal image need 64:36 widths for equal heights. Keep the links adjacent so both fit on one line. -->
  <a href="assets/setup-stationary.png"><img src="assets/setup-stationary.png" width="64%" alt="Stationary setup with the flight controller and ESC"></a><a href="assets/thermal_cam_64W_4A.png"><img src="assets/thermal_cam_64W_4A.png" width="36%" alt="Thermal image of the 4-in-1 ESC at 64W power consumption and 4A current"></a>
  <br>
  <sub><strong>Left:</strong> Drone setup at 18°C ambient temperature. BetaFPV 22KV motors and Air75 freestyle props.</sub>
  <br>
  <sub><strong>Right:</strong> Thermal image of the 4-in-1 ESC at 64W power consumption and 4A current.</sub>
</p>

We can typically see 64W total power draw at full throttle at 4V battery voltage, which is about 16W (4A) per motor. Our ESC is designed to handle 8A continuously and can probably easily handle 10-15A per channel, so we have plenty of headroom. PCB temperature (stationary, no airflow) is about 35°C at full throttle, and the MOSFET hotspot temperature is about 45°C. The 1.25mm PicoBlade motor connectors are the bottleneck here, with sometimes as much as 75°C at the connector itself (60°C typical). In flight, due to much stronger airflow and thus convective cooling, we expect the PCB and MOSFETs to run even cooler. No issues here.

## Limitations

Some limitations of this design are inherent to the 1S architecture, while others are due to the choice of components and the two-board design:

### Limited thrust

We went with two separate PCBs because we wanted to build and test the FCU first, then tackle the ESC and test that on its own. Another very important reason was that we also wanted all the tricky-to-solder parts, like QFN and BGA packages, on just one side of each board. A single 5-in-1 PCB with components on both sides would've made that much less practical to assemble and solder by hand.

The tradeoff is more PCB mass and board space than you'd normally see on a tinywhoop, plus a heavier frame. The small 22,000 KV motors and 1S batteries are yoinked from an Air75, where they have a lot less weight to carry. They're a bit underpowered for our setup. The drone still shreds, but it's more sluggish, and battery capacity and flight time are somewhat limited. More powerful motors could help with the thrust while still using 1S batteries, but they'd need a lot of current.

### FCU performance

A goal of this project was to simply test how well an RP2040 would perform in a drone application (and add support to Betaflight). The RP2040 offers less processing headroom for demanding flight-control configurations than more powerful MCUs, and it has no built-in floating-point unit. The RP2040 is still a very capable MCU, but it may not be the best choice for a high-performance flight controller. We can easily fly drones in acro mode with 1.6kHz PID loops, but we may run into performance issues with more complex flight-control algorithms, higher loop rates, or when also using the accelerometer (e.g., when using angle or horizon modes). 3.2kHz PID loops are out of the question. The RP2350 already has Betaflight firmware support, and it would be interesting to see how well it performs in a similar drone application.

### Thermal performance
´#
The ESC is designed to handle 8A continuously, and the half bridges handle this with ease. However, the 1.25mm PicoBlade connectors (taken from the Air75 design for compatibility) are the bottleneck here, and we might want to consider soldering the motors to the ESC directly. This is not strictly necessary even with higher current draws and also a bit inconvenient, but we can certainly add the SMD pads even if we don't use them.

### Are we FPV yet?

Hell nah - No camera, OSD, and VTX yet. We may want to add a small camera and VTX in the future - either as a separate module or integrated into the main FCU PCB. For now, we're limited to LoS flight :)

### Future Directions

There are two main directions for future development of this project: improving the 1S design, or moving to a 2S design. Improving the 1S design is easy because we only really need to combine our two PCBs into a single 5-in-1 PCB, which would reduce weight and board space, and then design a lighter frame. At that point, we would basically be redesigning existing 1S tinywhoops.

A 2S design will be a more interesting future project, with an RP2350, a revised ESC power stage, and larger motor/propeller options, because higher voltage allows us to use more powerful motors with better thrust and efficiency. At that point, the relatively heavy split design as well as heavier 3D printed frames will be far less of a concern.

## License

This project's hardware design files and accompanying documentation are licensed under the **CERN Open Hardware Licence Version 2 - Permissive**, SPDX identifier **`CERN-OHL-P-2.0`**. See [LICENSE](LICENSE) for the complete, unmodified [official license text](https://ohwr.org/cern_ohl_p_v2.txt).

This source describes Open Hardware. You may redistribute and modify it and make products using it under the terms of CERN-OHL-P-2.0. It is provided without any express or implied warranty; see the license for the applicable conditions.

Third-party firmware, libraries, and other third-party materials remain subject to their respective licenses.
