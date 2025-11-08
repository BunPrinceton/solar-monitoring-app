# Solar Integration Quick Reference Guide

## TL;DR - Fastest Path Forward

### For Tesla Powerwall Data (Inverter Production)
**Best Option: Local Gateway API (pypowerwall)**
- Time to implement: 30 minutes
- Cost: $0
- Works offline: Yes
- Data latency: 5 seconds

**Step 1: Install pypowerwall**
```bash
pip3 install pypowerwall
python3 -m pypowerwall scan  # Find gateway IP
```

**Step 2: Get gateway IP and password**
- Find IP: Check router or use scan command above
- Password: Login at https://<GATEWAY_IP> and set customer password

**Step 3: Simple Python script**
```python
import pypowerwall

pw = pypowerwall.Powerwall(
    host="10.0.1.50",
    password="your_password",
    email="your@email.com",
    timezone="America/Los_Angeles"
)

print(f"Solar: {pw.solar()}W")
print(f"Battery: {pw.level()}%")
print(f"Grid: {pw.grid()}W")
```

---

### For Utility Meter Reading
**Best Option: AI-on-Edge OCR (ESP32)**
- Time to implement: 2-3 hours
- Cost: $15-20
- Accuracy: 99%+
- No electrical work needed

**Step 1: Order hardware**
- ESP32-CAM module (~$10)
- USB cable (~$3)
- LED light (optional, ~$2)

**Step 2: Flash firmware**
```bash
pip install esptool
# Download firmware from: https://github.com/jomjol/AI-on-the-edge-device
esptool.py --chip esp32 --port /dev/ttyUSB0 --baud 460800 \
  write_flash -z 0x1000 ai_thinker_esp32_cam.bin
```

**Step 3: Setup**
1. Device creates WiFi "AI-on-the-edge"
2. Connect and visit http://192.168.4.1
3. Configure WiFi, MQTT broker (optional)
4. Mount camera near meter
5. Define digit regions in web UI

---

## Decision Tree

```
Which Powerwall do you have?
├─ Powerwall 2 or 2+ (most common)
│  └─ Use LOCAL GATEWAY API (recommended)
│     - Pros: Fastest, offline, free, 5-sec updates
│     - Try: python3 -m pypowerwall scan
│
├─ Powerwall 3
│  └─ Option A: Use FLEET API (recommended)
│     - Pros: Official, future-proof
│     - Con: Requires Tesla dev account, 5-min delay
│     └─ Or Option B: TEDAPI (advanced)
│        - Con: Must connect to WiFi directly
│
└─ Not sure
   └─ Run: python3 -m pypowerwall scan
      It will auto-detect and show version
```

```
What's your utility meter type?
├─ Analog/dial meter (classic spinning wheels)
│  └─ Use AI-ON-EDGE ESP32 OCR (recommended)
│     - Cost: $15-20
│     - Accuracy: 99%
│     - Install: No electrical work
│
├─ Digital display meter
│  ├─ Option A: AI-on-Edge (best accuracy)
│  └─ Option B: Shelly EM CT Clamp (if panel access available)
│     - Cost: $50-100
│     - Requires panel access
│
├─ Smart meter (networked)
│  └─ Check if utility has API or MQTT support
│     - Some utilities provide direct access
│
└─ Not sure
   └─ Take photo and check:
      - Spinning dial? → AI-on-Edge
      - Digital numbers? → AI-on-Edge or Shelly
      - Has "smart meter" on label? → Check utility
```

---

## API Quick Reference

### Tesla Powerwall Local Gateway
```
Base URL: https://<GATEWAY_IP>/api
Auth: POST /login/Basic with customer credentials

Endpoints:
GET /system_status/soe              → Battery % (0-100)
GET /meters/aggregates              → All power flows (site, solar, battery, load)
GET /system_status/grid_status      → "SystemGridConnected" or "SystemIslandedActive"
GET /device/vitals                  → System health
```

**Sample Response: /api/meters/aggregates**
```json
{
  "site": {"instant_power": -450},    // Negative = exporting
  "solar": {"instant_power": 2150},
  "battery": {"instant_power": 0},
  "load": {"instant_power": 1700}
}
```

### Tesla Fleet API (Cloud)
```
Base URL: https://api.tesla.com/api/1
Auth: Bearer token (OAuth 2.0)

Endpoints:
GET /products                                    → List all products
GET /energy_sites/{site_id}/live_status        → Current readings
```

