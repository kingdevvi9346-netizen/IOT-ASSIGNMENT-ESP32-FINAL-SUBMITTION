# IoT Assignment - ESP32 Temperature Monitoring System with ESP-NOW

## 📋 Project Overview
This project implements a **peer-to-peer temperature and humidity monitoring system** using two ESP32 microcontrollers. The boards communicate wirelessly using **ESP-NOW protocol** to exchange sensor data and provide visual feedback through RGB LEDs based on temperature thresholds.

### Team Members
- **Vivek Devarapalli** - Board 1 Implementation (VIVEK_ESP_FINALCODE)
- **Suhanth** - Board 2 Implementation (SUHANTH_ESP_FINAL-CODE)

### Course Information
- **Course**: B31OT - Internet of Things
- **Group**: Group 2
- **Submission Date**: December 5, 2025

---

## 🎯 Key Features

- **ESP-NOW Communication**: Low-latency peer-to-peer wireless communication without WiFi infrastructure
- **DHT11 Temperature & Humidity Monitoring**: Real-time environmental sensing on both boards
- **RGB LED Visual Feedback**: Color-coded temperature indication system with 5 temperature zones
- **Deep Sleep Power Management**: Energy-efficient operation with 10-second sleep/awake cycles
- **Broadcast Communication**: Both boards can send and receive data simultaneously (~20 packets per cycle)
- **Bidirectional Monitoring**: Each board displays the remote board's temperature on its LED
- **Temperature Threshold Alerts**: Multiple temperature zones with distinct LED colors
- **Automatic Synchronization**: Phase offset prevents transmission collisions

---

## 🛠️ Hardware Components

### Each ESP32 Board Requires:
1. **ESP32 Development Board** (x2)
2. **DHT11 Temperature & Humidity Sensor** (x2)
3. **WS2812B NeoPixel RGB LED** (x2)
4. **Connecting Wires**
5. **USB Cable for Programming**
6. **Power Supply** (USB or battery - 5V recommended)

### Pin Configuration:
| Component | GPIO Pin | Configuration |
|-----------|----------|---------------|
| DHT11 Sensor Data | GPIO 2 | `#define DHT_PIN 2` |
| NeoPixel LED Data | GPIO 5 | `#define RGB_PIN 5` |
| DHT11 VCC | 3.3V | Power supply |
| DHT11 GND | GND | Ground |
| NeoPixel VCC | 5V | Power supply (or 3.3V) |
| NeoPixel GND | GND | Ground |

### Wiring Diagram:
```
ESP32                DHT11 Sensor
─────                ────────────
GPIO 2  ────────────> Data Pin
3.3V    ────────────> VCC
GND     ────────────> GND

ESP32                NeoPixel LED
─────                ────────────
GPIO 5  ────────────> Data In (DI)
5V      ────────────> VCC (or 3.3V)
GND     ────────────> GND
```

---

## 📋 Code Variables and Constants

### Configuration Constants:
```cpp
// Board identification
#define MY_NAME "VIVEK"        // or "SUHANTH"

// Hardware pins
#define RGB_PIN     5          // NeoPixel data pin
#define LED_COUNT   1          // Single RGB LED
#define LED_BRIGHT  100        // Brightness (0-255)
#define DHT_PIN     2          // DHT11 data pin

// Timing (microseconds and milliseconds)
const uint64_t SLEEP_DURATION_US   = 10ULL * 1000000ULL;  // 10s sleep
const uint32_t AWAKE_DURATION_MS   = 10000UL;             // 10s awake
const uint32_t SEND_INTERVAL_MS    = 500UL;               // 500ms between sends

// Temperature thresholds (Celsius)
const float ALARM_COLD = 0.0;    // Below this = BLUE
const float WARN_COLD  = 10.0;   // Below this = TURQUOISE
const float WARN_HOT   = 25.0;   // Above this = YELLOW
const float ALARM_HOT  = 30.0;   // Above this = RED
```

### Global Objects:
```cpp
Adafruit_NeoPixel pixel(LED_COUNT, RGB_PIN, NEO_GRB + NEO_KHZ800);
DHTesp dht;
```

### Message Buffers:
```cpp
message_t outgoingMessage;           // Data to send
message_t incomingMessage;           // Received data
```

