# Agent Guidelines & Hardware Workflow: ESP32 & ESP-01S Automation v2.1

## 1. Hardware Port Mappings & Upload Targets
- **ESP32 Master Controller:** `COM9` (Silicon Labs CP210x, FQBN: `esp32:esp32:esp32`)
  - Target Path: `d:\Automation\v2.1\esp32_wifi_test`
  - Upload Command:
    ```powershell
    & "C:\Users\verma\AppData\Local\Programs\Arduino IDE\resources\app\lib\backend\resources\arduino-cli.exe" upload -p COM9 --fqbn esp32:esp32:esp32 "d:\Automation\v2.1\esp32_wifi_test"
    ```
- **ESP-01S Slave Node (Room 1):** `COM8` / `COM5` (USB-SERIAL CH340, FQBN: `esp8266:esp8266:generic:FlashMode=dout,baud=115200`)
  - Target Path: `d:\Automation\v2.1\esp01_node`
  - MAC Address: `CC:50:E3:C6:EA:D0`
  - Upload Command:
    ```powershell
    & "C:\Users\verma\AppData\Local\Programs\Arduino IDE\resources\app\lib\backend\resources\arduino-cli.exe" upload -p COM8 --fqbn esp8266:esp8266:generic:FlashMode=dout,baud=115200 "d:\Automation\v2.1\esp01_node"
    ```

---

## 2. ESP-01S Physical Button Flashing Protocol
The user's ESP-01 programmer board features two physical buttons: **RESET** and **FLASH**.
The exact hardware boot strapping sequence is:
1. **Preparation Phase:**
   - Prompt the user to hold **BOTH [RESET] and [FLASH]** buttons down simultaneously.
   - Insert the USB adapter into the PC (`COM8` / `COM5`).
   - Release **[RESET]**, while **CONTINUING TO HOLD [FLASH]**.
   - User says `"READY"` or `"GO"`.
2. **Flash Phase:**
   - Immediately trigger the upload via `arduino-cli`.
3. **Execution Phase (Activation):**
   - Once upload reaches 100%, instruct user:
     - **Release the [FLASH] button**.
     - **Click the [RESET] button once** to boot into SPI Flash Run Mode and start firmware execution.
4. **Serial Lock Prevention:**
   - Always ensure no active Serial Monitor (e.g. Arduino IDE monitor tab or Python scripts) is locking `COM8` before initiating upload. A locked port returns `PermissionError(13, 'Access is denied')`.

---

## 3. Hardware Pinout & Wiring Specifications

### A. ESP-01S to 74HC595 Shift Register (DIP-16)
```
          74HC595 Pinout (Top Notch [ U ])
                   ┌───┬───┐
     Relay 2 (QB) ─┤ 1 │ 16├── 3.3V (VCC)
     Relay 3 (QC) ─┤ 2 │ 15├── Relay 1 (QA / Q0)
     Relay 4 (QD) ─┤ 3 │ 14├── ESP-01 GPIO0 (SER / Serial Data)
     Relay 5 (QE) ─┤ 4 │ 13├── GND (/OE - Output Enable, active-low)
     Relay 6 (QF) ─┤ 5 │ 12├── ESP-01 RXD / GPIO3 (RCLK / Latch Clock)
     Relay 7 (QG) ─┤ 6 │ 11├── ESP-01 GPIO2 (SRCLK / Shift Clock)
     Relay 8 (QH) ─┤ 7 │ 10├── 3.3V (/MR - Master Reset, active-low disabled)
              GND ─┤ 8 │  9├── (Unconnected / Serial Out Cascade)
                   └───┴───┘
```

### B. Shift Register to 8-Channel Relay Module
- `Pin 15 (QA)` $\rightarrow$ Relay 1 `IN1`
- `Pin 1  (QB)` $\rightarrow$ Relay 2 `IN2`
- `Pin 2  (QC)` $\rightarrow$ Relay 3 `IN3`
- `Pin 3  (QD)` $\rightarrow$ Relay 4 `IN4`
- `Pin 4  (QE)` $\rightarrow$ Relay 5 `IN5`
- `Pin 5  (QF)` $\rightarrow$ Relay 6 `IN6`
- `Pin 6  (QG)` $\rightarrow$ Relay 7 `IN7`
- `Pin 7  (QH)` $\rightarrow$ Relay 8 `IN8`

---

