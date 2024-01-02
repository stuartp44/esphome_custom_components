## updated for *[👉 AOKE WP-H01 WP-CB01-001 Desktronic/JSDRIVE DCU_G-PRT5G](https://community.home-assistant.io/t/turn-your-desktronic-standing-desk-into-a-smart-desk/530732/45)* (read on Home Assistant forums)

# Added template cover to control it nicer from UI. Yaml below. 

*** 

<a name="readme_top"></a>

# esphome_desktronic_custom_component

[![Contributors][contributors_shield]][contributors_url]
[![Forks][forks_shield]][forks_url]
[![Stargazers][stars_shield]][stars_url]
[![Issues][issues_shield]][issues_url]
<br>

## 📑 About the project

This is an esphome-custom-component for the [desktronic](https://desktronic.de/)-desk-controller (type of `jsdrive`). If you are using another desk-controller I recommend this [repository](https://github.com/ssieb/custom_components) and this [HomeAssistant-Forum-Thread](https://community.home-assistant.io/t/desky-standing-desk-esphome-works-with-desky-uplift-jiecang-assmann-others/383790). The making of this project (learnings, the hardware-structure, questions, software, etc.) can be viewed on this [HomeAssistant-Forum-Thread](https://community.home-assistant.io/t/turn-your-desktronic-standing-desk-into-a-smart-desk/530732).

I can recommend the [esphome8266 d1-mini](https://www.azdelivery.de/products/d1-mini). It has enough flash-memory and is easy to use.

<p align="right">(<a href="#readme_top">back to top</a>)</p>

## 📝 Disclaimer

I can not guarantee that it works with other desk-controllers. If you have a desk-controller from desktronic and it does not work, please open an [issue][issues_url]. Then I will try to fix it as soon as possible.

<p align="right">(<a href="#readme_top">back to top</a>)</p>

## 💡 Usage (Example)

You can use the custom-component e. g. with [HomeAssistant](https://www.home-assistant.io/). Just add the following lines to your `configuration.yaml`.

### General

```yaml
substitutions:
  friendly_name: 'Seisuk'
  device_name: 'seisuk'
  node_name: 'seisuk'
  device_description: 'Seisuk AOKE WP-H01 WP-CB01-001 Desktronic/JSDRIVE DCU_G-PRT5G'
  project_base: 'AOKE' # https://aoke-europe.com/troubleshooting
  project_name: 'WP-H01' # panel
  project_version: 'WP-CB01-001' # control box # https://aoke-europe.com/mwdownloads/download/link/id/88/

    # https://profeqprofessional.nl/media/productattach//e/r/error_codes_wp-cb01-001_2.pdf
    # https://motionwise-products.com/wp-content/uploads/2018/11/MotionWise_sf_Manual_081318_E_Rev-USB.pdf
    # https://profeqprofessional.nl/media/productattach//e/r/error_codes_wp-cb01-001_2.pdf

  min_height: "65.5" # real 65  # Min height + 0.5
  max_height: "129.5" # real 130  # Max height - 0.5
  initial_value: "90"

globals:
  - id: height
    type: float
    restore_value: yes
    initial_value: '100.0'

  - id: min_height
    type: float
    restore_value: yes
    initial_value: '65.5'

  - id: max_height
    type: float
    restore_value: yes
    initial_value: '129.5'

esphome:
  name: ${device_name}
  friendly_name: ${friendly_name}
  name_add_mac_suffix: false
  comment: ${device_description}
  project:
    name: ${project_base}.${project_name}
    version: $project_version
  on_boot:
    - priority: 600.0 
      then:
        - button.press: stop
    - priority: -200.0 
      then:
        - lambda: |-
            id(uart_height).publish_state( id(height) );

esp8266:
  board: nodemcu
  early_pin_init: false
  restore_from_flash: true

packages:
  device_base: !include common/device.base.yaml

ota:
  on_begin:
    then:
      - button.press: stop

logger:
  esp8266_store_log_strings_in_flash: false  
  level: DEBUG
  baud_rate: 0

uart:
  - id: remote_bus
    tx_pin: 1 # WHITE (2nd pin)
    rx_pin: 3 # N/A (3rd pin)
    baud_rate: 9600

  - id: desk_bus
    tx_pin: 4 # N/A (D2)
    rx_pin: 5 # BLACK (D1) 
    baud_rate: 9600
    rx_buffer_size: 256

external_components:
  - source: components
    components: [ desktronic ]  

desktronic:
  id: seisuk
  desk_uart: desk_bus
  remote_uart: remote_bus
  height:
    name: Wnotworkign desktronic Height
    id: desk_height_sensor
    icon: mdi:human-male-height
    device_class: distance
    unit_of_measurement: cm
    accuracy_decimals: 1    
    internal: true
  move_pin:
    number: 14 # (D5)

binary_sensor:
  - id: moving
    name: Moving
    platform: template
    device_class: moving
    icon: mdi:hand-back-right-off
    lambda: return id(seisuk).current_operation != desktronic::DESKTRONIC_OPERATION_IDLE;
    on_state:
      - while:
          condition:
            binary_sensor.is_on: moving
          then:
            - script.execute: get_debug_height
            - delay: 50ms

  - id: going_up
    name: Moving up
    platform: template
    device_class: moving
    icon: mdi:transfer-up
    entity_category: diagnostic
    lambda: return (id(seisuk).current_operation == desktronic::DESKTRONIC_OPERATION_RAISING) || (id(seisuk).current_operation == desktronic::DESKTRONIC_OPERATION_UP);

  - id: going_down
    name: Moving down
    platform: template
    device_class: moving
    icon: mdi:transfer-down
    entity_category: diagnostic
    lambda: return (id(seisuk).current_operation == desktronic::DESKTRONIC_OPERATION_LOWERING) || (id(seisuk).current_operation == desktronic::DESKTRONIC_OPERATION_DOWN);

sensor:
  - id: uart_height
    platform: template
    name: Height
    icon: mdi:human-male-height
    device_class: distance
    unit_of_measurement: cm
    accuracy_decimals: 1
    update_interval: never
    lambda: |-
      float current = id(height);
      return current;

number:
  - platform: template
    id: desk_height
    name: Height
    icon: mdi:human-male-height-variant
    min_value: $min_height
    max_value: $max_height
    lambda: |-
      return id(height);
    device_class: distance
    unit_of_measurement: cm
    update_interval: never
    mode: box
    step: 0.1
    set_action:
      then:
        - while:
            condition: 
              - lambda: |-
                  float to = std::round(x * 10) / 10;
                  float from = std::round(id(height) * 10) / 10;
                  if ( to != from ) {
                    return true;
                  }
                  else{
                    return false;
                  }
            then:
              - script.execute: 
                  id: move_it
                  to: !lambda 'return x;'
              - delay: 150ms

cover:
  - platform: template
    name: None
    optimistic: true
    device_class: damper
    icon: mdi:desk
    assumed_state: true
    has_position: true    
    lambda: |-
      auto value = (int)id(height);
      float percentage = (float)(value - id(min_height)) / (id(max_height) - id(min_height));
      return percentage;
    open_action:
      - number.set:
          id: desk_height
          value: $max_height
    close_action:
      - number.set:
          id: desk_height
          value: $min_height
    stop_action:
      - button.press: stop
    position_action:
      - number.set:
          id: desk_height
          value: !lambda |-
            int value = id(min_height) + (int)( ( pos ) * (id(max_height) - id(min_height)) );
            return value;

button:
  - platform: template
    name: Stop
    icon: mdi:stop
    entity_category: config
    id: stop
    on_press:
      then:
        - switch.turn_off: takeover
        - script.stop: move_it
        - switch.turn_off: switch_up
        - switch.turn_off: switch_down
        - lambda: |-
            id(seisuk).stop();
        - number.set: 
            id: desk_height
            value: !lambda |-
              float x = id(height);
              return x;
        - component.update: desk_height

switch:
 - platform: gpio
   name: Takeover
   icon: mdi:gesture-tap
   id: takeover
   pin:
     number: 14 # (D5)
     mode: OUTPUT 
   restore_mode:  ALWAYS_OFF   
   entity_category: diagnostic
   disabled_by_default: true
   on_turn_on:
    then:
      - delay: 50ms
      - switch.turn_off: takeover

 - platform: template
   optimistic: true
   name: Move Up
   entity_category: config
   restore_mode: ALWAYS_OFF
   icon: mdi:arrow-up-bold-box
   id: switch_up
   on_turn_on:
     then:
       - switch.turn_on: takeover
       - switch.turn_off: switch_down
       - lambda: |-
           id(seisuk).move_up();
   on_turn_off:
     then:
       button.press: stop
 
 - platform: template
   name: Move Down
   entity_category: config
   restore_mode: ALWAYS_OFF
   optimistic: true
   icon: mdi:arrow-down-bold-box
   id: switch_down
   on_turn_on:
     then:
       - switch.turn_on: takeover
       - switch.turn_off: switch_up #  interlock: [up]
       - lambda: |-
           id(seisuk).move_down();
   on_turn_off:
     then:
       button.press: stop

script:
  - id: move_it
    parameters:
      to: float
    mode: restart
    then:
      - if:
          condition:
              - binary_sensor.is_off: moving
          then:
            - lambda: |-
                float target = std::round(to * 10) / 10;
                float from = std::round(id(height) * 10) / 10;
      
                ESP_LOGI("seisuk", "target %.1f from %.1f", target, from);
      
                if ( target != from ) {  
                  if ( target < from ) { 
                    id(switch_down).turn_on();
                  }
                  else if ( target > from ) {
                    id(switch_up).turn_on();
                  }
                  else if ( target == from ) {
                    id(stop).press();
                    ESP_LOGI("seisuk", "target == from");
                  }          
                  else {
                    id(stop).press();
                    ESP_LOGI("seisuk", "else height same");
                  }
                }
                else{
                  id(stop).press();
                  ESP_LOGI("seisuk", "height not not same");
                }
          else:
          - if:
              condition:
                  - binary_sensor.is_on: going_up 
              then:
              - lambda: |-
                  float target = std::round(to * 10) / 10;
                  float from = std::round(id(uart_height).state * 10) / 10;

                  ESP_LOGI("seisuk", "moving to %.1f from %.1f", target, from);

                  if ( target <= from ) { 
                    id(stop).press();
                    ESP_LOGE("seisuk", "height more");

                    auto call = id(desk_height).make_call();
                    call.set_value(id(uart_height).state);
                    call.perform();                  
                  }
          - if:
              condition:
                  - binary_sensor.is_on: going_down
              then:
              - lambda: |-
                  float target = std::round(to * 10) / 10;
                  float from = std::round(id(uart_height).state * 10) / 10;

                  ESP_LOGI("seisuk", "moving to %.1f from %.1f", target, from);

                  if ( target >= from ) { 
                    id(stop).press();
                    ESP_LOGE("seisuk", "height less");

                    auto call = id(desk_height).make_call();
                    call.set_value(id(uart_height).state);
                    call.perform();                  
                  }
      - delay: 100ms

  - id: get_debug_height
    mode: single
    then: 
      - lambda: |- 

          // before continuing tell its not really moving

          // id(moving).publish_state(false);
          // id(going_up).publish_state(false);
          // id(going_down).publish_state(false);

          uint8_t data[5];
          while (id(desk_bus).available() >= 5) {

            id(desk_bus).read_array(data, 5);
            if (data[0] != 0x5A) {
              break;
            }
            if ((data[1] | data[2] | data[3]) == 0x00) {
              break;
            }

            enum Segment : uint8_t
            {
                SEGMENT_INVALID = 0x00,
                SEGMENT_0 = 0x3f,
                SEGMENT_1 = 0x06,
                SEGMENT_2 = 0x5b,
                SEGMENT_3 = 0x4f,
                SEGMENT_4 = 0x67,
                SEGMENT_5 = 0x6d,
                SEGMENT_6 = 0x7d,
                SEGMENT_7 = 0x07,
                SEGMENT_8 = 0x7f,
                SEGMENT_9 = 0x6f,
            };
          
            auto segment_to_number = [](const uint8_t segment) {
                switch (segment & 0x7f)
                {
                case SEGMENT_0:
                    return 0;
                case SEGMENT_1:
                    return 1;
                case SEGMENT_2:
                    return 2;
                case SEGMENT_3:
                    return 3;
                case SEGMENT_4:
                    return 4;
                case SEGMENT_5:
                    return 5;
                case SEGMENT_6:
                    return 6;
                case SEGMENT_7:
                    return 7;
                case SEGMENT_8:
                    return 8;
                case SEGMENT_9:
                    return 9;
                default:
                    ESP_LOGD("desktronic", "idk");
                }
                return -1;
            };            

            int data0 = segment_to_number(data[1]);
            int data1 = segment_to_number(data[2]);
            int data2 = segment_to_number(data[3]);

            float got_height = 0.0;

            if (data0 < 0x00 || data1 < 0x00 || data2 < 0x00) {
              break;
            }

            got_height = data0 * 100 + data1 * 10 + data2; // + decimal * 0.1;

            // flip height for below 100
            if (data[2] & 0x80 ) { 
              got_height /= 10.0;
            }
            
            // If all correct, publish values
            if (got_height <= id(max_height) - 0.5 && got_height >= id(min_height) + 0.5 ){
              id(height) = got_height;
              id(uart_height).publish_state(got_height);              
            }

            if (got_height >= id(max_height) || got_height <= id(min_height) ){
              // id(stop).press();
              ESP_LOGE("seisuk","stopped, out of bounds");
            }

            // now its moving
            // id(moving).publish_state(true);
            // id(going_up).publish_state(true);
            // id(going_down).publish_state(true);            

          }
      - delay: 50ms
```

### HomeAssistant UI

Just add the given (by the custom-component) entities to your HomeAssistant UI:

![esphome](https://github.com/velijv/esphome_custom_components/assets/5716539/4c068e60-8ea9-4e1a-b667-7864d948e6c0)
![hass](https://github.com/velijv/esphome_custom_components/assets/5716539/f1ca8456-7e04-4c1a-9332-40f96a471977)
![mobile](https://github.com/velijv/esphome_custom_components/assets/5716539/ec28d894-8dc2-4812-bf1a-9cd448048ac8)




<p align="right">(<a href="#readme_top">back to top</a>)</p>

## 🔢 Getting started for contributing

1. Clone the repository

   ```sh
   git clone https://github.com/MhouneyLH/esphome_custom_components.git
   ```

2. Install the esphome-SourceCode from the [latest release](https://github.com/esphome/esphome/releases/).

3. Copy the directory `esphome` into the directory of the cloned repository.
   ```sh
   cp -r directory_of_esphome_release/esphome directory_of_cloned_repo
   ```

<p align="right">(<a href="#readme_top">back to top</a>)</p>

## 👨🏻‍💼 Contributing

Contributions are always welcome! Please look at following commit-conventions, while contributing: https://www.conventionalcommits.org/en/v1.0.0/#summary 😃

1. Fork the project.
2. Pick or create an [issue][issues_url] you want to work on.
3. Create your Feature-Branch. (`git checkout -b feat/best_feature`)
4. Commit your changes. (`git commit -m 'feat: add some cool feature'`)
5. Push to the branch. (`git push origin feat/best_feature`)
6. Open a Pull-Request into the Develop-Branch.
<p align="right">(<a href="#readme_top">back to top</a>)</p>

<!-- Links and Images -->

[contributors_shield]: https://img.shields.io/github/contributors/MhouneyLH/esphome_custom_components.svg?style=for-the-badge
[contributors_url]: https://github.com/MhouneyLH/esphome_custom_components/graphs/contributors
[forks_shield]: https://img.shields.io/github/forks/MhouneyLH/esphome_custom_components.svg?style=for-the-badge
[forks_url]: https://github.com/MhouneyLH/esphome_custom_components/network/members
[stars_shield]: https://img.shields.io/github/stars/MhouneyLH/esphome_custom_components.svg?style=for-the-badge
[stars_url]: https://github.com/MhouneyLH/esphome_custom_components/stargazers
[issues_shield]: https://img.shields.io/github/issues/MhouneyLH/esphome_custom_components.svg?style=for-the-badge
[issues_url]: https://github.com/MhouneyLH/esphome_custom_components/issues
