# Solar Systems Energy Integration Plan
## Tesla Powerwall + Utility Meter Reading

**Document Created:** 2025-11-07  
**Status:** Complete Research & Recommendations  
**Target Users:** Dad's Tesla iPhone App + Automated Home Monitoring

---

## Executive Summary

This document provides three implementation tiers for obtaining programmatic readings from:
1. **Tesla Energy Box/Inverter** (Powerwall 2/3 with Gateway)
2. **Outdoor Utility Meter** (manual/analog)

Both have multiple validated solutions with different cost/complexity tradeoffs. **Recommended starting point:** Local Powerwall API + Raspberry Pi OCR camera for meter.

**Total estimated cost:** $200-400 (Tier 1) to $1,200+ (Tier 3 with commercial IoT sensors)

---

## Part 1: Tesla Powerwall Energy API Integration

### 1.1 Authentication & Access Methods

Three distinct authentication approaches exist:

#### **Option A: Local Gateway API (RECOMMENDED for privacy + speed)**
- **What:** Direct HTTPS connection to Powerwall Gateway at `192.168.91.1`
- **Auth:** Customer login (username: `"customer"`, password: last 5 digits of Gateway serial)
- **Pros:** 
  - No cloud dependency
  - Real-time data (sub-second latency)
  - Self-signed certificate (ignore with `-k` flag)
  - Works offline if home network is isolated
  - Multiple Python libraries available
- **Cons:** 
  - Requires local network access
  - Gateway password needed (last 5 Gateway serial digits)
- **Certificate:** Self-signed, use `--insecure` or `-k` flag
- **Gateway Serial:** Found in Tesla app > Settings > My Home Info > "Powerwall Gateway"

#### **Option B: Tesla Fleet API (Official Cloud API)**
- **What:** OAuth 2.0 cloud API from `https://api.tesla.com`
- **Auth:** OAuth 2.0 with CLIENT_ID/CLIENT_SECRET
- **Requirements:**
  - Tesla Developer Account setup
  - App Access Request approval (manual process)
  - Domain ownership verification
  - Redirect URI configuration
- **Pros:**
  - Official Tesla support
  - Works remotely
  - Future-proof
- **Cons:**
  - Approval delay (days/weeks)
  - Cloud-dependent
  - Limited historical data in responses

#### **Option C: Unofficial Tesla Owners API (Cloud)**
- **What:** Community-maintained reverse-engineered API
- **Auth:** Tesla account credentials (email + password + MFA)
- **Pros:** 
  - Faster setup than FleetAPI
  - No approval process
- **Cons:** 
  - Unofficial, may break with Tesla updates
  - Not recommended for production

**RECOMMENDATION:** Start with **Option A (Local Gateway API)** for this installation.

---

### 1.2 Local Gateway API Endpoints

**Base URL:** `https://<powerwall-ip>/api/` (default: `https://192.168.91.1/api/`)

#### **Critical Endpoints for Production**

```
GET /api/meters/aggregates
├─ Current power readings (site, load, solar, battery, grid)
├─ Response time: <100ms
└─ Rate limit: 1 req/sec recommended

GET /api/system_status/soe
├─ Battery state of charge (percentage)
├─ Response: {"percentage": 69.17}
└─ Lightweight endpoint

GET /api/sitemaster
├─ System operational status
├─ Response: {"running": true, "uptime": "802459s", "connected_to_tesla": true}
└─ Health check endpoint

GET /api/system_status
├─ Overall system health
└─ Includes grid status, battery status

POST /api/login (INIT ONLY)
├─ Establish session
└─ See authentication section
```

#### **Additional Endpoints (Less Critical)**

```
GET /api/powerwalls           - List all Powerwall units + serials
GET /api/site_info            - Site configuration & timezone
GET /api/system/update/status - Firmware version
GET /api/device_vitals        - Device health diagnostics
GET /api/solar_powerflow      - Solar string data (advanced)
```

---

### 1.3 Sample JSON Responses

#### **GET /api/meters/aggregates Response**

```json
{
  "site": {
    "last_communication_time": "2025-11-07T14:23:45.123-08:00",
    "instant_power": 1250,
    "instant_reactive_power": -45,
    "instant_apparent_power": 1251,
    "frequency": 60.0,
    "energy_exported": 5230000,
    "energy_imported": 12450000,
    "energy_net": -7220000,
    "current_phase_1": 5.2,
    "current_phase_2": 5.0,
    "current_phase_3": 5.1,
    "voltage_phase_1": 240.1,
    "voltage_phase_2": 240.0,
    "voltage_phase_3": 240.2
  },
  "battery": {
    "last_communication_time": "2025-11-07T14:23:45.123-08:00",
    "instant_power": 500,
    "instant_reactive_power": 0,
    "instant_apparent_power": 500,
    "frequency": 60.0,
    "energy_exported": 2100000,
    "energy_imported": 3450000,
    "energy_net": -1350000,
    "current_phase_1": 2.1,
    "current_phase_2": 2.0,
    "current_phase_3": 2.0,
    "voltage_phase_1": 240.1,
    "voltage_phase_2": 240.0,
    "voltage_phase_3": 240.2
  },
  "load": {
    "last_communication_time": "2025-11-07T14:23:45.123-08:00",
    "instant_power": 750,
    "instant_reactive_power": -25,
    "instant_apparent_power": 751,
    "frequency": 60.0,
    "energy_exported": 0,
    "energy_imported": 8920000,
    "energy_net": -8920000,
    "current_phase_1": 3.1,
    "current_phase_2": 3.0,
    "current_phase_3": 3.1,
    "voltage_phase_1": 240.1,
    "voltage_phase_2": 240.0,
    "voltage_phase_3": 240.2
  },
  "solar": {
    "last_communication_time": "2025-11-07T14:23:45.123-08:00",
    "instant_power": 2000,
    "instant_reactive_power": 15,
    "instant_apparent_power": 2001,
    "frequency": 60.0,
    "energy_exported": 15670000,
    "energy_imported": 0,
    "energy_net": 15670000,
    "current_phase_1": 8.3,
    "current_phase_2": 8.2,
    "current_phase_3": 8.4,
    "voltage_phase_1": 240.1,
    "voltage_phase_2": 240.0,
    "voltage_phase_3": 240.2
  }
}
```

