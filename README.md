# PVOutput-upload-from-Home-Assistant
Upload SunPower Reserve output to PVoutput.org from Home Assistant

My configuration files for Home Assistant.

These leverage the SunPower Reserve API integration available in Home Assistant, and uploads PV data to PVoutput.org.

You'll need to know you way around Home Assistant a little bit to make these changes to your YAML configurations.

Settings -> Devices and Services -> Helpers. Input text helpers used are:

input_text.pvoutput_api_key - set this value to your pvoutput API key

input_text.pvoutput_system_id - set this value to pvoutput system id

The remainder of this document has been prepared thanks to claude.ai

# SunPower Reserve to PVOutput.org Integration for Home Assistant

This Home Assistant configuration enables automatic upload of SunPower Reserve system data to PVOutput.org, providing comprehensive monitoring of your solar generation, consumption, and battery storage performance.

## Overview

This integration uploads extended data from your SunPower Reserve system to PVOutput.org every 5 minutes, including:
- Solar panel generation (energy and power)
- Home energy consumption
- Battery storage power and charge level
- Battery energy usage
- Temperature data

## Features

- **Automated uploads** every 5 minutes
- **Extended PVOutput data** with 9 parameters
- **Secure credential management** using Home Assistant input helpers
- **Battery monitoring** for SunPower Reserve systems
- **Real-time and cumulative data** tracking

## Prerequisites

1. **SunPower Reserve system** integrated with Home Assistant
2. **PVOutput.org account** with API access enabled
3. **Home Assistant** with access to SunPower sensor entities

### Required PVOutput.org Setup

