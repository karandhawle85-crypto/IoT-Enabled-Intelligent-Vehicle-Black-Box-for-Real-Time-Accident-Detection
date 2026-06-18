# IoT-Enabled-Intelligent-Vehicle-Black-Box-for-Real-Time-Accident-Detection
#code
// ======================================================
// ESP32 Smart Vehicle Safety Monitoring System
// WITH WEB SERVER & ThingSpeak - Live Data & Fault Display
// ======================================================
// Features:
// MQ6 Alcohol Detection
// DS18B20 Temperature Sensor
// ADXL345 Accident Detection
// GPS Live Location
// GSM SMS Alert
// SD Card Data Logging
// 16x2 I2C LCD
// Relay Control
// Buzzer Alert
// WEB SERVER with Live Data & Fault Display
// ThingSpeak IoT Platform Integration
// ======================================================

#include <Wire.h>
#include <Adafruit_Sensor.h>
#include <Adafruit_ADXL345_U.h>
#include <OneWire.h>
#include <DallasTemperature.h>
#include <LiquidCrystal_I2C.h>
#include <TinyGPS++.h>
#include <HardwareSerial.h>
#include "FS.h"
#include "SD.h"
#include "SPI.h"
#include <WiFi.h>
#include <WebServer.h>
#include <ThingSpeak.h>

// ======================================================
// FORWARD DECLARATION
// ======================================================

void sendSMS(String locationLink);
void sendToThingSpeak();

// ======================================================
// WiFi CREDENTIALS - Connect to Aimers Networks
// ======================================================

const char* ssid1 = "Aimers-2GHz";
const char* password1 = "Aimers@254";
const char* ssid2 = "";
const char* password2 = "";

// ======================================================
// ThingSpeak Credentials
// ======================================================

unsigned long myChannelNumber = 3389393;  // Your channel number
const char* myWriteAPIKey = "32ESXTQMINGMA6HW";     // Your Write API Key

// ======================================================
// PIN DEFINITIONS
// ======================================================

// DS18B20
#define DS18B20_PIN 27

// MQ6
#define MQ6_PIN 34

// Relay
#define RELAY_PIN 14

// Buzzer
#define BUZZER_PIN 32

// GPS
#define GPS_RX 25
#define GPS_TX 26

// GSM
#define GSM_RX 16
#define GSM_TX 17

// SD Card
#define SD_CS 5

// ======================================================
// THRESHOLD
// ======================================================

#define ALCOHOL_THRESHOLD 2000

// ======================================================
// ThingSpeak Field Mapping
// Field 1: Temperature
// Field 2: Alcohol Value
// Field 3: Accel X
// Field 4: Accel Y
// Field 5: Accel Z
// Field 6: Latitude
// Field 7: Longitude
// Field 8: Status (0=Normal, 1=Alcohol, 2=Temp, 3=Accident)
// ======================================================

// ======================================================
// OBJECTS
// ======================================================

// DS18B20
OneWire oneWire(DS18B20_PIN);
DallasTemperature sensors(&oneWire);

// LCD
LiquidCrystal_I2C lcd(0x27, 16, 2);

// ADXL345
Adafruit_ADXL345_Unified accel = Adafruit_ADXL345_Unified(12345);

// GPS
TinyGPSPlus gps;

// GPS Serial
HardwareSerial gpsSerial(2);

// GSM Serial
HardwareSerial gsmSerial(1);

// Web Server
WebServer server(80);

// WiFi Client for ThingSpeak
WiFiClient client;

// ======================================================
// VARIABLES
// ======================================================

bool accidentDetected = false;
bool smsSent = false;

// Default GPS
String latitude = "18.729050357849335";
String longitude = "73.42941678597792";

// Mobile Number
String phoneNumber = "+918999161502";

// Sensor Variables
float currentTemp = 0;
int currentAlcohol = 0;
float currentAccelX = 0;
float currentAccelY = 0;
float currentAccelZ = 0;
float currentSpeed = 0;
int currentSatellites = 0;
float currentAltitude = 0;

// Fault Flags
bool alcoholFault = false;
bool tempFault = false;
bool gpsFault = false;
bool accelFault = false;

// WiFi status
bool wifiConnected = false;

