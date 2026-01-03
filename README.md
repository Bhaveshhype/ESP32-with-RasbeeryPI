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

