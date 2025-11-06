/*
  Smart Irrigation System - HOME NODE (Base Station)
  Features: WiFi Server, Web Dashboard, Weather API, Telegram Bot, Pump Control, AI Decision System
  Receives data from Field Node and controls irrigation
*/

// ===== INCLUDES =====
#include <WiFi.h>
#include <WiFiClient.h>
#include <HTTPClient.h>
#include <WiFiClientSecure.h>
#include <UniversalTelegramBot.h>
#include <WebServer.h>
#include <Preferences.h>
#include <ArduinoJson.h>

// ===== CONFIGURATION =====
#define WIFI_SSID "abcd"                              // CHANGE THIS
#define WIFI_PASSWORD "12345678"                      // CHANGE THIS
#define BOT_TOKEN "8350601538:AAAHTc1fpRKBm7XEdUYXwzKBiLSq2xT3StVw"  // CHANGE THIS
const String WEATHER_API_KEY = "6929ff9c9300336462813ae4059e2deac1";  // CHANGE THIS
const String WEATHER_CITY = "Bengaluru";

// ===== PIN DEFINITIONS =====
#define PUMP_PIN 16
#define RELAY_ACTIVE_STATE HIGH

// ===== TIMING CONSTANTS =====
const unsigned long BOT_MTBS = 500;
const unsigned long IRRIGATION_CHECK_INTERVAL = 2000;
const unsigned long WEATHER_UPDATE_INTERVAL = 15UL * 60UL * 1000UL;  // 15 minutes
const unsigned long HISTORY_UPDATE_INTERVAL = 3600000;               // 1 hour
const unsigned long DAILY_RESET_CHECK_INTERVAL = 60000;              // 1 minute
const unsigned long ALERT_THROTTLE_MS = 30000;                       // 30 seconds
const unsigned long FIELD_NODE_TIMEOUT = 30UL * 60UL * 1000UL;       // 30 minutes

// ===== GLOBAL OBJECTS =====
WebServer server(80);
Preferences preferences;
WiFiClientSecure secured_client;
UniversalTelegramBot bot(BOT_TOKEN, secured_client);

// ===== TIMING VARIABLES =====
unsigned long bot_lasttime = 0;
unsigned long irrigation_lasttime = 0;
unsigned long lastWeatherFetch = 0;
unsigned long lastHistoryUpdate = 0;
unsigned long lastAlertTime = 0;
unsigned long lastDailyResetCheck = 0;
unsigned long lastFieldDataReceived = 0;

// ===== SENSOR DATA FROM FIELD NODE =====
float temperature = NAN;
float humidity = NAN;
int soilMoisturePercent = -1;
float rainIntensity = 0.0;
float flowRate = 0.0;
float tankLevelLiters = 0.0;
float tankLevelPercent = 0.0;
bool vibrationDetected = false;
int fieldNodeRSSI = 0;
float fieldNodeBattery = 0.0;

// ===== IRRIGATION CONTROL VARIABLES =====
bool pumpState = false;
bool autoMode = true;
unsigned long irrigationStartTime = 0;
float irrigationDurationS = 0.0;
float totalWaterUsed = 0.0;
float dailyWaterUsed = 0.0;

// ===== ALERT SYSTEM =====
String lastAlert = "";
String alertHistory[5] = {"System Initialized."};
int alertHistoryCount = 1;
String TELEGRAM_CHAT_ID = "";

// ===== WEATHER DATA =====
String weatherMain = "N/A";
float rain1h_mm = 0.0;
float rainProb_percent = 0.0;
float windSpeed = 0.0;
int windDeg = 0;
float pressure = 0.0;
int lastDay = -1;

// ===== AI DECISION VARIABLES =====
String aiRecommendation = "System booting...";
String fertilizerRecommendation = "Checking...";
int daysTankLasts = 0;
float farmEfficiencyScore = 0.0;
float predictedWaterNeedLiters = 0.0;

// ===== HISTORICAL DATA =====
float tripleHistory[24 * 3] = {0};  // Soil, Temp, Hum for 24 hours
int historyIndex = 0;

// ===== FARMER PROFILE =====
struct FarmerProfile {
  char name[50] = "Farmer";
  char phone[20] = "+91-XXXXXXXXXX";
  char location[100] = "Karnataka, India";
  float landSize = 1.0;
  char cropType[50] = "Maize";
  char soilType[20] = "Red";
  char cropStartDate[20] = "2025-10-01";
  char cropEndDate[20] = "2026-02-01";
} farmerProfile;