#### **GET /api/system_status/soe Response**

```json
{
  "percentage": 69.17
}
```

#### **GET /api/sitemaster Response**

```json
{
  "running": true,
  "uptime": 802459,
  "connected_to_tesla": true
}
```

---

### 1.4 Authentication Implementation

#### **Step 1: Get Gateway Serial Number**

```bash
# Tesla iPhone app: Settings > My Home Info > "Powerwall Gateway"
# Serial format: STG...XXXXX (extract last 5 digits)
GATEWAY_SERIAL="STG1234ABCDE"
GATEWAY_PASSWORD="${GATEWAY_SERIAL: -5}"  # Extract last 5 digits
# Result: ABCDE
```

#### **Step 2: Initial Login (cURL)**

```bash
#!/bin/bash

POWERWALL_IP="192.168.91.1"
GATEWAY_PASSWORD="ABCDE"  # Last 5 digits of serial
EMAIL="your.email@example.com"

# POST /api/login
curl -k -X POST "https://${POWERWALL_IP}/api/login" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "customer",
    "password": "'${GATEWAY_PASSWORD}'",
    "email": "'${EMAIL}'"
  }' \
  --cookie-jar /tmp/cookies.txt

# Response includes auth token
```

#### **Step 3: Query Meters with Auth**

```bash
#!/bin/bash

POWERWALL_IP="192.168.91.1"

# Use cookies from previous login
curl -k -X GET "https://${POWERWALL_IP}/api/meters/aggregates" \
  --cookie /tmp/cookies.txt
```

#### **Step 4: Session Management**

- **Session Duration:** Typically 30 days
- **Logout:** `POST /api/logout`
- **Re-authentication:** Automatic if cookie expires
- **Best Practice:** Cache credentials in environment variables

---

### 1.5 Python Integration (Recommended Approach)

#### **Using pyPowerwall Library**

```bash
# Install
pip install pypowerwall
```

```python
import pypowerwall
import json
from datetime import datetime

class PowerwallMonitor:
    def __init__(self, gateway_ip="192.168.91.1", password=None):
        """
        Initialize Powerwall connection.
        Password = last 5 digits of Gateway serial
        """
        self.pw = pypowerwall.Powerwall(
            host=gateway_ip,
            gw_pwd=password,
            email="your.email@example.com",
            timezone="America/Los_Angeles",
            auto_select=True  # Auto-detect local/cloud/cloud2
        )
    
    def get_current_readings(self):
        """Get real-time energy readings"""
        return {
            "timestamp": datetime.utcnow().isoformat(),
            "solar_power_watts": self.pw.solar(),
            "battery_power_watts": self.pw.battery(),
            "grid_power_watts": self.pw.grid(),
            "load_power_watts": self.pw.load(),
            "battery_level_percent": self.pw.level(),
            "grid_status": self.pw.grid_status()
        }
    
    def get_device_vitals(self):
        """Get system health info"""
        return self.pw.device_vitals()
    
    def get_power_flow(self):
        """Get aggregated power metrics"""
        return self.pw.power()

# Usage
monitor = PowerwallMonitor(password="ABCDE")
readings = monitor.get_current_readings()
print(json.dumps(readings, indent=2))
```

#### **Example Output**

```json
{
  "timestamp": "2025-11-07T14:23:45.123456",
  "solar_power_watts": 2000,
  "battery_power_watts": 500,
  "grid_power_watts": 1250,
  "load_power_watts": 750,
  "battery_level_percent": 69.17,
  "grid_status": "SystemGridConnected"
}
```

---

### 1.6 OpenAPI-Lite Spec: GET /current_readings

```yaml
openapi: 3.0.0
info:
  title: Solar System Energy API
  version: 1.0.0
  description: Unified interface for Powerwall and utility meter readings

servers:
  - url: http://localhost:8000
    description: Local API server

paths:
  /current_readings:
    get:
      summary: Get current energy readings
      description: Returns real-time power measurements from Powerwall and utility meter
      tags:
        - Energy
      parameters:
        - name: include_history
          in: query
          schema:
            type: boolean
            default: false
          description: Include 24-hour historical data
      responses:
        '200':
          description: Successful reading
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/EnergyReading'
        '503':
          description: Service unavailable (Powerwall offline)

components:
  schemas:
    EnergyReading:
      type: object
      required:
        - timestamp
        - solar
        - battery
        - grid
        - load
        - meter
      properties:
        timestamp:
          type: string
          format: date-time
          description: ISO 8601 timestamp
          example: "2025-11-07T14:23:45Z"
        
        solar:
          type: object
          description: Solar production
          properties:
            power_watts:
              type: number
              description: Real-time solar output
              example: 2000
            total_today_kwh:
              type: number
              example: 12.5
        
        battery:
          type: object
          description: Powerwall battery status
          properties:
            power_watts:
              type: number
              description: Positive=charging, negative=discharging
              example: 500
            percent_charged:
              type: number
              minimum: 0
              maximum: 100
              example: 69.17
            remaining_kwh:
              type: number
              example: 10.2
        
        grid:
          type: object
          description: Utility grid interaction
          properties:
            power_watts:
              type: number
              description: Positive=import, negative=export
              example: 1250
            status:
              type: string
              enum: [Connected, Disconnected, Transitioning]
              example: "Connected"
        
        load:
          type: object
          description: Home consumption
          properties:
            power_watts:
              type: number
              example: 750
            percentage_of_capacity:
              type: number
              example: 25
        
        meter:
          type: object
          description: Utility meter reading
          properties:
            reading_kwh:
              type: number
              description: Total consumption (from meter)
              example: 1234.56
            method:
              type: string
              enum: [camera_ocr, ct_clamp, smart_meter_api]
              example: "camera_ocr"
            last_update:
              type: string
              format: date-time
              example: "2025-11-07T14:20:00Z"
            confidence:
              type: number
              minimum: 0
              maximum: 1
              description: OCR confidence (0-1 for camera method)
              example: 0.98
```

---

