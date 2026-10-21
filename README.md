# Solar weather station MISOL
Based on an ESP32-C3, ESPHome and Home Assistant. The station measures temperature, humidity, atmospheric pressure, illuminance, wind speed, wind direction and rainfall.

![First look weather station](https://raw.githubusercontent.com/ukiddingbitch/Solar_weather_station_MISOL/main/Images/First%20look%20weather%20station.jpg)

### ESPHome full YAML configuration here:

[Weather station yaml for ESPHome.txt](./yaml/Weather%20station%20yaml%20for%20ESPHome.txt)

### Features

- Outdoor temperature and relative humidity measurement
- Atmospheric pressure measurement
- Wind speed and direction measurement
- Rainfall measurement, including daily rainfall and precipitation rate
- Ambient illuminance measurement
- Battery voltage monitoring
- Wi-Fi recovery mechanism
- Home Assistant integration through the native ESPHome API
- Solar-powered operation


## Hardware/Software

| Component | Description |
| --- | --- |
| Microcontroller | ESP32-C3 DevKitM-1 |
| Weather station | MISOL WH-SP-WS02 |
| Temperature and humidity sensor | HTU21D |
| Atmospheric pressure sensor | BMP390 |
| Illuminance sensor | BH1750 |
| Wind speed sensor | Anemometer |
| Wind direction sensor | Analog wind direction sensor |
| Rain gauge | Tipping-bucket rain gauge |
| Solar panel (6V/5W)| Solar power supply |
| Battery | 2x parallel Li-ion 18650 |
| Charging module | CN3158 |
| Voltage converter | MT3608 |
| Firmware | ESPHome |
| Smart home platform | Home Assistant |

**Note:** Verify the actual components and wiring of your hardware revision before using the configuration.

### ESPHome Configuration

The following sections explain selected parts of the configuration and the calibration values that may need adjustment.

### 1. Battery Voltage Measurement

The battery voltage is measured using the ESP32-C3 ADC and a voltage divider.
```yaml
  - platform: adc
    pin: GPIO4
    id: battery_adc
    name: "Battery ADC"
    attenuation: 2.5db
    update_interval: 1s
    samples: 32
    filters:
      - sliding_window_moving_average:
          window_size: 10
          send_every: 1

  - platform: template
    name: "Battery Voltage"
    unit_of_measurement: "V"
    device_class: voltage
    state_class: measurement
    accuracy_decimals: 2
    update_interval: 1s
    lambda: |-
      return id(battery_adc).state * 11.35f;
```
**Calibration warning!:** Battery voltage is measured via a **100 kΩ / 33 kΩ** voltage divider with a **100 nF** ceramic capacitor on **GPIO4 (ADC)**. The theoretical multiplier is 4.03. Calibrate against a multimeter and ensure the ADC input voltage stays within the ESP32-C3's permitted range.

### 2. Wind Direction

Wind direction is measured using an analog input on GPIO3. The measured voltage is compared with calibrated voltage ranges to determine the compass direction.
```yaml
  - platform: adc
    pin: GPIO3
    id: wind_direction_adc
    name: "Wind Direction ADC"
    attenuation: 11db
    update_interval: 500ms
    samples: 5
    filters:
      - sliding_window_moving_average:
          window_size: 5
          send_every: 1
```
*Example voltage reference values:*

<table>
<tr>
<th colspan="2">ADC Voltage Calibration</th>
<th></th>
<th>Measured Resistance (kΩ)<br><sub>Independent of voltage values; listed in measurement order.</sub></th></tr>
<tr>
<th>Voltage (V)</th>
<th>Wind Direction</th>
<th></th>
<th>Resistance (kΩ)</th>
</tr>
<tr><td>2.984</td><td>North</td><td></td><td>3.89</td></tr>
<tr><td>2.920</td><td>Northeast</td><td></td><td>15.95</td></tr>
<tr><td>2.559</td><td>East</td><td></td><td>119.8</td></tr>
<tr><td>1.510</td><td>Southeast</td><td></td><td>64.6</td></tr>
<tr><td>0.303</td><td>South</td><td></td><td>32.94</td></tr>
<tr><td>0.606</td><td>Southwest</td><td></td><td>8.21</td></tr>
<tr><td>0.935</td><td>West</td><td></td><td>0.99</td></tr>
<tr><td>2.050</td><td>Northwest</td><td></td><td>2.193</td></tr>
</table>

**Calibration warning:** These values are specific to the sensor and circuit used in this build. Measure the actual output voltages and adjust the thresholds in the full YAML configuration if necessary.

### 3. Wind Speed

Wind speed is calculated by counting anemometer pulses. The configuration calculates speed in metres per second and kilometres per hour and uses a rolling history of measurements.
```yaml
  - platform: template
    name: "Wind Speed"
    id: wind_speed_kmh
    unit_of_measurement: "km/h"
    device_class: wind_speed
    state_class: measurement
    accuracy_decimals: 1
    lambda: |-
      return id(wind_speed_kmh_value);
```
The conversion used in the configuration is:
```yaml
wind_speed_kmh = pulses_per_second * 2.4f;
wind_speed_ms = wind_speed_kmh / 3.6f;
```
**Calibration warning:** The conversion factor "2.4" must match the characteristics of the anemometer. Compare the readings with a reference instrument or a known calibration method before relying on the measurements.

### 4. Rainfall Measurement

Rainfall is measured using a tipping-bucket rain gauge. Each detected pulse increases the accumulated rainfall.
```yaml
id(rain_count) += 0.2794f;
id(rain_count_today) += 0.2794f;
```
The configuration calculates accumulated rainfall, daily rainfall, recent rainfall and precipitation rate.

**Calibration warning:** The value "0.2794 mm" per pulse must match the rain gauge calibration. The rainfall counters are held in runtime globals in this implementation, so accumulated values may be lost after a restart unless persistence is explicitly configured. Daily reset also depends on a valid time source.

### 5. I²C Sensors

The I²C bus connects the BMP390, HTU21D and BH1750 sensors.
```yaml
i2c:
  sda: GPIO8
  scl: GPIO9
  scan: true
```
- **HTU21D:** Temperature and relative humidity
- **BMP390:** Atmospheric pressure and temperature
- **BH1750:** Ambient illuminance

The full configuration defines the individual sensor platforms and their update intervals.

**Hardware warning:** Confirm the I²C addresses, supply voltage and wiring for the specific sensor modules being used.

### 6. Illuminance Measurement

The BH1750 measures ambient illuminance in lux.
```yaml
  - platform: bh1750
    name: "Illuminance"
    address: 0x23
    update_interval: 10s
```
The address shown is a common BH1750 address. Use the address detected by your actual hardware if it differs.

### 7. Wi-Fi Reliability

The station includes a recovery mechanism intended to restore connectivity after repeated Wi-Fi failures.
```yaml
wifi:
  power_save_mode: light
  reboot_timeout: 0s
```
The full YAML also contains an automation that monitors connectivity and temporarily disables and re-enables Wi-Fi after repeated failed checks.

**Warning:** Disabling the automatic reboot timeout means that Wi-Fi disconnection alone will not trigger the usual ESPHome reboot behavior. Test the recovery automation under real network failure conditions.

### 8. Home Assistant Integration

The station uses the native ESPHome API to expose its sensor readings to Home Assistant.

The configuration also includes OTA updates and a fallback access point for recovery when the configured Wi-Fi network is unavailable.

Keep API encryption keys, Wi-Fi credentials and OTA passwords private. Never publish real credentials in a public repository.

## GPIO Pinout

| ESP32-C3 GPIO | Function |
| --- | --- |
| GPIO3 | Wind direction ADC |
| GPIO4 | Battery voltage ADC |
| GPIO5 | Rain gauge input |
| GPIO6 | Anemometer input |
| GPIO8 | I²C SDA |
| GPIO9 | I²C SCL |

Check the complete YAML and the wiring of your particular board before connecting hardware. Avoid assigning the same GPIO to incompatible functions.

### 9. Assembling

![Weather station circuit board](https://raw.githubusercontent.com/ukiddingbitch/Solar_weather_station_MISOL/main/Images/board.jpg)

![Circuit board](https://raw.githubusercontent.com/ukiddingbitch/Solar_weather_station_MISOL/main/Images/board_1.jpg)

![Circuit board with ESP32-C3](https://raw.githubusercontent.com/ukiddingbitch/Solar_weather_station_MISOL/main/Images/board%20with%20esp_1.jpg)

![Circuit board with ESP32-C3 assembled](https://raw.githubusercontent.com/ukiddingbitch/Solar_weather_station_MISOL/main/Images/board%20with%20esp.jpg)
### . Limitations and Notes

- Sensor accuracy depends on calibration, component tolerances and installation.
  
- Wind readings depend on the anemometer's pulse characteristics and exposure to the wind.
  
- Rainfall accuracy depends on the tipping-bucket calibration and pulse detection.
  
- Battery voltage readings depend on the voltage divider and ADC calibration.
  
- Solar charging performance depends on panel output, battery capacity, charging electronics and weather conditions.
  
- Wi-Fi availability and Home Assistant connectivity depend on the local network.
  
- Runtime counters may reset when the device restarts unless persistence is implemented.