// ===== FORWARD DECLARATIONS =====
void ai_water_budgeting();
void getFertilizerRecommendation();
void controlIrrigation();
void startPump(unsigned long duration);
void stopPump();
void sendAlert(String title, String message);
void fetchWeather();
void loadProfileFromFlash();
void saveProfileToFlash();
void updateHistoricalData();
void checkAndResetDailyUsage();
void checkFieldNodeStatus();

// ================================================================
// FERTILIZER RECOMMENDATION MODULE
// ================================================================
void getFertilizerRecommendation() {
  int temp = (int)temperature;
  int hum = (int)humidity;
  int soil = soilMoisturePercent;

  if (soil < 35) {
    fertilizerRecommendation = "Urgent Water needed. Apply N (Urea) post-watering.";
    return;
  }
  
  if (soil >= 35 && soil <= 60 && temp > 20 && temp < 35) {
    if (strcmp(farmerProfile.soilType, "Sandy") == 0) {
      fertilizerRecommendation = "Light N (Urea) dose. Sandy soil leaks.";
    } else {
      fertilizerRecommendation = "Moderate NPK (20-10-10) for growth.";
    }
    return;
  }

  if (temp >= 35 && hum < 50) {
    fertilizerRecommendation = "P/K Focus: Apply DAP (Phosphorus) for stress resistance.";
    return;
  }
  
  if (soil > 70 || hum > 80) {
    fertilizerRecommendation = "High Moisture Risk! Reduce watering. Use Compost/Fungicide.";
    return;
  }

  fertilizerRecommendation = "Optimal conditions. Maintenance NPK (15-15-15).";
}

// ================================================================
// AI WATER BUDGETING & IRRIGATION DECISION
// ================================================================
void ai_water_budgeting() {
  float baseWaterNeedLiters = 0.0;
  float evaporationFactor = 1.0;
  float soilFactor = 1.0;
  int targetMoistureMin = 50;

  // Determine base crop need and target moisture
  if (strcmp(farmerProfile.cropType, "Maize") == 0) {
    baseWaterNeedLiters = 25000.0 * farmerProfile.landSize;
    targetMoistureMin = 50;
  } else {
    baseWaterNeedLiters = 20000.0 * farmerProfile.landSize;
    targetMoistureMin = 50;
  }
  
  // Environmental Factor Adjustment
  if (temperature > 35.0 && humidity < 40.0) {
    evaporationFactor = 1.25;
  } else if (temperature < 20.0 || humidity > 70.0) {
    evaporationFactor = 0.90;
  }

  // Soil Factor Adjustment
  if (strcmp(farmerProfile.soilType, "Red") == 0 || strcmp(farmerProfile.soilType, "Sandy") == 0) {
    soilFactor = 1.15;
  } else {
    soilFactor = 0.85;
  }

  // Final Budget Calculation (Normalized for demo tank)
  predictedWaterNeedLiters = 0.5 * evaporationFactor * soilFactor;

  // Irrigation Duration Recommendation (Dynamic)
  float moistureDeficit = (float)targetMoistureMin - soilMoisturePercent;

  if (moistureDeficit > 5.0) {
    float requiredWaterLiters = moistureDeficit * 0.01;
    irrigationDurationS = requiredWaterLiters / 0.1667;  // Pump flow rate
    irrigationDurationS = constrain(irrigationDurationS, 5.0, 60.0);
    aiRecommendation = "Run pump for " + String((int)irrigationDurationS) + " seconds.";
  } else {
    irrigationDurationS = 0.0;
    aiRecommendation = "Soil optimal, irrigation paused. Target: " + String(targetMoistureMin) + "%";
  }

  // RAIN PREDICTION FACTOR
  if (rain1h_mm > 0.5 || rainProb_percent > 75.0 || rainIntensity > 0.5) {
    if (soilMoisturePercent < 45 && irrigationDurationS > 0) {
      irrigationDurationS *= 0.5;  // Halve duration
      aiRecommendation = "Rain expected, reduced irrigation to " + String((int)irrigationDurationS) + "s.";
    } else {
      aiRecommendation = "Rain detected/predicted, irrigation paused.";
      irrigationDurationS = 0.0;
    }
  }
  
  // Calculate efficiency score
  farmEfficiencyScore = constrain(100.0 - (abs(soilMoisturePercent - targetMoistureMin) * 2.0), 0.0, 100.0);

  // Days Tank Lasts
  if (predictedWaterNeedLiters > 0.001) {
    daysTankLasts = (int)(tankLevelLiters / predictedWaterNeedLiters);
  } else {
    daysTankLasts = 999;
  }
}