// ThingSpeak timer
unsigned long lastThingSpeakUpdate = 0;
const unsigned long thingSpeakInterval = 15000; // Update every 15 seconds

// Status code for ThingSpeak
// 0 = Normal
// 1 = Alcohol Detected
// 2 = High Temperature
// 3 = Accident Detected
int systemStatus = 0;

// ======================================================
// HTML WEB PAGE (Without Map)
// ======================================================

String getHTMLPage() {
  String html = R"rawliteral(
<!DOCTYPE html>
<html>
<head>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ESP32 Vehicle Monitor</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #1a1a2e, #16213e);
            color: #fff;
            min-height: 100vh;
            padding: 20px;
        }
        .container {
            max-width: 1200px;
            margin: 0 auto;
        }
        h1 {
            text-align: center;
            margin-bottom: 20px;
            font-size: 1.8em;
            color: #00d4ff;
        }
        .status-bar {
            background: #0f3460;
            border-radius: 10px;
            padding: 15px;
            margin-bottom: 20px;
            display: flex;
            justify-content: space-between;
            flex-wrap: wrap;
            gap: 10px;
        }
        .status-item {
            padding: 8px 15px;
            border-radius: 20px;
            font-size: 14px;
        }
        .status-normal {
            background: #00b894;
        }
        .status-warning {
            background: #e17055;
        }
        .status-danger {
            background: #d63031;
        }
        .sensor-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
            gap: 20px;
            margin-bottom: 20px;
        }
        .sensor-card {
            background: rgba(255,255,255,0.1);
            border-radius: 15px;
            padding: 20px;
            backdrop-filter: blur(10px);
            border: 1px solid rgba(255,255,255,0.2);
        }
        .sensor-title {
            font-size: 1.2em;
            margin-bottom: 15px;
            color: #00d4ff;
            border-bottom: 1px solid rgba(255,255,255,0.2);
            padding-bottom: 8px;
        }
        .sensor-value {
            font-size: 2em;
            font-weight: bold;
            margin: 10px 0;
        }
        .sensor-unit {
            font-size: 0.5em;
            color: #aaa;
        }
        .location-container {
            background: rgba(255,255,255,0.1);
            border-radius: 15px;
            padding: 20px;
            margin-bottom: 20px;
            text-align: center;
        }
        .location-coords {
            font-size: 1.2em;
            margin: 10px 0;
            font-family: monospace;
        }
        .google-link {
            display: inline-block;
            margin-top: 10px;
            padding: 10px 20px;
            background: #00d4ff;
            color: #1a1a2e;
            text-decoration: none;
            border-radius: 25px;
            font-weight: bold;
        }
        .faults-section {
            background: rgba(255,255,255,0.1);
            border-radius: 15px;
            padding: 20px;
        }
        .fault-item {
            padding: 10px;
            margin: 8px 0;
            border-radius: 8px;
            background: rgba(0,0,0,0.3);
        }
        .fault-active {
            border-left: 4px solid #d63031;
            background: rgba(214,48,49,0.2);
        }
        .refresh-btn {
            display: block;
            width: 200px;
            margin: 20px auto;
            padding: 12px;
            background: #00d4ff;
            color: #1a1a2e;
            border: none;
            border-radius: 25px;
            font-size: 16px;
            font-weight: bold;
            cursor: pointer;
            text-align: center;
        }
        .refresh-btn:hover {
            background: #00b8d4;
        }
        .footer {
            text-align: center;
            margin-top: 20px;
            padding: 15px;
            font-size: 12px;
            color: #888;
        }
        @media (max-width: 768px) {
            .sensor-value {
                font-size: 1.5em;
            }
        }
    </style>
    <script>
        function refreshData() {
            fetch('/data')
                .then(response => response.json())
                .then(data => {
                    document.getElementById('tempValue').innerHTML = data.temp + '<span class="sensor-unit">°C</span>';
                    document.getElementById('alcoholValue').innerHTML = data.alcohol;
                    document.getElementById('accelX').innerHTML = data.accelX.toFixed(2);
                    document.getElementById('accelY').innerHTML = data.accelY.toFixed(2);
                    document.getElementById('accelZ').innerHTML = data.accelZ.toFixed(2);
                    document.getElementById('latitude').innerHTML = data.lat;
                    document.getElementById('longitude').innerHTML = data.lng;
                    document.getElementById('googleLink').href = 'https://maps.google.com/?q=' + data.lat + ',' + data.lng;
                    
                    var faultsHtml = '';
                    if(data.alcoholFault) faultsHtml += '<div class="fault-item fault-active">ALCOHOL DETECTED - Vehicle Immobilized</div>';
                    if(data.tempFault) faultsHtml += '<div class="fault-item fault-active">HIGH TEMPERATURE - Engine Overheat Risk</div>';
                    if(data.accident) faultsHtml += '<div class="fault-item fault-active">ACCIDENT DETECTED - Emergency Alert Sent</div>';
                    if(data.gpsFault) faultsHtml += '<div class="fault-item fault-active">GPS SIGNAL LOST - Location May Be Inaccurate</div>';
                    if(!data.alcoholFault && !data.tempFault && !data.accident && !data.gpsFault) {
                        faultsHtml += '<div class="fault-item">All Systems Normal - No Faults Detected</div>';
                    }
                    document.getElementById('faultsList').innerHTML = faultsHtml;
                    
                    var statusHtml = '';
                    statusHtml += '<div class="status-item ' + (data.alcoholFault ? 'status-danger' : 'status-normal') + '">Alcohol: ' + (data.alcoholFault ? 'DETECTED' : 'OK') + '</div>';
                    statusHtml += '<div class="status-item ' + (data.tempFault ? 'status-danger' : 'status-normal') + '">Temp: ' + (data.tempFault ? 'HIGH' : 'Normal') + '</div>';
                    statusHtml += '<div class="status-item ' + (data.accident ? 'status-danger' : 'status-normal') + '">Accident: ' + (data.accident ? 'YES' : 'NO') + '</div>';
                    statusHtml += '<div class="status-item ' + (data.gpsFault ? 'status-warning' : 'status-normal') + '">GPS: ' + (data.gpsFault ? 'Weak' : 'Locked') + '</div>';
                    document.getElementById('statusBar').innerHTML = statusHtml;
                })
                .catch(error => console.log('Error:', error));
        }
        
        setInterval(refreshData, 3000);
        window.onload = refreshData;
    </script>
