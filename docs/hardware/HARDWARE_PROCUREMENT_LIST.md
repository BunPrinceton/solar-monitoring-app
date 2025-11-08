# Hardware Procurement List
## Solar Energy Integration System

### Tier 1: Camera OCR System (RECOMMENDED)
**Total: $150-250 | Setup Time: 4-6 hours | Accuracy: 95-99%**

#### Core Components

| Item | Qty | Unit Cost | Total | Supplier | Notes |
|------|-----|-----------|-------|----------|-------|
| Raspberry Pi 4B (2GB) | 1 | $40 | $40 | Pi Foundation | Recommended over Pi 5 for stability |
| Raspberry Pi Camera v2 8MP | 1 | $30 | $30 | Pi Foundation | CSI ribbon connector |
| MicroSD Card 32GB Class 10 | 1 | $10 | $10 | Amazon/Newegg | OS installation |
| USB-C Power Supply 5V 3A | 1 | $15 | $15 | Amazon | Official Pi supply |
| White LED Strip 12V 5m | 1 | $12 | $12 | Amazon | 60 LED/m minimum |
| Weatherproof ABS Enclosure | 1 | $25 | $25 | Amazon | IP65+ rated |
| Camera Bracket Mount | 1 | $15 | $15 | Amazon | Adjustable angle |
| Cat6 Ethernet Cable 50ft | 1 | $10 | $10 | Amazon | Or USB-C power extension |
| Power Supply Step-Down Converter | 1 | $8 | $8 | Amazon | 12V to 5V for LEDs |
| Cable Glands & Connectors | 1 | $8 | $8 | Amazon | M20/M25 glands |
| Mounting Hardware (nuts/bolts) | 1 | $5 | $5 | Hardware store | Stainless steel |
| Heatsink + Fan for Pi (optional) | 1 | $15 | $15 | Amazon | For hot climates |
| | | **SUBTOTAL** | **$193** | | |
| | | **+ 10% contingency** | **$212** | | |

#### Software (Free)
- Raspberry Pi OS Lite
- Python 3 + OpenCV
- PyTorch
- Flask

#### Setup Time Breakdown
- Hardware assembly: 45 minutes
- Pi OS installation: 30 minutes
- Software installation: 45 minutes
- Camera calibration: 60 minutes
- Testing & mounting: 60 minutes

#### Where to Buy
- **Raspberry Pi:** raspberrypi.com or Adafruit
- **Camera:** Arducam, Adafruit, Pi Foundation
- **Power/Housing:** Amazon, Newegg
- **Electronics:** Amazon, Sparkfun, Adafruit

#### Procurement Timeline
- Order: 1 day
- Shipping: 2-5 days
- Assembly: 1 day

---

### Tier 2: CT Clamp IoT System
**Total: $200-400 | Setup Time: 6-8 hours | Accuracy: 95-98% (real-time)**

#### Core Components

| Item | Qty | Unit Cost | Total | Supplier | Notes |
|------|-----|-----------|-------|----------|-------|
| ESP32 DevKit v1 | 1 | $12 | $12 | Amazon | 30-pin version |
| SCT-013-030 Current Sensor | 2 | $12 | $24 | Amazon | 30A max, ±1V output |
| ADS1115 16-bit ADC | 1 | $8 | $8 | Amazon | I2C 4-channel converter |
| 10K Ω 1% Resistor (pack) | 1 | $3 | $3 | Amazon | For voltage divider |
| 10µF Electrolytic Capacitor | 1 | $2 | $2 | Amazon | Filtering |
| 33Ω 1/4W Resistor | 1 | $2 | $2 | Amazon | Burden resistor |
| Dupont Jumper Wires | 1 | $5 | $5 | Amazon | 40 pin pack |
| Breadboard (optional) | 1 | $3 | $3 | Amazon | For prototyping |
| USB Micro-B Cable | 1 | $3 | $3 | Amazon | Programming ESP32 |
| 5V USB Power Supply 2A | 1 | $10 | $10 | Amazon | For ESP32 |
| DIN Rail Enclosure 250x175 | 1 | $35 | $35 | Amazon | IP66 rated |
| Weatherproof Terminal Box | 1 | $25 | $25 | Amazon | Outdoor mounting |
| Cable Glands M20 | 4 | $3 | $12 | Amazon | For CT clamp cables |
| Mounting Rail & Clamps | 1 | $20 | $20 | Amazon | DIN rail components |
| Solder & Flux | 1 | $10 | $10 | Adafruit | For assembly |
| | | **SUBTOTAL** | **$215** | | |
| | | **+ 15% contingency** | **$247** | | |

#### Software (Free)
- MicroPython
- EmonLib (if using Arduino)

#### Setup Time Breakdown
- Component sourcing: 1 day
- Soldering circuit: 2 hours
- Programming & testing: 2 hours
- CT clamp installation: 2 hours
- Calibration: 1 hour

#### Sourcing Notes
- Buy CT clamps from YHDC direct or Amazon
- ADS1115 widely available (also ADS1015 for lower cost)
- Soldering iron & solder required (cost ~$20 if not already owned)

---

### Tier 3: Commercial Smart Gateway
**Total: $800-1500 | Setup Time: 1-2 hours | Accuracy: 99%+**