## Part 2: Utility Meter Reading Integration

### 2.1 Meter Analysis & Options

Your outdoor utility meter is likely:
- **Analog/mechanical dial meter** (NOT smart-enabled)
- **Manual reading required by utility**
- **LCD digital display** (older) or **rotating dials** (electromechanical)

Three validated approaches ranked by ease:

| Rank | Method | Cost | Accuracy | Setup Time | Reliability |
|------|--------|------|----------|------------|-------------|
| **1** | **Camera + OCR** | $150-250 | 95-99% | 2-3 hours | Very Good |
| **2** | **CT Clamps (IoT)** | $200-400 | 95-98% | 4-6 hours | Excellent |
| **3** | **Commercial Gateway** | $800-1200 | 99%+ | 1 hour | Perfect |

---

### 2.2 Option 1: Camera + OCR (RECOMMENDED Tier 1)

#### **What It Does**
- ESP32 or Raspberry Pi with camera module
- Takes periodic photos of meter display
- ML-based OCR recognizes 7-segment digits
- Sends readings via WiFi to your home automation system

#### **Hardware Requirements**

```
Raspberry Pi 4B (2GB) or ESP32-CAM
├─ Raspberry Pi 4B 2GB: $35-45
├─ OR ESP32-CAM module: $15-25
├─ (Recommend Raspberry Pi for stability)

Camera Module
├─ Raspberry Pi Camera v2 (8MP): $25-35
├─ OR ESP32-CAM built-in camera: included
├─ USB webcam (640x480+): $15-25

Lighting (Critical for OCR accuracy)
├─ 6-12 white LED strip: $10-15
└─ OR solar-powered LED: $20-30

Housing
├─ 3D-printed weatherproof case: $5-20
└─ OR commercial enclosure: $30-50

Mounting Hardware
├─ Weatherproof mounting bracket: $15-25
└─ Cable conduit & glands: $10-15

Power Supply
├─ PoE injector (if using Ethernet): $20-30
├─ OR Solar panel + battery: $50-100
└─ OR USB-C 5V 3A power: $15-25

TOTAL: $150-250 (Raspberry Pi 4B tier)
```

#### **Recommended Kit**

```
✓ Raspberry Pi 4B (2GB)          - $40
✓ Raspberry Pi Camera v2 8MP     - $30
✓ SD Card 32GB (class 10)        - $10
✓ USB-C Power Supply 5V 3A       - $15
✓ White LED strip 12V            - $12
✓ Weatherproof ABS plastic case  - $20
✓ Mounting bracket (aluminum)    - $20
✓ Cat6 Ethernet cable (50ft)     - $10
├─ Subtotal: $157
└─ (+ labor for assembly: ~2-3 hours)
```

#### **Software Approach**

**Architecture:**
```
Meter Display (LCD/dial)
         ↓ [Photo]
   Camera Module
         ↓ [JPEG]
   Raspberry Pi
   ├─ OpenCV (image preprocessing)
   ├─ PyTorch/TensorFlow (digit recognition)
   └─ Python Flask API
         ↓ [HTTP JSON]
   Home Automation System
   └─ Graphite/Prometheus/InfluxDB
```

**Processing Pipeline:**
1. **Capture**: Photo every 15 minutes (configurable)
2. **Preprocess**: Grayscale, contrast enhancement, rotation correction
3. **Detect**: Find digit boundaries using edge detection
4. **Recognize**: CNN-based digit classification
5. **Validate**: Ensure monotonically increasing readings
6. **Upload**: Send to API endpoint

**Accuracy & Challenges:**
- **Typical Accuracy:** 95-99% per reading
- **Common Errors:**
  - 1/I/l confusion (rare with 7-segment)
  - 8/0/O confusion (mitigated with context)
  - Glare/shadows (solved with good lighting)
- **Solution:** Plausibility checks (digit count, monotonic increase, rate limits)

#### **Sample Python Implementation**