### AI-on-Edge (OCR Meter)
```
MQTT Topic: home/meter/power
Payload: {
  "device": "ai-edge-meter-1",
  "readings": {
    "meter_value": 12345.67,
    "confidence": 0.987
  }
}

Or REST API: http://<ESP32_IP>/api/status
```

### Shelly EM (CT Clamp)
```
Base URL: http://<DEVICE_IP>/rpc

Endpoints:
GET /EMData.GetStatus               → Current readings
```

---

## Hardware Comparison Table

| Method | Cost | Setup Time | Accuracy | Latency | Offline | Notes |
|--------|------|-----------|----------|---------|---------|-------|
| **Tesla Local Gateway** | $0 | 15 min | 99.9% | 5 sec | Yes | Best for PW2/+ |
| **Tesla Fleet API** | $0 | 30 min | 99% | 5 min | No | Best for PW3 |
| **AI-on-Edge OCR** | $15-20 | 2-3 hrs | 99% | Real-time | Yes | Best for analog meter |
| **Shelly EM Clamp** | $50-100 | 1-2 hrs | 95% | Real-time | Yes | Needs panel access |
| **Raspberry Pi OCR** | $95 | 3-4 hrs | 90-95% | Real-time | Yes | DIY customizable |
| **Manual reading** | $0 | - | 100% | Manual | Yes | Current approach |

---

## File Structure for Implementation

```
solar-monitor/
├── powerwall_api.py          # Tesla gateway connection
├── meter_reader.py           # Meter reading integration
├── solar_api.py              # Unified Flask API
├── requirements.txt          # Python dependencies
├── config.py                 # Configuration (passwords in env vars)
├── docker-compose.yml        # Optional: InfluxDB + Grafana
└── dashboards/
    ├── grafana_solar.json
    └── home_assistant.yaml
```

---

## Common Issues & Solutions

### "Connection refused" to Powerwall
```
1. Check IP is correct: ping <GATEWAY_IP>
2. Try https:// instead of http://
3. Check firmware version >= 20.49.0
4. Reset password at https://<GATEWAY_IP>/login
```

### OCR reading is wrong
```
1. Improve lighting (add LED)
2. Clean meter display
3. Reposition camera (perpendicular)
4. Recalibrate digit regions
5. Check confidence score (should be > 0.8)
```

### Data not updating
```
Powerwall:
- Check gateway is online (green LED)
- Verify auth token didn't expire
- Check network connectivity

Meter:
- Verify MQTT broker is running
- Check WiFi connection
- Review device logs at http://<ESP32_IP>
```

---

## One-Liner Commands

### Test Powerwall Connection
```bash
python3 -c "import pypowerwall; pw = pypowerwall.Powerwall('192.168.1.50', 'pwd', 'email@example.com'); print(f'Solar: {pw.solar()}W')"
```

### Scan for Powerwalls
```bash
python3 -m pypowerwall scan
```

### Get Powerwall Live Data (JSON)
```bash
python3 -m pypowerwall get -format json
```

### Flash ESP32 (AI-on-Edge)
```bash
esptool.py --chip esp32 --port /dev/ttyUSB0 --baud 460800 write_flash -z 0x1000 firmware.bin
```

### Test MQTT Connection
```bash
mosquitto_sub -h localhost -t "home/meter/power"
```

---

## Security Checklist

- [ ] Store passwords in environment variables, not in code
- [ ] Use HTTPS for remote API access
- [ ] Enable MQTT authentication if exposed to internet
- [ ] Rotate Tesla API tokens regularly
- [ ] Use firewall to restrict API port access
- [ ] Never commit .env files to git
- [ ] Use VPN for remote access

---

## Next Steps

1. **Identify Powerwall Model**
   ```bash
   python3 -m pypowerwall scan
   ```

2. **Choose Meter Solution**
   - Analog meter? → AI-on-Edge
   - Digital meter? → AI-on-Edge or Shelly
   - Smart meter? → Check utility provider

3. **Order Hardware** (if needed)
   - AI-on-Edge: Search "Freenove ESP32 WROVER CAM"
   - Shelly EM: Search "Shelly EM Gen3"

4. **Set Up Data Collection**
   - Start with single source (inverter OR meter)
   - Get working, then add second source

5. **Create Dashboard**
   - Grafana for historical trends
   - Home Assistant for automations
   - Simple web UI for quick stats

---

**Questions?**
- Tesla API: https://developer.tesla.com/
- AI-on-Edge: https://github.com/jomjol/AI-on-the-edge-device
- pypowerwall: https://github.com/jasonacox/pypowerwall