### Network Configuration:
```cpp
uint8_t broadcastAddress[] = {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF};
volatile uint8_t rxCountThisCycle = 0;  // Received packet counter
```

### DHT Sensor Reading:
```cpp
TempAndHumidity dhtData = dht.getTempAndHumidity();
float temp = dhtData.temperature;    // Local temperature
float hum  = dhtData.humidity;       // Local humidity
```

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
- **Encryption**: None (unencrypted)
- **Max Range**: Up to 200m (line of sight)
- **Packet Size**: 18 bytes (10 bytes name + 4 bytes temp + 4 bytes hum)

#### **Message Structure**:
```cpp
typedef struct {
  char  from[10];    // Sender name ("VIVEK" or "SUHANTH")
  float temp;        // Temperature in Celsius (4 bytes)
  float hum;         // Humidity percentage (4 bytes)
} message_t;
```

#### **Global Message Variables**:
```cpp
message_t outgoingMessage;  // Buffer for sending
message_t incomingMessage;  // Buffer for receiving
```

#### **Broadcast Address**:
```cpp
uint8_t broadcastAddress[] = {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF};
```
This allows any ESP32 on the same channel to receive the message without pairing.

### Timing Configuration:
- **Sleep Duration**: 10 seconds (deep sleep mode) = `10,000,000 μs`
- **Awake Duration**: 10 seconds (active communication) = `10,000 ms`
- **Send Interval**: 500ms (20 packets per wake cycle)
- **Callback Delay**: 10ms between sends (allows background tasks)
- **DHT Stabilization**: 1,500ms initial sensor warm-up

### Power Management:
```cpp
// Configure deep sleep timer
esp_sleep_enable_timer_wakeup(SLEEP_DURATION_US);
// Enter deep sleep (execution stops here)
esp_deep_sleep_start();
// Wakes up → restarts from setup()
```

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
```cpp
void setLedByTemperature(float tempC) {
  // Determines RGB values based on temperature zones
  // Updates NeoPixel display accordingly
}
```

#### 2. **onSent(const wifi_tx_info_t *tx_info, esp_now_send_status_t status)**
Callback function triggered when ESP-NOW message is sent.
- Reports transmission success/failure to Serial Monitor
- Helps debug communication issues

#### 3. **onReceive(const esp_now_recv_info *recv_info, const uint8_t *data, int len)**
Callback function triggered when receiving ESP-NOW messages from peer board.
- Increments receive counter (`rxCountThisCycle`)
- Parses incoming temperature and humidity data
- **Updates LED based on remote sensor readings** (key feature!)
- Prints detailed packet information to Serial Monitor
- Validates packet size before processing

#### 4. **setup()**
Main initialization and execution function (runs once per wake cycle):
- Initializes Serial communication (115200 baud)
- Configures NeoPixel LED (dim grey at boot)
- Initializes DHT11 sensor with 1.5s stabilization delay
- Sets up WiFi in Station mode on channel 1
- Initializes ESP-NOW protocol with broadcast peer
- Applies phase offset for SUHANTH board (1-second delay)
- Reads DHT11 sensor data
- Sets local LED color based on own temperature
- **Enters 10-second awake window**:
  - Broadcasts data packets every 500ms (~20 packets)
  - Listens for incoming packets from peer board
  - Allows callbacks to process received data
- Reports statistics (sends, receives)
- **Enters deep sleep for 10 seconds**
- Cycle repeats automatically on wake

#### 5. **loop()**
Empty function - all logic runs in `setup()` due to deep sleep operation.

---

## 🔄 Detailed System Workflow

### Complete Operation Cycle Explained:

#### **Phase 1: Boot/Wake-up**
```
1. ESP32 wakes from deep sleep (or initial power-on)
2. Serial communication starts (115200 baud)
3. System identifies itself (prints "Booting VIVEK" or "Booting SUHANTH")
```

#### **Phase 2: Hardware Initialization**
```
4. NeoPixel LED initializes → Shows dim grey (50,50,50)
5. DHT11 sensor initializes → Waits 1.5 seconds for stabilization
6. WiFi configures → Station mode, Channel 1, no connection
7. ESP-NOW initializes → Registers callbacks, adds broadcast peer
```

