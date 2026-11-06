This file contains the functional version of the code currently implemented in my Ice-Maker prototype. It is a YAML configuration file that allows an ESP32 microcontroller to connect to a Home Assistant server and transmit real-time sensor data. It also includes features to track and reset total water consumption via the flow sensor, as well as remotely control the fan and water valve relays directly through the Home Assistant interface.

```yaml
esphome:
  name: ice-maker
  friendly_name: ice-maker
  min_version: 2026.4.0
  name_add_mac_suffix: false

esp32:
  variant: esp32
  framework:
    type: esp-idf

# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: !secret ice_maker__encryption_key

# Allow Over-The-Air updates
ota:
- platform: esphome

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

# --- TIME SOURCE (Required for total_daily_energy) ---
time:
  - platform: sntp
    id: my_time

# --- GLOBAL VARIABLES ---
globals:
  - id: moisture_lockout
    type: bool
    initial_value: 'false'

# --- SENSORS ---
sensor:
  # 1. Flow Sensor (GPIO 5)
  - platform: pulse_counter
    pin:
      number: GPIO5
      mode:
        input: true
        pullup: true
    name: "Ice Maker Flow Rate"
    id: flow_rate
    unit_of_measurement: "L/min"
    update_interval: 1s
    filters:
      - multiply: 0.0005022

  # Total Water Volume Tracker
  - platform: integration
    name: "Ice Maker Total Water Used"
    id: total_water_used
    sensor: flow_rate
    time_unit: min
    unit_of_measurement: "L"

  # 2. Pressure Transducer 1 (P1 - GPIO 35)
  - platform: adc
    pin: GPIO35
    name: "Pressure Sensor P1"
    id: p1
    attenuation: 11db
    unit_of_measurement: "PSI"
    update_interval: 1s
    filters:
      - calibrate_linear:
          - 0.8 -> 0.0
          - 4.5 -> 80.0

  # 3. Pressure Transducer 2 (P2 - GPIO 34)
  - platform: adc
    pin: GPIO34
    name: "Pressure Sensor P2"
    id: p2
    attenuation: 11db
    unit_of_measurement: "PSI"
    update_interval: 1s
    filters:
      - calibrate_linear:
          - 0.8 -> 0.0
          - 4.5 -> 80.0

  # 4. Pressure Difference (P1 - P2)
  - platform: template
    name: "Pressure Difference"
    id: pressure_difference
    unit_of_measurement: "PSI"
    accuracy_decimals: 1
    update_interval: 1s
    lambda: |-
      return id(p1).state - id(p2).state;

  # 5. DHT22 Temperature & Humidity Sensor (GPIO 26)
  - platform: dht
    pin:
      number: GPIO26
      mode:
        input: true
        pullup: true
    model: DHT22
    temperature:
      name: "Ambient Temperature"
      id: ambient_temperature
      on_value_range:
        - above: 28.0
          then:
            - switch.turn_on: fan_relay
        - below: 26.0
          then:
            - switch.turn_off: fan_relay
    humidity:
      name: "Ambient Humidity"
      id: ambient_humidity
    update_interval: 10s

  # 6. Analog Soil Moisture Sensor (GPIO 32)
  - platform: adc
    pin: GPIO32
    name: "Soil Moisture Raw Digital Value"
    id: soil_moisture_digital
    attenuation: 11db
    raw: true
    unit_of_measurement: "counts"
    update_interval: 1s
    accuracy_decimals: 0
    on_value_range:
      - above: 2550.0
        then:
          - globals.set:
              id: moisture_lockout
              value: 'true'
          - script.execute: stop_pumping
      - below: 0.5
        then:
          - globals.set:
              id: moisture_lockout
              value: 'false'

# --- RELAYS / ACTUATORS ---
switch:
  # Valve Relay (GPIO 21) - Active LOW (Direct Water Source Control)
  - platform: gpio
    pin:
      number: GPIO21
      inverted: true
    name: "Ice Maker Valve Relay"
    id: valve_relay
    restore_mode: ALWAYS_OFF

  # Fan Relay (GPIO 18) - Active LOW
  - platform: gpio
    pin:
      number: GPIO18
      inverted: true
    name: "Ice Maker Fan Relay"
    id: fan_relay
    restore_mode: ALWAYS_OFF

# --- AUTOMATION SCRIPTS ---
script:
  - id: start_pumping
    mode: single
    then:
      - if:
          condition:
            lambda: 'return !id(moisture_lockout);'
          then:
            - switch.turn_on: valve_relay

  - id: stop_pumping
    mode: single
    then:
      - switch.turn_off: valve_relay

# --- FLOAT SWITCH & TRIGGER LOGIC ---
binary_sensor:
  - platform: gpio
    pin:
      number: GPIO27
      mode:
        input: true
        pullup: true
      inverted: true
    name: "Ice Maker Water Level Low"
    id: float_switch
    filters:
      - delayed_on: 300ms
      - delayed_off: 300ms
    on_press:
      then:
        - script.execute: start_pumping
    on_release:
      then:
        - script.execute: stop_pumping

# --- INTERFACE BUTTONS ---
button:
  - platform: template
    name: "Reset Total Water Meter"
    icon: "mdi:refresh"
    on_press:
      then:
        - sensor.integration.reset: total_water_used
        - pulse_counter.set_total_pulses:
            id: flow_rate
            value: 0
```