# DSC-HomeAssistant-MQTT-Ethernet

A DSC security system interface for Home Assistant using MQTT over Ethernet, specifically designed for the WT32-ETH01 ESP32 board with built-in Ethernet capabilities.

## Overview

This project enables integration of DSC PowerSeries and Classic series security systems with Home Assistant via MQTT using a reliable Ethernet connection instead of WiFi. It's based on the excellent [dscKeybusInterface](https://github.com/taligentx/dscKeybusInterface) library but modified to work with Ethernet-enabled ESP32 boards.

## Features

- **Ethernet Connectivity**: Reliable wired connection using WT32-ETH01 board
- **Full DSC Integration**: Monitor and control all partitions (1-8)
- **Zone Monitoring**: Track status of up to 64 zones
- **Virtual Keypad**: Arm/disarm system remotely via MQTT
- **Fire & Trouble Alerts**: Monitor fire alarms and system trouble states  
- **PGM Outputs**: Monitor programmable outputs (PGM 1-14)
- **Home Assistant Ready**: Includes complete configuration examples
- **Panic Alarm Support**: Trigger panic alarms remotely

## Supported DSC Systems

### PowerSeries Panels
- PC585, PC1555MX, PC1565, PC1565-2P, PC5005, PC5010, PC5015, PC5020
- PC1616, PC1808, PC1832, PC1864
- All DSC PowerSeries panels should work

### Classic Series Panels  
- PC1500, PC1550, PC2550 (requires PC16-OUT configuration)

## Hardware Requirements

### Required Components
- **WT32-ETH01** ESP32 board with Ethernet
- **Resistors**: 33kΩ (2x), 10kΩ (2x), 1kΩ (1x for virtual keypad)
- **NPN Transistor**: 2N3904, BC547/548/549, or similar (for virtual keypad)
- **5V Voltage Regulator**: LM2596 buck converter (recommended)

### Wiring Diagram

```
DSC Panel Wiring:
DSC Aux(+) ---+--- 5v voltage regulator --- WT32-ETH01 5v pin
              |
              +--- 1kΩ resistor --- DSC PGM (Classic series only)

DSC Aux(-) --- WT32-ETH01 Ground

                                      +--- GPIO 5 (dscClockPin)
DSC Yellow --- 33kΩ resistor ---------|
                                      +--- 10kΩ resistor --- Ground

                                      +--- GPIO 35 (dscReadPin)  
DSC Green ---- 33kΩ resistor ---------|
                                      +--- 10kΩ resistor --- Ground

Classic Series Only (PC16-OUT):
                                      +--- GPIO 32 (dscPC16Pin)
DSC PGM ------ 33kΩ resistor ---------|
                                      +--- 10kΩ resistor --- Ground

Virtual Keypad (Optional):
DSC Green ---- NPN collector ----\
                                  |-- NPN base --- 1kΩ resistor --- GPIO 33 (dscWritePin)
Ground ------- NPN emitter ------/
```

### Pin Configuration
| Function | WT32-ETH01 GPIO | DSC Connection |
|----------|-----------------|----------------|
| Clock    | 5               | Yellow (Clock) |
| Data     | 35              | Green (Data)   |
| PC16     | 32              | PGM (Classic)  |
| Write    | 33              | Data (via NPN) |

## Software Requirements

- **Arduino IDE** 1.8.19 or newer
- **ESP32 Arduino Core** 2.0.17 (ESP32 Core 3.x not supported)
- **Libraries**:
  - dscKeybusInterface
  - PubSubClient (MQTT)
  - WiFi (for event handling)

## Installation

### 1. Arduino IDE Setup
```bash
# Install ESP32 Arduino Core 2.0.17 via Boards Manager
# Tools > Board > Boards Manager > Search "ESP32" > Install version 2.0.17
```

### 2. Install Libraries
```bash
# Via Library Manager (Tools > Manage Libraries):
# Search and install:
# - "DSC" (dscKeybusInterface by taligentx)
# - "PubSubClient" by Nick O'Leary
```

### 3. Configuration
Edit the following settings in the sketch:
```cpp
// Security Settings
const char* accessCode = "1234";          // Your DSC access code

// MQTT Settings  
const char* mqttServer = "192.168.1.100"; // Your MQTT broker IP
const int   mqttPort = 1883;              // MQTT port
const char* mqttUsername = "username";     // MQTT username (optional)
const char* mqttPassword = "password";     // MQTT password (optional)

// Optional: Static IP Configuration
IPAddress local_IP(192, 168, 1, 200);     // Static IP (leave empty for DHCP)
IPAddress gateway(192, 168, 1, 1);        // Gateway IP
IPAddress subnet(255, 255, 255, 0);       // Subnet mask
```

### 4. Upload & Connect
1. Connect WT32-ETH01 to computer via USB
2. Select **ESP32 Dev Module** as board type
3. Upload the sketch
4. Connect Ethernet cable and DSC wiring
5. Power on the system

## Home Assistant Configuration

Add to your `configuration.yaml`:

```yaml
# MQTT Configuration
mqtt:
  broker: 192.168.1.100  # Your MQTT broker IP
  client_id: homeAssistant

# Alarm Control Panels
alarm_control_panel:
  - platform: mqtt
    name: "Security Partition 1"
    state_topic: "dsc/Get/Partition1"
    availability_topic: "dsc/Status"
    command_topic: "dsc/Set"
    payload_disarm: "1D"
    payload_arm_home: "1S"
    payload_arm_away: "1A"
    payload_arm_night: "1N"

# Sensors for Status Messages
sensor:
  - platform: mqtt
    name: "Security Status"
    state_topic: "dsc/Get/Partition1/Message"
    availability_topic: "dsc/Status"
    icon: "mdi:shield"

# Binary Sensors for Zones & Alarms
binary_sensor:
  - platform: mqtt
    name: "Security Trouble"
    state_topic: "dsc/Get/Trouble"
    device_class: "problem"
    payload_on: "1"
    payload_off: "0"
    
  - platform: mqtt
    name: "Front Door"
    state_topic: "dsc/Get/Zone1"
    device_class: "door"
    payload_on: "1"
    payload_off: "0"
    
  - platform: mqtt
    name: "Smoke Detector"
    state_topic: "dsc/Get/Fire1"
    device_class: "smoke"
    payload_on: "1"
    payload_off: "0"
```

## MQTT Topics

### Published Topics (Status Updates)
| Topic | Description | Values |
|-------|-------------|--------|
| `dsc/Status` | System availability | `online`/`offline` |
| `dsc/Get/Partition[1-8]` | Partition armed state | `disarmed`/`armed_home`/`armed_away`/`armed_night`/`pending`/`triggered` |
| `dsc/Get/Partition[1-8]/Message` | Partition status message | Text status |
| `dsc/Get/Zone[1-64]` | Zone status | `0` (closed) / `1` (open) |
| `dsc/Get/Fire[1-8]` | Fire alarm status | `0` (normal) / `1` (alarm) |
| `dsc/Get/Trouble` | System trouble | `0` (normal) / `1` (trouble) |
| `dsc/Get/PGM[1-14]` | PGM output status | `0` (off) / `1` (on) |

### Subscribed Topics (Commands)
| Topic | Command | Description |
|-------|---------|-------------|
| `dsc/Set` | `[1-8]D` | Disarm partition |
| `dsc/Set` | `[1-8]S` | Arm stay |
| `dsc/Set` | `[1-8]A` | Arm away |
| `dsc/Set` | `[1-8]N` | Arm night |
| `dsc/Set` | `P` | Panic alarm |

## Panel Configuration

### PowerSeries Panels
Configure via `*8` + installer code:

**Section 377 (PC1616/1832/1864) or 370 (PC1555MX/5015):**
- **Swinger Shutdown**: Set to `000` to disable (recommended)
- **AC Failure Delay**: Set to `000` for immediate reporting

### Classic Series Panels  
**Required configuration via `*8` + installer code:**

- **Section 12, Option 1**: Enable Communicator
- **Section 13, Option 4**: Enable PC16-OUT mode  
- **Section 24, Option 8**: Set PGM to trigger on system alarm

## Troubleshooting

### No Data from Panel
1. Check wiring connections (solder recommended over breadboard)
2. Verify resistor values (33kΩ and 10kΩ)
3. Run `KeybusReader` example to verify data capture

### Virtual Keypad Not Working
1. Check NPN transistor pinout and connections
2. Verify 1kΩ base resistor
3. Test with `KeybusReader` example

### Ethernet Connection Issues
1. Verify Ethernet cable connection
2. Check network settings (DHCP vs static IP)
3. Monitor serial output for connection status

### MQTT Connection Problems
1. Verify MQTT broker settings
2. Check network connectivity
3. Confirm MQTT credentials if authentication enabled

## Development & Contributing

This project is based on the [dscKeybusInterface](https://github.com/taligentx/dscKeybusInterface) library by taligentx. 

### Key Modifications
- Replaced WiFi with Ethernet connectivity
- Updated pin assignments for WT32-ETH01
- Added Ethernet event handling
- Optimized for wired network reliability

### Contributing
- Issues and pull requests welcome
- Test with different DSC panel models
- Improvements to documentation and examples

## License

This project maintains the same public domain license as the original dscKeybusInterface library.

## Acknowledgments

- **taligentx** for the excellent [dscKeybusInterface](https://github.com/taligentx/dscKeybusInterface) library
- **DSC Keybus Protocol** research community on AVR Freaks
- Contributors to the original dscKeybusInterface project

## Support

- Create an [Issue](../../issues) for bugs or feature requests
- Check [dscKeybusInterface Discussions](https://github.com/taligentx/dscKeybusInterface/discussions) for general questions
- Review DSC panel manuals for configuration options

---

**⚠️ Disclaimer:** This interface connects directly to your security system. Test thoroughly before relying on it for security purposes. The authors assume no responsibility for any security implications or system malfunctions.