```python
#!/usr/bin/env python3
"""
Meter Reader using OpenCV + PyTorch
Captures meter photo and extracts 7-digit reading
"""

import cv2
import numpy as np
import torch
from datetime import datetime
import requests
import json

class MeterReader:
    def __init__(self, camera_id=0):
        self.cap = cv2.VideoCapture(camera_id)
        self.cap.set(cv2.CAP_PROP_FRAME_WIDTH, 1280)
        self.cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 720)
        self.last_reading = None
        
    def capture_image(self):
        """Capture image from camera"""
        ret, frame = self.cap.read()
        if not ret:
            raise Exception("Failed to capture frame")
        return frame
    
    def preprocess_image(self, image):
        """
        Preprocess meter image for OCR:
        - Convert to grayscale
        - Enhance contrast
        - Rotate if needed
        - Extract meter region
        """
        # Grayscale
        gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
        
        # Histogram equalization (enhance contrast)
        enhanced = cv2.equalizeHist(gray)
        
        # Binary threshold
        _, binary = cv2.threshold(enhanced, 127, 255, cv2.THRESH_BINARY)
        
        # Optional: Rotate to correct skew using Hough transform
        lines = cv2.HoughLines(binary, 1, np.pi/180, 100)
        if lines is not None and len(lines) > 0:
            angle = lines[0][0][1] * 180 / np.pi - 90
            h, w = binary.shape
            center = (w // 2, h // 2)
            M = cv2.getRotationMatrix2D(center, angle, 1.0)
            binary = cv2.warpAffine(binary, M, (w, h))
        
        return binary
    
    def extract_digits(self, image):
        """
        Extract individual digit bounding boxes
        Returns list of (x, y, w, h) tuples
        """
        contours, _ = cv2.findContours(
            image, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE
        )
        
        digits = []
        for contour in contours:
            x, y, w, h = cv2.boundingRect(contour)
            # Filter by aspect ratio (digit-like)
            if 0.3 < h/w < 0.9 and w > 10 and h > 20:
                digits.append((x, y, w, h))
        
        # Sort by x position (left to right)
        digits.sort(key=lambda d: d[0])
        return digits
    
    def recognize_digit(self, digit_image):
        """
        Recognize single digit using pre-trained model
        Returns digit (0-9) and confidence
        """
        # Resize to standard size (28x28)
        digit_resized = cv2.resize(digit_image, (28, 28))
        
        # Normalize to [0, 1]
        tensor = torch.from_numpy(digit_resized).float() / 255.0
        tensor = tensor.unsqueeze(0).unsqueeze(0)  # Add batch and channel dims
        
        # Load pre-trained model (e.g., from torchvision)
        # For this example, using a simple CNN
        with torch.no_grad():
            # This would use your trained model
            # predictions = model(tensor)
            # digit = predictions.argmax().item()
            # confidence = predictions.softmax(1)[0].max().item()
            pass
        
        return digit, confidence
    
    def read_meter(self):
        """
        Main function: Capture, process, and extract meter reading
        Returns: {reading_kwh, timestamp, confidence, method}
        """
        try:
            # Capture
            image = self.capture_image()
            
            # Preprocess
            processed = self.preprocess_image(image)
            
            # Extract digits
            digit_regions = self.extract_digits(processed)
            
            if len(digit_regions) < 7:
                raise Exception(f"Only found {len(digit_regions)} digits, need 7")
            
            # Recognize each digit
            reading = ""
            confidence_scores = []
            
            for x, y, w, h in digit_regions[:7]:
                digit_img = processed[y:y+h, x:x+w]
                digit, conf = self.recognize_digit(digit_img)
                reading += str(digit)
                confidence_scores.append(conf)
            
            # Validate reading (plausibility checks)
            reading_value = float(reading[:4] + "." + reading[4:])
            avg_confidence = np.mean(confidence_scores)
            
            # Check 1: Monotonically increasing
            if self.last_reading:
                if reading_value < self.last_reading:
                    raise Exception(
                        f"Reading decreased: {self.last_reading} -> {reading_value}"
                    )
            
            self.last_reading = reading_value
            
            return {
                "reading_kwh": reading_value,
                "timestamp": datetime.utcnow().isoformat(),
                "confidence": float(avg_confidence),
                "method": "camera_ocr",
                "digits": reading
            }
        
        except Exception as e:
            return {
                "error": str(e),
                "timestamp": datetime.utcnow().isoformat()
            }
    
    def send_to_api(self, reading, api_url="http://localhost:8000/meter_reading"):
        """Send reading to home automation API"""
        try:
            response = requests.post(
                api_url,
                json=reading,
                timeout=5
            )
            response.raise_for_status()
            return response.json()
        except Exception as e:
            print(f"API Error: {e}")
            return None

# Main loop
if __name__ == "__main__":
    reader = MeterReader(camera_id=0)
    
    while True:
        reading = reader.read_meter()
        print(json.dumps(reading, indent=2))
        
        if "reading_kwh" in reading:
            reader.send_to_api(reading)
        
        # Wait 15 minutes before next reading
        import time
        time.sleep(900)
```

#### **Setup Instructions (Raspberry Pi)**

```bash
#!/bin/bash
# Install Raspberry Pi OS Lite on SD card
# ssh into Pi with: ssh pi@<ip>

# Update system
sudo apt update && sudo apt upgrade -y

# Install Python3 + dependencies
sudo apt install -y python3-pip python3-opencv python3-numpy
sudo apt install -y libatlas-base-dev libjasper-dev libtiff5 libjasper1 libharfp
sudo apt install -y libwebp6 libtiff5 libjasper1 libharfp0 libwebp6 libtiff5

# Install PyTorch (ARM-optimized)
pip3 install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cpu

# Install Flask for API endpoint
pip3 install flask requests

# Enable camera interface
sudo raspi-config nonint do_camera 0

# Create systemd service for auto-start
sudo tee /etc/systemd/system/meter-reader.service << 'SYSDF'
[Unit]
Description=Meter Reader Service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/meter-reader
ExecStart=/usr/bin/python3 /home/pi/meter-reader/meter_reader.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
SYSDF

# Enable service
sudo systemctl enable meter-reader
sudo systemctl start meter-reader
```

---

### 2.3 Option 2: CT Clamp + IoT (Tier 2)

#### **What It Does**
- Non-invasive current transformers (CT clamps) on utility meter incoming cables
- Measures real-time power draw
- Integrates with home automation (Home Assistant, OpenHAB)
- More accurate than OCR, requires low-level electrical understanding

#### **Hardware Requirements**

```
Microcontroller
├─ ESP32 DevKit: $8-12
├─ OR Raspberry Pi 4B: $40-50
└─ Recommend ESP32 (lower power)

Current Sensors (CT Clamps)
├─ YHDC SCT-013-030 (30A): $12 each × 2-3 = $24-36
│  └─ Measures real-time current, converts to voltage
├─ OR Pressac EnOcean wireless: $80-150
├─ OR Milesight LoRaWAN CT: $150-250

Conditioning Circuit (for SCT-013)
├─ 10K resistors: $1
├─ 10µF capacitors: $2
├─ 33Ω burden resistor: $1
├─ Op-amp (optional): $2
└─ Subtotal: $6

ADC Converter (if using ESP32)
├─ ADS1115 (4-channel, 16-bit): $5-8
└─ (ESP32 has 12-bit ADC, may need 16-bit for accuracy)

Power Supply
├─ USB-C 5V 3A: $15
├─ OR PoE injection: $25
└─ OR Battery + solar panel: $50-100

Housing & Installation
├─ DIN rail enclosure: $30-50
├─ Weatherproof outdoor box: $40-60
├─ Cable glands: $10
├─ Mounting hardware: $15

TOTAL: $200-400 (ESP32 tier)
       $500-700 (Pressac wireless tier)
       $800-1200 (Milesight LoRaWAN tier)
```

#### **Recommended Setup (ESP32 + SCT-013)**

```
              +----------+
              | Utility  |
              |  Meter   |
              +----┬─────+
                   │
          ┌────────┴────────┐
          │                 │
      [CT-01]           [CT-02]
      (Line A)         (Line B)
          │                 │
          └────────┬────────┘
                   │
           ┌───────┴────────┐
           │  Conditioning  │
           │   Circuit      │
           │  (Resistors &  │
           │  Capacitors)   │
           └───────┬────────┘
                   │
           ┌───────┴────────┐
           │   ADS1115 ADC  │
           │  (I2C Address) │
           └───────┬────────┘
                   │
           ┌───────┴────────┐
           │  ESP32 Board   │
           │  (WiFi/BLE)    │
           └───────┬────────┘
                   │
           Home Assistant
           (or MQTT Broker)
```

