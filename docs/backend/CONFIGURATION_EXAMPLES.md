# Configuration Examples & Code Snippets

## Central Energy API Server Configuration

### app.py - Complete Flask Application

```python
#!/usr/bin/env python3
"""
Solar Energy Monitoring System
Unified REST API for Powerwall + Utility Meter Readings
"""

from flask import Flask, jsonify, request
from flask_cors import CORS
from datetime import datetime, timedelta
import pypowerwall
import requests
import json
import logging
from functools import lru_cache
import threading
import time

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

# Initialize Flask
app = Flask(__name__)
CORS(app)

class EnergyMonitor:
    """Main monitoring class"""
    
    def __init__(self):
        self.pw = pypowerwall.Powerwall(
            host="192.168.91.1",
            gw_pwd="ABCDE",  # Last 5 digits of gateway serial
            email="your@email.com",
            timezone="America/Los_Angeles",
            auto_select=True
        )
        self.meter_endpoint = "http://192.168.1.50:5000/meter"
        self.cache_ttl = 30  # seconds
        self.last_meter_reading = None
        self.meter_readings_history = []
        
    @lru_cache(maxsize=1)
    def get_powerwall_readings(self, cache_key=None):
        """Get current Powerwall readings"""
        try:
            return {
                "timestamp": datetime.utcnow().isoformat() + "Z",
                "solar_power_watts": self.pw.solar(),
                "battery_power_watts": self.pw.battery(),
                "grid_power_watts": self.pw.grid(),
                "load_power_watts": self.pw.load(),
                "battery_level_percent": self.pw.level(),
                "grid_status": self.pw.grid_status(),
                "sitemaster": self.pw.sitemaster()
            }
        except Exception as e:
            logger.error(f"Powerwall error: {e}")
            return {"error": str(e), "timestamp": datetime.utcnow().isoformat()}
    
    def get_meter_readings(self):
        """Get utility meter readings from camera/CT clamp system"""
        try:
            response = requests.get(self.meter_endpoint, timeout=5)
            if response.status_code == 200:
                reading = response.json()
                self.last_meter_reading = reading
                self.meter_readings_history.append({
                    "timestamp": datetime.utcnow().isoformat(),
                    "reading": reading
                })
                # Keep only last 100 readings in memory
                if len(self.meter_readings_history) > 100:
                    self.meter_readings_history.pop(0)
                return reading
        except requests.exceptions.Timeout:
            logger.warning("Meter endpoint timeout")
        except Exception as e:
            logger.error(f"Meter error: {e}")
        
        return self.last_meter_reading or {
            "error": "No meter data available",
            "method": "unknown"
        }
    
    def calculate_net_flow(self, pw_data, meter_data):
        """Calculate net energy flow"""
        return {
            "solar_production_watts": pw_data.get("solar_power_watts", 0),
            "battery_charge_watts": pw_data.get("battery_power_watts", 0),
            "grid_import_watts": pw_data.get("grid_power_watts", 0),
            "home_load_watts": pw_data.get("load_power_watts", 0),
            "net_grid_flow": pw_data.get("grid_power_watts", 0),
            "description": "positive=grid export, negative=grid import"
        }

# Initialize monitor
monitor = EnergyMonitor()

# ============================================================================
# ENDPOINTS
# ============================================================================

@app.route('/health', methods=['GET'])
def health_check():
    """Simple health check"""
    try:
        pw_data = monitor.pw.sitemaster()
        return jsonify({
            "status": "healthy",
            "powerwall": "online" if pw_data.get("running") else "offline",
            "timestamp": datetime.utcnow().isoformat()
        }), 200
    except:
        return jsonify({
            "status": "unhealthy",
            "powerwall": "offline"
        }), 503

@app.route('/current_readings', methods=['GET'])
def get_current_readings():
    """
    GET /current_readings
    
    Returns combined Powerwall + Meter readings
    
    Query Parameters:
    - include_history (bool): Include 24h historical data
    
    Response: 200 OK with JSON payload
    """
    try:
        # Add cache key with current timestamp (rounded to nearest 30s)
        cache_key = int(time.time()) // 30
        pw_data = monitor.get_powerwall_readings(cache_key)
        meter_data = monitor.get_meter_readings()
        
        if "error" in pw_data:
            return jsonify({
                "status": "error",
                "message": "Powerwall offline",
                "powerwall_error": pw_data["error"],
                "timestamp": datetime.utcnow().isoformat()
            }), 503
        
        net_flow = monitor.calculate_net_flow(pw_data, meter_data)
        
        response = {
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "status": "ok",
            "powerwall": {
                "solar": {
                    "power_watts": pw_data["solar_power_watts"],
                    "status": "producing" if pw_data["solar_power_watts"] > 50 else "idle"
                },
                "battery": {
                    "power_watts": pw_data["battery_power_watts"],
                    "percent_charged": pw_data["battery_level_percent"],
                    "status": "charging" if pw_data["battery_power_watts"] > 0 else "discharging"
                },
                "grid": {
                    "power_watts": pw_data["grid_power_watts"],
                    "status": pw_data["grid_status"]
                },
                "load": {
                    "power_watts": pw_data["load_power_watts"]
                }
            },
            "utility_meter": {
                "reading_kwh": meter_data.get("reading_kwh"),
                "method": meter_data.get("method"),
                "last_update": meter_data.get("last_update"),
                "confidence": meter_data.get("confidence")
            },
            "net_flow": net_flow,
            "system_summary": {
                "mode": _get_system_mode(pw_data, meter_data),
                "efficiency": _calculate_efficiency(pw_data),
                "grid_tied": pw_data["grid_status"] == "SystemGridConnected"
            }
        }
        
        # Optional: include history
        if request.args.get('include_history', 'false').lower() == 'true':
            response["history"] = monitor.meter_readings_history[-24:]
        
        return jsonify(response), 200
    
    except Exception as e:
        logger.error(f"Error in current_readings: {e}")
        return jsonify({
            "status": "error",
            "error": str(e),
            "timestamp": datetime.utcnow().isoformat()
        }), 500

@app.route('/meter_reading', methods=['POST'])
def receive_meter_reading():
    """
    POST /meter_reading
    
    Receive meter readings from camera/CT clamp system
    
    Body: JSON with fields:
    {
        "reading_kwh": 1234.56,
        "timestamp": "2025-11-07T14:23:45Z",
        "confidence": 0.98,
        "method": "camera_ocr",
        "digits": "1234567"
    }
    """
    try:
        data = request.get_json()
        
        # Validate
        if not data or "reading_kwh" not in data:
            return jsonify({
                "status": "rejected",
                "reason": "Missing required field: reading_kwh"
            }), 400
        
        # Check confidence
        confidence = data.get("confidence", 0)
        if confidence < 0.85:
            logger.warning(f"Low confidence reading: {confidence}")
            # Still accept but flag
        
        # Store reading
        monitor.last_meter_reading = data
        logger.info(f"Meter reading received: {data['reading_kwh']} kWh")
        
        return jsonify({
            "status": "accepted",
            "message": "Reading stored"
        }), 200
    
    except Exception as e:
        logger.error(f"Error receiving meter reading: {e}")
        return jsonify({
            "status": "error",
            "error": str(e)
        }), 500

@app.route('/powerwall/status', methods=['GET'])
def powerwall_status():
    """Get detailed Powerwall status"""
    try:
        pw_data = monitor.get_powerwall_readings()
        return jsonify(pw_data), 200
    except Exception as e:
        return jsonify({"error": str(e)}), 503

@app.route('/powerwall/soe', methods=['GET'])
def powerwall_soe():
    """Get battery state of charge"""
    try:
        soe = monitor.pw.level()
        return jsonify({
            "battery_percent": soe,
            "timestamp": datetime.utcnow().isoformat()
        }), 200
    except Exception as e:
        return jsonify({"error": str(e)}), 503

@app.route('/powerwall/vitals', methods=['GET'])
def powerwall_vitals():
    """Get device health information"""
    try:
        vitals = monitor.pw.device_vitals()
        return jsonify(vitals), 200
    except Exception as e:
        return jsonify({"error": str(e)}), 503

@app.route('/history/meter', methods=['GET'])
def meter_history():
    """Get historical meter readings"""
    try:
        days = int(request.args.get('days', 7))
        # Return last N readings from history
        history = monitor.meter_readings_history[-min(96, days * 24):]
        
        return jsonify({
            "readings": history,
            "count": len(history),
            "timestamp": datetime.utcnow().isoformat()
        }), 200
    except Exception as e:
        return jsonify({"error": str(e)}), 400

# ============================================================================
# HELPER FUNCTIONS
# ============================================================================

def _get_system_mode(pw_data, meter_data):
    """Determine system operating mode"""
    solar = pw_data["solar_power_watts"]
    battery = pw_data["battery_power_watts"]
    grid = pw_data["grid_power_watts"]
    
    if solar > 1000:
        if battery > 0:
            return "solar_charging_battery"
        elif grid < 0:
            return "solar_exporting"
        else:
            return "solar_powering_home"
    elif grid < 0:
        return "exporting_to_grid"
    elif battery < 0:
        return "drawing_from_battery"
    else:
        return "grid_powered"

def _calculate_efficiency(pw_data):
    """Calculate approximate system efficiency"""
    solar = pw_data["solar_power_watts"]
    load = pw_data["load_power_watts"]
    
    if solar > 0:
        return min(100, (load / solar) * 100)
    return 0

# ============================================================================
# ERROR HANDLERS
# ============================================================================

@app.errorhandler(404)
def not_found(error):
    return jsonify({
        "status": "error",
        "message": "Endpoint not found",
        "timestamp": datetime.utcnow().isoformat()
    }), 404

@app.errorhandler(500)
def server_error(error):
    return jsonify({
        "status": "error",
        "message": "Internal server error",
        "timestamp": datetime.utcnow().isoformat()
    }), 500

# ============================================================================
# MAIN
# ============================================================================

if __name__ == '__main__':
    logger.info("Starting Energy Monitor API")
    logger.info("Tesla Powerwall: 192.168.91.1")
    logger.info("Meter endpoint: http://192.168.1.50:5000/meter")
    
    app.run(host='0.0.0.0', port=8000, debug=False, threaded=True)
```

