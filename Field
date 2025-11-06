/*
  Smart Irrigation System - FIELD NODE (Solar Powered)
  Features: DHT11, Soil Moisture, HC-SR04, Flow Sensor, Rain Sensor, Vibration Sensor
  Communication: WiFi transmission to Home Node via HTTP POST
  Power: Solar panel with deep sleep between transmissions
*/

// ===== INCLUDES =====
#include <WiFi.h>
#include <HTTPClient.h>
#include <DHT.h>
#include <ArduinoJson.h>

// ===== CONFIGURATION =====
#define WIFI_SSID "abcd"                    // CHANGE THIS
#define WIFI_PASSWORD "12345678"            // CHANGE THIS
#define HOME_NODE_IP "192.168.1.100"        // CHANGE TO HOME NODE IP
#define HOME_NODE_PORT 80

// ===== PIN DEFINITIONS =====
#define DHT_PIN 4
#define DHT_TYPE DHT11
#define SOIL_MOISTURE_PIN 34
#define RAIN_SENSOR_PIN 35
#define FLOW_SENSOR_PIN 27
#define VIBRATION_PIN 32
#define TRIG_PIN 12
#define ECHO_PIN 13

// ===== TIMING & CALIBRATION =====
#define SLEEP_INTERVAL_S 600                // 10 minutes between readings
#define SENSOR_STABILIZATION_MS 2000        // Wait for sensors to stabilize
#define FLOW_PULSES_PER_LITER 450.0        // YF-S201 calibration
#define RAIN_CAL_FACTOR 0.01               // Rain sensor mm per unit
#define MOISTURE_MIN 1200                   // Dry soil ADC value
#define MOISTURE_MAX 3100                   // Wet soil ADC value
#define MAX_WIFI_ATTEMPTS 20                // Max connection attempts

// Tank geometry (conical frustum)
#define TANK_HEIGHT 12.0                    // cm active height
#define SENSOR_TO_FULL 2.0                  // cm from sensor to full
#define SENSOR_TO_EMPTY (SENSOR_TO_FULL + TANK_HEIGHT)
#define TANK_R_BOTTOM 3.0                   // cm bottom radius
#define TANK_R_TOP 4.0                      // cm top radius

// ===== GLOBAL OBJECTS =====
DHT dht(DHT_PIN, DHT_TYPE);
HTTPClient http;

// ===== GLOBAL VARIABLES =====
volatile unsigned long flowPulseCount = 0;
unsigned long flowStartTime = 0;
bool flowMeasurementActive = false;

struct SensorData {
  float temperature = 0.0;
  float humidity = 0.0;
  int soilMoisturePercent = 0;
  float rainIntensity = 0.0;
  float flowRate = 0.0;
  float distance = 0.0;
  float tankLevelPercent = 0.0;
  float tankLevelLiters = 0.0;
  bool vibrationDetected = false;
  int wifiRSSI = 0;
  float batteryVoltage = 0.0;
} sensorData;

// ===== INTERRUPT SERVICE ROUTINE =====
void IRAM_ATTR flowPulseCounter() {
  flowPulseCount++;
}

// ===== TANK LEVEL CALCULATION =====
float calculateConicalTankLevel(float dist_cm) {
  float h_water = SENSOR_TO_EMPTY - dist_cm;
  h_water = constrain(h_water, 0.0, TANK_HEIGHT);
  if (h_water <= 0) return 0.0;

  const float H = TANK_HEIGHT;
  float r_h = TANK_R_BOTTOM + (TANK_R_TOP - TANK_R_BOTTOM) * (h_water / H);
  float volume_cm3 = (PI / 3.0) * h_water * 
                     (TANK_R_BOTTOM * TANK_R_BOTTOM + 
                      TANK_R_BOTTOM * r_h + 
                      r_h * r_h);
  return volume_cm3 / 1000.0; // Convert to liters
}

// ===== SENSOR READING FUNCTIONS =====

// Read DHT11 Temperature & Humidity
void readDHT() {
  sensorData.temperature = dht.readTemperature();
  sensorData.humidity = dht.readHumidity();
  
  if (isnan(sensorData.temperature)) {
    Serial.println("DHT read error - Temperature");
    sensorData.temperature = 0.0;
  }
  if (isnan(sensorData.humidity)) {
    Serial.println("DHT read error - Humidity");
    sensorData.humidity = 0.0;
  }
  
  Serial.printf("DHT: Temp=%.1f°C, Humidity=%.1f%%\n", 
                sensorData.temperature, sensorData.humidity);
}