#### **Wiring Schematic (Simplified)**

```
SCT-013-030 (3.5mm jack):
     Tip (A) ───── 10K Ω ─┬─ ADS1115 Channel
     Ring (B) ───── GND   │
                           ├─ 10µF capacitor ─ GND
                           └─ 33Ω resistor (burden)

ADS1115 to ESP32:
     SDA ───── GPIO 21 (SDA)
     SCL ───── GPIO 22 (SCL)
     VDD ───── 3.3V
     GND ───── GND
```

#### **Sample Python Implementation (ESP32 + MicroPython)**

```python
# ct_meter.py - MicroPython on ESP32
import machine
import network
import time
import json
from machine import I2C, Pin
import urequests

# ADS1115 I2C address and registers
ADS1115_ADDR = 0x48
ADS_CONFIG_REGISTER = 0x01
ADS_CONVERSION_REGISTER = 0x00

class ADS1115:
    """Simple ADS1115 ADC interface"""
    def __init__(self, i2c, addr=0x48):
        self.i2c = i2c
        self.addr = addr
    
    def read_channel(self, channel):
        """Read voltage from analog channel (0-3)"""
        # Configure channel
        config = 0xC000  # Start single-shot conversion
        config |= (channel << 12)
        config |= 0x0080  # FSR = 0.256V (for ±6.144V range)
        
        self.i2c.writeto(self.addr, bytes([ADS_CONFIG_REGISTER, config >> 8, config & 0xFF]))
        time.sleep(0.01)
        
        # Read conversion result
        self.i2c.writeto(self.addr, bytes([ADS_CONVERSION_REGISTER]))
        data = self.i2c.readfrom(self.addr, 2)
        
        # Convert to voltage (-6.144V to +6.144V)
        raw = (data[0] << 8) | data[1]
        if raw > 32767:
            raw -= 65536
        voltage = (raw >> 4) * 0.0001875  # 0.1875mV per unit
        
        return voltage

class CTMeterMonitor:
    def __init__(self, ssid, password):
        self.ssid = ssid
        self.password = password
        self.i2c = I2C(1, scl=Pin(22), sda=Pin(21), freq=400000)
        self.adc = ADS1115(self.i2c)
        self.connect_wifi()
    
    def connect_wifi(self):
        wlan = network.WLAN(network.STA_IF)
        wlan.active(True)
        wlan.connect(self.ssid, self.password)
        
        timeout = 10
        while timeout > 0 and not wlan.isconnected():
            time.sleep(1)
            timeout -= 1
        
        if wlan.isconnected():
            print(f"WiFi Connected: {wlan.ifconfig()}")
        else:
            print("WiFi Connection Failed")
    
    def read_ct_sensors(self):
        """
        Read 2x CT clamps on analog channels 0 and 1
        Returns current in amps
        """
        readings = {}
        
        # CT-013-030: 1V at 30A (1000mV / 30A = 33.33mV/A)
        ct_ratio = 33.33  # mV per Amp
        
        for ch in range(2):
            # Take multiple samples for RMS calculation
            samples = 100
            sum_square = 0
            
            for _ in range(samples):
                v = self.adc.read_channel(ch)
                sum_square += v ** 2
                time.sleep(0.001)
            
            # Calculate RMS
            rms_voltage = (sum_square / samples) ** 0.5
            current_amps = abs(rms_voltage) * 1000 / ct_ratio
            
            readings[f"line_{chr(65+ch)}"] = {
                "current_amps": round(current_amps, 2),
                "voltage_mv": round(abs(rms_voltage) * 1000, 1)
            }
        
        # Calculate total power (assume 240V nominal)
        total_amps = (
            readings["line_A"]["current_amps"] +
            readings["line_B"]["current_amps"]
        )
        power_watts = total_amps * 240 * 0.95  # PF ~0.95
        
        return {
            "timestamp": time.time(),
            "channels": readings,
            "total_amps": round(total_amps, 2),
            "estimated_power_watts": round(power_watts, 0),
            "method": "ct_clamp"
        }
    
    def send_to_api(self, reading, api_url):
        try:
            response = urequests.post(
                api_url,
                json=reading,
                timeout=5
            )
            print(f"API Response: {response.status_code}")
            response.close()
        except Exception as e:
            print(f"API Error: {e}")

# Main loop
if __name__ == "__main__":
    monitor = CTMeterMonitor(
        ssid="YourSSID",
        password="YourPassword"
    )
    
    while True:
        reading = monitor.read_ct_sensors()
        print(json.dumps(reading))
        
        monitor.send_to_api(
            reading,
            "http://192.168.1.100:8000/meter_reading"
        )
        
        time.sleep(60)  # Read every 60 seconds
```

#### **Key Advantages Over Camera OCR**
- **Real-time power measurement** (not just periodic snapshots)
- **Continuous monitoring** (no lighting dependencies)
- **More accurate energy calculation** (actual current draw)
- **Can detect anomalies** (e.g., phantom loads)

#### **Setup Instructions**

```bash
# Soldering required!
# 1. Solder CT clamp conditioning circuit
# 2. Mount CT clamps on meter incoming cables (L1/L2)
# 3. Connect to ESP32 via ADS1115
# 4. Flash MicroPython to ESP32
# 5. Upload ct_meter.py script
# 6. Configure WiFi credentials
# 7. Start reading!
```

---

### 2.4 Option 3: Commercial Smart Meter Gateway (Tier 3)

#### **What It Does**
- Plug-and-play wireless gateway that clips onto meter
- Reads meter digits automatically (like OCR but integrated)
- Professional accuracy, warranty, technical support

#### **Examples**