// ================================================================
// IRRIGATION CONTROL
// ================================================================
void startPump(unsigned long duration) {
  if (tankLevelLiters < 0.05) {
    sendAlert("⚠ Cannot Start Pump", "Tank level too low for irrigation.");
    return;
  }
  
  digitalWrite(PUMP_PIN, RELAY_ACTIVE_STATE);
  pumpState = true;
  irrigationStartTime = millis();
  
  if (duration > 0) {
    irrigationDurationS = (float)duration / 1000.0;
  } else {
    irrigationDurationS = 3600.0;  // 1 hour safety limit
  }
  
  Serial.println("Pump started");
}

void stopPump() {
  if (!pumpState) return;

  digitalWrite(PUMP_PIN, !RELAY_ACTIVE_STATE);
  pumpState = false;
  
  if (irrigationStartTime > 0) {
    unsigned long run_time_ms = millis() - irrigationStartTime;
    float water_used_liters = (float)run_time_ms / 1000.0 * 0.1667;  // L/sec
    totalWaterUsed += water_used_liters;
    dailyWaterUsed += water_used_liters;
    
    tankLevelLiters -= water_used_liters;
    tankLevelLiters = constrain(tankLevelLiters, 0.0, 0.5);
    tankLevelPercent = (tankLevelLiters / 0.5) * 100.0;

    Serial.println("Pump stopped. Used: " + String(water_used_liters, 3) + "L");
    sendAlert("⏹ Irrigation Stopped", "Water used: " + String(water_used_liters, 2) + "L");
  }
  
  irrigationStartTime = 0;
}

void controlIrrigation() {
  // Emergency stop if tank is almost empty
  if (pumpState && tankLevelLiters < 0.025) {
    stopPump();
    sendAlert("⚠ EMERGENCY STOP", "Tank almost empty!");
    return;
  }
  
  // Check for stop condition
  if (pumpState) {
    unsigned long currentRunTime = (millis() - irrigationStartTime);
    
    // Time-based stop
    if (currentRunTime >= (unsigned long)(irrigationDurationS * 1000.0)) {
      stopPump();
    }
    return;
  }

  if (!autoMode) return;

  // Auto-Mode Logic (Start)
  if (irrigationDurationS > 0.0 && !pumpState) {
    startPump((unsigned long)(irrigationDurationS * 1000.0));
    sendAlert("🌱 Irrigation Started (AI)", "Running for " + String((int)irrigationDurationS) + "s. Soil: " + String(soilMoisturePercent) + "%");
  }
}

// ================================================================
// ALERT SYSTEM
// ================================================================
void sendAlert(String title, String message) {
  if (millis() - lastAlertTime < ALERT_THROTTLE_MS) return;
  
  String newAlert = title + ": " + message;
  lastAlert = newAlert;
  lastAlertTime = millis();
  Serial.println("ALERT: " + newAlert);
  
  // Shift old alerts and add new
  for (int i = alertHistoryCount; i > 0; i--) {
    if (i < 5) alertHistory[i] = alertHistory[i - 1];
  }
  alertHistory[0] = newAlert;
  if (alertHistoryCount < 5) alertHistoryCount++;
  
  // Send Telegram notification
  if (TELEGRAM_CHAT_ID.length() > 0) {
    bot.sendMessage(TELEGRAM_CHAT_ID, "🚨 " + newAlert, "");
  }
}

// ================================================================
// CHECK FIELD NODE STATUS
// ================================================================
void checkFieldNodeStatus() {
  if (millis() - lastFieldDataReceived > FIELD_NODE_TIMEOUT && lastFieldDataReceived > 0) {
    sendAlert("⚠ Field Node Offline", "No data received for 30+ minutes.");
    
    // Pause irrigation for safety
    if (pumpState) {
      stopPump();
    }
    autoMode = false;
  }
}