#### **Phase 3: Sensor Reading**
```
8. Reads DHT11 sensor
   ├─ Gets temperature (°C)
   ├─ Gets humidity (%)
   └─ Validates data (checks for NaN)
9. Updates local LED based on OWN temperature
10. Prepares message structure with name, temp, hum
```

#### **Phase 4: Communication Window (10 seconds)**
```
11. Resets receive counter (rxCountThisCycle = 0)
12. Enters transmission loop:
    ├─ Every 500ms: Broadcasts message via ESP-NOW
    ├─ Logs send status (OK/ERROR)
    ├─ Total ~20 packets sent per cycle
    └─ Between sends: delay(10ms) allows callbacks to run
    
13. Concurrent receiving:
    ├─ onReceive() callback fires when packet arrives
    ├─ Parses sender name, temperature, humidity
    ├─ **Updates LED to show REMOTE temperature**
    ├─ Increments rxCountThisCycle
    └─ Prints detailed packet info to Serial
```

#### **Phase 5: Statistics & Sleep**
```
14. Communication window ends (10 seconds elapsed)
15. Prints statistics:
    ├─ Total sends: ~20
    └─ Packets received: varies (typically 15-20)
16. Enters deep sleep for 10 seconds
17. Timer wakes ESP32 automatically
18. Cycle repeats from Phase 1
```

### Visual Timeline:
```
Time: 0s─────────5s─────────10s────────────────20s────────────25s→
      │ Awake & Sending    │   Deep Sleep      │ Awake & Sending
      │  TX: ▲▲▲▲▲▲▲▲▲▲   │   (Power Save)    │  TX: ▲▲▲▲▲▲▲▲▲▲
      │  RX: ▼▼▼▼▼▼▼▼▼▼   │                   │  RX: ▼▼▼▼▼▼▼▼▼▼
      │ LED: Remote Temp    │   (LED Off)       │ LED: Remote Temp
```

---

## 💡 How the Code Files Work

### **VIVEK_ESP_FINALCODE** - Board 1 Operation:
1. **Identifies as**: "VIVEK"
2. **No phase offset**: Starts immediately after initialization
3. **Reads local sensor**: Gets temperature/humidity from connected DHT11
4. **Sets local LED**: Based on its own temperature initially
5. **Broadcasts data**: Sends "VIVEK" + temp + humidity every 500ms
6. **Receives from SUHANTH**: Listens for packets from Board 2
7. **Updates LED**: Changes color to reflect **SUHANTH's temperature**
8. **Deep sleeps**: Power-saving mode for 10 seconds
9. **Repeats**: Automatically wakes and cycles again

### **SUHANTH_ESP_FINAL-CODE** - Board 2 Operation:
1. **Identifies as**: "SUHANTH"
2. **Phase offset**: Waits 1 second before starting sends (reduces collisions)
3. **Reads local sensor**: Gets temperature/humidity from connected DHT11
4. **Sets local LED**: Based on its own temperature initially
5. **Broadcasts data**: Sends "SUHANTH" + temp + humidity every 500ms
6. **Receives from VIVEK**: Listens for packets from Board 1
7. **Updates LED**: Changes color to reflect **VIVEK's temperature**
8. **Deep sleeps**: Power-saving mode for 10 seconds
9. **Repeats**: Automatically wakes and cycles again

### **Key Insight - Bidirectional Monitoring:**
```
VIVEK Board                          SUHANTH Board
───────────                          ─────────────
Sensor: 23°C ──────sends───────────> Receives: 23°C
LED: Shows 26°C <────sends────────── Sensor: 26°C
(from SUHANTH)                       LED: Shows 23°C
                                     (from VIVEK)
```

**Each board displays the OTHER board's temperature on its LED**, creating a bidirectional remote monitoring system!

---

## 🎨 LED Color Logic Implementation

The `setLedByTemperature()` function implements a 5-zone temperature monitoring system:

```cpp
void setLedByTemperature(float tempC) {
  uint8_t r = 0, g = 0, b = 0;
  
  if (tempC < ALARM_COLD) {           // < 0°C
    r = 0; g = 0; b = 255;            // BLUE
    Serial.println("LED: BLUE (Freezing)");
  } 
  else if (tempC < WARN_COLD) {       // 0°C - 10°C
    r = 0; g = 150; b = 150;          // TURQUOISE
    Serial.println("LED: TURQUOISE (Cold)");
  } 
  else if (tempC <= WARN_HOT) {       // 10°C - 25°C
    r = 0; g = 255; b = 0;            // GREEN
    Serial.println("LED: GREEN (Normal)");
  } 
  else if (tempC < ALARM_HOT) {       // 25°C - 30°C
    r = 255; g = 200; b = 0;          // YELLOW
    Serial.println("LED: YELLOW (Hot)");
  } 
  else {                              // > 30°C
    r = 255; g = 0; b = 0;            // RED
    Serial.println("LED: RED (Very Hot)");
  }
  
  pixel.setPixelColor(0, pixel.Color(r, g, b));
  pixel.show();
}
```

This function is called:
1. **Initially**: Based on local temperature reading
2. **On receive**: Based on remote board's temperature (in `onReceive()` callback)

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

## ✅ Expected Behavior & Testing

### Normal Operation Indicators:

#### **Serial Monitor Output Should Show:**
1. ✓ Boot message with board name
2. ✓ WiFi channel: 1
3. ✓ MAC address
4. ✓ "ESP-NOW Ready" message
5. ✓ Temperature and humidity readings (not NaN)
6. ✓ LED color changes based on temperature
7. ✓ Send status: "OK" (not "ERROR")
8. ✓ Received packets from peer board
9. ✓ "Entering deep sleep" message

#### **LED Behavior Should Show:**
1. ✓ Dim grey at boot (50,50,50)
2. ✓ Changes to temperature-based color after sensor read
3. ✓ Updates to remote board's temperature color when receiving packets
4. ✓ Color matches reported temperature zone in Serial Monitor

#### **Communication Success:**
- Each board should receive **15-20 packets per cycle** (out of ~20 sent)
- Some packet loss is normal (WiFi interference, timing)
- LED should update to reflect remote temperature during awake window

### Testing Steps:

1. **Individual Board Test:**
   ```
   - Upload code to one board
   - Open Serial Monitor
   - Verify sensor readings
   - Check LED shows correct color for local temp
   - Should see 0 packets received (no peer yet)
   ```

2. **Paired Board Test:**
   ```
   - Upload code to both boards
   - Power both simultaneously
   - Observe Serial Monitors
   - VIVEK should receive from SUHANTH
   - SUHANTH should receive from VIVEK
   - LEDs should show each other's temperatures
   ```

3. **Temperature Change Test:**
   ```
   - Heat one sensor (breath on it or hold near warm object)
   - Watch remote board's LED change color
   - Verify color matches expected zone
   - Check Serial Monitor confirms temperature reading
   ```

4. **Deep Sleep Test:**
   ```
   - Observe "Entering deep sleep" message
   - Wait 10 seconds
   - Board should automatically wake
   - Cycle repeats indefinitely
   ```

---

## 🔍 Troubleshooting

### Issue: Boards not communicating
**Symptoms**: 
- "Packets received this cycle: 0"
- No "RECEIVED PACKET" messages in Serial Monitor
- LED doesn't change to remote temperature

**Solutions**: 
- ✓ Ensure both boards are on the same WiFi channel (channel 1)
- ✓ Check Serial Monitor for MAC addresses
- ✓ Verify both boards are powered on and running simultaneously
- ✓ Keep boards within ~10m range during testing
- ✓ Reduce WiFi interference from other devices

### Issue: DHT11 returning NaN values
**Symptoms**: 
- "DHT read FAILED (NaN)" in Serial Monitor
- LED stays dim grey
- No temperature/humidity values displayed

**Solutions**: 
- ✓ Check wiring connections (Data → GPIO 2, VCC → 3.3V, GND → GND)
- ✓ Ensure 1.5-second stabilization delay in setup()
- ✓ Wait longer for sensor to warm up (some DHT11 need more time)
- ✓ Try different DHT11 sensor (might be faulty)
- ✓ Check if DHT11 requires pull-up resistor (10kΩ on data line)

### Issue: LED not changing colors
**Symptoms**: 
- LED stays dim grey or doesn't change
- No visual feedback from temperature changes