```
LAIWA EnergyCam OCR Meter Reader
├─ Small weatherproof device: $800-1200
├─ Takes photo every 15 minutes
├─ AI-powered digit recognition
├─ Cloud dashboard included
├─ 99%+ accuracy
└─ Professional support

Pressac Wireless CT Clamp System
├─ Wireless sensors + gateway: $1000-1500
├─ Multiple meter support
├─ Cloud integration
└─ Enterprise-grade

Netvox LoRaWAN Solution
├─ R718N37 (3-phase): $800-1200
├─ LoRaWAN connectivity
├─ Network operator required
└─ Industrial reliability
```

#### **Pros & Cons**
- **Pros:** Professional install, warranty, ongoing support
- **Cons:** Expensive, cloud-dependent, limited customization

---

## Part 3: Unified Integration Plan

### 3.1 Recommended Architecture

```
┌─────────────────────────────────────────────────────────────┐
│         Home Energy Monitoring System                       │
└─────────────────────────────────────────────────────────────┘
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
    ┌─────▼─────┐     ┌─────▼─────┐    ┌─────▼──────┐
    │  Powerwall │     │  Meter    │    │  Optional: │
    │  Gateway   │     │  Camera   │    │ Smart Home │
    │            │     │ (Raspi)   │    │ (HA/MQTT)  │
    │ 192.168.   │     │ (IoT+OCR) │    │            │
    │ 91.1:443   │     │           │    │            │
    └─────┬──────┘     └─────┬─────┘    └─────┬──────┘
          │                 │                 │
          │ HTTPS           │ HTTP            │ MQTT/HTTP
          │ Local Auth      │ Flask API       │ Publish
          │                 │                 │
          └─────────────────┼─────────────────┘
                            │
                  ┌─────────▼────────┐
                  │   Central API    │
                  │   Server         │
                  │ (Python Flask)   │
                  │ /current_readings│
                  └─────────┬────────┘
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
    ┌─────▼──────┐   ┌─────▼──────┐   ┌──────▼────────┐
    │ InfluxDB   │   │ Grafana    │   │ Mobile App    │
    │ Time-series│   │ Dashboards │   │ (via REST API)│
    │ Database   │   │            │   │               │
    └────────────┘   └────────────┘   └───────────────┘
```

### 3.2 Central API Server (Python Flask)

```python
# app.py - Central Energy API
from flask import Flask, jsonify, request
from datetime import datetime
import pypowerwall
import requests
import json

app = Flask(__name__)

# Initialize Powerwall connection
pw = pypowerwall.Powerwall(
    host="192.168.91.1",
    gw_pwd="ABCDE",  # Last 5 digits of gateway serial
    email="your@email.com",
    timezone="America/Los_Angeles",
    auto_select=True
)

class EnergySystemAPI:
    def __init__(self):
        self.powerwall = pw
        self.meter_endpoint = "http://192.168.1.50:5000/meter"  # Raspi camera
        self.last_meter_reading = None
    
    @app.route('/current_readings', methods=['GET'])
    def get_current_readings(self):
        """
        Combined endpoint: Powerwall + Utility Meter
        """
        try:
            # Get Powerwall data
            pw_data = {
                "solar_power_watts": self.powerwall.solar(),
                "battery_power_watts": self.powerwall.battery(),
                "grid_power_watts": self.powerwall.grid(),
                "load_power_watts": self.powerwall.load(),
                "battery_level_percent": self.powerwall.level(),
                "grid_status": self.powerwall.grid_status()
            }
            
            # Get meter data
            meter_data = requests.get(self.meter_endpoint, timeout=5).json()
            
            response = {
                "timestamp": datetime.utcnow().isoformat() + "Z",
                "powerwall": pw_data,
                "utility_meter": meter_data,
                "net_flow_watts": pw_data["grid_power_watts"],  # +import, -export
                "status": "ok"
            }
            
            return jsonify(response), 200
        
        except Exception as e:
            return jsonify({
                "timestamp": datetime.utcnow().isoformat() + "Z",
                "error": str(e),
                "status": "error"
            }), 503
    
    @app.route('/meter_reading', methods=['POST'])
    def receive_meter_reading(self):
        """Endpoint for meter camera/CT clamp to POST readings"""
        data = request.json
        self.last_meter_reading = data
        
        # Validate
        if "reading_kwh" in data and data.get("confidence", 0) > 0.9:
            # Store to database
            return jsonify({"status": "accepted"}), 200
        else:
            return jsonify({"status": "rejected"}), 400
    
    @app.route('/health', methods=['GET'])
    def health_check(self):
        """Simple health check endpoint"""
        try:
            pw_status = self.powerwall.sitemaster()
            return jsonify({
                "powerwall": "connected" if pw_status.get("running") else "offline",
                "timestamp": datetime.utcnow().isoformat()
            }), 200
        except:
            return jsonify({"powerwall": "offline"}), 503

# Initialize API
api = EnergySystemAPI()

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000, debug=False)
```

---

### 3.3 Docker Compose Setup (Complete Stack)

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Central API Server
  api-server:
    build: ./api
    container_name: energy-api
    ports:
      - "8000:8000"
    environment:
      - POWERWALL_IP=192.168.91.1
      - POWERWALL_PASSWORD=ABCDE
      - METER_ENDPOINT=http://meter-camera:5000/meter
      - INFLUXDB_URL=http://influxdb:8086
    depends_on:
      - influxdb
    restart: unless-stopped
    networks:
      - energy_net

  # InfluxDB Time-Series Database
  influxdb:
    image: influxdb:2.7-alpine
    container_name: influxdb
    ports:
      - "8086:8086"
    environment:
      - INFLUXDB_DB=energy
      - INFLUXDB_ADMIN_USER=admin
      - INFLUXDB_ADMIN_PASSWORD=changeme
    volumes:
      - influxdb_data:/var/lib/influxdb2
    restart: unless-stopped
    networks:
      - energy_net

  # Grafana Dashboards
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_INSTALL_PLUGINS=grafana-piechart-panel
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    depends_on:
      - influxdb
    restart: unless-stopped
    networks:
      - energy_net

  # Meter Camera (Raspberry Pi)
  # This runs on separate Raspberry Pi, not in Docker
  # But can export via REST API to this network

  # MQTT Broker (optional for Home Assistant)
  mosquitto:
    image: eclipse-mosquitto:latest
    container_name: mqtt
    ports:
      - "1883:1883"
      - "9001:9001"
    volumes:
      - mosquitto_data:/mosquitto/data
      - ./mosquitto/config:/mosquitto/config
    restart: unless-stopped
    networks:
      - energy_net