// ================================================================
// WEATHER FETCH
// ================================================================
void fetchWeather() {
  if (WiFi.status() != WL_CONNECTED) return;
  
  HTTPClient http;
  String url = "http://api.openweathermap.org/data/2.5/weather?q=" + WEATHER_CITY + 
               "&appid=" + WEATHER_API_KEY + "&units=metric";

  http.begin(url);
  int httpCode = http.GET();
  
  if (httpCode == HTTP_CODE_OK) {
    String payload = http.getString();
    DynamicJsonDocument doc(4096);
    DeserializationError err = deserializeJson(doc, payload);
    
    if (!err) {
      weatherMain = doc["weather"][0]["main"].as<String>();
      pressure = doc["main"]["pressure"] | 0.0;
      windSpeed = doc["wind"]["speed"] | 0.0;
      windDeg = doc["wind"]["deg"] | 0;
      rain1h_mm = doc["rain"]["1h"] | 0.0;
      rainProb_percent = doc["clouds"]["all"] | 0;
      
      Serial.println("Weather: " + weatherMain + " Rain1h:" + String(rain1h_mm) + " Prob:" + String(rainProb_percent));
    } else {
      Serial.println("Weather JSON parse error.");
    }
  } else {
    Serial.println("Weather fetch failed, code: " + String(httpCode));
  }
  
  http.end();
}

// ================================================================
// TELEGRAM BOT HANDLERS
// ================================================================
void getChatId(int numNewMessages) {
  if (TELEGRAM_CHAT_ID.length() > 0) return;
  
  for (int i = 0; i < numNewMessages; i++) {
    TELEGRAM_CHAT_ID = bot.messages[i].chat_id;
    break;
  }
}

void handleNewMessages(int numNewMessages) {
  // Capture Chat ID if missing
  if (TELEGRAM_CHAT_ID.length() == 0) {
    getChatId(numNewMessages);
    if (TELEGRAM_CHAT_ID.length() > 0) {
      bot.sendMessage(TELEGRAM_CHAT_ID, "✅ System connected and ready! Send /help for commands.", "");
    }
  }

  for (int i = 0; i < numNewMessages; i++) {
    String chat_id = bot.messages[i].chat_id;
    String text = bot.messages[i].text;
    text.trim();
    
    if (TELEGRAM_CHAT_ID.length() > 0 && chat_id != TELEGRAM_CHAT_ID) continue;
    
    // Command handling
    if (text == "/sensors") {
      String msg = "📊 Sensor Data\n\n";
      msg += "🌡 Temp: " + String(temperature, 1) + "°C\n";
      msg += "💨 Humidity: " + String(humidity, 1) + "%\n";
      msg += "🌱 Soil: " + String(soilMoisturePercent) + "%\n";
      msg += "💧 Tank: " + String(tankLevelPercent, 1) + "% (" + String(tankLevelLiters, 2) + "L)\n";
      msg += "☔ Rain: " + String(rainIntensity, 1) + "mm\n";
      msg += "💦 Flow: " + String(flowRate, 2) + " L/min\n";
      msg += "🔋 Field Battery: " + String(fieldNodeBattery, 1) + "V\n";
      msg += "📶 RSSI: " + String(fieldNodeRSSI) + " dBm\n";
      msg += "💦 Pump: " + String(pumpState ? "ON" : "OFF") + "\n";
      bot.sendMessage(chat_id, msg, "Markdown");
    }
    else if (text == "/water") {
      String msg = "💧 Water Usage & Budget\n\n";
      msg += "Today: " + String(dailyWaterUsed, 3) + " L\n";
      msg += "Total: " + String(totalWaterUsed, 3) + " L\n";
      msg += "Tank: " + String(tankLevelLiters, 3) + " L\n";
      msg += "Tank Lasts: " + String(daysTankLasts) + " days (Est.)";
      bot.sendMessage(chat_id, msg, "Markdown");
    }
    else if (text == "/pumpon") {
      if (tankLevelPercent < 10) {
        bot.sendMessage(chat_id, "⚠ Cannot start - Tank level too low!", "");
      } else {
        autoMode = false;
        startPump(0);
        bot.sendMessage(chat_id, "💧 Pump turned ON (Manual mode, 1hr limit). Use /pumpoff to stop.", "");
      }
    }
    else if (text == "/pumpoff") {
      stopPump();
      bot.sendMessage(chat_id, "⏹ Pump turned OFF", "");
    }
    else if (text == "/auto") {
      autoMode = true;
      bot.sendMessage(chat_id, "🤖 Auto mode ENABLED.", "");
    }
    else if (text == "/manual") {
      autoMode = false;
      bot.sendMessage(chat_id, "👤 Manual mode ENABLED.", "");
    }
    else if (text == "/ai") {
      String msg = "🧠 AI Decision\n";
      msg += "Reco: " + aiRecommendation + "\n";
      msg += "Duration: " + String((int)irrigationDurationS) + "s\n";
      msg += "Fert Reco: " + fertilizerRecommendation;
      bot.sendMessage(chat_id, msg, "Markdown");
    }
    else if (text == "/fertilizer") {
      bot.sendMessage(chat_id, "🧪 " + fertilizerRecommendation, "");
    }
    else if (text == "/alerts") {
      String msg = "🔔 Recent Alerts (Max 5):\n";
      for (int j = 0; j < alertHistoryCount; j++) {
        msg += "- " + alertHistory[j] + "\n";
      }
      bot.sendMessage(chat_id, msg, "Markdown");
    }
    else if (text == "/help" || text == "/start") {
      String welcome = "🌱 Smart Irrigation System\n\n";
      welcome += "🎛 Control:\n/pumpon | /pumpoff | /auto | /manual\n";
      welcome += "📊 Data:\n/sensors | /water | /ai | /fertilizer\n";
      welcome += "🚨 Alerts:\n/alerts\n";
      bot.sendMessage(chat_id, welcome, "Markdown");
    }
    else {
      bot.sendMessage(chat_id, "Unknown command. Send /help", "");
    }
  }
}

