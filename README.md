# YAML
# Example YAML code to create an esp32 based Current and Voltage CANBus controller for the 4000W Huawai R4875G1 battery charger
esphome:
  name: can-bus01
  friendly_name: can-bus01

esp32:
  board: esp32dev
  framework:
    type: arduino

logger:
  level: ERROR


# Enable Home Assistant API
api:
 

ota:
  - platform: esphome


wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Can-Bus01 Fallback Hotspot"


captive_portal:


mqtt:
   broker: !secret mqtt_host
   username: !secret mqtt_username
   password: !secret mqtt_password
   id: mqtt_client

canbus:
  - platform: esp32_can
    tx_pin: GPIO19
    rx_pin: GPIO02
    can_id: 0x108180FE
    bit_rate: 125kbps
    use_extended_id: True
    on_frame:
    - can_id: 0x1081407F
      use_extended_id: true
      then:
        - lambda: |-
            if (x.size() >= 8) {
              uint8_t byte_0 = static_cast<uint8_t>(x[0]);
              uint8_t byte_1 = static_cast<uint8_t>(x[1]);

              // Combine the last four bytes into a 32-bit value
              uint32_t value = (static_cast<uint32_t>(x[4]) << 24) | 
                               (static_cast<uint32_t>(x[5]) << 16) | 
                               (static_cast<uint32_t>(x[6]) << 8) | 
                               static_cast<uint32_t>(x[7]);

              // Handle sensor updates based on byte_0 and byte_1
              if (byte_0 == 0x01 && byte_1 == 0x71) {
                id(grid_frequency_sensor).publish_state(value);
              } else if (byte_0 == 0x01 && byte_1 == 0x72) {
                id(input_current_sensor).publish_state(value);
              } else if (byte_0 == 0x01 && byte_1 == 0x75) {
                id(output_voltage_sensor).publish_state(value);
              } else if (byte_0 == 0x01 && byte_1 == 0x76) {
                id(set_max_output_current_sensor).publish_state(value);
              } else if (byte_0 == 0x01 && byte_1 == 0x78) {
                id(input_grid_voltage_sensor).publish_state(value);
              } else if (byte_0 == 0x01 && byte_1 == 0x7F) {
                id(output_temperature_sensor).publish_state(value);
              } else if (byte_0 == 0x01 && byte_1 == 0x81) {
                id(output_current_sensor).publish_state(value);
              }
            }


# Define sensors to expose them via API
sensor:
  - platform: template
    name: "Grid Frequency"
    id: grid_frequency_sensor
    unit_of_measurement: "Hz"
    accuracy_decimals: 1
    filters:
    - multiply: 0.00097656225
    icon: "mdi:frequency"

  - platform: template
    name: "Input Current"
    id: input_current_sensor
    unit_of_measurement: "A"
    accuracy_decimals: 1
    filters:
    - multiply: 0.00097656225
    icon: "mdi:current-ac"

  - platform: template
    name: "Output Voltage"
    id: output_voltage_sensor
    unit_of_measurement: "V"
    accuracy_decimals: 1
    filters:
    - multiply: 0.00097656225
    icon: "mdi:voltage"

  - platform: template
    name: "Set Max Output Current"
    id: set_max_output_current_sensor
    unit_of_measurement: "A"
    accuracy_decimals: 1
    filters:
    - multiply: 0.05
    icon: "mdi:current-ac"

  - platform: template
    name: "Input Grid Voltage"
    id: input_grid_voltage_sensor
    unit_of_measurement: "V"
    accuracy_decimals: 2
    filters:
    - multiply: 0.00097656225
    icon: "mdi:voltage"
    
  - platform: template
    name: "Output Temperature"
    id: output_temperature_sensor
    unit_of_measurement: "°C"
    accuracy_decimals: 1
    filters:
    - multiply: 0.00097656225
    icon: "mdi:thermometer"

  - platform: template
    name: "Output Current"
    id: output_current_sensor
    unit_of_measurement: "A"
    accuracy_decimals: 1
    filters:
    - multiply: 0.00097656225
    icon: "mdi:current-ac"

