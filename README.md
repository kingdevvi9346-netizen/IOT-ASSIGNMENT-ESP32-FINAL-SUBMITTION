# IoT Assignment - ESP32 Temperature Monitoring System with ESP-NOW

## 📋 Project Overview
This project implements a **peer-to-peer temperature and humidity monitoring system** using two ESP32 microcontrollers. The boards communicate wirelessly using **ESP-NOW protocol** to exchange sensor data and provide visual feedback through RGB LEDs based on temperature thresholds.

### Team Members
- **Vivek Devarapalli** - Board 1 Implementation
- **Suhanth** - Board 2 Implementation

### Course Information
- **Course**: B31OT - Internet of Things
- **Group**: Group 2

---

## 🎯 Key Features

- **ESP-NOW Communication**: Low-latency peer-to-peer wireless communication without WiFi infrastructure
- **DHT11 Temperature & Humidity Monitoring**: Real-time environmental sensing
- **RGB LED Visual Feedback**: Color-coded temperature indication system
- **Deep Sleep Power Management**: Energy-efficient operation with 10-second sleep cycles
- **Broadcast Communication**: Both boards can send and receive data simultaneously
- **Temperature Threshold Alerts**: Multiple temperature zones with distinct LED colors

---

## 🛠️ Hardware Components

### Each ESP32 Board Requires:
1. **ESP32 Development Board** (x2)
2. **DHT11 Temperature & Humidity Sensor** (x2)
3. **WS2812B NeoPixel RGB LED** (x2)
4. **Connecting Wires**
5. **USB Cable for Programming**

### Pin Configuration:
| Component | GPIO Pin |
|-----------|----------|
| DHT11 Sensor | GPIO 2 |
| NeoPixel LED | GPIO 5 |

---

## 📚 Software Dependencies

### Arduino Libraries Required:
```cpp
#include <WiFi.h>              // ESP32 WiFi library (built-in)
#include <esp_now.h>           // ESP-NOW protocol library (built-in)
#include "esp_wifi.h"          // ESP32 WiFi configuration (built-in)
#include <Adafruit_NeoPixel.h> // NeoPixel LED control
#include <DHTesp.h>            // DHT11 sensor library
```

### Installation Instructions:
1. Open Arduino IDE
2. Go to **Sketch → Include Library → Manage Libraries**
3. Install the following libraries:
   - **Adafruit NeoPixel** by Adafruit
   - **DHTesp** by beegee_tokyo

---

## 🚀 System Architecture

### Communication Protocol: ESP-NOW
- **Channel**: Fixed WiFi Channel 1
- **Mode**: Broadcast (FF:FF:FF:FF:FF:FF)
- **Message Structure**:
```cpp
typedef struct {
  char  from[10];    // Sender name ("VIVEK" or "SUHANTH")
  float temp;        // Temperature in Celsius
  float hum;         // Humidity percentage
} message_t;
```

### Timing Configuration:
- **Sleep Duration**: 10 seconds (deep sleep mode)
- **Awake Duration**: 10 seconds (active communication)
- **Send Interval**: 500ms (20 packets per wake cycle)

---

## 🌡️ Temperature Threshold System

The LED color changes based on temperature readings:

| Temperature Range | LED Color | Status |
|------------------|-----------|--------|
| < 0°C | 🔵 BLUE (0, 0, 255) | Freezing / Alarm Cold |
| 0°C - 10°C | 💠 TURQUOISE (0, 150, 150) | Cold Warning |
| 10°C - 25°C | 🟢 GREEN (0, 255, 0) | Normal / Optimal |
| 25°C - 30°C | 🟡 YELLOW (255, 200, 0) | Hot Warning |
| > 30°C | 🔴 RED (255, 0, 0) | Very Hot / Alarm Hot |

### LED Behavior:
- **Boot**: Dim grey (50, 50, 50)
- **Local Temperature**: LED shows own sensor reading
- **Remote Temperature**: LED updates when receiving data from peer board

---

## 📝 Code Implementation

### Board Configuration
Each board must be configured with a unique name. Modify this line before uploading:

**For Vivek's Board:**
```cpp
#define MY_NAME "VIVEK"
```

**For Suhanth's Board:**
```cpp
#define MY_NAME "SUHANTH"
```

### Key Functions:

#### 1. **setLedByTemperature(float tempC)**
Updates the NeoPixel LED color based on temperature thresholds.

#### 2. **onSent()**
Callback function triggered when ESP-NOW message is sent, reports success/failure.

#### 3. **onReceive()**
Callback function triggered when receiving ESP-NOW messages from peer board.
- Parses incoming temperature and humidity data
- Updates LED based on remote sensor readings
- Prints received data to Serial Monitor

#### 4. **setup()**
- Initializes Serial communication (115200 baud)
- Configures NeoPixel LED
- Initializes DHT11 sensor
- Sets up ESP-NOW protocol
- Reads sensor data
- Enters awake/send cycle

---

## 🔧 Setup and Installation

### Step 1: Hardware Assembly
1. Connect DHT11 sensor to GPIO 2 on each ESP32
2. Connect NeoPixel LED to GPIO 5 on each ESP32
3. Ensure proper power connections (3.3V for DHT11, 5V for NeoPixel if needed)

### Step 2: Software Setup
1. Install Arduino IDE (1.8.x or 2.x)
2. Add ESP32 board support:
   - Go to **File → Preferences**
   - Add to Additional Board Manager URLs:
     ```
     https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
     ```
   - Go to **Tools → Board → Boards Manager**
   - Search for "ESP32" and install

3. Install required libraries (see Software Dependencies)