// ================================================================
// WEB SERVER HANDLERS
// ================================================================

// API endpoint to receive field data
void handleFieldData() {
  if (server.method() != HTTP_POST) {
    server.send(405, "text/plain", "Method Not Allowed");
    return;
  }
  
  String body = server.arg("plain");
  StaticJsonDocument<512> doc;
  DeserializationError error = deserializeJson(doc, body);
  
  if (error) {
    server.send(400, "text/plain", "Invalid JSON");
    return;
  }
  
  // Parse field node data
  temperature = doc["temperature"] | 0.0;
  humidity = doc["humidity"] | 0.0;
  soilMoisturePercent = doc["soilMoisture"] | 0;
  rainIntensity = doc["rainIntensity"] | 0.0;
  flowRate = doc["flowRate"] | 0.0;
  tankLevelPercent = doc["tankLevelPercent"] | 0.0;
  tankLevelLiters = doc["tankLevelLiters"] | 0.0;
  vibrationDetected = doc["vibration"] | 0;
  fieldNodeRSSI = doc["rssi"] | 0;
  fieldNodeBattery = doc["battery"] | 0.0;
  
  lastFieldDataReceived = millis();
  
  Serial.println("\n=== Field Data Received ===");
  Serial.printf("Temp: %.1f°C, Humidity: %.1f%%, Soil: %d%%\n", temperature, humidity, soilMoisturePercent);
  Serial.printf("Tank: %.1f%% (%.2fL), Rain: %.1fmm, Flow: %.2f L/min\n", 
                tankLevelPercent, tankLevelLiters, rainIntensity, flowRate);
  Serial.printf("Battery: %.1fV, RSSI: %d dBm\n", fieldNodeBattery, fieldNodeRSSI);
  
  // Run AI decision
  ai_water_budgeting();
  getFertilizerRecommendation();
  
  server.send(200, "text/plain", "Data received successfully");
}