**Solutions**:
- ✓ Verify NeoPixel wiring (Data → GPIO 5, VCC → 5V, GND → GND)
- ✓ Check power supply (NeoPixels need adequate current, use external power if needed)
- ✓ Ensure brightness is set correctly (LED_BRIGHT = 100)
- ✓ Test NeoPixel with simple example sketch first
- ✓ Try different NeoPixel LED (might be damaged)

### Issue: ESP-NOW initialization failed
**Symptoms**: 
- "ESP-NOW init failed!" in Serial Monitor
- No send/receive operations occur

**Solutions**:
- ✓ Restart both boards (power cycle)
- ✓ Re-upload code with correct board settings
- ✓ Check ESP32 board package version (should be 2.0.0 or higher)
- ✓ Verify WiFi.mode(WIFI_STA) is set before esp_now_init()
- ✓ Try different ESP32 board

### Issue: Send status shows "ERROR"
**Symptoms**: 
- "[VIVEK] Send #1 -> ERROR" in Serial Monitor
- Low packet reception rate

**Solutions**:
- ✓ Check peer address is correctly set (broadcast: FF:FF:FF:FF:FF:FF)
- ✓ Verify esp_now_add_peer() was successful
- ✓ Ensure channel is consistent (channel 1)
- ✓ Reduce distance between boards
- ✓ Check for WiFi interference

### Issue: Boards wake at different times
**Symptoms**: 
- Boards don't seem to communicate consistently
- Varying packet reception counts

**Solutions**:
- ✓ This is partially intentional (phase offset for SUHANTH)
- ✓ Deep sleep timing may vary slightly (±100ms normal)
- ✓ Power both boards simultaneously for initial sync
- ✓ Longer awake window (10s) allows overlap

### Issue: High power consumption
**Symptoms**: 
- Battery drains quickly
- ESP32 feels warm

**Solutions**:
- ✓ Verify deep sleep is actually occurring (check Serial Monitor)
- ✓ Check for proper esp_deep_sleep_start() call
- ✓ Ensure WiFi is not reconnecting unnecessarily
- ✓ Monitor current draw with multimeter during sleep (<100μA expected)

---

## 🧪 Debugging Tips

### Enable Verbose Output:
Add more debug prints to understand behavior:
```cpp
Serial.print("Free heap: ");
Serial.println(ESP.getFreeHeap());
Serial.print("Wake count: ");
Serial.println(esp_sleep_get_wakeup_cause());
```

### Test Without Deep Sleep:
Comment out sleep to keep board awake for debugging:
```cpp
// esp_deep_sleep_start();  // Temporarily disabled for testing
delay(10000);  // Wait instead of sleeping
```

### Force LED Colors:
Test LED independently:
```cpp
setColor(255, 0, 0);  // Red
delay(1000);
setColor(0, 255, 0);  // Green
delay(1000);
setColor(0, 0, 255);  // Blue
```

### Monitor WiFi Channel:
Verify channel is correct:
```cpp
uint8_t ch;
wifi_second_chan_t sch;
esp_wifi_get_channel(&ch, &sch);
Serial.printf("Current channel: %d\n", ch);
```

---

## 🌟 Key Differences Between Boards

Both boards run nearly identical code with these differences:

| Feature | Vivek's Board | Suhanth's Board |
|---------|--------------|-----------------|
| **Board Name** | `#define MY_NAME "VIVEK"` | `#define MY_NAME "SUHANTH"` |
| **Phase Offset** | None - immediate start | 1-second delay before sends |
| **Code Lines** | 267 lines | 269 lines |
| **Commented Code** | Clean | Has test comment block (lines 130-133) |
| **Functionality** | Identical | Identical |

### Why Phase Offset?
```cpp
// In SUHANTH's code (line 200-202):
if (String(MY_NAME) == "SUHANTH") {
  Serial.println("[SUHANTH] Applying 1s phase offset before sends...");
  delay(1000);  // Prevents simultaneous broadcasts
}
```

**Purpose**: 
- Reduces packet collisions when both boards wake simultaneously
- Improves communication reliability
- Increases successful packet reception rate
- Creates temporal separation in transmission windows

### Code Comparison:

#### **VIVEK_ESP_FINALCODE** (Lines 1-267):
- Line 11: `#define MY_NAME "VIVEK"`
- Line 133: Clean implementation, no test code
- Line 200-202: No phase offset logic
- Otherwise identical to SUHANTH code