---

## Raspberry Pi Configuration Files

### /etc/systemd/system/meter-reader.service

```ini
[Unit]
Description=Meter Reader OCR Service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/meter-reader
ExecStart=/usr/bin/python3 /home/pi/meter-reader/meter_reader.py
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

### /home/pi/meter-reader/config.json

```json
{
  "camera": {
    "device_id": 0,
    "width": 1280,
    "height": 720,
    "fps": 30,
    "flip_vertical": false,
    "flip_horizontal": false
  },
  "ocr": {
    "expected_digits": 7,
    "confidence_threshold": 0.85,
    "monotonic_check": true,
    "max_daily_increase_kwh": 100
  },
  "lighting": {
    "enabled": true,
    "brightness_target": 200,
    "auto_adjust": true
  },
  "upload": {
    "api_endpoint": "http://192.168.1.100:8000/meter_reading",
    "interval_seconds": 900,
    "timeout_seconds": 5
  },
  "storage": {
    "save_images": false,
    "image_path": "/var/log/meter-reader/images",
    "keep_days": 7
  },
  "logging": {
    "level": "INFO",
    "log_file": "/var/log/meter-reader/meter.log"
  }
}
```

### /etc/cron.d/meter-reader-tasks

```cron
# Daily image cleanup
0 3 * * * pi find /var/log/meter-reader/images -mtime +7 -delete