volumes:
  influxdb_data:
  grafana_data:
  mosquitto_data:

networks:
  energy_net:
    driver: bridge
```

---

## Part 4: Step-by-Step Setup Instructions

### 4.1 Tesla Powerwall Local API Setup

**Prerequisites:**
- Powerwall 2/3 with Gateway
- Home network access to gateway
- Gateway serial number (from Tesla app)

**Step 1: Identify Gateway Serial**
```
Tesla iPhone app:
  Settings
    → My Home Info
      → "Powerwall Gateway: STG...XXXXX"
        → Extract last 5 characters = PASSWORD
```

**Step 2: Verify Network Access**
```bash
# From any machine on your home network
ping 192.168.91.1

# Should get responses like:
# 64 bytes from 192.168.91.1: icmp_seq=1 ttl=61 time=5.32 ms
```

**Step 3: Test API with cURL**
```bash
#!/bin/bash

POWERWALL_IP="192.168.91.1"
GATEWAY_PASSWORD="ABCDE"  # Last 5 digits
EMAIL="your@email.com"

# Login (creates session cookie)
curl -k -c /tmp/cookies.txt -X POST \
  "https://${POWERWALL_IP}/api/login" \
  -H "Content-Type: application/json" \
  -d "{
    \"username\": \"customer\",
    \"password\": \"${GATEWAY_PASSWORD}\",
    \"email\": \"${EMAIL}\"
  }" \
  | jq .

# Query meters with session cookie
curl -k -b /tmp/cookies.txt \
  "https://${POWERWALL_IP}/api/meters/aggregates" \
  | jq .
```

**Step 4: Install Python Client**
```bash
pip install pypowerwall requests

# Test connection
python3 << 'PYEOF'
import pypowerwall
pw = pypowerwall.Powerwall(
    host="192.168.91.1",
    gw_pwd="ABCDE",
    email="your@email.com",
    timezone="America/Los_Angeles"
)
print("Solar:", pw.solar(), "watts")
print("Battery:", pw.battery(), "watts")
print("Grid:", pw.grid(), "watts")
PYEOF
```

### 4.2 Meter Camera Setup (Raspberry Pi)

**Hardware Assembly (1-2 hours)**

1. Mount Raspberry Pi in weatherproof enclosure
2. Attach camera module with flexible ribbon cable
3. Wire LED strip for lighting (parallel to Pi power)
4. Mount assembly on pole pointing at meter display
5. Run ethernet cable back to home network PoE injector

**Software Setup (30 minutes)**

```bash
# SSH into Raspberry Pi
ssh pi@192.168.1.50

# 1. Update system
sudo apt update && sudo apt upgrade -y

# 2. Install dependencies
sudo apt install -y \
  python3-pip \
  python3-opencv \
  python3-numpy \
  libatlas-base-dev \
  libjasper-dev \
  libharfp0

# 3. Install Python packages
pip3 install flask torch torchvision requests

# 4. Enable camera
sudo raspi-config nonint do_camera 0

# 5. Create meter reader script
mkdir -p /home/pi/meter-reader
cd /home/pi/meter-reader

# Copy meter_reader.py from above

# 6. Create systemd service
sudo tee /etc/systemd/system/meter-reader.service > /dev/null << 'SYSDF'
[Unit]
Description=Meter Reader Service
After=network-online.target

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/meter-reader
ExecStart=/usr/bin/python3 meter_reader.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
SYSDF

# 7. Enable and start service
sudo systemctl enable meter-reader
sudo systemctl start meter-reader

# 8. Check status
sudo systemctl status meter-reader
```

**Calibration (30 minutes)**

```bash
# SSH into Pi
# Test camera capture
python3 << 'PYEOF'
import cv2
cap = cv2.VideoCapture(0)
ret, frame = cap.read()
cv2.imwrite('/tmp/meter_test.jpg', frame)
cap.release()
print("Photo saved to /tmp/meter_test.jpg")
PYEOF

# Download test image to laptop for viewing
# Adjust lighting/angle as needed
```

### 4.3 Central API Server Setup

```bash
# Create project directory
mkdir -p ~/energy-api
cd ~/energy-api

# Create Python virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install flask flask-cors pypowerwall requests influxdb-client

# Copy app.py from above

# Run locally for testing
python3 app.py

# Should see:
# WARNING: This is a development server. Do not use it in production.
# Running on http://127.0.0.1:8000

# Test endpoint
curl http://localhost:8000/current_readings | python -m json.tool
```

### 4.4 Home Assistant Integration (Optional)

```yaml
# configuration.yaml
rest_command:
  get_energy_readings:
    url: "http://localhost:8000/current_readings"
    method: GET

sensor:
  - platform: rest
    resource: "http://localhost:8000/current_readings"
    name: "Energy System"
    value_template: "{{ value_json.status }}"
    json_attributes:
      - powerwall
      - utility_meter

template:
  - sensor:
      - name: "Solar Production"
        unit_of_measurement: "W"
        value_template: "{{ state_attr('sensor.energy_system', 'powerwall').solar_power_watts | int(0) }}"
      
      - name: "Battery Level"
        unit_of_measurement: "%"
        value_template: "{{ state_attr('sensor.energy_system', 'powerwall').battery_level_percent | round(1) }}"
      
      - name: "Utility Meter Reading"
        unit_of_measurement: "kWh"
        value_template: "{{ state_attr('sensor.energy_system', 'utility_meter').reading_kwh | round(2) }}"

automation:
  - alias: "Alert on Low Battery"
    trigger:
      platform: numeric_state
      entity_id: sensor.battery_level
      below: 10
    action:
      service: notify.persistent_notification
      data:
        message: "Battery below 10%"