#### **SUHANTH_ESP_FINAL-CODE** (Lines 1-269):
- Line 9: `#define MY_NAME "SUHANTH"`
- Line 130-133: Commented test code for magenta LED test
  ```cpp
  /*Serial.println("[VIVEK] RX TEST -> setting NeoPixel to MAGENTA");
  pixel.setPixelColor(0, pixel.Color(0, 0, 255)); 
  pixel.show();*/
  ```
- Line 200-202: 1-second phase offset implementation
- Otherwise identical to VIVEK code

**Result**: Both boards have identical functionality despite minor code differences

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

## � Performance Metrics

### Communication Statistics:
| Metric | Value | Notes |
|--------|-------|-------|
| **Packets Sent per Cycle** | ~20 | Every 500ms for 10s |
| **Packets Received per Cycle** | 15-20 | Depends on RF conditions |
| **Success Rate** | 75-100% | Typical in good conditions |
| **Latency** | <10ms | ESP-NOW is very fast |
| **Range** | Up to 200m | Line of sight, outdoor |
| **Indoor Range** | 50-100m | Varies with obstacles |

### Power Consumption Analysis:
| State | Current Draw | Duration | Energy per Cycle |
|-------|--------------|----------|------------------|
| **Deep Sleep** | ~20μA | 10s | 0.2mJ |
| **Awake (TX/RX)** | ~100mA | 10s | 5,000mJ |
| **Average** | ~50mA | 20s | ~5,000mJ |

**Battery Life Estimation** (with 2000mAh battery):
- Average current: ~50mA
- Expected runtime: ~40 hours
- Can be improved by increasing sleep duration

### Memory Usage:
```
Sketch uses ~XXXXX bytes (X%) of program storage space.
Global variables use XXXX bytes (X%) of dynamic memory.
ESP-NOW overhead: ~100 bytes per message
```

---

## 🎯 Practical Applications

### Where This System Can Be Used:

1. **Multi-Room Temperature Monitoring**
   - Monitor temperatures across different rooms
   - Visual alert system (LED colors) for quick status check
   - No WiFi network required

2. **Greenhouse/Plant Monitoring**
   - Monitor conditions in different greenhouse zones
   - Remote display of temperature without infrastructure
   - Battery-powered for portability

3. **Cold Chain Monitoring**
   - Monitor refrigerator/freezer temperatures
   - LED alerts for out-of-range conditions
   - Backup monitoring when WiFi unavailable

4. **HVAC System Monitoring**
   - Compare temperatures before/after HVAC units
   - Visual feedback on system performance
   - Energy-efficient continuous monitoring

5. **Smart Home Automation**
   - Integrate with home automation systems
   - Trigger actions based on temperature thresholds
   - Mesh network of sensors throughout home

6. **Educational Projects**
   - Learn IoT communication protocols
   - Understand sensor networks
   - Demonstrate ESP-NOW capabilities

7. **Industrial Monitoring**
   - Monitor equipment temperatures
   - Early warning system for overheating
   - Low-power operation for continuous monitoring

8. **Weather Stations**
   - Distributed temperature sensing
   - Compare indoor vs outdoor temperatures
   - Log environmental conditions

---

## 🚧 Future Enhancements

### Short-term Improvements:
1. **Adjustable Sleep Duration**: Configure via Serial commands
2. **Data Buffering**: Store multiple readings before transmission
3. **Retry Logic**: Resend failed packets
4. **Status LED Patterns**: Blink patterns for different states
5. **Battery Voltage Monitoring**: Track remaining power

### Medium-term Enhancements:
1. **Web Dashboard**: Display temperature data on a web interface via WiFi AP mode
2. **Data Logging**: Store historical data on SD card or SPIFFS
3. **MQTT Integration**: Connect to cloud services when WiFi available
4. **Multiple Sensors**: Support for 3+ boards in mesh network
5. **LCD Display**: Show current and remote temperatures on screen

### Long-term Vision:
1. **Mobile App**: Android/iOS app for monitoring and configuration
2. **Encrypted Communication**: AES encryption for ESP-NOW messages
3. **Configurable Thresholds**: Adjust temperature zones via Bluetooth or web interface
4. **Machine Learning**: Predict temperature trends and anomalies
5. **Cloud Integration**: AWS IoT, Google Cloud, or Azure IoT Hub
6. **Energy Harvesting**: Solar panel integration for perpetual operation
7. **Advanced Sensors**: Add CO2, pressure, light sensors