1. Create a free account at [PVOutput.org](https://pvoutput.org)
2. Navigate to **Settings** → **API Settings**
3. Enable **API Access**
4. Note your **API Key** and **System ID**
5. Ensure **Extended Data** is enabled for your system

## Installation

### Step 1: Add Configuration to `configuration.yaml`

Add the following sections to your `configuration.yaml` file:

```yaml
# REST command for PVOutput upload
rest_command:
  pvoutput_extended_upload:
    url: 'https://pvoutput.org/service/r2/addstatus.jsp'
    method: POST
    headers:
      X-Pvoutput-Apikey: "{{ states('input_text.pvoutput_api_key') }}"
      X-Pvoutput-SystemId: "{{ states('input_text.pvoutput_system_id') }}"
      Content-Type: 'application/x-www-form-urlencoded'
    payload: "d={{ now().strftime('%Y%m%d') }}&t={{ now().strftime('%H:%M') }}&v2={{ states('sensor.sunpower_system_panel_power_real_time') | float(0) | int }}&v3={{ states('sensor.sunpower_reserve_total_consumption') | float(0) * 1000 | int }}&v5={{ states('sensor.home_outside_temperature') | float(0) | int }}&v7={{ states('sensor.sunpower_reserve_storage_power_real_time') | float(0) | int }}&v8={{ states('sensor.sunpower_reserve_storage_charge') | float(0) | int }}&v9={{ states('sensor.sunpower_battery_energy_today') | float(0) * 1000 | int }}&c1=1"

# Utility sensor for tracking upload status
sensor:
  - platform: template
    sensors:
      pvoutput_last_upload:
        friendly_name: "PVOutput Last Upload"
        value_template: "{{ now().strftime('%H:%M:%S') }}"

# Input helpers for secure credential storage
input_text:
  pvoutput_api_key:
    name: "PVOutput API Key"
    max: 40
  pvoutput_system_id:
    name: "PVOutput System ID"
    max: 10
```

### Step 2: Add Automation to `automations.yaml`

Add this automation to your `automations.yaml` file:

```yaml
- id: pvoutput_extended_upload_sunpower
  alias: "Upload Extended SunPower Data to PVOutput"
  description: "Uploads additional SunPower Reserve data including battery info"
  trigger:
    - platform: time_pattern
      minutes: "/5"  # Every 5 minutes for extended data
  action:
    - service: rest_command.pvoutput_extended_upload
      data: {}
```

### Step 3: Configure Your Sensor Entities

**Important**: You must update the sensor entity names in the configuration to match your specific SunPower system. The example uses:

- `sensor.sunpower_system_panels_generation`
- `sensor.sunpower_system_panel_power_real_time`
- `sensor.sunpower_reserve_total_consumption`
- `sensor.sunpower_reserve_consumption_power_real_time`
- `sensor.home_outside_temperature`
- `sensor.sunpower_reserve_storage_power_real_time`
- `sensor.sunpower_reserve_storage_charge`
- `sensor.sunpower_battery_energy_today`

**To find your sensor names:**
1. Go to **Developer Tools** → **States** in Home Assistant
2. Search for your SunPower sensors
3. Replace the sensor names in the configuration with your actual entity IDs

### Step 4: Set Up Credentials

1. Restart Home Assistant to load the new configuration
2. Navigate to **Settings** → **Devices & Services** → **Helpers**
3. Find the "PVOutput API Key" and "PVOutput System ID" helpers
4. Enter your actual PVOutput credentials:
   - **API Key**: Your 40-character PVOutput API key
   - **System ID**: Your PVOutput system identifier

### Step 5: Test the Integration

1. Go to **Developer Tools** → **Services**
2. Find `rest_command.pvoutput_extended_upload`
3. Click **Call Service** to test the upload
4. Check your PVOutput dashboard for new data

## Data Mapping

The integration maps SunPower data to PVOutput parameters as follows:

| PVOutput Parameter | Description | SunPower Source |
|-------------------|-------------|-----------------|
| v1 | Energy Generation (Wh) | Daily solar generation |
| v2 | Power Generation (W) | Real-time solar power |
| v3 | Energy Consumption (Wh) | Daily home consumption × 1000 |
| v4 | Power Consumption (W) | Real-time consumption power |
| v5 | Temperature (°C) | Outside temperature |
| v7 | Extended Value 1 (W) | Battery storage power |
| v8 | Extended Value 2 (%) | Battery charge level |
| v9 | Extended Value 3 (Wh) | Daily battery energy × 1000 |
| c1 | Cumulative Flag | Set to 1 for cumulative data |

## Troubleshooting

### Common Issues

**Service Not Found Error:**
- Ensure the `rest_command` section is properly added to `configuration.yaml`
- Restart Home Assistant after configuration changes

**401 Authentication Error:**
- Verify your PVOutput API key and System ID are correct
- Check that API access is enabled in your PVOutput account

**400 Bad Request Error:**
- Ensure all sensor entities exist and have valid numeric values
- Check that sensors are not returning "unknown" or "unavailable"
- Verify data types are correct (integers where required)

### Debugging

1. **Check sensor values:**
   ```
   Developer Tools → States → Search for your sensors
   ```

2. **Monitor automation logs:**
   ```
   Settings → System → Logs → Filter by "automation"
   ```

3. **Test REST command manually:**
   ```
   Developer Tools → Services → rest_command.pvoutput_extended_upload
   ```

## Customization

### Adjust Upload Frequency

To change the upload interval, modify the trigger in the automation:

```yaml
trigger:
  - platform: time_pattern
    minutes: "/10"  # Every 10 minutes instead of 5
```

### Conditional Uploads

To only upload during daylight hours, add this condition:

```yaml
condition:
  - condition: sun
    after: sunrise
    before: sunset
```

## Security Notes

- API credentials are stored in Home Assistant input helpers, not hardcoded in configuration
- Credentials are referenced using templating for security
- Consider using Home Assistant's secrets.yaml for additional security

## Contributing

If you encounter issues or have improvements:
1. Check that your SunPower sensor names match the configuration
2. Verify your PVOutput account settings
3. Submit issues with relevant log entries and sensor data

## License

This configuration is provided as-is for educational and personal use. Please ensure compliance with PVOutput.org terms of service and API usage limits.

---

**Note**: This integration requires a SunPower Reserve system with Home Assistant integration. Sensor entity names will vary based on your specific system configuration.