#### LAIWA EnergyCam Solution

| Item | Qty | Unit Cost | Total | Supplier | Notes |
|------|-----|-----------|-------|----------|-------|
| EnergyCam Device | 1 | $900 | $900 | LAIWA | Includes camera + AI |
| Installation Kit | 1 | $100 | $100 | LAIWA | Mounting, cables |
| Annual Cloud Subscription | 1 | $150 | $150 | LAIWA | Dashboard + API access |
| | | **SUBTOTAL** | **$1150** | | |

#### Pressac Wireless Solution

| Item | Qty | Unit Cost | Total | Supplier | Notes |
|------|-----|-----------|-------|----------|-------|
| Pressac Wireless CT Clamps (2x) | 2 | $150 | $300 | Pressac | Wireless transmitters |
| Smart Gateway | 1 | $200 | $200 | Pressac | WiFi/Ethernet bridge |
| Installation | 1 | $300 | $300 | Pressac | Professional install |
| | | **SUBTOTAL** | **$800** | | |

---

## Complete System Architecture

### Bill of Materials Summary

```
RECOMMENDED TIER 1 (Camera OCR)
├─ Raspberry Pi 4B: $40
├─ Camera Module: $30
├─ Lighting: $12
├─ Housing & Mounting: $45
├─ Power/Cabling: $30
├─ Conditioning: $8
└─ TOTAL: $165-200

TESLA POWERWALL (Already owned)
└─ Local API: Free

CENTRAL API SERVER
├─ Laptop/NUC with Python: $0-800 (optional)
├─ InfluxDB: Free (Docker)
├─ Grafana: Free (Docker)
└─ Flask API: Free

SYSTEM TOTAL (Tier 1): $165-200
```

---

## Supplier Recommendations

### Tier 1 Suppliers (Camera OCR)

| Supplier | Best For | Pros | Cons |
|----------|----------|------|------|
| **Raspberry Pi Foundation Store** | Official hardware | Authentic, guaranteed | Premium pricing |
| **Adafruit** | Components + kits | Good guides + support | Premium pricing |
| **Amazon** | General electronics | Fast shipping, variety | Quality varies |
| **Newegg** | Bulk components | Good prices | Longer shipping |
| **SparkFun** | DIY electronics | Educational resources | Premium pricing |

### Tier 2 Suppliers (CT Clamps)

| Supplier | Product | Cost | Lead Time |
|----------|---------|------|-----------|
| **Amazon** | SCT-013, ADS1115 | $12-15/sensor | 2-5 days |
| **eBay** | Chinese vendors | $8-10/sensor | 7-14 days |
| **YHDC Direct** | OEM sensors | $10-12/sensor | 5-10 days |
| **Aliexpress** | Bulk pricing | $3-5/sensor | 14-30 days |

### Tier 3 Suppliers (Commercial)

| Supplier | Product | Website |
|----------|---------|---------|
| **LAIWA Metering** | EnergyCam | laiwa-metering.com |
| **Pressac** | Wireless CT | pressac.com |
| **Milesight** | LoRaWAN Sensors | milesight.com |

---

## Cost Analysis by Tier

### Tier 1: Camera OCR
```
One-time costs:     $165-200
Annual costs:       $0
ROI (5 years):      Excellent - near free monitoring
```

### Tier 2: CT Clamps
```
One-time costs:     $200-400
Annual costs:       $0
ROI (5 years):      Good - more features than Tier 1
```

### Tier 3: Commercial
```
One-time costs:     $800-1500
Annual costs:       $150-300 (cloud subscription)
ROI (5 years):      Fair - professional but expensive
```

---

## Recommended Shopping List (Order Now)

### Priority 1 (Essential)
```
[ ] Raspberry Pi 4B 2GB
[ ] Raspberry Pi Camera v2
[ ] MicroSD Card 32GB
[ ] USB-C Power Supply 5V 3A
[ ] White LED Strip 12V
[ ] Weatherproof Enclosure IP65
```

### Priority 2 (Recommended)
```
[ ] Camera Mount Bracket
[ ] Cat6 Ethernet Cable 50ft
[ ] Power Converter 12V to 5V
[ ] Cable Glands & Connectors
```

### Priority 3 (Optional)
```
[ ] Pi Heatsink + Fan
[ ] Backup MicroSD Card
[ ] Extra Jumper Wires
[ ] Testing equipment (multimeter)
```

---

## Installation Verification Checklist

After procurement, verify:

```
Tier 1 (Camera OCR)
[ ] Raspberry Pi boots and connects to network
[ ] Camera interface enabled in raspi-config
[ ] Camera test image captures properly
[ ] LED lighting provides adequate brightness
[ ] Weatherproof enclosure seals properly
[ ] Network connectivity stable (WiFi/Ethernet)

Tier 2 (CT Clamps)
[ ] ESP32 recognized on USB
[ ] ADS1115 I2C communication verified
[ ] CT clamps read voltage when placed around conductor
[ ] WiFi connection stable
[ ] API endpoint responds to requests

Both Systems
[ ] Powerwall local API accessible at 192.168.91.1
[ ] HTTPS connection works with -k flag
[ ] Authentication successful
[ ] /api/meters/aggregates returns data
```

