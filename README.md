# Spin-BLDC-motor
This project is aim to spin BLDC motor by using smartphone/pc


#include <WiFi.h>
#include <ESPAsyncWebServer.h>

// AP credentials
const char* ssid = "ESP32-AP";
const char* password = "123456789";

// PWM configuration
const int pwmPin = 23;  // PWM output to ESC signal pin
const int pwmFreq = 50;  // 50Hz (20ms period) for ESC
const int pwmChannel = 0;
const int pwmResolution = 16;

// Web server
AsyncWebServer server(80);

// Microseconds to duty cycle conversion
void writeMicroseconds(uint8_t channel, int microseconds) {
  // Convert microseconds to 16-bit duty cycle for 50Hz
  int duty = (microseconds * 65536) / 20000;
  ledcWrite(channel, duty);
}

void setup() {
  Serial.begin(115200);

  // Start Access Point
  WiFi.softAP(ssid, password);
  delay(1000);  // Give it a moment to start

  IPAddress IP = WiFi.softAPIP(); // Get local IP
  Serial.println("=================================");
  Serial.println("ESP32 Access Point Started ✅");
  Serial.print("SSID: ");
  Serial.println(ssid);
  Serial.print("Password: ");
  Serial.println(password);
  Serial.print("Access Web Interface at: http://");
  Serial.println(IP); // <<< THIS IS YOUR IP
  Serial.println("=================================");

  // Setup PWM
  ledcSetup(pwmChannel, pwmFreq, pwmResolution);
  ledcAttachPin(pwmPin, pwmChannel);

  // Arm ESC with 1000us signal
  Serial.println("Sending arming signal to ESC...");
  writeMicroseconds(pwmChannel, 1000);
  delay(2000); // Wait for ESC to initialize

  // Web UI
  server.on("/", HTTP_GET, [](AsyncWebServerRequest *request){
    String html = "<html><body>";
    html += "<h2>BLDC Motor Speed Control</h2>";
    html += "<input type='range' min='0' max='255' value='0' id='slider' oninput='sendSpeed(this.value)'>";
    html += "<p>Speed: <span id='value'>0</span></p>";
    html += "<script>";
    html += "function sendSpeed(val){";
    html += "document.getElementById('value').innerText = val;";
    html += "fetch('/setSpeed?value=' + val);";
    html += "}";
    html += "</script></body></html>";
    request->send(200, "text/html", html);
  });

  // Handle slider value
  server.on("/setSpeed", HTTP_GET, [](AsyncWebServerRequest *request){
    if (request->hasParam("value")) {
      String val = request->getParam("value")->value();
      int speed = val.toInt();
      Serial.print("Received slider value: ");
      Serial.println(speed);

      // Map slider value to PWM signal (1000-2000µs range)
      int microseconds = map(speed, 0, 255, 1000, 2000);
      writeMicroseconds(pwmChannel, microseconds);
    }
    request->send(200, "text/plain", "OK");
  });

  server.begin();
}

void loop() {
  // Nothing needed here
}