</head>
<body>
    <div class="container">
        <h1>ESP32 Vehicle Safety Monitor</h1>
        
        <div id="statusBar" class="status-bar"></div>
        
        <div class="sensor-grid">
            <div class="sensor-card">
                <div class="sensor-title">Temperature (DS18B20)</div>
                <div class="sensor-value" id="tempValue">--<span class="sensor-unit">°C</span></div>
                <div>Status: <span id="tempStatus">--</span></div>
            </div>
            
            <div class="sensor-card">
                <div class="sensor-title">Alcohol (MQ6)</div>
                <div class="sensor-value" id="alcoholValue">--</div>
                <div>Threshold: 2000 | Status: <span id="alcoholStatus">--</span></div>
            </div>
            
            <div class="sensor-card">
                <div class="sensor-title">Accelerometer (ADXL345)</div>
                <div>X: <span id="accelX">--</span> g</div>
                <div>Y: <span id="accelY">--</span> g</div>
                <div>Z: <span id="accelZ">--</span> g</div>
            </div>
            
            <div class="sensor-card">
                <div class="sensor-title">GPS Location</div>
                <div>Latitude: <span id="latitude">--</span></div>
                <div>Longitude: <span id="longitude">--</span></div>
                <a id="googleLink" class="google-link" href="#" target="_blank">View on Google Maps</a>
            </div>
        </div>
        
        <div class="location-container">
            <div class="sensor-title">Live Vehicle Location</div>
            <div class="location-coords">
                Lat: <span id="latitude2">--</span> | Lng: <span id="longitude2">--</span>
            </div>
        </div>
        
        <div class="faults-section">
            <div class="sensor-title">System Faults & Alerts</div>
            <div id="faultsList"></div>
        </div>
        
        <button class="refresh-btn" onclick="refreshData()">Refresh Now</button>
        <div class="footer">
            ESP32 Vehicle Safety System | Connected to Aimers Network | Data sent to ThingSpeak every 15s
        </div>
    </div>