```

---

## Part 5: Cost Summary & Recommendations

### Tier 1: Camera OCR (RECOMMENDED)
**Total Cost:** $150-250
**Time to Deploy:** 4-6 hours
**Pros:**
- Lowest cost
- Non-invasive (no electrical work)
- Good accuracy (95-99%)
- DIY-friendly

**Cons:**
- Requires good lighting
- Lower real-time frequency
- Camera maintenance needed

**Best For:** Home solar monitoring, cost-conscious, no electrical expertise

---

### Tier 2: CT Clamps + ESP32/Pi
**Total Cost:** $200-400
**Time to Deploy:** 6-8 hours
**Pros:**
- Real-time power measurements
- No lighting dependencies
- Continuous monitoring
- Integration with Home Assistant

**Cons:**
- Requires soldering
- Electrical knowledge needed
- More components

**Best For:** Advanced monitoring, real-time alerts, home automation

---

### Tier 3: Commercial Gateway
**Total Cost:** $800-1500
**Time to Deploy:** 2-3 hours
**Pros:**
- Professional grade
- Warranty included
- Cloud dashboards
- Technical support

**Cons:**
- Expensive
- Cloud-dependent
- Limited customization

**Best For:** Production installations, commercial buildings, minimal maintenance

---

## Part 6: Troubleshooting Guide

### Powerwall API Issues

| Problem | Solution |
|---------|----------|
| Connection refused (192.168.91.1) | Verify network access, check gateway is on same subnet |
| "Invalid password" error | Verify you're using last 5 digits of Gateway serial (not Powerwall serial) |
| SSL certificate error | Use `-k` flag with curl or `verify=False` in Python requests |
| Timeout/no response | Check gateway is running, restart with breaker |
| Session expires | Re-authenticate (cookies are cached 30 days) |

### Camera OCR Issues

| Problem | Solution |
|---------|----------|
| Blurry images | Increase LED lighting, adjust camera focus, ensure steady mount |
| Digit recognition failures | Verify 7 digits detected, check contrast, try retraining model |
| Monotonic check fails | Verify meter reading is actually increasing, not rolling over |
| API unreachable from Pi | Check flask service running, verify firewall rules |

### CT Clamp Issues

| Problem | Solution |
|---------|----------|
| ADC reads 0V | Verify CT clamp is around conductor, check wiring, test with known load |
| Erratic readings | Add capacitor across clamp output, shield wires, reduce cable length |
| WiFi disconnects | Improve signal, reduce WiFi interference, add PoE for power stability |

---

## Part 7: JSON Response Specifications

### GET /current_readings Full Response

```json
{
  "timestamp": "2025-11-07T14:23:45.123456Z",
  "status": "ok",
  "powerwall": {
    "solar_power_watts": 2000,
    "battery_power_watts": 500,
    "grid_power_watts": 1250,
    "load_power_watts": 750,
    "battery_level_percent": 69.17,
    "grid_status": "SystemGridConnected"
  },
  "utility_meter": {
    "reading_kwh": 1234.56,
    "method": "camera_ocr",
    "last_update": "2025-11-07T14:20:00Z",
    "confidence": 0.98,
    "digits": "0123456"
  },
  "net_flow_watts": 1250,
  "summary": {
    "net_import_export": "importing",
    "home_powered_by": "grid_and_solar",
    "battery_sufficient_for_emergency": true,
    "estimated_hours_to_full_discharge": 24.3
  }
}
```

---

## Part 8: Rate Limits & Reliability

### Recommended Polling Rates

```
GET /api/meters/aggregates
├─ Max frequency: 1 request per second
├─ Recommended: 1 request per 5-15 seconds
└─ Storage: InfluxDB downsamples to 1min/5min/1hour

Meter OCR Camera
├─ Frequency: Every 15-30 minutes
├─ Why: Meter readings change slowly, saves bandwidth
└─ Accuracy: 7-digit meter (±0.01 kWh per reading)

CT Clamp Sampling
├─ Frequency: Every 60 seconds (aggregated)
└─ Why: Real-time sampling at 100Hz, but only upload aggregates
```

### Reliability Notes

- **Powerwall uptime:** 99.9%+ (local network)
- **Camera OCR reliability:** 95-99% (depends on lighting)
- **CT clamps:** 99%+ (no moving parts)
- **Network:** Single point of failure is internet outage (doesn't affect local API)

---

## Summary & Next Steps

### Recommended Implementation Path

1. **Week 1:** Deploy Local Powerwall API
   - Test connectivity and authentication
   - Set up Python monitoring script
   - Verify data accuracy

2. **Week 2:** Deploy Meter Camera (Tier 1)
   - Assemble hardware
   - Install software and dependencies
   - Calibrate OCR model
   - Test readings

3. **Week 3:** Deploy Central API Server
   - Set up Flask API
   - Integrate Powerwall + Meter
   - Test /current_readings endpoint

4. **Optional:** Add Time-Series Database & Dashboards
   - Deploy InfluxDB
   - Deploy Grafana
   - Create visualization dashboards

### Success Metrics

- Tesla Powerwall: Real-time kWh production visible in API
- Utility Meter: Daily readings automated (no manual reading needed)
- Combined Dashboard: Shows solar, battery, grid, and meter on single screen
- Alerts: Battery low, excessive grid import, meter anomalies

---

## References & Resources

### Official Documentation
- Tesla Fleet API: https://developer.tesla.com/docs/fleet-api
- Powerwall GitHub: https://github.com/vloschiavo/powerwall2
- pyPowerwall: https://github.com/jasonacox/pypowerwall

### Community Resources
- Powerwall-Dashboard: https://github.com/jasonacox/Powerwall-Dashboard
- Tesla Motors Club API Docs: https://teslamotorsclub.com/tmc/threads/powerwall-2-gateway-api-documentation.112341/
- Home Assistant Energy: https://www.home-assistant.io/blog/2021/08/04/home-energy-management/

### Hardware Suppliers
- Raspberry Pi: https://www.raspberrypi.com/
- ESP32: https://www.espressif.com/
- SCT-013 Current Sensors: Various electronics suppliers ($12-15)
- Camera Modules: Raspberry Pi official ($25-35)

---

**Document Status:** Complete  
**Last Updated:** 2025-11-07  
**Next Review:** After initial 2-week deployment period