number:
  # Voltage Setting
  - platform: template
    name: "CAN Voltage Set"
    id: can_voltage_input
    min_value: 49
    max_value: 58  # Max value for the voltage input
    step: 0.1  # 0.1V steps
    unit_of_measurement: "V"
    optimistic: true
    mode: BOX
    restore_value: True
    on_value:
      then:
        - lambda: |-
            float voltage_value = id(can_voltage_input).state;  // Get the current voltage value
            int scaled_value = (int)(voltage_value * 1020);  // Scale the voltage (voltage * 1020)
            ESP_LOGD("custom", "Received voltage: %.1fV, Scaled value: %d", voltage_value, scaled_value);
            
            uint8_t high_byte = (scaled_value >> 8) & 0xFF;
            uint8_t low_byte = scaled_value & 0xFF;
            ESP_LOGD("custom", "Encoded CAN message: [0x01, 0x00, 0x00, 0x00, 0x00, 0x00, 0x%02X, 0x%02X]", high_byte, low_byte);  // Log the CAN message data

        - canbus.send:
            use_extended_id: true
            can_id: 0x108180FE
            data: !lambda |-
              std::vector<uint8_t> can_data(8, 0x00);  // Initialize with 8 zeros
              can_data[0] = 0x01;  // First byte is always 0x01
              int scaled_value = (int)(id(can_voltage_input).state * 1020);  // Scale the voltage value
              can_data[6] = (scaled_value >> 8) & 0xFF;  // Encode the high byte in the 7th byte
              can_data[7] = scaled_value & 0xFF;         // Encode the low byte in the 8th byte
              return can_data;

  # Amp Setting
  - platform: template
    name: "CAN Amp Set"
    id: can_amp_input
    min_value: 1
    max_value: 50  # Max value for the Amp input
    step: 1  # 1 Amp steps
    unit_of_measurement: "A"
    optimistic: true
    mode: BOX
    restore_value: True
    on_value:
      then:
        - lambda: |-
            float amp_value = id(can_amp_input).state;  // Get the current Amp value
            int scaled_value = (int)(amp_value * 20);  // Scale the Amp value (Amp * 20)
            ESP_LOGD("custom", "Received Amp: %.1fA, Scaled value: %d", amp_value, scaled_value);
            
            uint8_t high_byte = (scaled_value >> 8) & 0xFF;
            uint8_t low_byte = scaled_value & 0xFF;
            ESP_LOGD("custom", "Encoded CAN message: [0x01, 0x03, 0x00, 0x00, 0x00, 0x00, 0x%02X, 0x%02X]", high_byte, low_byte);  // Log the CAN message data

        - canbus.send:
            use_extended_id: true
            can_id: 0x108180FE
            data: !lambda |-
              std::vector<uint8_t> can_data(8, 0x00);  // Initialize with 8 zeros
              can_data[0] = 0x01;  // First byte is always 0x01
              can_data[1] = 0x03;  // Second byte is 0x03
              int scaled_value = (int)(id(can_amp_input).state * 20);  // Scale the Amp value
              can_data[6] = (scaled_value >> 8) & 0xFF;  // Encode the high byte in the 7th byte
              can_data[7] = scaled_value & 0xFF;         // Encode the low byte in the 8th byte
              return can_data;
  

  # Fallback Amp Setting
  - platform: template
    name: "Fallback Amp Set"
    id: fallback_amp_input
    min_value: 1
    max_value: 50  # Adjust as needed
    step: 1  # 1A steps
    unit_of_measurement: "A"
    optimistic: true
    mode: BOX
    restore_value: True
    on_value:
      then:
        - lambda: |-
            float fallback_amp_value = id(fallback_amp_input).state;  // Get the fallback Amp value
            int fallback_scaled_value = (int)(fallback_amp_value * 20);  // Scale the Amp value (Amp * 20)
            ESP_LOGD("custom", "Fallback Amp: %.1fA, Scaled value: %d", fallback_amp_value, fallback_scaled_value);
            
            uint8_t high_byte = (fallback_scaled_value >> 8) & 0xFF;  // Get high byte
            uint8_t low_byte = fallback_scaled_value & 0xFF;  // Get low byte
            ESP_LOGD("custom", "Encoded CAN message: [0x01, 0x04, 0x00, 0x00, 0x00, 0x00, 0x%02X, 0x%02X]", high_byte, low_byte);  // Log the CAN message data

        - canbus.send:
            use_extended_id: true
            can_id: 0x108180FE
            data: !lambda |-
              std::vector<uint8_t> can_data(8, 0x00);  // Initialize with 8 zeros
              can_data[0] = 0x01;  // First byte is always 0x01
              can_data[1] = 0x04;  // Second byte is 0x04 for the Fallback Amps setting
              int fallback_scaled_value = (int)(id(fallback_amp_input).state * 20);  // Scale the Amp value
              can_data[6] = (fallback_scaled_value >> 8) & 0xFF;  // Encode the high byte in the 7th byte
              can_data[7] = fallback_scaled_value & 0xFF;         // Encode the low byte in the 8th byte
              return can_data;

  # Fallback Voltage Setting
  - platform: template
    name: "Fallback Voltage Set"
    id: fallback_voltage_input
    min_value: 49
    max_value: 58  # Adjust as needed for your voltage range
    step: 0.1  # 0.1V steps
    unit_of_measurement: "V"
    optimistic: true
    mode: BOX
    restore_value: True
    on_value:
      then:
        - lambda: |-
            float fallback_voltage_value = id(fallback_voltage_input).state;  // Get the fallback voltage value
            int fallback_scaled_value = (int)(fallback_voltage_value * 1020);  // Scale the voltage value (voltage * 1020)
            ESP_LOGD("custom", "Fallback Voltage: %.1fV, Scaled value: %d", fallback_voltage_value, fallback_scaled_value);
            
            uint8_t high_byte = (fallback_scaled_value >> 8) & 0xFF;  // Get high byte
            uint8_t low_byte = fallback_scaled_value & 0xFF;  // Get low byte
            ESP_LOGD("custom", "Encoded CAN message: [0x01, 0x01, 0x00, 0x00, 0x00, 0x00, 0x%02X, 0x%02X]", high_byte, low_byte);  // Log the CAN message data

        - canbus.send:
            use_extended_id: true
            can_id: 0x108180FE
            data: !lambda |-
              std::vector<uint8_t> can_data(8, 0x00);  // Initialize with 8 zeros
              can_data[0] = 0x01;  // First byte is always 0x01
              can_data[1] = 0x01;  // Second byte is 0x01 for the Fallback Voltage setting
              int fallback_scaled_value = (int)(id(fallback_voltage_input).state * 1020);  // Scale the voltage value
              can_data[6] = (fallback_scaled_value >> 8) & 0xFF;  // Encode the high byte in the 7th byte
              can_data[7] = fallback_scaled_value & 0xFF;         // Encode the low byte in the 8th byte
              return can_data;



