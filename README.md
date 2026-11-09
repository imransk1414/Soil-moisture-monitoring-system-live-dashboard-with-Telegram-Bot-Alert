# ESP32 Soil Moisture Monitoring System

An IoT soil moisture monitor built on the ESP32, with a live web dashboard hosted on the board itself and Telegram bot alerts. No cloud server, SD card or subscription needed.

Author: Shaik Imran

---

## Features

- Reads a two-prong soil probe (LM393 module) through the ESP32's 12-bit ADC
- Converts the raw reading to a moisture percentage using two-point calibration
- Responsive dashboard served directly from the ESP32 (phone and desktop, light and dark mode)
  - Live gauge with colour-coded condition
  - Raw ADC value, Wi-Fi signal (dBm), average / min / max, uptime
  - 20-minute history chart with alert thresholds marked
  - Dry-warning banner and alert log
  - "Live" / "Device offline" status
- Telegram bot with automatic dry and recovery alerts and on-demand commands
- Hysteresis and cooldown to avoid message spam
- Automatic Wi-Fi reconnection

---

## Hardware

| Component | Purpose |
| --- | --- |
| ESP32 development board | Reads the sensor, runs the web server and Telegram client |
| Soil moisture probe with LM393 module | Analog output falls as soil gets wetter |
| Jumper wires and breadboard | Connections |
| USB cable and 5 V supply | Power and programming |
| Wi-Fi network with internet access | Dashboard and Telegram |

### Wiring

| Sensor pin | ESP32 pin |
| --- | --- |
| VCC | 3V3 |
| GND | GND |
| AOUT | GPIO34 |

Note: GPIO34 is on ADC1, which works while Wi-Fi is active. ADC2 pins are unreliable with Wi-Fi on.

---

## Software Requirements

- Arduino IDE with the ESP32 board package
- WiFi.h, WebServer.h, WiFiClientSecure.h (included with the ESP32 core)
- UniversalTelegramBot by Brian Lough
- ArduinoJson 6.21.x by Benoit Blanchon
- Telegram app, @BotFather and @myidbot

---

## Setup

1. Create the Telegram bot: message @BotFather, copy the bot token, and get your chat ID from @myidbot.
2. Install UniversalTelegramBot and ArduinoJson (6.21.x) from the Arduino Library Manager.
3. Wire the sensor as shown above.
4. In the sketch's user-settings section, enter your Wi-Fi name and password, the bot token and the chat ID.
5. Calibrate: note the raw ADC value in open air (dry) and in water (wet), and enter them as the calibration values.
6. Upload to the ESP32, open the Serial Monitor at 115200 baud to see the IP address, then open it in a browser on the same Wi-Fi network.

---

## How It Works

### Calibration

Moisture (%) = (Raw - 3200) * 100 / (1400 - 3200), clamped to 0-100 %

| Point | Raw ADC | Moisture |
| --- | --- | --- |
| Dry (open air) | 3200 | 0 % |
| Wet (in water) | 1400 | 100 % |

These values are specific to the sensor used. Re-measure them for your own sensor and soil.

### Sensor reading

12-bit ADC (0-4095), 11 dB attenuation, each reading is the average of 20 samples taken 5 ms apart.

### Moisture levels

| Moisture | Condition |
| --- | --- |
| Below 30 % | Dry, needs water (alert) |
| 30 % - 59 % | Moist (good) |
| 60 % and above | Wet |

### Alert logic (hysteresis)

- Dry alert: sent when moisture drops below 30 %, repeated only after a 30-minute cooldown while the soil stays dry.
- Recovery message: sent only when moisture rises above 40 %.
- Dead band (30-40 %): no new messages.

### Timing

| Parameter | Value |
| --- | --- |
| Sensor check interval | 10 s |
| Telegram polling interval | 1 s |
| Dashboard refresh | 5 s |
| History buffer | 120 readings (about 20 min), kept in RAM |
| Alert cooldown | 30 min |

---

## Web Server Endpoints

| Address | Function |
| --- | --- |
| / | Dashboard page (HTML, CSS, JS stored in flash) |
| /data | JSON with raw value, Wi-Fi RSSI, uptime, thresholds, sampling step and moisture history |

---

## Telegram Commands

Only the authorised chat ID is accepted. Anyone else gets "Unauthorized user".

| Command | Reply |
| --- | --- |
| /start | Welcome message, command list and dashboard address |
| /moisture | Current moisture %, condition and raw ADC value |
| /status | Moisture details, Wi-Fi signal, uptime and dashboard address |
| anything else | "Unknown command" with a hint to send /start |

Automatic messages: online message on every start-up, dry alerts and recovery messages.

---

## Test Results

| Situation | Raw ADC | Moisture | Condition |
| --- | --- | --- | --- |
| Sensor dry / out of moisture | 4095 | 0 % | Dry, alert sent |
| Dry soil (Telegram test) | 2855 | 19 % | Dry |
| Moist soil | 2141 | 58 % | Moist (good) |
| Wet soil | 1969 | 68 % | Wet, recovery message |

Observations:

- The Telegram alert and dashboard banner appeared together, so both use the same sensor data.
- Recovery messages were sent only after moisture crossed 40 %, confirming the hysteresis logic.
- /moisture and /status matched the dashboard values at the same moment.

---

## Limitations

- Dashboard is reachable only on the same Wi-Fi network unless port forwarding or a VPN is set up.
- History is stored in RAM and is lost on reset or power loss.
- Resistive probes corrode over time in wet soil, causing drift.
- Calibration is sensor- and soil-specific.
- The system only monitors; it does not water the plant.

## Future Scope

- Relay and water pump for automatic watering
- LED or buzzer for local alerts
- Long-term storage on a cloud database or SD card
- DHT11 temperature and humidity sensor
- Battery and solar power with deep sleep
- Adjustable thresholds from the dashboard or Telegram

---

## References

- Espressif Systems, ESP32 Series Datasheet and Arduino-ESP32 documentation
- Brian Lough, UniversalTelegramBot library
- Benoit Blanchon, ArduinoJson library
- Telegram Bot API: https://core.telegram.org/bots/api
