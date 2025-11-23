# Micro-ESPectre

**Motion detection system based on Wi-Fi CSI (Channel State Information) - Pure Python implementation for MicroPython**

Micro-ESPectre is a lightweight Python port of [ESPectre](https://github.com/francescopace/espectre) that runs on MicroPython with the [esp32-microcsi](https://github.com/francescopace/esp32-microcsi) module. The name reflects both the MicroPython platform and the minimal feature set (MVS-only).

## 🎯 Features

- ✅ **Pure Python**: No C compilation required
- ✅ **MVS Segmentation**: Moving Variance Segmentation for motion detection
- ✅ **MQTT Integration**: Real-time publishing to Home Assistant or any MQTT broker
- ✅ **Smart Publishing**: Reduces bandwidth by only publishing significant changes
- ✅ **Easy Deployment**: Upload Python files via mpremote
- ✅ **Configurable**: All parameters adjustable in config.py

## 📋 Requirements

### Hardware
- ESP32-S3 or ESP32-C6 board
- 2.4GHz WiFi router

### Software
- MicroPython with esp32-microcsi module installed
- MQTT broker (Home Assistant, Mosquitto, etc.)

## 🚀 Quick Start

### 1. Install MicroPython with CSI Support

Follow the instructions at [esp32-microcsi](https://github.com/francescopace/esp32-microcsi):

```bash
# Clone esp32-microcsi repository
git clone https://github.com/francescopace/esp32-microcsi
cd esp32-microcsi

# Setup environment
./scripts/setup_env.sh

# Integrate CSI module
./scripts/integrate_csi.sh

# Build and flash (ESP32-S3)
./scripts/build_flash.sh -b ESP32_GENERIC_S3

# Or for ESP32-C6
./scripts/build_flash.sh -b ESP32_GENERIC_C6 --clean
```

### 2. Configure WiFi and MQTT

Create `config_local.py` from the template:

```bash
cp config_local.py.example config_local.py
```

Edit `config_local.py` with your credentials:

```python
# WiFi Configuration
WIFI_SSID = "YourWiFiSSID"
WIFI_PASSWORD = "YourWiFiPassword"

# MQTT Configuration
MQTT_BROKER = "192.168.1.100"  # Your MQTT broker IP or hostname
MQTT_PORT = 1883
MQTT_USERNAME = None  # Set to "username" if authentication required
MQTT_PASSWORD = None  # Set to "password" if authentication required
```

**Note**: `config_local.py` overrides the defaults in `config.py`. You can also customize other settings like topic, buffer size, etc. in `config.py`.

### 3. Upload Files to ESP32

Use the deployment script:

```bash
# Deploy only (upload files)
./deploy.sh /dev/cu.usbmodem*

# Deploy and run main application
./deploy.sh /dev/cu.usbmodem* --run

# Deploy and run debug script (diagnose CSI issues)
./deploy.sh /dev/cu.usbmodem* --debug
```

### 4. Run

```bash
# Run main application
mpremote connect /dev/cu.usbmodem* run src/main.py

# Run debug script (analyze CSI data)
mpremote connect /dev/cu.usbmodem* run examples/debug_csi.py

# Or connect to REPL and run
mpremote connect /dev/cu.usbmodem*
>>> from src import main
>>> main.main()
```

## 📁 Project Structure

```
micro-espectre/
├── src/                       # Main package
│   ├── __init__.py           # Package initialization
│   ├── main.py               # Main application entry point
│   ├── config.py             # Default configuration
│   ├── segmentation.py       # MVS segmentation logic
│   └── mqtt/                 # MQTT sub-package
│       ├── __init__.py       # MQTT package initialization
│       ├── handler.py        # MQTT connection and publishing
│       └── commands.py       # MQTT command processing (info, stats)
├── examples/                  # Utility scripts
│   └── debug_csi.py          # CSI data debug/diagnostic tool
├── config_local.py            # Local config override (gitignored)
├── config_local.py.example    # Configuration template
├── deploy.sh                  # Deployment script
├── .gitignore                 # Git ignore rules
└── README.md                  # This file
```

## ⚙️ Configuration

### Segmentation Parameters (config.py)

```python
SEG_K_FACTOR = 2.5        # Threshold sensitivity (0.5-5.0)
                          # Higher = less sensitive, fewer false positives
                          # Lower = more sensitive, more detections

SEG_WINDOW_SIZE = 30      # Moving variance window (3-50 packets)
                          # Larger = smoother, slower response
                          # Smaller = faster response, more noise

SEG_MIN_LENGTH = 10       # Min motion segment length (5-100 packets)
                          # Filters out very short movements

SEG_MAX_LENGTH = 60       # Max motion segment length (10-200 packets)
                          # Splits long continuous motion into segments

SEG_THRESHOLD = 3.0       # Base threshold 
                          # Lower values = more sensitive to motion
```

### Publishing Parameters

```python
SMART_PUBLISHING = True   # Only publish on significant changes
DELTA_THRESHOLD = 0.05    # Minimum variance change to trigger publish
MAX_PUBLISH_INTERVAL_MS = 5000  # Max time between publishes (heartbeat)
```

## 📊 MQTT Integration

### Published Data

The system publishes JSON payloads to the configured MQTT topic (default: `home/espectre/node1`):

```json
{
  "movement": 0.0234,            // Current moving variance
  "threshold": 3.0,              // Current threshold
  "state": "idle",               // "idle" or "motion"
  "packets_processed": 42,       // Packets since last publish
  "timestamp": 1700000000        // Unix timestamp
}
```

### MQTT Commands

The system listens for commands on `{topic}/cmd` and responds on `{topic}/response`.

#### Get System Information
```bash
mosquitto_pub -h broker.local -t "home/espectre/node1/cmd" -m '{"cmd":"info"}'
```

Response includes:
- Network configuration (IP address)
- MQTT topics
- Segmentation parameters
- Smart publishing settings
- Subcarrier selection

#### Get Runtime Statistics
```bash
mosquitto_pub -h broker.local -t "home/espectre/node1/cmd" -m '{"cmd":"stats"}'
```

Response includes:
- System uptime
- CPU usage (estimated)
- Memory usage (heap)
- Current state (idle/motion)
- Current turbulence and movement values
- Packets processed

Example response:
```json
{
  "timestamp": 1700000000,
  "uptime": "2h 15m 30s",
  "cpu_usage_percent": 12.5,
  "heap_usage_percent": 45.2,
  "state": "idle",
  "turbulence": 12.34,
  "movement": 0.0234,
  "threshold": 3.0,
  "packets_processed": 20
}
```

#### Configure Segmentation Parameters

**Set Threshold** (0.5-10.0):
```bash
mosquitto_pub -h broker.local -t "home/espectre/node1/cmd" -m '{"cmd":"segmentation_threshold","value":3.0}'
```

**Set K Factor** (0.5-5.0):
```bash
mosquitto_pub -h broker.local -t "home/espectre/node1/cmd" -m '{"cmd":"segmentation_k_factor","value":2.5}'
```

**Set Window Size** (3-50 packets):
```bash
mosquitto_pub -h broker.local -t "home/espectre/node1/cmd" -m '{"cmd":"segmentation_window_size","value":30}'
```

**Set Min Segment Length** (5-100 packets):
```bash
mosquitto_pub -h broker.local -t "home/espectre/node1/cmd" -m '{"cmd":"segmentation_min_length","value":10}'
```

**Set Max Segment Length** (0-200 packets, 0=no limit):
```bash
mosquitto_pub -h broker.local -t "home/espectre/node1/cmd" -m '{"cmd":"segmentation_max_length","value":60}'
```

**Set Subcarrier Selection** (array of indices 0-63):
```bash
mosquitto_pub -h broker.local -t "home/espectre/node1/cmd" -m '{"cmd":"subcarrier_selection","indices":[47,48,49,50,51,52,53,54,55,56,57,58]}'
```

#### Control Traffic Generator
```bash
# Enable traffic generator (15 packets/sec recommended)
mosquitto_pub -h broker.local -t "home/espectre/node1/cmd" -m '{"cmd":"traffic_generator_rate","value":15}'

# Change rate
mosquitto_pub -h broker.local -t "home/espectre/node1/cmd" -m '{"cmd":"traffic_generator_rate","value":20}'

# Disable traffic generator
mosquitto_pub -h broker.local -t "home/espectre/node1/cmd" -m '{"cmd":"traffic_generator_rate","value":0}'
```

The traffic generator sends UDP packets to the gateway to ensure continuous CSI data availability. This is useful when there's little ambient WiFi traffic.

#### Toggle Smart Publishing
```bash
# Enable smart publishing (reduce MQTT traffic)
mosquitto_pub -h broker.local -t "home/espectre/node1/cmd" -m '{"cmd":"smart_publishing","enabled":true}'

# Disable smart publishing (publish every second)
mosquitto_pub -h broker.local -t "home/espectre/node1/cmd" -m '{"cmd":"smart_publishing","enabled":false}'
```

#### Factory Reset
```bash
# Reset all parameters to defaults
mosquitto_pub -h broker.local -t "home/espectre/node1/cmd" -m '{"cmd":"factory_reset"}'
```

Resets:
- All segmentation parameters to defaults
- Subcarrier selection to default [47-58]
- Smart publishing to disabled
- Stops traffic generator if running
- Erases saved configuration file

### Configuration Persistence

All configuration changes made via MQTT commands are **automatically saved** to a JSON file (`espectre_config.json`) on the ESP32 filesystem. The configuration is **automatically loaded** on startup, so your settings persist across reboots.

**Saved Configuration Includes:**
- Segmentation parameters (threshold, K factor, window size, min/max length)
- Subcarrier selection
- Smart publishing setting
- Traffic generator rate

**Note:** Unlike the C version which uses NVS (Non-Volatile Storage), the MicroPython version uses a JSON file for simplicity. The functionality is equivalent.

## 🏠 Home Assistant Integration

Add to your `configuration.yaml`:

```yaml
mqtt:
  sensor:
    - name: "ESPectre Motion"
      state_topic: "home/espectre/sensor"
      value_template: "{{ 'motion' if value_json.state == 1 else 'idle' }}"
      json_attributes_topic: "home/espectre/sensor"
      json_attributes_template: "{{ value_json | tojson }}"
      
    - name: "ESPectre Variance"
      state_topic: "home/espectre/sensor"
      value_template: "{{ value_json.moving_variance }}"
      unit_of_measurement: ""
      
    - name: "ESPectre RSSI"
      state_topic: "home/espectre/sensor"
      value_template: "{{ value_json.rssi }}"
      unit_of_measurement: "dBm"
      device_class: signal_strength
```

## 🔧 Troubleshooting

### CSI Module Not Found

Make sure you've built MicroPython with the esp32-microcsi module:

```python
>>> import network
>>> wlan = network.WLAN(network.STA_IF)
>>> hasattr(wlan, 'csi')
True  # Should return True
```

### WiFi Connection Issues

Check your credentials in `config_local.py` and ensure the ESP32 is in range of your 2.4GHz WiFi network.

### MQTT Connection Issues

Verify your MQTT broker is running and accessible:

```bash
# Test with mosquitto_pub
mosquitto_pub -h 192.168.1.100 -t test -m "hello"
```

### Low Detection Accuracy

Adjust segmentation parameters in `config.py`:
- Increase `SEG_K_FACTOR` to reduce false positives
- Decrease `SEG_K_FACTOR` to increase sensitivity
- Adjust `SEG_WINDOW_SIZE` for smoother/faster response

## 📈 Performance

**ESP32-S3 (8MB PSRAM):**
- Memory usage: ~160KB
- Processing: ~5-10ms per frame
- Throughput: >100 frames/sec
- CSI buffer: 64 frames

**ESP32-C6 (no PSRAM):**
- Memory usage: ~135KB
- Processing: ~5-10ms per frame
- Throughput: >100 frames/sec
- CSI buffer: 32 frames (recommended)

## 🆚 Comparison with C Version

### Feature Comparison

| Feature | C (ESP-IDF) | Python (MicroPython) | Status |
|---------|-------------|----------------------|--------|
| **Core Algorithm** |
| MVS Segmentation | ✅ | ✅ | ✅ Aligned |
| Spatial Turbulence | ✅ | ✅ | ✅ Aligned |
| Moving Variance | ✅ | ✅ | ✅ Aligned |
| **Traffic Generator** |
| ICMP Ping | ✅ | ✅ (UDP) | ✅ Implemented |
| Configurable Rate | ✅ | ✅ | ✅ Implemented |
| **MQTT Commands** |
| `info` | ✅ | ✅ | ✅ Implemented |
| `stats` | ✅ | ✅ | ✅ Implemented |
| `segmentation_threshold` | ✅ | ✅ | ✅ Implemented |
| `segmentation_k_factor` | ✅ | ✅ | ✅ Implemented |
| `segmentation_window_size` | ✅ | ✅ | ✅ Implemented |
| `segmentation_min_length` | ✅ | ✅ | ✅ Implemented |
| `segmentation_max_length` | ✅ | ✅ | ✅ Implemented |
| `subcarrier_selection` | ✅ | ✅ | ✅ Implemented |
| `traffic_generator_rate` | ✅ | ✅ | ✅ Implemented |
| `smart_publishing` | ✅ | ✅ | ✅ Implemented |
| `factory_reset` | ✅ | ✅ | ✅ Implemented |
| `csi_raw_capture` | ✅ | ❌ | Not implemented |
| **Storage** |
| NVS Persistence | ✅ | ✅ (JSON file) | ✅ Implemented |
| Auto-save on config change | ✅ | ✅ | ✅ Implemented |
| Auto-load on startup | ✅ | ✅ | ✅ Implemented |
| **CSI Features** |
| `features_enable` | ✅ | ❌ | Not implemented |
| 10 CSI Features | ✅ | ❌ | Not implemented |
| Feature Extraction | ✅ | ❌ | Not implemented |
| Hampel Filter | ✅ | ❌ | Not implemented |
| Savitzky-Golay Filter | ✅ | ❌ | Not implemented |
| Butterworth Filter | ✅ | ❌ | Not implemented |
| Wavelet Filter | ✅ | ❌ | Not implemented |

### Performance Comparison

| Metric | C (ESP-IDF) | Python (MicroPython) |
|--------|-------------|----------------------|
| Performance | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Memory Usage | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| Ease of Use | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| Deployment | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| Code Size | ~3000 lines C | ~600 lines Python |
| Build Time | ~5 minutes | Instant (no build) |
| Update Time | ~5 minutes | ~10 seconds |

### Implementation Summary

**Micro-ESPectre (Python)** implements the **core motion detection** functionality with **11 MQTT commands** for runtime configuration. It focuses on simplicity and ease of deployment while maintaining the essential MVS algorithm aligned with the C version.

**ESPectre (C)** provides the **full feature set** including advanced filters, feature extraction, and NVS persistence, at the cost of more complex build and deployment process.

## 📚 References

- [ESPectre (C/ESP-IDF)](https://github.com/francescopace/espectre)
- [esp32-microcsi](https://github.com/francescopace/esp32-microcsi)
- [MicroPython](https://micropython.org/)

## 📄 License

GPLv3 - See LICENSE file for details

## 👤 Author

**Francesco Pace**  
📧 Email: francesco.pace@gmail.com  
💼 LinkedIn: [linkedin.com/in/francescopace](https://www.linkedin.com/in/francescopace/)