// Read Soil Moisture Sensor
void readSoilMoisture() {
  int rawValue = analogRead(SOIL_MOISTURE_PIN);
  sensorData.soilMoisturePercent = map(rawValue, MOISTURE_MAX, MOISTURE_MIN, 0, 100);
  sensorData.soilMoisturePercent = constrain(sensorData.soilMoisturePercent, 0, 100);
  
  Serial.printf("Soil Moisture: %d%% (Raw: %d)\n", 
                sensorData.soilMoisturePercent, rawValue);
}

// Read Rain Sensor
void readRainSensor() {
  int rainValue = analogRead(RAIN_SENSOR_PIN);
  sensorData.rainIntensity = map(rainValue, 4095, 0, 0, 100) * RAIN_CAL_FACTOR;
  sensorData.rainIntensity = constrain(sensorData.rainIntensity, 0.0, 100.0);
  
  Serial.printf("Rain Intensity: %.2fmm (Raw: %d)\n", 
                sensorData.rainIntensity, rainValue);
}

// Read Flow Sensor (YF-S201)
void readFlowSensor() {
  // Start flow measurement
  flowPulseCount = 0;
  flowStartTime = millis();
  attachInterrupt(digitalPinToInterrupt(FLOW_SENSOR_PIN), flowPulseCounter, RISING);
  
  delay(1000); // Measure for 1 second
  
  detachInterrupt(digitalPinToInterrupt(FLOW_SENSOR_PIN));
  unsigned long duration = millis() - flowStartTime;
  
  if (duration > 0) {
    float liters = flowPulseCount / FLOW_PULSES_PER_LITER;
    sensorData.flowRate = (liters / (duration / 1000.0)) * 60.0; // L/min
  } else {
    sensorData.flowRate = 0.0;
  }
  
  Serial.printf("Flow Rate: %.2f L/min (Pulses: %lu)\n", 
                sensorData.flowRate, flowPulseCount);
}

// Read Ultrasonic Distance Sensor (HC-SR04)
void readUltrasonicSensor() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  
  long duration = pulseIn(ECHO_PIN, HIGH, 30000);
  sensorData.distance = duration * 0.0343 / 2.0;
  
  if (sensorData.distance > 400 || sensorData.distance <= 0) {
    Serial.println("Ultrasonic: Out of range");
    sensorData.distance = -1;
    sensorData.tankLevelLiters = 0.0;
    sensorData.tankLevelPercent = 0.0;
  } else {
    sensorData.tankLevelLiters = calculateConicalTankLevel(sensorData.distance);
    sensorData.tankLevelPercent = (sensorData.tankLevelLiters / 0.5) * 100.0;
    sensorData.tankLevelPercent = constrain(sensorData.tankLevelPercent, 0, 100);
    
    Serial.printf("Tank: %.1fcm, %.2fL (%.1f%%)\n", 
                  sensorData.distance, 
                  sensorData.tankLevelLiters, 
                  sensorData.tankLevelPercent);
  }
}

// Read Vibration Sensor
void readVibrationSensor() {
  int vibValue = digitalRead(VIBRATION_PIN);
  sensorData.vibrationDetected = (vibValue == HIGH);
  
  Serial.printf("Vibration: %s\n", 
                sensorData.vibrationDetected ? "DETECTED" : "None");
}

// Read Battery Voltage (via voltage divider on GPIO33)
void readBatteryVoltage() {
  // Assuming a voltage divider for 12V battery monitoring
  // Adjust multiplier based on your voltage divider ratio
  int rawValue = analogRead(33);
  sensorData.batteryVoltage = (rawValue / 4095.0) * 3.3 * 4.0; // Example for 4:1 divider
  
  Serial.printf("Battery: %.2fV\n", sensorData.batteryVoltage);
}

// ===== READ ALL SENSORS =====
void readAllSensors() {
  Serial.println("\n=== Reading All Sensors ===");
  
  readDHT();
  delay(100);
  
  readSoilMoisture();
  delay(100);
  
  readRainSensor();
  delay(100);
  
  readFlowSensor();
  delay(100);
  
  readUltrasonicSensor();
  delay(100);
  
  readVibrationSensor();
  delay(100);
  
  readBatteryVoltage();
  
  sensorData.wifiRSSI = WiFi.RSSI();
  
  Serial.println("=== Sensor Reading Complete ===\n");
}

// ===== WIFI CONNECTION =====
bool connectToWiFi() {
  Serial.print("Connecting to WiFi");
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  
  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED && attempts < MAX_WIFI_ATTEMPTS) {
    delay(500);
    Serial.print(".");
    attempts++;
  }
  
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("\nWiFi Connected!");
    Serial.print("IP Address: ");
    Serial.println(WiFi.localIP());
    Serial.print("Signal Strength: ");
    Serial.println(WiFi.RSSI());
    return true;
  } else {
    Serial.println("\nWiFi Connection Failed!");
    return false;
  }
}