</body>
</html>
)rawliteral";
  return html;
}

// ======================================================
// JSON DATA ENDPOINT
// ======================================================

String getJSONData() {
  String json = "{";
  json += "\"temp\":" + String(currentTemp) + ",";
  json += "\"alcohol\":" + String(currentAlcohol) + ",";
  json += "\"accelX\":" + String(currentAccelX) + ",";
  json += "\"accelY\":" + String(currentAccelY) + ",";
  json += "\"accelZ\":" + String(currentAccelZ) + ",";
  json += "\"lat\":\"" + latitude + "\",";
  json += "\"lng\":\"" + longitude + "\",";
  json += "\"alcoholFault\":" + String(alcoholFault ? "true" : "false") + ",";
  json += "\"tempFault\":" + String(tempFault ? "true" : "false") + ",";
  json += "\"accident\":" + String(accidentDetected ? "true" : "false") + ",";
  json += "\"gpsFault\":" + String(gpsFault ? "true" : "false");
  json += "}";
  return json;
}

// ======================================================
// WEB SERVER HANDLERS
// ======================================================

void handleRoot() {
  server.send(200, "text/html", getHTMLPage());
}

void handleData() {
  server.send(200, "application/json", getJSONData());
}

// ======================================================
// CONNECT TO AIMERS NETWORKS
// ======================================================

void connectToWiFi() {
  Serial.println("\n========== CONNECTING TO AIMERS NETWORK ==========");
  
  Serial.print("Connecting to Aimers-2GHz");
  WiFi.begin(ssid1, password1);
  
  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED && attempts < 10) {
    delay(1000);
    Serial.print(".");
    attempts++;
  }
  
  if (WiFi.status() == WL_CONNECTED) {
    wifiConnected = true;
    Serial.println("\nConnected to Aimers-2GHz");
    Serial.print("IP Address: ");
    Serial.println(WiFi.localIP());
    
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("WiFi: Aimers-2GHz");
    lcd.setCursor(0, 1);
    lcd.print(WiFi.localIP());
    delay(3000);
    return;
  }
  
  Serial.print("\nConnecting to Aimers@254");
  WiFi.begin(ssid2, password2);
  
  attempts = 0;
  while (WiFi.status() != WL_CONNECTED && attempts < 10) {
    delay(1000);
    Serial.print(".");
    attempts++;
  }
  
  if (WiFi.status() == WL_CONNECTED) {
    wifiConnected = true;
    Serial.println("\nConnected to Aimers@254");
    Serial.print("IP Address: ");
    Serial.println(WiFi.localIP());
    
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("WiFi: Aimers@254");
    lcd.setCursor(0, 1);
    lcd.print(WiFi.localIP());
    delay(3000);
  } else {
    wifiConnected = false;
    Serial.println("\nFailed to connect to any Aimers network");
    
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("WiFi Failed");
    lcd.setCursor(0, 1);
    lcd.print("Running Offline");
    delay(3000);
  }
  
  Serial.println("==================================================\n");
}

// ======================================================
// SEND DATA TO THINGSPEAK
// ======================================================

void sendToThingSpeak() {
  if (!wifiConnected) {
    Serial.println("ThingSpeak: No WiFi connection, skipping update");
    return;
  }
  
  // Update system status code
  if (accidentDetected) {
    systemStatus = 3;
  } else if (alcoholFault) {
    systemStatus = 1;
  } else if (tempFault) {
    systemStatus = 2;
  } else {
    systemStatus = 0;
  }
  
  // Convert latitude and longitude to float for ThingSpeak
  float latFloat = latitude.toFloat();
  float lngFloat = longitude.toFloat();
  
  // Set ThingSpeak fields
  ThingSpeak.setField(1, currentTemp);
  ThingSpeak.setField(2, currentAlcohol);
  ThingSpeak.setField(3, currentAccelX);
  ThingSpeak.setField(4, currentAccelY);
  ThingSpeak.setField(5, currentAccelZ);
  ThingSpeak.setField(6, latFloat);
  ThingSpeak.setField(7, lngFloat);
  ThingSpeak.setField(8, systemStatus);
  
  // Write to ThingSpeak
  int httpCode = ThingSpeak.writeFields(myChannelNumber, myWriteAPIKey);
  
  if (httpCode == 200) {
    Serial.println("ThingSpeak: Data sent successfully!");
    Serial.print("  Temperature: ");
    Serial.print(currentTemp);
    Serial.print(" | Alcohol: ");
    Serial.print(currentAlcohol);
    Serial.print(" | Status: ");
    Serial.println(systemStatus);
    
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("TS Data Sent");
    lcd.setCursor(0, 1);
    lcd.print("Status: OK");
    delay(1000);
    lcd.clear();
  } else {
    Serial.print("ThingSpeak: Error sending data. HTTP code: ");
    Serial.println(httpCode);
    
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("TS Error");
    lcd.setCursor(0, 1);
    lcd.print("Code: ");
    lcd.print(httpCode);
    delay(1000);
  }
}