### Step 3: Upload Code
1. Open the respective code file:
   - `VIVEK_ESP_FINALCODE` for Board 1
   - `SUHANTH_ESP_FINAL-CODE` for Board 2

2. Select board: **Tools → Board → ESP32 Dev Module**

3. Select correct COM port: **Tools → Port**

4. Click **Upload** button

5. Open Serial Monitor (115200 baud) to view output

---

## 📊 System Operation

### Operation Cycle:
1. **Wake Up**: Board wakes from deep sleep
2. **Sensor Reading**: DHT11 reads temperature and humidity
3. **Local LED Update**: LED shows local temperature status
4. **Broadcasting**: Sends data packet every 500ms for 10 seconds
5. **Listening**: Receives packets from peer board
6. **Remote LED Update**: Updates LED based on received temperature
7. **Sleep**: Enters deep sleep for 10 seconds
8. **Repeat**: Cycle continues indefinitely

### Phase Offset:
Suhanth's board has a 1-second delay to prevent perfect synchronization, improving communication reliability.

---

## 🖥️ Serial Monitor Output Example

```
====================================
Booting VIVEK
====================================
WiFi channel: 1
My MAC: XX:XX:XX:XX:XX:XX
[INFO] ESP-NOW Ready

Temp: 23.5 °C, Humidity: 65.0 %
LED: GREEN (Normal)
Staying awake for ~10 seconds, sending every 500 ms...
[VIVEK] Send #1 -> OK
[VIVEK] Send #2 -> OK

========= RECEIVED PACKET =========
My Name: VIVEK
From MAC: XX:XX:XX:XX:XX:XX
Sender: SUHANTH
Temp: 26.2 °C
Hum: 62.5 %
LED: YELLOW (Hot)
====================================

Awake window done, total sends: 20
Packets received this cycle: 18
Entering deep sleep for 10 seconds...
====================================
```

---

## 🔍 Troubleshooting

### Issue: Boards not communicating
- **Solution**: Ensure both boards are on the same WiFi channel (channel 1)
- Check Serial Monitor for MAC addresses
- Verify both boards are powered on and running

### Issue: DHT11 returning NaN values
- **Solution**: 
  - Check wiring connections
  - Ensure 1.5-second stabilization delay in setup()
  - Try different DHT11 sensor

### Issue: LED not changing colors
- **Solution**:
  - Verify NeoPixel wiring (Data to GPIO 5)
  - Check power supply (NeoPixels need adequate current)
  - Ensure brightness is set correctly (LED_BRIGHT = 100)

### Issue: ESP-NOW initialization failed
- **Solution**:
  - Restart both boards
  - Re-upload code
  - Check ESP32 board package version compatibility

---

## 🌟 Key Differences Between Boards

Both boards run nearly identical code with these differences:

| Feature | Vivek's Board | Suhanth's Board |
|---------|--------------|-----------------|
| Board Name | `#define MY_NAME "VIVEK"` | `#define MY_NAME "SUHANTH"` |
| Phase Offset | None | 1-second delay |
| Functionality | Identical | Identical |

---

## 📈 Power Consumption

### Deep Sleep Mode:
- **Current Draw**: ~10-20μA
- **Duration**: 10 seconds

### Active Mode:
- **Current Draw**: ~80-100mA (with WiFi active)
- **Duration**: 10 seconds

### Average Power:
With 50% duty cycle, the system is highly energy-efficient for battery-powered operation.

---

## 🔐 Security Considerations

- **Unencrypted Communication**: ESP-NOW broadcasts are not encrypted in this implementation
- **MAC Address**: Uses broadcast address (FF:FF:FF:FF:FF:FF) - any ESP32 on channel 1 can receive
- **Future Enhancement**: Add encryption for sensitive deployments

---

## 🚧 Future Enhancements

1. **Web Dashboard**: Display temperature data on a web interface
2. **Data Logging**: Store historical temperature/humidity data
3. **MQTT Integration**: Connect to cloud services for remote monitoring
4. **Multiple Sensors**: Support for more than 2 boards
5. **Battery Level Monitoring**: Track power status
6. **Encrypted Communication**: Secure ESP-NOW messages
7. **Configurable Thresholds**: Adjust temperature zones via web interface

---

## 📄 File Structure

```
IOT-ASSIGNMENT-ESP32-FINAL-SUBMITTION/
├── README.md                          # This file
├── VIVEK_ESP_FINALCODE               # Board 1 source code (Vivek)
├── SUHANTH_ESP_FINAL-CODE            # Board 2 source code (Suhanth)
└── B31OT Group 2 Report.docx         # Project documentation
```

---

## 🤝 Contributing

This project is part of an academic assignment. For questions or improvements:
1. Contact team members
2. Refer to the detailed report: `B31OT Group 2 Report.docx`

---

## 📞 Support

For technical issues or questions:
- Review Serial Monitor output for debugging information
- Check ESP32 documentation: [ESP-NOW Protocol Guide](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/network/esp_now.html)
- DHTesp Library: [GitHub Repository](https://github.com/beegee-tokyo/DHTesp)
- Adafruit NeoPixel Guide: [Adafruit Learning System](https://learn.adafruit.com/adafruit-neopixel-uberguide)

---

## 📜 License

This project is developed for educational purposes as part of the B31OT IoT course.

---

## 🙏 Acknowledgments

- **Espressif Systems** - ESP32 platform and ESP-NOW protocol
- **Adafruit** - NeoPixel library and excellent documentation
- **beegee_tokyo** - DHTesp library
- Course instructors and teaching assistants

---

**Last Updated**: December 2025  
**Version**: 1.0  
**Status**: ✅ Fully Functional
