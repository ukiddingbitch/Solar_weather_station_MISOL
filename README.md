# Solar weather station MISOL
Using ESP32-C3 with ESPHome fimware

![First look weather station](https://raw.githubusercontent.com/ukiddingbitch/Solar_weather_station_MISOL/main/Images/First%20look%20weather%20station.jpg)

### ESPHome full YAML configuration here:

[Weather station yaml for ESPHome.txt](./yaml/Weather%20station%20yaml%20for%20ESPHome.txt)

GPIO4 was used for ADC battery voltage measurement

```yaml
- platform: adc
  pin: GPIO4
  id: battery_adc
  name: "Battery ADC"
  attenuation: 2.5db
  update_interval: 1s
  unit_of_measurement: "V"
  accuracy_decimals: 3
  samples: 32
  sampling_mode: avg

- platform: template
  id: battery_voltage
  name: "Battery Voltage"
  unit_of_measurement: "V"
  accuracy_decimals: 2
  update_interval: 1s
  lambda: |-
    float adc = id(battery_adc).state;

    if (isnan(adc))
      return NAN;

    return adc * 11.35f;
  filters:
    - sliding_window_moving_average:
        window_size: 10
        send_every: 1
        send_first_at: 1
```
Battery voltage measurement have to be done then change multiplier if this one will not work:
```yaml
return adc * 11.35f;
```
Images/board with esp.jpg