// ======================================================
// SETUP
// ======================================================

void setup() {

  Serial.begin(115200);

  sensors.begin();

  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  digitalWrite(RELAY_PIN, HIGH);
  digitalWrite(BUZZER_PIN, LOW);

  lcd.init();
  lcd.backlight();

  lcd.setCursor(0, 0);
  lcd.print("System Starting");
  delay(2000);
  lcd.clear();

  if (!accel.begin()) {
    lcd.setCursor(0, 0);
    lcd.print("ADXL Error");
    accelFault = true;
    Serial.println("ADXL345 Not Found");
  } else {
    accel.setRange(ADXL345_RANGE_16_G);
    accelFault = false;
  }

  gpsSerial.begin(9600, SERIAL_8N1, 26, 25);
  Serial.println("GPS Reading Started...");

  gsmSerial.begin(9600, SERIAL_8N1, GSM_RX, GSM_TX);

  Serial.println("Initializing SD Card...");

  if (!SD.begin(SD_CS)) {
    Serial.println("SD Card Mount Failed!");
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("SD Failed");
  } else {
    Serial.println("SD Card Ready");
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("SD Card Ready");

    File file = SD.open("/log.txt", FILE_APPEND);
    if (file) {
      file.println("===== SYSTEM STARTED =====");
      file.close();
    }
  }

  delay(2000);
  lcd.clear();

  connectToWiFi();

  // Initialize ThingSpeak
  if (wifiConnected) {
    ThingSpeak.begin(client);
    Serial.println("ThingSpeak initialized");
  }

  if (wifiConnected) {
    server.on("/", handleRoot);
    server.on("/data", handleData);
    server.begin();
    Serial.println("Web Server Started");
    
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Server Ready");
    lcd.setCursor(0, 1);
    lcd.print(WiFi.localIP());
    delay(3000);
  }

  lcd.clear();
  Serial.println("System Ready");
}

// ======================================================
// LOOP
// ======================================================

