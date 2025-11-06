# 🌾 **AgroSmart: Solar-Powered Water Budgeting Smart Irrigation System**

### 💡 *An IoT-based dual-ESP32 smart irrigation system powered by solar energy, featuring LoRa communication, weather prediction, and water budgeting logic.*

---

## 📘 **Project Overview**

**AgroSmart** is a solar-powered IoT irrigation system that automates and optimizes water use for agriculture.  
It monitors soil moisture, humidity, temperature, rainfall, tank level, and water flow to deliver the right amount of water at the right time.

The system uses **two ESP32 boards**:
- **Field Node** → Collects sensor data and sends it via **LoRa SX1278**
- **Home Node** → Receives data, fetches weather info, and controls irrigation using a **relay-controlled water pump**

It works **offline** via LoRa and **online** via Wi-Fi for cloud/Telegram features.

---

## ⚙️ **System Architecture**

### 🔹 Field Node
- Powered by **solar panel + battery**
- Equipped with:
  - Soil moisture sensor  
  - DHT11 (temperature & humidity)  
  - Rain sensor  
  - Flow sensor (water usage)  
  - Ultrasonic (tank level)  
  - Vibration sensor (end-line detection)
- Sends periodic readings to the home node via **LoRa SX1278**

### 🔹 Home Node
- Receives LoRa data and connects to Wi-Fi
- Uses **Visual Crossing Weather API** for rain prediction
- Performs **AI-inspired irrigation logic**  
- Controls **pump via relay**  
- Sends **Telegram alerts** for irrigation, water level, and rainfall updates
- Stores usage data in **EEPROM + Cloud**

---

## 📊 **Block Diagram**

<img width="761" height="438" alt="flowchart" src="https://github.com/user-attachments/assets/f96482c3-adf0-40b7-99e4-14c5ac86a6f3" />


> **Figure 1:** Block diagram showing field and home ESP32 modules.  
> The Field Node reads all sensors and transmits data to the Home Node.  
> The Home Node processes data, automates irrigation, and provides both online (cloud + web dashboard) and offline (OLED + alert) outputs.

---

## 🔄 **System Flow Diagram**

<img width="749" height="718" alt="Screenshot 2025-10-10 115117" src="https://github.com/user-attachments/assets/47fe328d-2971-4b98-9fb0-028d38b70344" />


> **Figure 2:** Data-flow representation of AgroSmart.  
> Sensor data → ESP32 Field → LoRa → ESP32 Home → Cloud/Telegram → Pump Control.  
> The system enables both **automatic** and **manual irrigation**, rain-based decision logic, and local voice notifications.

---

## 🔩 **Hardware Setup**

| Component | Description |
|------------|-------------|
| **ESP32-WROOM-32** | Dual microcontrollers for field and home nodes |
| **LoRa SX1278** | Long-range, low-power wireless data transfer |
| **Soil Moisture Sensor** | Detects soil wetness (0–100%) |
| **DHT11 Sensor** | Temperature & humidity measurement |
| **Flow Sensor (YF-S201)** | Monitors irrigation flow (L/min) |
| **Rain Sensor** | Detects rainfall to prevent over-watering |
| **Vibration Sensor** | Confirms water reaches drip endpoints |
| **Ultrasonic Sensor (HC-SR04)** | Measures water level in tank |
| **Relay Module** | Controls the water pump |
| **Solar Panel + Battery** | Sustainable power source |

---

## 🔌 **Pin Connections**

### 🧠 Field Node
| Component | ESP32 Pin | Notes |
|------------|------------|-------|
| Soil Moisture | GPIO34 | Analog input |
| DHT11 | GPIO4 | Digital input |
| Flow Sensor | GPIO27 | Interrupt pin |
| Rain Sensor | GPIO35 | Analog input |
| Vibration Sensor | GPIO32 | Digital input |
| Ultrasonic | Trig: GPIO12 / Echo: GPIO13 | Tank level detection |
| LoRa SX1278 | SCK:5, MISO:19, MOSI:27, CS:18, RST:14, DIO0:26 | SPI |
| Power | 5 V / 3.3 V | From solar battery |