# Weekly restart (maintenance)
0 4 * * 0 root systemctl restart meter-reader

# Daily log rotation
0 2 * * * pi /usr/sbin/logrotate -f /etc/logrotate.d/meter-reader
```

---

## Docker Compose Setup

### docker-compose.yml

```yaml
version: '3.8'

services:
  # Energy API Server
  api:
    build:
      context: ./api
      dockerfile: Dockerfile
    container_name: energy-api
    ports:
      - "8000:8000"
    environment:
      - POWERWALL_IP=192.168.91.1
      - POWERWALL_PASSWORD=ABCDE
      - METER_ENDPOINT=http://meter-camera:5000/meter
      - INFLUXDB_URL=http://influxdb:8086
      - INFLUXDB_TOKEN=${INFLUXDB_TOKEN}
      - INFLUXDB_ORG=solar
      - INFLUXDB_BUCKET=energy
    depends_on:
      - influxdb
    restart: unless-stopped
    networks:
      - energy_net
    volumes:
      - ./logs:/app/logs

  # Time Series Database
  influxdb:
    image: influxdb:2.7-alpine
    container_name: influxdb
    ports:
      - "8086:8086"
    environment:
      - INFLUXDB_DB=energy
      - INFLUXDB_ADMIN_USER=admin
      - INFLUXDB_ADMIN_PASSWORD=${INFLUXDB_PASSWORD}
      - INFLUXDB_HTTP_AUTH_ENABLED=true
    volumes:
      - influxdb_data:/var/lib/influxdb2
      - ./influxdb/init:/docker-entrypoint-initdb.d
    restart: unless-stopped
    networks:
      - energy_net
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8086/ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Visualization Dashboards
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_INSTALL_PLUGINS=grafana-piechart-panel
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards
    depends_on:
      - influxdb
    restart: unless-stopped
    networks:
      - energy_net

  # Message Broker (optional for Home Assistant)
  mosquitto:
    image: eclipse-mosquitto:2
    container_name: mqtt
    ports:
      - "1883:1883"
      - "9001:9001"
    volumes:
      - ./mosquitto/config/mosquitto.conf:/mosquitto/config/mosquitto.conf
      - mosquitto_data:/mosquitto/data
      - mosquitto_logs:/mosquitto/log
    restart: unless-stopped
    networks:
      - energy_net

volumes:
  influxdb_data:
  grafana_data:
  mosquitto_data:
  mosquitto_logs:

networks:
  energy_net:
    driver: bridge
```

### .env (Environment Variables)

```bash
# InfluxDB
INFLUXDB_PASSWORD=your_secure_password
INFLUXDB_TOKEN=your_influxdb_token