void handleRoot() {
  String html = R"rawliteral(<!DOCTYPE html><html><head><meta charset='UTF-8'><meta name='viewport' content='width=device-width,initial-scale=1'>
<title>Smart Irrigation Dashboard</title><style>*{margin:0;padding:0;box-sizing:border-box}body{font-family:'Segoe UI',system-ui,sans-serif;background:linear-gradient(135deg,#667eea 0%,#764ba2 100%);min-height:100vh}
.topbar{background:#fff;box-shadow:0 2px 10px rgba(0,0,0,.1);padding:15px 30px;display:flex;justify-content:space-between;align-items:center}
.status{display:flex;align-items:center;gap:8px}.status-dot{width:12px;height:12px;border-radius:50%;animation:pulse 2s infinite}
.status-active{background:#28a745}.status-inactive{background:#dc3545}@keyframes pulse{0%,100%{opacity:1}50%{opacity:.5}}
.container{max-width:1400px;margin:20px auto;padding:0 30px}
.grid{display:grid;grid-template-columns:repeat(auto-fit,minmax(250px,1fr));gap:20px;margin-bottom:30px}
.card{background:white;border-radius:15px;padding:25px;box-shadow:0 3px 15px rgba(0,0,0,.08);transition:transform .3s}
.card:hover{transform:translateY(-5px)}
.card-title{font-size:14px;color:#666;margin-bottom:10px}
.card-value{font-size:36px;font-weight:bold;color:#333;margin:10px 0}
.card-unit{font-size:18px;color:#999;font-weight:normal}
.btn{padding:12px 30px;border:none;border-radius:25px;font-weight:600;cursor:pointer;transition:all .3s;box-shadow:0 3px 10px rgba(0,0,0,.1);margin:5px}
.btn-primary{background:linear-gradient(135deg,#667eea,#764ba2);color:white}
.btn-danger{background:#dc3545;color:white}
h2{color:#fff;margin-bottom:20px;font-size:28px}
.control-panel{background:white;padding:20px;border-radius:15px;margin-bottom:20px}
</style></head><body>
<div class='topbar'><div class='status'><div class='status-dot status-active' id='statusDot'></div>
<span id='statusText'>Home Node Active</span></div></div>
<div class='container'>
<h2>🏠 Home Node - Irrigation Dashboard</h2>
<div class='grid'>
<div class='card'><div class='card-title'>🌡 Temperature</div><div class='card-value' id='temp'>--<span class='card-unit'>°C</span></div></div>
<div class='card'><div class='card-title'>💨 Humidity</div><div class='card-value' id='humidity'>--<span class='card-unit'>%</span></div></div>
<div class='card'><div class='card-title'>🌱 Soil Moisture</div><div class='card-value' id='soil'>--<span class='card-unit'>%</span></div></div>
<div class='card'><div class='card-title'>💧 Tank Level</div><div class='card-value' id='tank'>--<span class='card-unit'>%</span></div></div>
<div class='card'><div class='card-title'>☔ Rain Intensity</div><div class='card-value' id='rain'>--<span class='card-unit'>mm</span></div></div>
<div class='card'><div class='card-title'>💦 Flow Rate</div><div class='card-value' id='flow'>--<span class='card-unit'>L/min</span></div></div>
<div class='card'><div class='card-title'>🔋 Field Battery</div><div class='card-value' id='battery'>--<span class='card-unit'>V</span></div></div>
<div class='card'><div class='card-title'>💧 Water Used Today</div><div class='card-value' id='water'>--<span class='card-unit'>L</span></div></div>
</div>
<div class='control-panel'>
<h3>Pump Control</h3>
<div>Status: <span id='pumpStatus'>OFF</span></div>
<div>Mode: <span id='mode'>AUTO</span></div>
<div>AI Recommendation: <span id='ai'>--</span></div>
<button class='btn btn-primary' onclick='pumpOn()'>Pump ON</button>
<button class='btn btn-danger' onclick='pumpOff()'>Pump OFF</button>
<button class='btn btn-primary' onclick='toggleAuto()'>Toggle Auto</button>
</div>
</div>
<script>
function updateData(){fetch('/api/data').then(r=>r.json()).then(data=>{
document.getElementById('temp').innerHTML=data.temperature.toFixed(1)+'<span class="card-unit">°C</span>';
document.getElementById('humidity').innerHTML=data.humidity.toFixed(1)+'<span class="card-unit">%</span>';
document.getElementById('soil').innerHTML=data.soilMoisture+'<span class="card-unit">%</span>';
document.getElementById('tank').innerHTML=data.tankPercent.toFixed(1)+'<span class="card-unit">%</span>';
document.getElementById('rain').innerHTML=data.rain.toFixed(1)+'<span class="card-unit">mm</span>';
document.getElementById('flow').innerHTML=data.flow.toFixed(2)+'<span class="card-unit">L/min</span>';
document.getElementById('battery').innerHTML=data.battery.toFixed(1)+'<span class="card-unit">V</span>';
document.getElementById('water').innerHTML=data.dailyWater.toFixed(2)+'<span class="card-unit">L</span>';
document.getElementById('pumpStatus').textContent=data.pump?'ON':'OFF';
document.getElementById('mode').textContent=data.auto?'AUTO':'MANUAL';
document.getElementById('ai').textContent=data.ai;
}).catch(e=>console.error(e))}
function pumpOn(){fetch('/api/pump/on',{method:'POST'}).then(r=>r.text()).then(msg=>alert(msg))}
function pumpOff(){fetch('/api/pump/off',{method:'POST'}).then(r=>r.text()).then(msg=>alert(msg))}
function toggleAuto(){fetch('/api/auto',{method:'POST'}).then(r=>r.text()).then(msg=>alert(msg))}
setInterval(updateData,2000);updateData();
</script></body></html>
)rawliteral";
  server.send(200, "text/html", html);
}

void handleAPIData() {
  StaticJsonDocument<1024> doc;
  doc["temperature"] = temperature;
  doc["humidity"] = humidity;
  doc["soilMoisture"] = soilMoisturePercent;
  doc["tankPercent"] = tankLevelPercent;
  doc["tankLiters"] = tankLevelLiters;
  doc["rain"] = rainIntensity;
  doc["flow"] = flowRate;
  doc["battery"] = fieldNodeBattery;
  doc["dailyWater"] = dailyWaterUsed;
  doc["totalWater"] = totalWaterUsed;
  doc["pump"] = pumpState;
  doc["auto"] = autoMode;
  doc["ai"] = aiRecommendation;
  doc["fertilizer"] = fertilizerRecommendation;
  doc["daysLasts"] = daysTankLasts;
  doc["weatherMain"] = weatherMain;
  doc["rain1h"] = rain1h_mm;
  doc["rainProb"] = rainProb_percent;
  doc["windSpeed"] = windSpeed;
  doc["pressure"] = pressure;
  doc["rssi"] = fieldNodeRSSI;
  doc["vibration"] = vibrationDetected;
  
  String output;
  serializeJson(doc, output);
  server.send(200, "application/json", output);
}

void handleAPIPumpOn() {
  if (tankLevelPercent < 10) {
    server.send(400, "text/plain", "Cannot start - Tank level too low!");
    return;
  }
  autoMode = false;
  startPump(0);
  server.send(200, "text/plain", "Pump turned ON (Manual mode)");
}

void handleAPIPumpOff() {
  stopPump();
  server.send(200, "text/plain", "Pump turned OFF");
}

void handleAPIAuto() {
  autoMode = !autoMode;
  if (!autoMode && pumpState) {
    stopPump();
  }
  server.send(200, "text/plain", autoMode ? "Auto mode ENABLED" : "Manual mode ENABLED");
}

// ================================================================
// FLASH MEMORY FUNCTIONS
// ================================================================
void loadProfileFromFlash() {
  preferences.begin("irrigation", false);
  preferences.getString("name", farmerProfile.name, sizeof(farmerProfile.name));
  preferences.getString("phone", farmerProfile.phone, sizeof(farmerProfile.phone));
  preferences.getString("location", farmerProfile.location, sizeof(farmerProfile.location));
  farmerProfile.landSize = preferences.getFloat("landSize", 1.0);
  preferences.getString("cropType", farmerProfile.cropType, sizeof(farmerProfile.cropType));
  preferences.getString("soilType", farmerProfile.soilType, sizeof(farmerProfile.soilType));
  preferences.getString("cropStart", farmerProfile.cropStartDate, sizeof(farmerProfile.cropStartDate));
  preferences.getString("cropEnd", farmerProfile.cropEndDate, sizeof(farmerProfile.cropEndDate));
  preferences.end();
}

void saveProfileToFlash() {
  preferences.begin("irrigation", false);
  preferences.putString("name", farmerProfile.name);
  preferences.putString("phone", farmerProfile.phone);
  preferences.putString("location", farmerProfile.location);
  preferences.putFloat("landSize", farmerProfile.landSize);
  preferences.putString("cropType", farmerProfile.cropType);
  preferences.putString("soilType", farmerProfile.soilType);
  preferences.putString("cropStart", farmerProfile.cropStartDate);
  preferences.putString("cropEnd", farmerProfile.cropEndDate);
  preferences.end();
}

// ================================================================
// HISTORICAL DATA UPDATE & DAILY RESET
// ================================================================
void checkAndResetDailyUsage() {
  if (WiFi.status() != WL_CONNECTED) return;
  
  time_t now = time(NULL);
  struct tm *tm = localtime(&now);

  if (lastDay == -1 || tm->tm_mday != lastDay) {
    if (lastDay != -1) {
      dailyWaterUsed = 0.0;
      Serial.println("Daily water usage reset at midnight.");
      sendAlert("📅 Daily Reset", "Water usage counter reset.");
    }
    lastDay = tm->tm_mday;
  }
}

void updateHistoricalData() {
  if (millis() - lastHistoryUpdate >= HISTORY_UPDATE_INTERVAL) {
    historyIndex = (historyIndex + 1) % 24;
    int baseIndex = historyIndex * 3;

    tripleHistory[baseIndex] = soilMoisturePercent;
    tripleHistory[baseIndex + 1] = temperature;
    tripleHistory[baseIndex + 2] = humidity;
    
    lastHistoryUpdate = millis();
    Serial.println("History updated. Index: " + String(historyIndex));
  }
}

// ================================================================
// SETUP
// ================================================================
void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\n\n");
  Serial.println("========================================");
  Serial.println("  HOME NODE - Smart Irrigation System");
  Serial.println("  Control & Dashboard Unit");
  Serial.println("========================================");
  
  // Initialize pins
  pinMode(PUMP_PIN, OUTPUT);
  digitalWrite(PUMP_PIN, !RELAY_ACTIVE_STATE);
  
  // Load farmer profile from flash
  loadProfileFromFlash();
  
  // Connect to WiFi
  Serial.print("Connecting to WiFi");
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  secured_client.setInsecure();
  
  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED && attempts < 30) {
    Serial.print(".");
    delay(500);
    attempts++;
  }
  
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("\nWiFi connected!");
    Serial.print("IP Address: ");
    Serial.println(WiFi.localIP());
    Serial.println("Field Node should send data to this IP!");
  } else {
    Serial.println("\nWiFi Connection Failed!");
  }
  
  // Set up NTP for accurate time keeping
  configTime(5 * 3600 + 30 * 60, 0, "pool.ntp.org"); // IST (+5:30)
  
  // Initialize web server routes
  server.on("/", handleRoot);
  server.on("/api/data", handleAPIData);
  server.on("/api/field_data", HTTP_POST, handleFieldData);
  server.on("/api/pump/on", HTTP_POST, handleAPIPumpOn);
  server.on("/api/pump/off", HTTP_POST, handleAPIPumpOff);
  server.on("/api/auto", HTTP_POST, handleAPIAuto);
  server.begin();
  
  Serial.println("Web server started!");
  
  // Fetch initial weather data
  fetchWeather();
  
  // Check for pending Telegram messages
  int initialMessages = bot.getUpdates(0);
  if (initialMessages > 0) {
    getChatId(initialMessages);
    if (TELEGRAM_CHAT_ID.length() > 0) {
      bot.sendMessage(TELEGRAM_CHAT_ID, "✅ Home Node started! Waiting for Field Node data...", "");
    }
  }
  
  Serial.println("=== System Ready ===");
  Serial.println("Waiting for Field Node transmissions...\n");
}

// ================================================================
// MAIN LOOP
// ================================================================
void loop() {
  unsigned long now = millis();
  
  // 1. WEB SERVER
  server.handleClient();
  
  // 2. WEATHER FETCH
  if (now - lastWeatherFetch >= WEATHER_UPDATE_INTERVAL) {
    fetchWeather();
    lastWeatherFetch = now;
  }
  
  // 3. IRRIGATION CONTROL & HISTORY
  if (now - irrigation_lasttime >= IRRIGATION_CHECK_INTERVAL) {
    controlIrrigation();
    updateHistoricalData();
    checkFieldNodeStatus();
    irrigation_lasttime = now;
  }
  
  // 4. DAILY USAGE RESET
  if (now - lastDailyResetCheck >= DAILY_RESET_CHECK_INTERVAL) {
    checkAndResetDailyUsage();
    lastDailyResetCheck = now;
  }
  
  // 5. TELEGRAM BOT
  if (now - bot_lasttime >= BOT_MTBS) {
    int numNewMessages = bot.getUpdates(bot.last_message_received + 1);
    if (numNewMessages > 0) {
      handleNewMessages(numNewMessages);
    }
    bot_lasttime = now;
  }
}