---

## 📄 File Structure

```
IOT-ASSIGNMENT-ESP32-FINAL-SUBMITTION/
├── README.md                          # This file
├── VIVEK_ESP_FINALCODE               # Board 1 source code (Vivek)
├── SUHANTH_ESP_FINAL-CODE            # Board 2 source code (Suhanth)
└── B31OT Group 2 Report.pdf          # Project documentation
```

---

## 🤝 Contributing

This project is part of an academic assignment. For questions or improvements:
1. Contact team members
2. Refer to the detailed report: `B31OT Group 2 Report.pdf`

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

## 📖 Summary

This IoT project successfully demonstrates:

### ✅ **Implemented Features:**
- ✓ Peer-to-peer ESP-NOW communication between two ESP32 boards
- ✓ Real-time temperature and humidity monitoring using DHT11 sensors
- ✓ Visual feedback system with color-coded RGB LEDs (5 temperature zones)
- ✓ Bidirectional data exchange - each board displays remote temperature
- ✓ Energy-efficient operation with deep sleep power management
- ✓ Broadcast communication without requiring MAC address pairing
- ✓ Robust packet transmission with ~75-100% success rate
- ✓ Automatic wake/sleep cycling for continuous monitoring
- ✓ Serial debugging output for development and troubleshooting

### 🎓 **Learning Outcomes:**
- ESP-NOW protocol implementation and configuration
- ESP32 deep sleep modes and power management
- Wireless sensor network design and operation
- Callback-based asynchronous programming
- DHT11 sensor interfacing and data validation
- NeoPixel LED control and color mapping
- WiFi channel configuration and management
- Embedded systems debugging techniques

### 🏆 **Project Strengths:**
- No WiFi infrastructure required (works anywhere)
- Low latency communication (<10ms)
- Energy efficient (50% duty cycle with deep sleep)
- Simple hardware setup (minimal components)
- Scalable design (can add more boards easily)
- Visual feedback system (LED colors)
- Comprehensive error handling
- Well-documented code and operation

### 📈 **Performance Summary:**
- **Communication Range**: Up to 200m (outdoor, line-of-sight)
- **Packet Success Rate**: 75-100% under normal conditions
- **Power Consumption**: Average ~50mA (with deep sleep)
- **Operating Temperature Range**: -40°C to +125°C (DHT11: 0-50°C)
- **Update Frequency**: Every 10 seconds (20 packets per cycle)
- **Battery Life**: ~40 hours (with 2000mAh battery)

---

## 📞 Contact & Support

### Team Members:
- **Vivek Devarapalli** - [GitHub: kingdevvi9346-netizen]
- **Suhanth** - Board 2 Development

### Repository:
- **GitHub**: [IOT-ASSIGNMENT-ESP32-FINAL-SUBMITTION](https://github.com/kingdevvi9346-netizen/IOT-ASSIGNMENT-ESP32-FINAL-SUBMITTION)

### For Questions or Issues:
1. Check the **Troubleshooting** section above
2. Review **Serial Monitor** output for error messages
3. Verify **hardware connections** match pin configuration
4. Consult the **detailed documentation** in this README
5. Refer to **B31OT Group 2 Report.pdf** for additional details

### Additional Resources:
- [ESP-NOW Protocol Guide](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/network/esp_now.html)
- [ESP32 Arduino Core Documentation](https://docs.espressif.com/projects/arduino-esp32/en/latest/)
- [DHTesp Library GitHub](https://github.com/beegee-tokyo/DHTesp)
- [Adafruit NeoPixel Guide](https://learn.adafruit.com/adafruit-neopixel-uberguide)
- [ESP32 Deep Sleep Tutorial](https://randomnerdtutorials.com/esp32-deep-sleep-arduino-ide-wake-up-sources/)

---

**Last Updated**: December 5, 2025  
**Version**: 1.0  
**Status**: ✅ Fully Functional and Tested  
**Course**: B31OT - Internet of Things  
**Group**: Group 2  

---

**⭐ If you found this project helpful, please consider starring the repository!**
