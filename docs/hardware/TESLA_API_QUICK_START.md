# Tesla Powerwall API - Quick Start Guide

## 5-Minute Setup

### Step 1: Find Gateway Serial
```bash
# Open Tesla iPhone app
# Navigate: Settings > My Home Info
# Copy: "Powerwall Gateway" field (format: STG...XXXXX)

# Extract last 5 digits = password
GATEWAY_SERIAL="STG1234ABCDE"
PASSWORD="${GATEWAY_SERIAL: -5}"  # Result: ABCDE
```

### Step 2: Test Connectivity
```bash
# Ping gateway on local network
ping 192.168.91.1

# Should respond with:
# PING 192.168.91.1 (192.168.91.1): 56 data bytes
# 64 bytes from 192.168.91.1: icmp_seq=1 ttl=61 time=5.32 ms
```

### Step 3: Authenticate & Test
```bash
#!/bin/bash

POWERWALL_IP="192.168.91.1"
GATEWAY_PASSWORD="ABCDE"
EMAIL="your@email.com"

# Authenticate
curl -k -c /tmp/cookies.txt -X POST \
  "https://${POWERWALL_IP}/api/login" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "customer",
    "password": "'${GATEWAY_PASSWORD}'",
    "email": "'${EMAIL}'"
  }' 2>/dev/null | python3 -m json.tool

# Get current readings
curl -k -b /tmp/cookies.txt \
  "https://${POWERWALL_IP}/api/meters/aggregates" 2>/dev/null | \
  python3 -m json.tool | head -50
```

### Step 4: Python Integration
```python
#!/usr/bin/env python3

import pypowerwall
import json

# Connect
pw = pypowerwall.Powerwall(
    host="192.168.91.1",
    gw_pwd="ABCDE",
    email="your@email.com",
    timezone="America/Los_Angeles"
)

# Get readings
print("Solar:", pw.solar(), "W")
print("Battery:", pw.battery(), "W")
print("Grid:", pw.grid(), "W")
print("Load:", pw.load(), "W")
print("Battery %:", pw.level(), "%")
```

## Common Issues

| Issue | Solution |
|-------|----------|
| Connection timeout | Check network: `ping 192.168.91.1` |
| Auth error | Verify gateway password = last 5 serial digits |
| SSL cert error | Use `-k` flag with curl or `verify=False` in Python |
| Timeout | Gateway may be offline; check breaker or restart |