## 4. Electrical Truth Table & The 0.05V / Active-LOW Architecture

### A. Optocoupled Relay Logic
Standard 8-channel relay modules are **Active-LOW**:
$$\text{Relay Module Circuit: } \text{VCC} \longrightarrow \text{Resistor} \longrightarrow \text{Green Channel LED} \longrightarrow \text{Optocoupler Cathode} \longrightarrow \text{IN Pin}$$

| Desired State | 74HC595 Output | Output Voltage | Voltage Drop Across Optocoupler | Green LED | Relay Coil |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Relay OFF** | `HIGH` (Bit = 1) | **3.3V** | $V_{CC} - 3.3\text{V} \approx 0\text{V}$ (No conduction) | **OFF (Dark)** | **De-energized (OFF)** |
| **Relay ON**  | `LOW`  (Bit = 0) | **0.05V** ($V_{OL}$) | $V_{CC} - 0.05\text{V} \approx 3.25\text{V}$ (Forward biased) | **ON (Green)** | **Energized (ON)** |

### B. Critical Understanding of 0.05V
- **0.05V (50 mV) is NOT a leak or error.** It is standard CMOS logic LOW ($V_{OL}$ to Ground).
- On an Active-LOW module, **0.05V IS the signal that turns the relay ON.**
- Relays are turned **OFF** by outputting **3.3V (HIGH)**, which eliminates the voltage drop across the optocoupler diode.
- In `esp01_node.ino`, `#define RELAY_ACTIVE_LOW true` handles this inversion:
  `outByte = RELAY_ACTIVE_LOW ? ~relayStates : relayStates;`

---

## 5. Critical Engineering Rules for Future Agents

### Rule 1: NEVER Add a Startup Self-Test (`runStartupSelfTest`)
- **Reason:** Energizing all 8 relay coils simultaneously draws $8 \times 75\text{ mA} = 600\text{ mA}$.
- The USB programmer's 3.3V LDO regulator is rated for only $150\text{--}200\text{ mA}$.
- Turning on all relays at boot causes an instant brownout reset loop or freezes the ESP-01S with Relay 1 stuck on (`Mask: 0x1`).
- All relays must initialize strictly **OFF** at power-on.

### Rule 2: Microsecond-Zero Shift Register Initialization
- In `esp01_node.ino` `setup()`, claims pins and shift out `0xFF` (all OFF) **immediately at line 1**, before initializing Serial or delays:
  ```cpp
  pinMode(PIN_SER, OUTPUT);
  pinMode(PIN_SRCLK, OUTPUT);
  pinMode(PIN_RCLK, OUTPUT);
  digitalWrite(PIN_RCLK, LOW);
  digitalWrite(PIN_SRCLK, LOW);
  digitalWrite(PIN_SER, LOW);
  relayStates = 0x00;
  updateShiftRegister(); // Shifts 0xFF (all 3.3V = Relays OFF) in <50us
  ```

### Rule 3: NEVER Re-shift Bits in a Watchdog Loop
- An earlier bug re-shifted bits 10 times/second in `loop()`. Because `SRCLK` (Pin 11) and `RCLK` (Pin 12) are physically adjacent, high-frequency continuous shifting caused parasitic capacitive crosstalk on breadboards, causing other relays to blink or switch randomly.
- **Rule:** Call `updateShiftRegister()` **ONLY** when a genuine relay state transition occurs.

### Rule 4: Mandatory `SERIAL_TX_ONLY` on ESP-01S
- ESP-01S has only 4 I/O pins: `GPIO0`, `GPIO2`, `GPIO1` (TXD), and `GPIO3` (RXD).
- To use `GPIO3 (RXD)` as the 74HC595 Latch Clock (`RCLK`), you must start serial with:
  ```cpp
  Serial.begin(115200, SERIAL_8N1, SERIAL_TX_ONLY);
  ```
  This disables the UART RX receiver hardware, dedicating GPIO3 purely as a clean digital output pin.

### Rule 5: Silent Background Heartbeats
- The ESP-01S node sends an ESP-NOW heartbeat packet (`Dev: 255`, `State: 255`) every 3 seconds to the ESP32 Master to keep `nodeOnline = true` on the dashboard.
- **Do NOT print heartbeat logs to Serial.** Only log physical command executions (`device != 255`) to keep the Serial Monitor clean.