time:
  - platform: sntp
    on_time:
      - seconds: /5
        then:
          - canbus.send:
              use_extended_id: true
              can_id: 0x108040FE
              data: [0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00]

      - seconds: /30  # This will trigger every 30 seconds for voltage
        then:
          - canbus.send:
              use_extended_id: true
              can_id: 0x108180FE
              data: !lambda |-
                std::vector<uint8_t> can_data(8, 0x00);  // Initialize with 8 zeros
                can_data[0] = 0x01;  // First byte is always 0x01
                int scaled_value = (int)(id(can_voltage_input).state * 1020);  // Scale the voltage value
                can_data[6] = (scaled_value >> 8) & 0xFF;  // Encode the high byte in the 7th byte
                can_data[7] = scaled_value & 0xFF;         // Encode the low byte in the 8th byte
                return can_data;

      - seconds: /30  # This will trigger every 30 seconds for Amp setting
        then:
          - canbus.send:
              use_extended_id: true
              can_id: 0x108180FE
              data: !lambda |-
                std::vector<uint8_t> can_data(8, 0x00);  // Initialize with 8 zeros
                can_data[0] = 0x01;  // First byte is always 0x01
                can_data[1] = 0x03;  // Second byte is 0x03
                int scaled_value = (int)(id(can_amp_input).state * 20);  // Scale the Amp value
                can_data[6] = (scaled_value >> 8) & 0xFF;  // Encode the high byte in the 7th byte
                can_data[7] = scaled_value & 0xFF;         // Encode the low byte in the 8th byte
                return can_data;

# Define a button component for turning ON
button:
  - platform: template
    name: "CAN ON Button"
    on_press:
      then:
        - canbus.send:
            use_extended_id: true
            can_id: 0x108080FE
            data: [0x01, 0x32, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00]

# Define a button component for turning OFF
  - platform: template
    name: "CAN OFF Button"
    on_press:
      then:
        - canbus.send:
            use_extended_id: true
            can_id: 0x108080FE
            data: [0x01, 0x32, 0x00, 0x01, 0x00, 0x00, 0x00, 0x00]
  
  # Define a button component for setting the fan to full speed
  - platform: template
    name: "Fan Full Speed Button"
    on_press:
      then:
        - canbus.send:
            use_extended_id: true
            can_id: 0x108080FE
            data: [0x01, 0x34, 0x00, 0x01, 0x00, 0x00, 0x00, 0x00]

  # Define a button component for setting the fan to auto mode
  - platform: template
    name: "Fan Auto Mode Button"
    on_press:
      then:
        - canbus.send:
            use_extended_id: true
            can_id: 0x108080FE
            data: [0x01, 0x34, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00]
    
