Arduino Code

#include <WiFi.h>
#include <WebServer.h>
#include "DHT.h"

// ================= WIFI =================
const char* ssid = "Wokwi-GUEST";   // Change to your WiFi SSID
const char* password = "";           // Change to your WiFi password

// ================= DHT =================
#define DHTPIN 4
#define DHTTYPE DHT22
DHT dht(DHTPIN, DHTTYPE);

// ================= OUTPUT =================
#define LED_PIN 2
bool ledState = false;

WebServer server(80);

// ================= HTML DASHBOARD =================
String webpage = R"rawliteral(
<!DOCTYPE html>
<html>
<head>
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>ESP32 IoT Dashboard</title>
<style>
body { font-family: Arial; background:#f0f2f5; text-align:center; }
.container { max-width:400px; margin:auto; }
.card {
  background:white;
  padding:20px;
  margin:15px;
  border-radius:10px;
  box-shadow:0 4px 8px rgba(0,0,0,0.2);
}
button {
  padding:12px 20px;
  font-size:16px;
  border:none;
  border-radius:6px;
  cursor:pointer;
}
.on { background:green; color:white; }
.off { background:red; color:white; }
</style>
</head>
<body>

<h2>ESP32 Web Dashboard</h2>

<div class="container">
  <div class="card">
    <h3>Temperature</h3>
    <p id="temp">-- °C</p>
  </div>

  <div class="card">
    <h3>Humidity</h3>
    <p id="hum">-- %</p>
  </div>

  <div class="card">
    <h3>LED Control</h3>
    <button id="btn" onclick="toggleLED()">Toggle</button>
  </div>
</div>

<script>
function updateData(){
  fetch("/data")
  .then(response => response.json())
  .then(data => {
    document.getElementById("temp").innerHTML = data.temp + " °C";
    document.getElementById("hum").innerHTML = data.hum + " %";

    let btn = document.getElementById("btn");
    if(data.led == 1){
      btn.className="on";
      btn.innerHTML="ON";
    } else {
      btn.className="off";
      btn.innerHTML="OFF";
    }
  });
}

function toggleLED(){
  fetch("/toggle");
}

setInterval(updateData,2000);
updateData();
</script>

</body>
</html>
)rawliteral";

// ================= HANDLERS =================
void handleRoot() {
  server.send(200, "text/html", webpage);
}

void handleData() {
  float t = dht.readTemperature();
  float h = dht.readHumidity();

  // Print to Serial Monitor
  Serial.print("Temperature: ");
  Serial.print(t);
  Serial.print(" °C, Humidity: ");
  Serial.print(h);
  Serial.print(" %, LED: ");
  Serial.println(ledState ? "ON" : "OFF");

  String json = "{";
  json += "\"temp\":" + String(t) + ",";
  json += "\"hum\":" + String(h) + ",";
  json += "\"led\":" + String(ledState ?  0  :  1);
  json += "}";

  server.send(200, "application/json", json);
}

void handleToggle() {
  ledState = !ledState;
  digitalWrite(LED_PIN, ledState);
  Serial.print("LED Toggled: ");
  Serial.println(ledState ? "Off" : "On");
  server.send(200, "text/plain", "OK");
}

// ================= SETUP =================
void setup() {
  Serial.begin(115200);
  dht.begin();
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);

  Serial.println("Connecting to WiFi...");
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("\nConnected!");
  Serial.print("ESP32 IP Address: ");
  Serial.println(WiFi.localIP());
  Serial.println("Dashboard is ready!");

  server.on("/", handleRoot);
  server.on("/data", handleData);
  server.on("/toggle", handleToggle);

  server.begin();
}

// ================= LOOP =================
void loop() {
  server.handleClient();

  // Print sensor & LED every 2 seconds
  static unsigned long lastPrint = 0;
  if (millis() - lastPrint > 2000) {
    lastPrint = millis();
    float t = dht.readTemperature();
    float h = dht.readHumidity();
    Serial.print("Temperature: ");
    Serial.print(t);
    Serial.print(" °C, Humidity: ");
    Serial.print(h);
    Serial.print(" %, LED: ");
    Serial.println(ledState ? "ON" : "OFF");
  }
}
