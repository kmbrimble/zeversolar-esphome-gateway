# Zeversolar RS485 to ESPHome Gateway

This project intercepts the internal RS485 communication of a Zeversolar inverter (TL3000 series) and exposes all solar production, DC/AC diagnostics, and grid data to Home Assistant via ESPHome. 

## Hardware Components

* **Microcontroller:** ESP32-S3 Super Mini Development Board (SKU: TS-P1285). *Note: These are often sold mislabeled as C3 boards. This project uses the S3 configuration based on hardware boot logs.*
* **Buck Converter:** Mini-360 DC-DC Step Down Module (SKU: TS-P312).
* **RS485 Module:** TTL to RS485 Automatic Flow Control Module (SKU: TS-P1085).
* **Cabling:** Standard Cat5e/Cat6 Ethernet cable with RJ45 connector.

## Internal Inverter Connection

The gateway connects to the inverter's internal motherboard at **Port J105** (or J106). Do not use the external DRED or ETH ports, as they are isolated or carry incorrect logic voltages (-4.4V).

### Inverter J105 RJ45 Pinout
* **Pin 1 (White/Orange):** RS485 Data A (+)
* **Pin 2 (Orange):** RS485 Data B (-)
* **Pin 4 (Blue):** Ground (GND)
* **Pin 7 (Brown/White):** +7.38V DC Power

*Unused wires (Green, Green/White, Brown, Blue/White) can be severed and capped.*

## Wiring Diagram

### 1. Power Regulation
The inverter outputs ~7.38V on Pin 7, which must be stepped down to safely power the ESP32 and RS485 module.
* **Mini-360 IN (+):** Inverter Pin 7 (Brown/White)
* **Mini-360 IN (-):** Inverter Pin 4 (Blue)
* **Mini-360 OUT (+):** Tuned via potentiometer to **5.05V**
* **Mini-360 OUT (-):** Common Ground

### 2. Logic Connections
| Component A | Component B | Purpose |
| :--- | :--- | :--- |
| Mini-360 OUT (+) | ESP32 `5V` & RS485 `VCC` | 5V Power Distribution |
| Mini-360 OUT (-) | ESP32 `GND` | Ground Distribution |
| ESP32 `GND` | RS485 `GND` | Common Ground Bridge |
| ESP32 `TX` (GPIO 43) | RS485 `TXD` | Transmit Data |
| ESP32 `RX` (GPIO 44) | RS485 `RXD` | Receive Data |
| RS485 `A+` | Inverter Pin 1 (White/Orange) | Modbus Data A |
| RS485 `B-` | Inverter Pin 2 (Orange) | Modbus Data B |

## ESPHome Configuration

The ESP32-S3 Super Mini requires specific `platformio_options` to force `dio` flash mode and enable native USB serial logging, otherwise it will enter a continuous crash/boot loop.

```yaml
esphome:
  name: "solar-gateway"
  friendly_name: "Zeversolar Inverter"
  platformio_options:
    board_build.flash_mode: dio
    build_flags:
      - -DARDUINO_USB_MODE=1
      - -DARDUINO_USB_CDC_ON_BOOT=1

esp32:
  board: lolin_s3_mini
  variant: esp32s3
  framework:
    type: arduino

logger:
  hardware_uart: USB_SERIAL_JTAG
  level: DEBUG

api:
  password: ""

ota:
  - platform: esphome
    password: ""

wifi:
  ssid: "YOUR_WIFI_SSID"
  password: "YOUR_WIFI_PASSWORD"
  fast_connect: true
  ap:
    ssid: "Solar-Gateway-Fallback"
    password: "solar-recovery"

uart:
  - id: inverter_uart
    tx_pin: GPIO43
    rx_pin: GPIO44
    baud_rate: 9600
    stop_bits: 1

modbus:
  id: modbus1
  uart_id: inverter_uart

modbus_controller:
  - id: zeversolar
    address: 1
    modbus_id: modbus1
    command_throttle: 100ms
    update_interval: 10s

sensor:
  # --- Live Data ---
  - platform: modbus_controller
    modbus_controller_id: zeversolar
    name: "Solar Current Power"
    address: 0x100A
    register_type: read
    value_type: U_WORD
    unit_of_measurement: "W"
    device_class: power

  - platform: modbus_controller
    modbus_controller_id: zeversolar
    name: "Solar Daily Yield"
    address: 0x100C
    register_type: read
    value_type: U_WORD
    unit_of_measurement: "kWh"
    filters:
      - multiply: 0.1
    device_class: energy

  - platform: modbus_controller
    modbus_controller_id: zeversolar
    name: "Solar Total Yield"
    address: 0x101D
    register_type: read
    value_type: U_DWORD
    unit_of_measurement: "kWh"
    filters:
      - multiply: 0.1
    device_class: energy

  # --- Diagnostics ---
  - platform: modbus_controller
    modbus_controller_id: zeversolar
    name: "Solar Inverter Temp"
    address: 0x1002
    register_type: read
    value_type: U_WORD
    unit_of_measurement: "°C"
    filters:
      - multiply: 0.1
    device_class: temperature

  # --- DC String Data ---
  - platform: modbus_controller
    modbus_controller_id: zeversolar
    name: "Solar PV1 Voltage"
    address: 0x1004
    register_type: read
    value_type: U_WORD
    unit_of_measurement: "V"
    filters: { multiply: 0.1 }

  - platform: modbus_controller
    modbus_controller_id: zeversolar
    name: "Solar PV1 Current"
    address: 0x1005
    register_type: read
    value_type: U_WORD
    unit_of_measurement: "A"
    filters: { multiply: 0.1 }

  - platform: modbus_controller
    modbus_controller_id: zeversolar
    name: "Solar PV2 Voltage"
    address: 0x1006
    register_type: read
    value_type: U_WORD
    unit_of_measurement: "V"
    filters: { multiply: 0.1 }

  - platform: modbus_controller
    modbus_controller_id: zeversolar
    name: "Solar PV2 Current"
    address: 0x1007
    register_type: read
    value_type: U_WORD
    unit_of_measurement: "A"
    filters: { multiply: 0.1 }

  # --- AC Grid Data ---
  - platform: modbus_controller
    modbus_controller_id: zeversolar
    name: "Solar Grid Voltage"
    address: 0x1009
    register_type: read
    value_type: U_WORD
    unit_of_measurement: "V"
    filters: { multiply: 0.1 }

  - platform: modbus_controller
    modbus_controller_id: zeversolar
    name: "Solar Grid Frequency"
    address: 0x1013
    register_type: read
    value_type: U_WORD
    unit_of_measurement: "Hz"
    filters: { multiply: 0.01 }

text_sensor:
  - platform: wifi_info
    ip_address:
      name: "Solar Gateway IP"
