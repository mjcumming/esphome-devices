---
title: Sonoff THR320D
date-published: 2023-01-07
type: relay
standard: global
board: esp32
difficulty: 3
---

## Bootloop Workaround

Some people experience a boot loop when trying to flash esphome directly.
Here's a workaround: <https://community.home-assistant.io/t/bootloop-workaround-for-flashing-sonoff-th-elite-thr316d-thr320d-and-maybe-others-with-esphome-for-the-first-time/498868>

## GPIO Pinout

(Source: <https://templates.blakadder.com/sonoff_THR320D.html>)
Some GPIO are active-low, meaning they're "on" when they're pulled low. In ESPHome that's often called "inverted".
The relays GPIO are active-high.

The main relay is bistable/latching, meaning a pulse on pin 1 switches the
relay ON, and a pulse on pin 2 switches the relay OFF.
These two pins should never be active at the same time, or the device will become dangerously hot in a few minutes.

Note that until March 2024 there was an error in this page causing a safety issue:
The code was considering the relays GPIO as being active-low, when they are actually active-high. So the two main relay pins were stay simultaneously active most of the time, making the device dangerously hot.
If you copied the old version of the code from here, please remove the ```inverted: True``` line for the relays and update your devices as soon as possible.

| Pin    | Function                                                                  |
| ------ | ----------------------------------                                        |
| GPIO0  | Push Button (HIGH = off, LOW = on)                                        |
| GPIO4  | Small Relay (Dry Contact)                                                 |
| GPIO19 | Large/Main Relay pin 1, pull high for relay ON                            |
| GPIO22 | Large/Main Relay pin 2, pull high for relay OFF                           |
| GPIO5  | Display (TM1621) Data                                                     |
| GPIO17 | Display (TM1621) CS                                                       |
| GPIO18 | Display (TM1621) Write                                                    |
| GPIO23 | Display (TM1621) Read                                                     |
| GPIO16 | Left LED (Red)                                                            |
| GPIO15 | Middle LED (Blue)                                                         |
| GPIO13 | Right LED (Green)                                                         |

## Basic Configuration

Internal momentary switches are used to pulse the ON/OFF pins on the main relay.
A template switch is used to hide the complexity of controlling the two internal
momentary switches.

One shortcoming here is we don't have any way to confirm the true state of the
main relay, and so there is a possibility that our template switch could get out
of sync with the true state of the relay.

```yaml
substitutions:
  name: "sonoffth320d"
  friendly_name: "Sonoff THR320D"
  project_name: "thermostats"
  project_version: "1.0"
  light_restore_mode: RESTORE_DEFAULT_OFF

esphome:
  name: "${name}"
  # supply the external temp/hum sensor with 3v power by pulling this GPIO high
  on_boot:
    - priority: 90
      then:
      - switch.turn_on: ${name}_sensor_power

esp32:
  board: nodemcu-32s

api:
  encryption:
    key: !secret api_encryption_key

ota:
  password: "ota-password"

logger:
  baud_rate: 0

web_server:
  port: 80

wifi:
  ssid: "SSID"
  password: "PASSWORD"
  power_save_mode: none

captive_portal:

# This will take care of the display automatically.
# You don't need to tell it to print something to the display manually.
# It'll update every 60s or so.
display:
  platform: tm1621
  id: tm1621_display
  cs_pin: GPIO17
  data_pin: GPIO5
  read_pin: GPIO23
  write_pin: GPIO18
  lambda: |-
    it.printf(0, "%.1f", id(${name}_temp).state);
    it.display_celsius(true);
    it.printf(1, "%.1f", id(${name}_humi).state);
    it.display_humidity(true);

binary_sensor:
  # single main button that also puts device into flash mode when held on boot
  - platform: gpio
    pin:
      number: GPIO0
      mode: INPUT_PULLUP
      inverted: True
    name: "${friendly_name} Button"
    on_press:
      then:
        - switch.toggle: mainRelayVirt
  - platform: status
    name: "${friendly_name} Status"



switch:
  # virtual switch to represent the main relay
  # as far as I know, we have no way to confirm the real state
  - platform: template
    id: mainRelayVirt
    name: "Main Relay"
    turn_on_action:
      - switch.turn_on: mainRelayOn
      - switch.turn_on: ${name}_onoff_led
    turn_off_action:
      - switch.turn_on: mainRelayOff
      - switch.turn_off: ${name}_onoff_led
    assumed_state: True
    optimistic: True
    restore_state: True
  # internal momentary switch for main relay ON
  - platform: gpio
    id: mainRelayOn
    internal: True
    pin:
      number: GPIO19
    on_turn_on:
      - delay: 500ms
      - switch.turn_off: mainRelayOn
    restore_mode: ALWAYS_OFF
  # internal momentary switch for main relay OFF
  - platform: gpio
    id: mainRelayOff
    internal: True
    pin:
      number: GPIO22
    on_turn_on:
      - delay: 500ms
      - switch.turn_off: mainRelayOff
    restore_mode: ALWAYS_OFF
  # dry contact relay switch
  - platform: gpio
    id: dryContRelay
    name: "Dry Contact Relay"
    pin:
      number: GPIO4
    on_turn_on:
      - switch.turn_on: ${name}_idk_led
    on_turn_off:
      - switch.turn_off: ${name}_idk_led
  # Rightmost (green) LED; use as dry contact indicator
  - platform: gpio
    id: ${name}_idk_led
    pin:
      number: GPIO13
      inverted: true
  # Leftmost (red) LED that's used to indicate the relay being on/off
  - platform: gpio
    id: ${name}_onoff_led
    pin:
      number: GPIO16
      inverted: true
  # This is needed to power the external temp/humidity sensor.
  # It receives 3v from this pin, which is pulled up on boot.
  # TODO: This should probably be an internal switch.
  - platform: gpio
    pin: GPIO27
    id: ${name}_sensor_power
    restore_mode: ALWAYS_ON


light:
  # The middle (blue) LED is used as wifi status indicator.
  - platform: status_led
    name: "${friendly_name} State"
    pin:
      number: GPIO15
      inverted: true


sensor:
  # You need to specify here that it's an SI7021 sensor.
  # This assumes you're using their device "Sonoff THS01"
  - platform: dht
    pin: GPIO25
    model: SI7021
    temperature:
      name: "${friendly_name} Temperature"
      id: ${name}_temp
    humidity:
      name: "${friendly_name} Humidity"
      id: ${name}_humi
    update_interval: 60s

climate:
  - platform: thermostat
    name: "${friendly_name} Climate"
    sensor: ${name}_temp
    default_preset: Home
    preset:
      - name: Home
        default_target_temperature_low: 21 °C
        mode: heat
    min_heating_off_time: 300s
    min_heating_run_time: 300s
    min_idle_time: 30s
    heat_action:
      - switch.turn_on: mainRelayVirt
    idle_action:
      - switch.turn_off: mainRelayVirt
    heat_deadband: 0.5 # how many degrees can we go under the temp before starting to heat
    heat_overrun: 0.5 # how many degrees can we go over the temp before stopping

text_sensor:
  - platform: wifi_info
    ip_address:
      name: "${friendly_name} IP Address"
      disabled_by_default: true
```

Here is an alternative configuration, set up to control a geyser, with an
ATTiny85 acting as a DS18B20 1-wire probe, using OneWireHub. The intent is
to use excess solar power to heat the geyser in Boost mode, revert to Eco
overnight, and default to Home in case there is no external controller.

```yaml
substitutions:
  name: "geyser"
  friendly_name: "Geyser Thermostat"
  project_name: "thermostats"
  project_version: "1.0"

packages:
  # contains basic setup, WiFi, etc
  common: !include .common.yaml

esphome:
  name: "${name}"
  friendly_name: "${friendly_name}"
  on_boot:
    - priority: 90
      then:
      # supply the external sensor with 3v power by pulling this GPIO high
      - output.turn_on: sensor_power
      # make sure the relay is in a known state at startup
      - switch.turn_off: main_relay
      # Default to running the geyser in Home mode
      - climate.control:
          id: geyser_climate
          preset: "Home"

esp32:
  board: nodemcu-32s

logger:
  # It's in the ceiling, nobody is listening to the UART
  baud_rate: 0
  level: DEBUG

web_server:
  port: 80

captive_portal:

binary_sensor:
  # single main button that also puts device into flash mode when held on boot
  # For someone in the ceiling, this can be used to turn the climate control
  # into OFF or HEAT modes. It does NOT directly control the relay.
  - platform: gpio
    pin:
      number: GPIO0
      mode: INPUT_PULLUP
      inverted: True
    id: button0
    filters:
      - delayed_on_off: 50ms
    on_press:
      then:
        - if:
            condition:
              lambda: |-
                return id(geyser_climate).mode != CLIMATE_MODE_OFF;
            then:
              - logger.log: "Button deactivates climate control"
              - climate.control:
                  id: geyser_climate
                  mode: "OFF"
            else:
              - logger.log: "Button activates climate control"
              - climate.control:
                  id: geyser_climate
                  mode: "HEAT"

switch:
  # template switch to represent the main relay
  # this is synchronised with the RED LED
  # Note: this is controlled by the climate entity, and is not exposed
  # for direct manipulation, otherwise it could be left on permanently
  - platform: template
    id: main_relay
    turn_on_action:
      - button.press: main_relay_on
      - light.turn_on: onoff_led
    turn_off_action:
      - button.press: main_relay_off
      - light.turn_off: onoff_led
    assumed_state: True
    optimistic: True
    restore_state: True

output:
  # Ideally, these two relay GPIOs should be interlocked to prevent
  # simultaneous operation. ESPhome currently does not support
  # interlocks at an output: level, or even at a button: level
  # BE CAREFUL!
  - platform: gpio
    id: main_relay_on_output
    pin:
      number: GPIO19

  - platform: gpio
    id: main_relay_off_output
    pin:
      number: GPIO22

  - platform: ledc
    id: red_led_output
    pin:
      number: GPIO16
      inverted: true

  - platform: ledc
    id: green_led_output
    pin:
      number: GPIO13
      inverted: true

  # This is needed to power the external sensor.
  # It receives 3v3 from this pin, which is pulled up on boot.
  - platform: gpio
    pin: GPIO27
    id: sensor_power

button:
  # See note above about interlocks!
  - platform: output
    id: main_relay_on
    output: main_relay_on_output
    duration: 100ms

  - platform: output
    id: main_relay_off
    output: main_relay_off_output
    duration: 100ms

# The middle (blue) LED is used as wifi status indicator.
status_led:
  pin:
    number: GPIO15
    inverted: true

light:
  # Leftmost (red) LED that's used to indicate the relay being on/off
  - platform: binary
    id: onoff_led
    output: red_led_output
    internal: true

  # Rightmost (green) LED used to indicate climate control being active
  - platform: binary
    id: auto_led
    output: green_led_output
    internal: true

sensor:
  # Geyser temperature
  # Has some failsafes to disable climate control if the temperature
  # being reported is unreasonable. Below 10C suggests that the ATTiny85
  # is either not connected to the thermistor, or is otherwise reporting
  # incorrect values, and should be investigated.
  #
  # NOTE: This can be overridden, but care should be taken when doing so
  # because these only apply when the temperature ENTERS these ranges
  # If it REMAINS in the range, and climate is turned on manually, these
  # failsafes will not apply!
  - platform: dallas_temp
    address: 0x1e11223344550028
    id: temp
    name: "Temperature"
    on_value_range:
      - below: 10.0
        then:
          - logger.log: "Temperature too low, disabling climate!"
          - climate.control:
              id: geyser_climate
              mode: "OFF"
      - above: 70.0
        then:
          - logger.log: "Temperature too high, disabling climate!"
          - climate.control:
              id: geyser_climate
              mode: "OFF"

  # The THR320 appears to run quite hot, let's just keep an eye on it
  - platform: internal_temperature
    name: "Internal Temperature"

climate:
  - platform: thermostat
    id: geyser_climate
    name: "Climate"
    sensor: temp
    visual:
      min_temperature: 45C
      max_temperature: 70C
      temperature_step:
        target_temperature: 1
        current_temperature: 1
    default_preset: Home
    preset:
      - name: Home
        default_target_temperature_low: 55C
        mode: heat
      - name: Boost
        default_target_temperature_low: 65C
        mode: heat
      - name: Eco
        default_target_temperature_low: 45C
        mode: heat
    min_heating_off_time: 0s
    min_heating_run_time: 60s
    min_idle_time: 30s
    heat_action:
      - switch.turn_on: main_relay
    idle_action:
      - switch.turn_off: main_relay
    heat_deadband: 2 # how many degrees can we go under the temp before starting to heat
    heat_overrun: 0.5 # how many degrees can we go over the temp before stopping
    off_mode:
      - switch.turn_off: main_relay
    on_state:
    - if:
        condition:
          lambda: |-
            return id(geyser_climate).mode == CLIMATE_MODE_OFF;
        then:
          - logger.log: "Climate control OFF"
          - light.turn_off: auto_led
    - if:
        condition:
          lambda: |-
            return id(geyser_climate).mode == CLIMATE_MODE_HEAT;
        then:
          - logger.log: "Climate control ON"
          - light.turn_on: auto_led

one_wire:
  pin: GPIO25
  update_interval: 10s

```

## Garage Heater Configuration with Fahrenheit Display

This configuration transforms the Sonoff THR320D into a dedicated frost protection system for garages or other outbuildings. It features a Fahrenheit-based interface with persistent settings that survive power outages, automatic sensor error detection, and intuitive LED status indicators.

### Key Features

- **Fahrenheit-based display and control**: Unlike the default Celsius configurations, this setup is designed for regions using Fahrenheit scales.
- **Persistent settings**: Climate mode and target temperature are stored in flash memory and restored after power outages.
- **Enhanced reliability**:
  - Automatic error detection for temperature sensors
  - Visual LED indicators for device status (red for relay, blue for WiFi, green for sensor)
  - Self-healing with automatic reboot if sensor errors persist
- **Simple physical control**: Single button press toggles the heater on/off
- **Anti-freeze focus**: Temperature range optimized for freeze protection (36°F-50°F)

### GPIO Usage

This configuration follows the standard Sonoff THR320D pinout with the bistable relay properly handled to prevent overheating issues.

```yaml
#############################################
# Basic device definitions
#############################################
substitutions:
  name: "garageheater"
  friendly_name: "Garage Thermostat"

esphome:
  name: "${name}"
  on_boot:
    priority: 90
    then:
      - switch.turn_on: sensor_power  # Enable external DHT power line
      - delay: 5s  # Give sensors time to initialize
      - lambda: |-
          // Restore the previous climate mode
          if (id(saved_climate_mode) == 1) {
            id(${name}_climate).mode = climate::CLIMATE_MODE_HEAT;
            // Also restore the saved target temperature
            id(${name}_climate).target_temperature = id(saved_target_temp);
            ESP_LOGI("boot", "Restored previous mode: HEAT - Target: %.1f°C/%.1f°F - Setting RED LED ON", 
                     id(saved_target_temp), 
                     id(saved_target_temp) * (9.0/5.0) + 32.0);
            id(relay_led_light).turn_on();
          } else {
            id(${name}_climate).mode = climate::CLIMATE_MODE_OFF;
            ESP_LOGI("boot", "Restored previous mode: OFF - Setting RED LED OFF");
            id(relay_led_light).turn_off();
          }
      - logger.log: "Initializing GREEN LED for sensor status indication"
      - light.turn_on: sensor_led_light

preferences:
  flash_write_interval: 1h  # Only write to flash at most once per hour

esp32:
  board: nodemcu-32s

wifi:
  networks:
    - ssid: "4254"
      password: "6129206467"
    - ssid: "FlatRockPoint"
      password: "8072243274"
  ap:
    ssid: "${name}"
  on_connect:
    then:
      - logger.log: "WiFi CONNECTED - Blue LED should be ON"
  on_disconnect:
    then:
      - logger.log: "WiFi DISCONNECTED - Blue LED should be OFF or flashing"

logger:
  baud_rate: 0
  level: DEBUG
  logs:
    climate: DEBUG
    thermostat: DEBUG

api:

ota:
  platform: esphome

web_server:
  port: 80

captive_portal:

###########################
# GLOBALS
###########################
globals:
  - id: last_valid_reading_time
    type: int
    restore_value: no
    initial_value: '0'

  - id: dht_error_count
    type: int
    restore_value: no
    initial_value: '0'
    
  - id: saved_climate_mode
    type: int
    restore_value: yes  # <-- This will persist across reboots
    initial_value: '1'  # 1 = HEAT mode (default), 0 = OFF mode

  - id: saved_target_temp
    type: float
    restore_value: yes  # Store target temperature in flash
    initial_value: '3.33'  # Default target temp (38°F in Celsius)

###########################
# OUTPUTS & LIGHTS
###########################
output:
  # Red LED for relay status (active low)
  - platform: gpio
    id: relay_led_output
    pin:
      number: GPIO16
      inverted: true
  
  # Blue LED as Wi-Fi status
  - platform: gpio
    id: wifi_led_output
    pin:
      number: GPIO15
      inverted: true
      ignore_strapping_warning: true
  
  # Green LED as sensor status
  - platform: gpio
    id: sensor_led_output
    pin:
      number: GPIO13
      inverted: true

light:
  # Red LED to indicate relay state
  - platform: binary
    id: relay_led_light
    name: "${friendly_name} Relay LED"
    output: relay_led_output
    internal: true

  # Blue LED as Wi-Fi status
  - platform: status_led
    id: wifi_led
    name: "${friendly_name} Wi-Fi LED"
    internal: true
    output: wifi_led_output

  # Green LED as sensor status
  - platform: binary
    id: sensor_led_light
    name: "${friendly_name} Sensor LED"
    output: sensor_led_output
    internal: true

###########################
# BINARY SENSORS
###########################
binary_sensor:
  # The device's push button on GPIO0
  - platform: gpio
    id: thermostat_button
    name: "${friendly_name} Button"
    internal: true
    pin:
      number: GPIO0
      mode: INPUT_PULLUP
      inverted: true
      ignore_strapping_warning: true
    filters:
      - delayed_off: 10ms
    on_click:
      - min_length: 10ms
        max_length: 350ms
        then:
          - logger.log: 
              format: "Button pressed -> toggling Thermostat Mode from %s - Temp: %.1f°F"
              args: ["id(${name}_climate).mode == climate::CLIMATE_MODE_HEAT ? \"HEAT\" : \"OFF\"", "id(${name}_temp).state"]
          - if:
              condition:
                lambda: 'return id(${name}_climate).mode == climate::CLIMATE_MODE_HEAT;'
              then:
                - climate.control:
                    id: ${name}_climate
                    mode: "OFF"
              else:
                - climate.control:
                    id: ${name}_climate
                    mode: HEAT

  # Wi-Fi/Device status
  - platform: status
    id: wifi_status
    name: "${friendly_name} Status"

  # Template sensor error indicator
  - platform: template
    name: "THS01 Sensor Error"
    id: sensor_error
    device_class: problem
    internal: false
    lambda: |-
      // Return true if it's been more than 10 minutes since last valid reading
      return (millis() - id(last_valid_reading_time) > 600000);
    on_state:
      then:
        - if:
            condition:
              lambda: 'return id(sensor_error).state;'
            then:
              - logger.log: "THS01 SENSOR ERROR DETECTED - Setting GREEN LED OFF"
              - light.turn_off: sensor_led_light
              - script.execute: start_reboot_timer
            else:
              - logger.log: "THS01 SENSOR NOW OK - Setting GREEN LED ON"
              - light.turn_on: sensor_led_light
              - script.execute: cancel_reboot_timer

###########################
# SWITCHES
###########################
switch:
  # Bistable relay ON control
  - platform: gpio
    id: thermostat_relay_on
    name: "Thermostat Relay ON"
    pin: GPIO19
    restore_mode: RESTORE_DEFAULT_OFF
    on_turn_on:
      - logger.log:
          format: "Turning ON thermostat relay (GPIO19 pulse) - Setting RED LED ON - Temp: %.0f°F"
          args: ["id(${name}_temp).state"]
      - light.turn_on: relay_led_light
      - delay: 100ms
      - switch.turn_off: thermostat_relay_on
      - logger.log: "Relay ON pulse completed"
    internal: true

  # Bistable relay OFF control
  - platform: gpio
    id: thermostat_relay_off
    name: "Thermostat Relay OFF"
    pin: GPIO22
    restore_mode: RESTORE_DEFAULT_OFF
    on_turn_on:
      - logger.log:
          format: "Turning OFF thermostat relay (GPIO22 pulse) - Setting RED LED OFF - Temp: %.1f°F"
          args: ["id(${name}_temp).state"]
      - light.turn_off: relay_led_light
      - delay: 100ms
      - switch.turn_off: thermostat_relay_off
      - logger.log: "Relay OFF pulse completed"
    internal: true

  # External sensor power line
  - platform: gpio
    pin: GPIO27
    id: sensor_power
    name: "${friendly_name} Sensor Power"
    restore_mode: ALWAYS_ON
    internal: true

  # Optional device restart
  - platform: restart
    id: restart_esp
    name: "${friendly_name} Restart"

###########################
# SENSORS
###########################
sensor:
  # Simple SI7021 sensor configuration
  - platform: dht
    pin: GPIO25  # Use simple pin definition
    model: SI7021
    update_interval: 60s  # Longer interval for more stability
    temperature:
      name: "${friendly_name} Temperature"
      id: ${name}_temp
      unit_of_measurement: "°F"  # Keep Fahrenheit for consistency
      device_class: "temperature"
      filters:
        - lambda: return x * (9.0/5.0) + 32.0;  # Simple C to F conversion
      on_value:
        then:
          - lambda: 'id(last_valid_reading_time) = millis(); id(dht_error_count) = 0;'
    humidity:
      name: "${friendly_name} Humidity"
      id: ${name}_humi

  # Wi-Fi RSSI
  - platform: wifi_signal
    name: "${friendly_name} Wi-Fi RSSI"
    update_interval: 60s

  # Create a template sensor for Celsius conversion with less processing
  - platform: template
    name: "${friendly_name} Temperature Celsius"
    id: ${name}_temp_celsius
    unit_of_measurement: "°C"
    device_class: "temperature"
    state_class: "measurement"
    accuracy_decimals: 1
    lambda: |-
      // Use integer math where possible and cache results
      if (id(${name}_temp).has_state()) {
        return (id(${name}_temp).state - 32.0f) * 0.5555f; // 5/9 as float constant
      } else {
        return NAN;
      }
    update_interval: 120s  # Increased from 30s to reduce climate state updates

  # Add these debug sensors to verify unit handling
  - platform: template
    name: "${friendly_name} Target Temperature Debug"
    id: target_temp_debug
    unit_of_measurement: "°F"
    accuracy_decimals: 1
    lambda: |-
      if (id(${name}_climate).mode == climate::CLIMATE_MODE_HEAT) {
        // Convert target temp from internal C to display F
        return id(${name}_climate).target_temperature * (9.0/5.0) + 32.0;
      } else {
        return 0; // Off mode
      }
    update_interval: 60s  # Increased from 10s to reduce frequency

  # DHT Error Counter Display
  - platform: template
    name: "${friendly_name} DHT Error Count"
    id: dht_error_count_sensor
    unit_of_measurement: "errors"
    icon: "mdi:alert-circle"
    accuracy_decimals: 0
    lambda: |-
      return id(dht_error_count);
    update_interval: 60s  # Increased from 10s to reduce frequency

# Keep the improved DHT error detection in the interval
interval:
  - interval: 30s
    then:
      - lambda: |-
          // Check for invalid readings (NaN or timeout)
          bool has_error = false;
          
          // Check if temperature reading is invalid
          if (isnan(id(${name}_temp).state)) {
            ESP_LOGW("sensor", "Temperature reading is NaN");
            has_error = true;
          }
          
          // Check if we haven't received a valid reading for more than 60 seconds
          if (millis() - id(last_valid_reading_time) > 60000) {
            ESP_LOGW("sensor", "No valid reading in %d ms", (millis() - id(last_valid_reading_time)));
            has_error = true;
          }
          
          // Update error count and LED if we have errors
          if (has_error) {
            id(dht_error_count)++;
            ESP_LOGW("sensor", "DHT error count: %d - Setting GREEN LED OFF", id(dht_error_count));
            id(sensor_led_light).turn_off();
          } else {
            ESP_LOGD("sensor", "Sensor reading is valid");
          }

###########################
# CLIMATE (Thermostat)
###########################
climate:
  - platform: thermostat
    name: "${friendly_name} Climate"
    id: ${name}_climate
    sensor: ${name}_temp_celsius  # Use the Celsius sensor for internal calculations
    
    # Reduced timing values for electric heater
    min_idle_time: 5s
    min_heating_off_time: 5s
    min_heating_run_time: 5s
    
    # Adjust hysteresis values - make sure they're working properly
    heat_deadband: 0.3  # Reduced from 0.5°C
    heat_overrun: 0.3   # Reduced from 0.5°C
    
    default_preset: Home
    preset:
      - name: Home
        default_target_temperature_low: 4    # 40°F in Celsius
        mode: heat
    
    # Visual configuration - these must be in Celsius
    visual:
      min_temperature: 2     # 36°F in Celsius
      max_temperature: 10    # 50°F in Celsius
      temperature_step: 0.5  # Smaller steps for Celsius
    
    # Fix: Using on_state instead of on_update with corrected method name
    on_state:
      then:
        - logger.log:
            format: "THERMOSTAT DEBUG: Current:%.1f°C/%.1f°F, Target:%.1f°C/%.1f°F, Mode:%s, State:%s, Ready:%s"
            args: [
              "id(${name}_temp_celsius).state", 
              "id(${name}_temp).state", 
              "id(${name}_climate).target_temperature", 
              "id(${name}_climate).target_temperature * (9.0/5.0) + 32.0",
              "id(${name}_climate).mode == climate::CLIMATE_MODE_HEAT ? \"HEAT\" : \"OFF\"",
              "id(${name}_climate).action == climate::CLIMATE_ACTION_HEATING ? \"HEATING\" : \"IDLE\"",
              "id(${name}_climate).is_ready() ? \"YES\" : \"NO\""
            ]
            level: DEBUG
        # Save target temperature when it changes
        - lambda: |-
            // Only save if target temp has changed to reduce flash writes
            if (id(${name}_climate).target_temperature != id(saved_target_temp)) {
              id(saved_target_temp) = id(${name}_climate).target_temperature;
              ESP_LOGI("climate", "Saved target temperature %.1f°C to flash", id(saved_target_temp));
            }
    
    heat_action:
      - logger.log:
          format: "TRANSITION: OFF → HEAT - Thermostat calling for HEAT (Current: %.1f°F, Target: %.1f°F, Delta: %.1f°F)"
          args: ["id(${name}_temp).state", "id(${name}_climate).target_temperature * (9.0/5.0) + 32.0", "((id(${name}_climate).target_temperature * (9.0/5.0) + 32.0) - id(${name}_temp).state)"]
          level: INFO
      - switch.turn_on: thermostat_relay_on

    idle_action:
      - logger.log:
          format: "TRANSITION: HEAT → OFF - Thermostat going idle (Current: %.1f°F, Target: %.1f°F, Delta: %.1f°F)"
          args: ["id(${name}_temp).state", "id(${name}_climate).target_temperature * (9.0/5.0) + 32.0", "(id(${name}_temp).state - (id(${name}_climate).target_temperature * (9.0/5.0) + 32.0))"]
          level: INFO
      - switch.turn_on: thermostat_relay_off
      
    # Mode-specific handlers (add these back)
    heat_mode:
      - logger.log:
          format: "Climate mode changed to HEAT - Setting RED LED ON - Current temp: %.1f°F"
          args: ["id(${name}_temp).state"]
      - light.turn_on: relay_led_light
      - lambda: |-
          // Only update flash if the mode is actually changing
          if (id(saved_climate_mode) != 1) {
            id(saved_climate_mode) = 1;
            ESP_LOGI("climate", "Saved HEAT mode (1) to persistent storage");
          }
          // Restore saved target temperature
          id(${name}_climate).target_temperature = id(saved_target_temp);
    
    off_mode:
      - logger.log:
          format: "Climate mode changed to OFF - Setting RED LED OFF - Current temp: %.1f°F"
          args: ["id(${name}_temp).state"]
      - light.turn_off: relay_led_light
      - lambda: |-
          // Only update flash if the mode is actually changing
          if (id(saved_climate_mode) != 0) {
            id(saved_climate_mode) = 0;
            ESP_LOGI("climate", "Saved OFF mode (0) to persistent storage");
          }

###########################
# TEXT SENSORS
###########################
text_sensor:
  - platform: wifi_info
    ip_address:
      name: "${friendly_name} IP Address"
      disabled_by_default: true

###########################
# SCRIPTS (Optional)
###########################
script:
  - id: start_reboot_timer
    mode: single
    then:
      - delay: 15min
      - if:
          condition:
            binary_sensor.is_on: sensor_error
          then:
            - logger.log: "Sensor inactive for 15 minutes, rebooting device..."
            - delay: 2s
            - switch.turn_on: restart_esp

  - id: cancel_reboot_timer
    mode: single
    then:
      - script.stop: start_reboot_timer

###########################
# DISPLAY
###########################
display:
  platform: tm1621
  id: tm1621_display
  cs_pin: GPIO17
  data_pin:
    number: GPIO5
    ignore_strapping_warning: true
  read_pin: GPIO23
  write_pin: GPIO18
  lambda: |-
    // Show temperature in Fahrenheit as whole number (no decimals)
    it.printf(0, "%.0f", id(${name}_temp).state);
    it.display_fahrenheit(true);
    // Show humidity
    it.printf(1, "%.0f", id(${name}_humi).state);
    it.display_humidity(true);
```

### Important Safety Note

This configuration properly handles the bistable relay by using separate GPIO pins for the ON and OFF operations. Each activation includes a short pulse duration (100ms) followed by deactivation to prevent the device from overheating. The configuration correctly implements the safety recommendations from the pinout documentation.

### Notable Improvements

1. **Fahrenheit-Celsius Conversion**: The device internally works in Celsius but displays and reports in Fahrenheit for user convenience.
2. **Persistent State**: Uses ESPHome's `restore_value: yes` feature to maintain settings across power cycles.
3. **Temperature Range**: Optimized for freeze protection with a range of 36°F (2°C) to 50°F (10°C).
4. **Self-diagnostics**: Tracks sensor errors and attempts recovery, with automatic reboot as a last resort.
5. **Visual Feedback**: Uses the three built-in LEDs for comprehensive status information:
   - Red LED: Relay/heater status
   - Blue LED: WiFi connection status
   - Green LED: Sensor status