// ===== SEND DATA TO HOME NODE =====
bool sendDataToHomeNode() {
  if (WiFi.status() != WL_CONNECTED) {
    Serial.println("ERROR: WiFi not connected");
    return false;
  }
  
  // Create JSON payload
  StaticJsonDocument<512> doc;
  doc["temperature"] = sensorData.temperature;
  doc["humidity"] = sensorData.humidity;
  doc["soilMoisture"] = sensorData.soilMoisturePercent;
  doc["rainIntensity"] = sensorData.rainIntensity;
  doc["flowRate"] = sensorData.flowRate;
  doc["tankLevelPercent"] = sensorData.tankLevelPercent;
  doc["tankLevelLiters"] = sensorData.tankLevelLiters;
  doc["vibration"] = sensorData.vibrationDetected ? 1 : 0;
  doc["rssi"] = sensorData.wifiRSSI;
  doc["battery"] = sensorData.batteryVoltage;
  doc["timestamp"] = millis() / 1000;
  
  String jsonPayload;
  serializeJson(doc, jsonPayload);
  
  Serial.println("\n=== Sending Data to Home Node ===");
  Serial.println("Payload: " + jsonPayload);
  
  // Send HTTP POST request
  String url = "http://" + String(HOME_NODE_IP) + ":" + String(HOME_NODE_PORT) + "/api/field_data";
  http.begin(url);
  http.addHeader("Content-Type", "application/json");
  
  int httpCode = http.POST(jsonPayload);
  
  if (httpCode > 0) {
    Serial.printf("HTTP Response Code: %d\n", httpCode);
    
    if (httpCode == HTTP_CODE_OK) {
      String response = http.getString();
      Serial.println("Server Response: " + response);
      http.end();
      return true;
    }
  } else {
    Serial.printf("HTTP POST failed: %s\n", http.errorToString(httpCode).c_str());
  }
  
  http.end();
  return false;
}

// ===== DEEP SLEEP =====
void enterDeepSleep() {
  Serial.println("\n=== Entering Deep Sleep ===");
  Serial.printf("Sleep Duration: %d seconds\n", SLEEP_INTERVAL_S);
  Serial.println("==========================\n");
  
  // Configure wake-up timer
  esp_sleep_enable_timer_wakeup(SLEEP_INTERVAL_S * 1000000ULL);
  
  // Disconnect WiFi to save power
  WiFi.disconnect(true);
  WiFi.mode(WIFI_OFF);
  
  delay(100);
  
  // Enter deep sleep
  esp_deep_sleep_start();
}

// ===== SETUP =====
void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\n\n");
  Serial.println("========================================");
  Serial.println("  FIELD NODE - Smart Irrigation System");
  Serial.println("  Solar Powered Sensor Unit");
  Serial.println("========================================");
  
  // Print wake-up reason
  esp_sleep_wakeup_cause_t wakeup_reason = esp_sleep_get_wakeup_cause();
  switch(wakeup_reason) {
    case ESP_SLEEP_WAKEUP_TIMER: 
      Serial.println("Wake-up: Timer"); 
      break;
    case ESP_SLEEP_WAKEUP_EXT0: 
      Serial.println("Wake-up: External Signal"); 
      break;
    default: 
      Serial.println("Wake-up: Power On/Reset"); 
      break;
  }
  
  // Initialize pins
  pinMode(SOIL_MOISTURE_PIN, INPUT);
  pinMode(RAIN_SENSOR_PIN, INPUT);
  pinMode(FLOW_SENSOR_PIN, INPUT);
  pinMode(VIBRATION_PIN, INPUT);
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  // Initialize DHT sensor
  dht.begin();
  
  // Wait for sensors to stabilize
  Serial.println("Waiting for sensors to stabilize...");
  delay(SENSOR_STABILIZATION_MS);
  
  // Connect to WiFi
  if (!connectToWiFi()) {
    Serial.println("Failed to connect to WiFi. Skipping transmission.");
    enterDeepSleep();
    return;
  }
  
  // Read all sensors
  readAllSensors();
  
  // Send data to Home Node
  bool success = sendDataToHomeNode();
  
  if (success) {
    Serial.println("\n✓ Data transmission successful!");
  } else {
    Serial.println("\n✗ Data transmission failed!");
  }
  
  // Enter deep sleep
  enterDeepSleep();
}

// ===== LOOP =====
void loop() {
  // Empty - device wakes from deep sleep and runs setup() each time
}
