# ESP32-with-RasbeeryPI
# 🔧 ESP32-CAM Tool Detection Dashboard

A smart dashboard that uses an ESP32-CAM to **detect tools like pliers, multimeters, wires, etc.**, using a machine learning model trained with **Edge Impulse**. The project shows real-time results in your browser — with tool name, confidence, motion detection, and detection history.

---

## 🌟 What This Project Does

- 📷 Uses the ESP32-CAM to capture images
- 🧠 Runs a trained ML model (Edge Impulse) on the device
- 🛠 Detects tools like:
  - Multimeter
  - Wire Roll
  - Servo Motor
  - Soldering Iron
- 👀 Tracks motion using frame difference
- 📊 Displays everything on a simple web dashboard:
  - Live camera feed
  - Detected tool and confidence
  - Motion status
  - Last 5 detections

---

## 🧰 What You Need

- ESP32-CAM (AI Thinker module)
- FTDI programmer for flashing
- 5V power supply (recommended for stable camera)
- Edge Impulse account (to train and export the model)

---

## 🔌 Wiring (ESP32-CAM to FTDI)

| ESP32-CAM | FTDI       |
|-----------|------------|
| 5V        | VCC        |
| GND       | GND        |
| U0R       | TX         |
| U0T       | RX         |
| GPIO 0    | GND (only while flashing) |

> ⚠️ Use external 5V power if the camera isn’t stable. Disconnect GPIO 0 from GND after flashing.

---

## 🚀 How to Use

1. **Train your model** on Edge Impulse  
   (Image classification project, export as Arduino C++)

2. **Clone this repo**  
   ```bash
  
// main.ino
#include <WiFi.h>
#include <WebServer.h>
#include "esp_camera.h"
#include "edge-impulse-sdk/classifier/ei_run_classifier.h"

// Replace with your network credentials
const char* ssid = "YOUR_SSID";
const char* password = "YOUR_PASSWORD";

WebServer server(80);
String latestLabel = "None";
bool motionDetected = false;
float lastConfidence = 0;
String history[5];
int historyIndex = 0;

// Simple motion detection (frame diffing)
#define MOTION_THRESHOLD 20
static camera_fb_t* lastFrame = NULL;

// Camera config
void startCamera() {
  camera_config_t config;
  config.ledc_channel = LEDC_CHANNEL_0;
  config.ledc_timer = LEDC_TIMER_0;
  config.pin_d0 = 5;
  config.pin_d1 = 18;
  config.pin_d2 = 19;
  config.pin_d3 = 21;
  config.pin_d4 = 36;
  config.pin_d5 = 39;
  config.pin_d6 = 34;
  config.pin_d7 = 35;
  config.pin_xclk = 0;
  config.pin_pclk = 22;
  config.pin_vsync = 25;
  config.pin_href = 23;
  config.pin_sscb_sda = 26;
  config.pin_sscb_scl = 27;
  config.pin_pwdn = 32;
  config.pin_reset = -1;
  config.xclk_freq_hz = 20000000;
  config.pixel_format = PIXFORMAT_JPEG;
  config.frame_size = FRAMESIZE_QVGA;
  config.jpeg_quality = 12;
  config.fb_count = 1;

  // Init camera
  esp_err_t err = esp_camera_init(&config);
  if (err != ESP_OK) {
    Serial.printf("Camera init failed\n");
    return;
  }
}

void handleRoot() {
  camera_fb_t* fb = esp_camera_fb_get();
  if (!fb) return;

  server.sendHeader("Content-Type", "text/html");
  String html = "<!DOCTYPE html><html><head><title>ESP32 Tool Tracker</title></head><body>";
  html += "<h1>Tool Detection Dashboard</h1><img src='/cam.jpg' width='320'/><br>";
  html += "<p>Detected Tool: <strong>" + latestLabel + "</strong><br>Confidence: " + String(lastConfidence) + "%<br>Motion: " + (motionDetected ? "Yes" : "No") + "</p>";
  html += "<h3>History</h3><ul>";
  for (int i = 0; i < 5; i++) html += "<li>" + history[i] + "</li>";
  html += "</ul></body></html>";

  server.send(200, "text/html", html);
  esp_camera_fb_return(fb);
}

void handleJpg() {
  camera_fb_t* fb = esp_camera_fb_get();
  if (!fb) return;
  server.sendHeader("Content-Type", "image/jpeg");
  server.send_P(200, "image/jpeg", (char*)fb->buf, fb->len);
  esp_camera_fb_return(fb);
}

void updateMotion(camera_fb_t* fb) {
  motionDetected = false;
  if (!lastFrame) {
    lastFrame = fb;
    return;
  }

  int changes = 0;
  for (int i = 0; i < fb->len; i++) {
    if (abs(fb->buf[i] - lastFrame->buf[i]) > 30) changes++;
  }
  if (changes > MOTION_THRESHOLD) motionDetected = true;
  lastFrame = fb;
}

void runInference() {
  camera_fb_t* fb = esp_camera_fb_get();
  if (!fb) return;

  ei::signal_t signal;
  signal.total_length = fb->len;
  signal.get_data = [](size_t offset, size_t length, float* out_ptr) {
    for (size_t i = 0; i < length; i++) out_ptr[i] = fb->buf[offset + i] / 255.0f;
    return 0;
  };

  ei_impulse_result_t result;
  EI_IMPULSE_ERROR res = run_classifier(&signal, &result, false);
  if (res != EI_IMPULSE_OK) return;

  for (size_t ix = 0; ix < result.classification->size; ix++) {
    if (result.classification->value[ix] > 0.7f) {
      latestLabel = result.classification->label[ix];
      lastConfidence = result.classification->value[ix] * 100.0f;
      history[historyIndex] = latestLabel + " - " + String(lastConfidence, 1) + "%";
      historyIndex = (historyIndex + 1) % 5;
      break;
    }
  }
  updateMotion(fb);
  esp_camera_fb_return(fb);
}

void setup() {
  Serial.begin(115200);
  startCamera();

  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) delay(500);
  Serial.println(WiFi.localIP());

  server.on("/", handleRoot);
  server.on("/cam.jpg", handleJpg);
  server.begin();
}

void loop() {
  server.handleClient();
  runInference();
}