### Rule 6: Outbound Queue & Frame Pacing on ESP32 Master
- When spamming or tapping multiple buttons rapidly, transmitting consecutive ESP-NOW frames in $<2\text{ms}$ overwhelms the internal ESP-NOW radio buffer and throws `ESP_ERR_ESPNOW_INTERNAL`.
- The ESP32 master firmware implements a 32-slot `txQueue` with a mandatory **6ms pacing delay** in `loop()` to guarantee 100% packet delivery.

---

## 6. Network & Web Dashboard Configuration
- **ESP32 IP (Router Mode):** `http://192.168.1.76/` (SSID: `Afinity 4G`)
- **ESP32 IP (Hotspot Mode):** `http://192.168.4.1/` (SSID: `ESP32-SmartHome-Offline`, Pass: `12345678`)
- **WebSocket Port:** `81` (Handles real-time closed-loop state sync)
- **Dashboard Buttons:** State-linked buttons displaying direct hardware state:
  - **`🟢 ON`** (Glowing green `#22c55e`)
  - **`⚪ OFF`** (Dark slate `#1e293b`)
  - Clicking sends instantaneous WebSocket commands without client-side lockouts.

---

## 7. Commercial IoT Smart Water Tank Auto-Cut Controller (ESP-01S Architecture v2.9.2)
- **Target Path:** `d:\Automation\v2.1\esp01_commercial_node\esp01_commercial_node.ino`
- **Active Device Profile:**
  - Microcontroller: ESP8266EX (ESP-01S, 1MB Flash, CPU overclocked to 160MHz)
  - Active Device ID: `TANK-3C05EA`
  - MAC Address: `84:F3:EB:3C:05:EA`
  - Firmware Version: `v2.9.2` (Build 292)
- **Fail-Safe Pin Assignment Architecture:**
  - **`GPIO0` (Pin 3 - Relay Output):** Safe Hi-Z Relay Driver
    - **When ON:** `pinMode(OUTPUT)`, `digitalWrite(LOW)` (Pulls to 0V ground, sinks current -> Clicks ON).
    - **When OFF:** `pinMode(INPUT)` (High-Impedance Hi-Z open circuit -> Zero current -> Relay OFF).
    - **CRITICAL WARNING:** NEVER drive GPIO0 `OUTPUT HIGH (3.3V)` on an Active-LOW 5V relay module! $5.0\text{V} - 3.24\text{V} = 1.76\text{V} > 1.2\text{V}$ forward drop of PC817 optocoupler, which causes continuous conduction and leaves the relay **STUCK ON FOREVER**. Setting `pinMode(INPUT)` breaks the circuit to Ground and guarantees 100% cutoff.
  - **`GPIO2` (Pin 2 - Visual Status Indicator):** ESP-01S Onboard Blue LED
    - Hardwired internally on the ESP-01S PCB to the blue SMD LED (Active-LOW).
    - Mapped to motor state: `digitalWrite(PIN_STATUS_LED, on ? LOW : HIGH)`.
    - **Motor Running:** Blue LED glows solid bright blue.
    - **Motor Stopped:** Blue LED turns completely OFF.
    - Left empty externally on the 2x4 header so it floats HIGH at boot, preventing SDIO flash boot freezes.
  - **`GPIO3 (RXD)` (Pin 4 - Water Tank Float Sensor):**
    - Configured with internal `INPUT_PULLUP` through a 1kΩ current-limiting surge resistor (Brown-Black-Red-Gold).
    - Float switch closes to GND when water reaches 100% full.
    - 5-second continuous wave turbulence filter (`TANK_FULL_DEBOUNCE_MS = 5000`).
    - Eliminates boot restrictions completely because GPIO3 has NO boot strapping dependencies!
  - **`GPIO1 (TXD)` (Pin 8 - Debug Serial):**
    - Strictly initialized with `Serial.begin(115200, SERIAL_8N1, SERIAL_TX_ONLY)`.
    - Dedicates GPIO3 cleanly as digital input without UART RX interference.

---

## 8. Physical 2x4 Header Mapping (Silkscreen Orientation)
Hold the ESP-01 with the **black PCB facing you, antenna at the top, and the 8 pins at the bottom**:

```text
               ┌──────────────────────┐
               │     PCB ANTENNA      │
               │                      │
               │   ESP8266EX CHIP     │
               │                      │
    [Pin 1]    │ [GND]         [TX]   │ [Pin 8 - GPIO1 Debug TX]
    [Pin 2]    │ [GPIO2]      [CH_PD] │ [Pin 7 - EN / 3.3V]
    [Pin 3] ──►│ [GPIO0]      [RESET] │ [Pin 6 - Reset / 3.3V]
    [Pin 4] ──►│ [RX]          [VCC]  │ [Pin 5 - Power 3.3V]
               └──────────────────────┘
                  LEFT          RIGHT
                 COLUMN        COLUMN
```
* **Pin 1 (GND):** Power GND + Float Switch Wire 2 + 100µF Capacitor ($-$)
* **Pin 2 (GPIO2):** Onboard Blue LED (Leave header pin unconnected!)
* **Pin 3 (GPIO0):** Relay Module `IN` control wire
* **Pin 4 (RX / GPIO3):** Float Switch Wire 1 via 1kΩ Resistor
* **Pin 5 (VCC):** Regulated 3.3V Power Supply + 100µF Capacitor ($+$)
* **Pin 6 (RESET):** Connected to 3.3V (prevents electrostatic resets)
* **Pin 7 (CH_PD / EN):** Connected to 3.3V (chip enable)
* **Pin 8 (TX / GPIO1):** Serial TX (leave empty during normal operation)

> [!CAUTION]
> Online AI models (e.g. ChatGPT) frequently generate arbitrary 1–8 numbering tables that do NOT match the real physical connector. NEVER tell the user to count pins 1 to 8! Always refer to the exact printed silkscreen labels (`GND, GPIO2, GPIO0, RX, TX, CH_PD, RESET, VCC`).

---

## 9. Autonomous Firmware Versioning & GitHub Fleet OTA Release Protocol (MANDATORY FOR ALL AGENTS)

When any future agent modifies or upgrades the ESP-01S commercial controller firmware, **NEVER ASK THE USER TO REINSERT THE CHIP OR PRESS PHYSICAL BUTTONS.** 
The chip is permanently deployed in the customer's home. Follow this exact 6-step zero-touch release protocol:

### Step 1: Bump Version Numbers in Firmware
In `d:\Automation\v2.1\esp01_commercial_node\esp01_commercial_node.ino`:
```cpp
#define FIRMWARE_VERSION "2.9.2" // Increment Semantic Version (MAJOR.MINOR.PATCH)
#define FIRMWARE_BUILD 292       // Increment Integer Build Number (Always > previous)
```

### Step 2: Compile & Export 1MB Binary
Execute via PowerShell:
```powershell
& "C:\Users\verma\AppData\Local\Programs\Arduino IDE\resources\app\lib\backend\resources\arduino-cli.exe" compile --fqbn esp8266:esp8266:generic:eesz=1M,ssl=basic,FlashMode=dout,baud=115200 --export-binaries "d:\Automation\v2.1\esp01_commercial_node"
```

### Step 3: Copy Binary to Cloud App
```powershell
Copy-Item "d:\Automation\v2.1\esp01_commercial_node\build\esp8266.esp8266.generic\esp01_commercial_node.ino.bin" -Destination "d:\Automation\v2.1\smarttank_cloud_app\firmware.bin" -Force
```

### Step 4: Update Version Manifest (`version.json`)
In `d:\Automation\v2.1\smarttank_cloud_app\version.json`:
- Update `"version"` and `"build"`.
- Set `"release_date"` to the current ISO date (`YYYY-MM-DD`).
- Add a descriptive `"release_notes"` string.
- Prepend the new entry to the `"changelog"` array.

### Step 5: Deploy to GitHub Pages
```powershell
cd "d:\Automation\v2.1\smarttank_cloud_app"
git add .
git commit -m "release: vX.X.X (Build XXX) - <Feature summary>"
git push origin main
git push origin main:master
```

### Step 6: Broadcast Fleet OTA Update (Zero Touch!)
Once pushed, send a single MQTT message to trigger immediate fleet-wide update:
- Topic: `smarttank/TANK-3C05EA/cmd` or `smarttank/fleet/cmd`
- Payload: `{"cmd": "check_ota"}` or `{"ota_url": "https://pardeepsinghverma.github.io/automate.github.io/firmware.bin"}`
All connected customer devices globally will silently query GitHub, download `firmware.bin` over HTTPS in ~10 seconds using a memory-safe 4KB BearSSL buffer (`setBufferSizes(4096, 512)`), and reboot into the new version automatically without touching any buttons!