void loop() {

  if (wifiConnected) {
    server.handleClient();
  }

  while (gpsSerial.available() > 0) {
    char c = gpsSerial.read();
    Serial.write(c);
    gps.encode(c);

    if (gps.location.isUpdated()) {
      latitude = String(gps.location.lat(), 6);
      longitude = String(gps.location.lng(), 6);
      currentSpeed = gps.speed.kmph();
      currentSatellites = gps.satellites.value();
      currentAltitude = gps.altitude.meters();
      
      if (currentSatellites < 3) {
        gpsFault = true;
      } else {
        gpsFault = false;
      }

      Serial.println("\n========== GPS DATA ==========");
      Serial.print("Latitude: ");
      Serial.println(latitude);
      Serial.print("Longitude: ");
      Serial.println(longitude);
      Serial.println("==============================");
    }
  }

  sensors.requestTemperatures();
  currentTemp = sensors.getTempCByIndex(0);
  
  if (currentTemp > 60) {
    tempFault = true;
  } else {
    tempFault = false;
  }

  currentAlcohol = analogRead(MQ6_PIN);
  
  if (currentAlcohol > ALCOHOL_THRESHOLD) {
    alcoholFault = true;
  } else {
    alcoholFault = false;
  }

  if (!accelFault) {
    sensors_event_t event;
    accel.getEvent(&event);
    currentAccelX = event.acceleration.x;
    currentAccelY = event.acceleration.y;
    currentAccelZ = event.acceleration.z;
  }

  String googleLink = "https://maps.google.com/?q=" + latitude + "," + longitude;

  if (!accelFault && (currentAccelX < -5 || currentAccelX > 5 ||
       currentAccelY < -5 || currentAccelY > 5) &&
      !accidentDetected) {

    accidentDetected = true;

    Serial.println("Accident Detected!");

    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Accident Alert");
    lcd.setCursor(0, 1);
    lcd.print("Sending SMS");

    digitalWrite(RELAY_PIN, HIGH);
    digitalWrite(BUZZER_PIN, HIGH);

    if (!smsSent) {
      sendSMS(googleLink);
      smsSent = true;
    }
  }

  // Send data to ThingSpeak at regular intervals
  if (wifiConnected && (millis() - lastThingSpeakUpdate >= thingSpeakInterval)) {
    sendToThingSpeak();
    lastThingSpeakUpdate = millis();
  }

  if (!accidentDetected) {
    lcd.clear();

    if (!alcoholFault && !tempFault) {
      lcd.setCursor(0, 0);
      lcd.print("Alcohol OK");
      lcd.setCursor(0, 1);
      lcd.print("Temp:");
      lcd.print(currentTemp);
      lcd.print((char)223);
      lcd.print("C");

      digitalWrite(RELAY_PIN, LOW);
      digitalWrite(BUZZER_PIN, LOW);
    } else {
      lcd.setCursor(0, 0);
      if (alcoholFault) {
        lcd.print("Alcohol Found");
      } else if (tempFault) {
        lcd.print("Temp High");
      }
      lcd.setCursor(0, 1);
      lcd.print("Problem Detect");

      digitalWrite(RELAY_PIN, HIGH);
      digitalWrite(BUZZER_PIN, HIGH);
    }
  }

  File file = SD.open("/log.txt", FILE_APPEND);
  if (file) {
    file.print("Temp: ");
    file.print(currentTemp);
    file.print(" , Alcohol: ");
    file.print(currentAlcohol);
    file.print(" , AccelX: ");
    file.print(currentAccelX);
    file.print(" , AccelY: ");
    file.print(currentAccelY);
    file.print(" , Latitude: ");
    file.print(latitude);
    file.print(" , Longitude: ");
    file.print(longitude);
    file.print(" , Link: ");
    file.print(googleLink);

    if (accidentDetected) {
      file.print(" , STATUS: ACCIDENT");
    } else if (alcoholFault) {
      file.print(" , STATUS: ALCOHOL DETECTED");
    } else if (tempFault) {
      file.print(" , STATUS: HIGH TEMP");
    } else {
      file.print(" , STATUS: NORMAL");
    }
    file.println();
    file.close();
    Serial.println("Data Saved to SD Card");
  } else {
    Serial.println("SD Write Failed");
  }

  Serial.println("\n========== LIVE DATA ==========");
  Serial.print("Temperature: ");
  Serial.println(currentTemp);
  Serial.print("Alcohol Value: ");
  Serial.println(currentAlcohol);
  Serial.print("Accel X: ");
  Serial.println(currentAccelX);
  Serial.print("Accel Y: ");
  Serial.println(currentAccelY);
  Serial.print("Latitude: ");
  Serial.println(latitude);
  Serial.print("Longitude: ");
  Serial.println(longitude);
  Serial.println("================================");

  delay(1000);
}

// ======================================================
// SEND SMS FUNCTION
// ======================================================

void sendSMS(String locationLink) {
  Serial.println("Sending SMS...");

  gsmSerial.println("AT");
  delay(1000);

  gsmSerial.println("AT+CMGF=1");
  delay(1000);

  gsmSerial.print("AT+CMGS=\"");
  gsmSerial.print(phoneNumber);
  gsmSerial.println("\"");N
  delay(1000);

  gsmSerial.println("Accident Detected!");
  gsmSerial.println("Vehicle Location:");
  gsmSerial.println(locationLink);
  delay(500);

  gsmSerial.write(26);
  Serial.println("SMS Sent");
}