### 🏠 Home Node
| Component | ESP32 Pin | Notes |
|------------|------------|-------|
| LoRa SX1278 | SCK:5, MISO:19, MOSI:27, CS:18, RST:14, DIO0:26 | SPI |
| Relay Module | GPIO23 | Controls water pump |
| Flow Sensor | GPIO25 | Tracks pump output |
| Wi-Fi | Built-in | For Weather API + Telegram |
| Power | 5 V | Solar/mains |

---

## 📚 **Libraries Used**

| Function | Library |
|-----------|----------|
| LoRa Communication | `<LoRa.h>` |
| Sensors | `<DHT.h>`, `<Ultrasonic.h>` |
| Networking | `<WiFi.h>`, `<HTTPClient.h>`, `<ArduinoJson.h>` |
| Data Storage | `<EEPROM.h>` |
| Alerts | `<UniversalTelegramBot.h>` |
| Core | `<Arduino.h>` |

> 💻 **IDE:** Arduino IDE / PlatformIO  
> 🧰 **Board:** ESP32 Dev Module  

---

## 💧 **Water Budgeting Logic**

1. **Soil moisture** determines dryness threshold.  
2. **Weather API** predicts rain; irrigation is skipped if rain is likely.  
3. **Flow sensor** measures water in liters/minute.  
4. **Tank level** is checked to prevent dry runs.  
5. **Water usage** is logged and used to compute the next irrigation cycle.  

**Formula:**
```text
WaterBudget = (TankCapacity - TotalUsed) + RainfallContribution
````

---

## ☁️ **Rain Prediction Integration**

* Uses **Visual Crossing Weather API** for rainfall probability & intensity.
* The system delays watering if precipitation is expected within 12 hours.
* Reduces unnecessary pump cycles and saves water.

---

## 🔔 **Telegram Bot Alerts**

Real-time messages include:

* 🌦 *“Soil Dry – Pump ON”*
* ✅ *“Irrigation Complete – 3.8 L Used”*
* ⚠️ *“Low Tank Level – Refill Required”*
* 💧 *“Rain Expected – Irrigation Postponed”*

---

## 🧮 **Expected Serial Output**

### 🔹 Field Node

```
Temp: 28.4°C | Humidity: 60% | Soil: 47% | Flow: 0.00 L/min | Rain: 0 | Tank: 82% | Vib: OK
[LoRa TX] Packet Sent → 28.4,60,47,0.0,0,82,1
```

### 🔹 Home Node

```
[LoRa RX] 28.4,60,47,0.0,0,82,1
[Weather] Rain Forecast: 5%
[Decision] Soil dry, no rain → Pump ON
[Flow] 3.8 L used | [EEPROM] Data Saved | [Telegram] Alert Sent
```

---

## 🌱 **Key Features**

* ✅ Dual-node LoRa architecture (offline operation)
* ☀️ Solar-powered, energy-independent
* 🌦 Rain-aware adaptive irrigation
* 💧 Real-time water budgeting and logging
* 📲 Telegram & dashboard alerts
* 🔄 Scalable for multi-farm setup

---

## 🔮 **Future Enhancements**

* Integrate **TinyML model** for crop-specific irrigation prediction
* Add **GSM fallback** for no-Wi-Fi regions
* Include **pH/TDS water quality sensors**
* Develop a **mobile app dashboard** for farmers

---

## 👨‍🔧 **Developed By – Team Tech_Spark**

| Member              | Role                           |
| ------------------- | ------------------------------ |
| **Afnan Jhadwale**  | Team Lead / Embedded Developer |
| **Tech_Spark Team** | IoT & Software Integration     |

---

## 📜 **License**

Released under the **MIT License** – free to use, modify, and share with credit.