# Grafana
GRAFANA_ADMIN_PASSWORD=your_grafana_password

# System
TIMEZONE=America/Los_Angeles
LOG_LEVEL=INFO
```

---

## Home Assistant Integration

### configuration.yaml

```yaml
# REST API endpoints
rest_command:
  get_energy_readings:
    url: "http://192.168.1.100:8000/current_readings"
    method: GET
    
  get_powerwall_vitals:
    url: "http://192.168.1.100:8000/powerwall/vitals"
    method: GET

# Sensor - Energy readings
sensor:
  - platform: rest
    name: "Energy System State"
    resource: "http://192.168.1.100:8000/current_readings"
    value_template: "{{ value_json.status }}"
    json_attributes:
      - powerwall
      - utility_meter
      - net_flow
    scan_interval: 30
    timeout: 10

# Template sensors - Extract individual values
template:
  - sensor:
      - name: "Solar Production"
        unit_of_measurement: "W"
        icon: "mdi:sun-clock"
        state_class: "measurement"
        device_class: "power"
        value_template: >
          {{ state_attr('sensor.energy_system_state', 'powerwall')['solar']['power_watts'] | int(0) }}

      - name: "Battery Level"
        unit_of_measurement: "%"
        icon: "mdi:battery"
        device_class: "battery"
        value_template: >
          {{ state_attr('sensor.energy_system_state', 'powerwall')['battery']['percent_charged'] | round(1) }}

      - name: "Grid Power Flow"
        unit_of_measurement: "W"
        icon: "mdi:transmission-tower"
        state_class: "measurement"
        device_class: "power"
        value_template: >
          {{ state_attr('sensor.energy_system_state', 'powerwall')['grid']['power_watts'] | int(0) }}

      - name: "Home Power Consumption"
        unit_of_measurement: "W"
        icon: "mdi:home-lightning-bolt"
        state_class: "measurement"
        device_class: "power"
        value_template: >
          {{ state_attr('sensor.energy_system_state', 'powerwall')['load']['power_watts'] | int(0) }}

      - name: "Utility Meter Reading"
        unit_of_measurement: "kWh"
        icon: "mdi:meter-electric"
        state_class: "total_increasing"
        device_class: "energy"
        value_template: >
          {{ state_attr('sensor.energy_system_state', 'utility_meter')['reading_kwh'] | round(2) }}

      - name: "System Mode"
        icon: "mdi:power-settings"
        value_template: >
          {{ state_attr('sensor.energy_system_state', 'system_summary')['mode'] }}

# Automation examples
automation:
  - alias: "Alert: Battery Low"
    trigger:
      platform: numeric_state
      entity_id: sensor.battery_level
      below: 15
    action:
      service: notify.mobile_app_iphone
      data:
        message: "Battery below 15%"
        title: "Powerwall Alert"

  - alias: "Log: Grid Export Started"
    trigger:
      platform: numeric_state
      entity_id: sensor.grid_power_flow
      below: -500
    action:
      service: logger.write
      data:
        level: "info"
        message: "Exporting to grid: {{ states('sensor.grid_power_flow') }}W"

  - alias: "Notify: High Grid Import"
    trigger:
      platform: numeric_state
      entity_id: sensor.grid_power_flow
      above: 5000
    action:
      service: notify.mobile_app_iphone
      data:
        message: "High grid import: {{ states('sensor.grid_power_flow') }}W"
        title: "Power Alert"
```

---

## Prometheus Metrics Export (Optional)

### /app/metrics.py

```python
from prometheus_client import Counter, Gauge, generate_latest
from flask import Response
import time

# Define metrics
solar_production = Gauge('solar_production_watts', 'Solar production in watts')
battery_level = Gauge('battery_level_percent', 'Battery charge percentage')
grid_power = Gauge('grid_power_watts', 'Grid import/export in watts')
home_load = Gauge('home_load_watts', 'Home load in watts')
meter_reading = Gauge('meter_reading_kwh', 'Utility meter reading in kWh')

readings_total = Counter('energy_readings_total', 'Total readings processed')
api_errors = Counter('api_errors_total', 'Total API errors')

def update_metrics(pw_data, meter_data):
    """Update all metrics from reading data"""
    solar_production.set(pw_data.get("solar_power_watts", 0))
    battery_level.set(pw_data.get("battery_level_percent", 0))
    grid_power.set(pw_data.get("grid_power_watts", 0))
    home_load.set(pw_data.get("load_power_watts", 0))
    meter_reading.set(meter_data.get("reading_kwh", 0))
    readings_total.inc()

@app.route('/metrics', methods=['GET'])
def metrics():
    """Prometheus metrics endpoint"""
    return Response(generate_latest(), mimetype='text/plain')
```

