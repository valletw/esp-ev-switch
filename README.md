# ESP Electric Vehicle charge controller

The purpose of this project is to control a EVSE (Electric Vehicle Supply
Equipment), aka. "wall-box", to charge an electric vehicle. It will open/close
the Proximity Pilot (PP), the Control Pilot (CP) and the AC coil for the main
power. This project will not managed the EV protocol and be integrated only as a
switch between the vehicle and the EVSE. The control will be done with Home
Assistant by using [ESPHome](https://esphome.io) firmware.

Custom software could be written by using ESP-IDF SDK but it is not the purpose
of this project.

![Board](img/board-v1.1.0.jpg)

## Signals description

**Proximity Pilot**, also known as "Plug Present", is used to signal the
connection / disconnection of the plug. It can be used to simulate a manual
disconnection by opening the circuit as the "release" button.

**Control Pilot** is used to negotiate the charge and exchange information by
the vehicle and the EVSE. In case of proximity pilot is not detected, the signal
can be opened to trigger an error on the charge controller.

**AC coil** is used to open/close the main power relay. In case of power is still
remaining after PP and CP are opened.

## Disclaimer

The authors and contributors of this repository disclaim any and all
responsibility for the misuse of the information, tools, or techniques described
herein. Users are strictly advised to utilize this information in accordance with
applicable laws and regulations and only on systems for which they have explicit
authorization.

Neither the authors nor contributors shall be held liable for any damages, direct
or indirect, resulting from the misuse or unauthorized application of the
knowledge contained herein.

These device is powered from 230V line. Every flash and debug steps shall be
take with extreme caution with main power supply disconnected or by using
an isolator on the USB connector !

## Hardware

Wago connector is used for easy plug with the different wall-box wires:

- `L`: Line (230V)
- `N`: Neutral
- `RELAY I/O`: EVSE main power supply coil control Input/Output
- `CP I/O`: EVSE Control Pilot Input/Output
- `PP I/O`: EVSE Proximity Pilot Input/Output

3.5mm jack is available for current transformer measure. It's compatible with the
sensor SCT-013-050. Different current limit could be selected depending of the
maximum voltage of your wall-box: for a 7kW charge, select at least a probe for
50 A.

Three status LED are available:

- `PWR`: 3.3V state
- `STAT`: System status controlled by software
- `CHG`: Charge indicator (all circuits are open/close)

Two buttons are available for ESP32 boot sequence:

- `BOOT`: Control processor download mode on boot
- `EN`: Control processor reset

For advanced debug, a 2x5 1.27 pins header connectors is available with the
Cortex-M JTAG/SWD pinout for external probe.

![Board top render](img/render-top.jpg)
![Board bottom render](img/render-bottom.jpg)

### Wiring

The board will be placed between the EVSE and the EV plug. It will open/close
the different signals to control the main power relay, the control signal and
the proximity signal.

![EV Switch Wiring](img/wiring.jpg)

Note: This wiring could be adapted for tri-phase system and should be able
to control the EVSE. The power measure can only be done on one phase.

You can see my setup on a QUBEV wallbox. The clamp take some space and I had to
put it on the input instead of the relay output because of the EV plug when I
closed the enclosure.

Also, the jack for the clamp broke when I installed the board, I was not careful
enough. As it's not through hole connector, it break easily. I added glue and a wire
as quick fix.

Setup | Home Assistant
:----:|:--------------:
![Setup on QUBEV](img/setup-qubev.jpg) | ![Home Assistant Screenshot](img/homeassistant.jpg)

Note: I limited my wallbox at 20A with an hardware switch inside.

### Components choice

This project is based on an ESP32-C3 module to simplify design and ensure good
WiFi performance (with external antenna connector). ESP32 has been selected for
the big community to generate software and the frameworks available. Espressif
hardware design rules has been take into account following there
[documentation](https://docs.espressif.com/projects/esp-hardware-design-guidelines/en/latest/esp32c3/pcb-layout-design.html)

For easy debug and flashing, ESP32 with USB-CDC and JTAG support has been selected.

The 5V power rail is generated from a Mean Well AC-DC power supply. It has been
selected for the "ready to use" and the security / certification. The 5V is used
to power the relays.
The 3.3V power rail is generated from a LDO using the 5V power rail. It's used for
the controller part: ESP32, measure and signals.

By default, the 5V from the USB connector is not connected. For debug, it can
be forced with a jumper to solder on the PCB. In this case, do not connect / solder
both 5V USB & 5V AC-DC.

All others components has been selected from [JLCPCB](https://jlcpcb.com/parts/)
catalogue to minimize the cost and the assembly fees.

### Manufacturing

The board is manufactured by [JLCPCB](https://jlcpcb.com), here is the details
you will require to generate an order.

*Note: JLCPCB requests some modifications on output files, do not use direct
export from KiCad (check the FAQ)*

#### PCB

Files: [Gerbers](jlcpcb/gerbers.zip)

Configuration              | Value
---------------------------|---------------------------------
Base Materiel              | FR-4
Layers                     | 4
Dimensions                 | 99.92 x 58.42 mm
Different design           | 1
Delivery format            | Single PCB
PCB thickness              | 1.6
PCB color                  | Green (other colors have fees)
Silkscreen                 | White
Materiel type              | FR4-Standard TG 135-140
Surface finish             | HASL
Specify layer sequence     | F_Cu / In1_Cu / In2_Cu / B_Cu
Impedance control          | Yes
Layer stack-up             | JLC04161H-7628
Via covering               | Tented
Min via hole size/diameter | 0.3mm/0.45mm
Remove order number        | Specify a location

*Note: All options have not been detail here, keep default value.*

#### Assembly

Files: [BOM](jlcpcb/bom.csv), [CPL](jlcpcb/cpl.csv)

Configuration       | Value
--------------------|--------------------
PCBA type           | Economic
Assembly side       | Top

Verify "pick & place" orientations and positions on the web viewer

*Note: All options have not been detail here, keep default value.*

### Calibration

For power consumption measurement, a BL0942 chip is used and connected to the
current transformer probe. Default values are set for the voltage, current, power
and energy reference. The BL0942 is calibration free and default value could
work. If better accuracy is required, the default value can be changed by following
a calibration process:

1. Voltage reference can be adapted by measuring the input voltage with a multimeter,
2. Current can be adapted by using a well known load: example 10 A,
3. When your voltage/current are correctly calibrated, the power can be computed
with `P = U * I`

#### Voltage reference

The voltage RMS is measured from VP/GND pins connected through a voltage transformer
for galvanic isolation. The primary side uses a resistive divider 235 kOhms (5 * 47k)
to limit the current through the transformer primary. On the secondary side, a 56 Ohms
resistor sets the output voltage, followed by a 1 kOhms series resistor on the VP pin
for protection.

The transformer is designed as a current source: the primary voltage drives a current
through R_in, which is then converted back to a voltage across R_out on the secondary:

```math
U_{out} = \frac{U_{in}}{R_{in}} \times R_{out}
```

```math
U_{out} = \frac{230}{235000} \times 56 = 54.8 mV
```

The BL0942 VP input full-scale is 70 mV RMS / 100 mV peak-to-peak. The operating
points relative to full-scale are:

Input voltage | VP voltage | % of full-scale
:------------:|:----------:|:---------------:
230 V         | 54.8 mV    | 78 %
260 V         | 61.9 mV    | 88 %

From the datasheet, the internal register value is computed with the following formula:

```math
V_{RMS} = \frac{73989 \times V(mV)}{V_{ref}}
```

```math
V_{ref} = 1.218 V
```

With the previous values (expected VP voltage and chip reference voltage) we can
estimate the voltage reference for the ESPHome configuration:

```math
Reference = \frac{V_{RMS}}{V_{in}}
```

```math
Reference = \frac{\frac{73989 \times 54.8}{1.218}}{230} = 14477
```

After test on the received board, the measured voltage reference is **14400**,
consistent with the theoretical value (< 0.5 % deviation). This value will be used
as a base to improve accuracy by adjusting it while measuring the input voltage with
a calibrated True RMS multimeter.

#### Current reference

The current RMS is measured from IP/IN pins connected to a `SCT-013-050` current
transformer clamp through a signal conditioning circuit.

##### SCT-013-050 model

The `SCT-013-050` is a voltage output clamp with a built-in burden resistor `Rb`.
It is designed as a current source with the following characteristics:

Parameter       | Value
----------------|-------------------------------
Rated input     | 50 A RMS
Rated output    | 1 V RMS
Internal burden | Rb = ~37 Ohms
Turns ratio     | N = 50 / (1V / 37 Ohms) = 1850
Sensitivity     | 20 mV/A RMS

The secondary current for a given primary current is:

```math
I_s = \frac{I_{in}}{N} = \frac{I_{in}}{1850}
```

##### Signal conditioning circuit

The IP/IN pins are connected as follows:

- 1 Ohms (`R201`) in parallel with CT_K/CT_L (external burden)
- 1 kOhms in series on each line (pin protection)

`R201` forms a parallel combination with `Rb`, reducing the effective burden:

```math
R_{burden} = \frac{Rb \times R201}{Rb + R201} = \frac{37 \times 1}{37 + 1} = 0.974 Ω
```

The differential voltage on IP/IN for a given primary current is:

```math
U_{out} = \frac{I_{in}}{N} \times R_{burden} = \frac{I_{in}}{1850} \times 0.974
```

The BL0942 IP/IN full-scale is 30 mV RMS / 42 mV peak-to-peak. The operating
points relative to full-scale are:

Primary current | IP/IN voltage | % of full-scale
:--------------:|:-------------:|:---------------:
50 A            | 26 mV         | 90 %
32 A            | 17 mV         | 57 %
16 A            | 8 mV          | 27 %
2 A             | 1 mV          | 3 %

##### Current reference calculation

From the datasheet, the internal register value is computed with the following formula:

```math
I_{RMS} = \frac{305978 \times U_{out}(mV)}{V_{ref}}
```

```math
V_{ref} = 1.218 V
```

With the previous values we can estimated the current reference for the ESPHome
configuration:

```math
Reference = \frac{I_{RMS}}{I_{in}} = \frac{305978 \times R_{burden}}{V_{ref} \times N}
```

```math
Reference = \frac{305978 \times 974}{1.218 \times 1850} = 132260
```

After test on the received board, the measured current reference is **119500**,
representing a ~10 % deviation from the theoretical value, consistent with the
combined tolerances of `Rb` (10 %), `R201` (1 %), and the BL0942 internal reference.
This value will be used as a base to improve accuracy by adjusting it while
measuring the input current with a calibrated True RMS clamp meter.

### ESP32 Pinout

For those who want to reuse this project, here is the ESP32 pinout to manage the
different input/output.

Name               | Pin  | Direction
-------------------|------|----------
Relay ON           | IO0  | Output
Control Pilot ON   | IO1  | Output
Charge LED         | IO2  | Output (strapping pin)
Proximity Pilot ON | IO3  | Output
System Status LED  | IO8  | Output (strapping pin)
BL0942 CF1         | IO10 | Input
BL0942 Tx          | RX   | Input
BL0942 Rx          | TX   | Output

## ESPHome configuration

Device must be setup via USB connection the first time to flash the customized
firmware with sensors/buttons configuration.

Once it has been configured successfully, you should fix the device IP on your
router for easier Home Assistant configuration.

Note: Installation can be bypassed if already done or you can use docker image.
Check [ESPHome documentation](https://esphome.io/guides/getting_started_command_line.html)
for more details.

### Installation

```shell
pip install esphome
# Ensure your user session has the correct permission for serial port.
sudo usermod -a -G dialout <USERNAME>
```

### Customize

Depending of your installation and network, a `secrets.yaml` file must be
create with your specific configuration.

```yaml
# secrets.yaml
wifi_ssid: MySSID
wifi_password: MyStrongPassphrase
wifi_fb_ssid: ATX Controller Fallback Hotspot
wifi_fb_password: StrongPassphrase
ota_password: MyOTAPassphrase
api_key: 32BytesBase64String
```

### Compile and flash

```shell
# Generate the firmware.
esphome compile esphome.yaml
# Flash the firmware to the ESP.
esphome run esphome.yaml
# Get logs.
esphome logs esphome.yaml
```
