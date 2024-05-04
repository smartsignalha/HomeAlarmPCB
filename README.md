### BASIC  SETUP   USING  ESPHOME###
###HOME ALARM  VERSION  3.0
######################################


esphome:
  name: homealarm
  friendly_name: HOMEALARM

    
esp32:
  board: esp32dev
  framework:
    type: arduino

logger🔢

api:
  encryption:
    key: !SECRET

ota:
  password: !SECRET

#ETHERNET#
ethernet:
  id: eth
  type: LAN8720
  mdc_pin: GPIO23
  mdio_pin: GPIO18
  clk_mode: GPIO0_IN
  phy_addr: 1
  power_pin: GPIO16

#CHECK GATEWAY#
  manual_ip:
    static_ip: 000.000.0.000
    gateway: 000.000.0.000
    subnet: 255.255.255.0 

#TEXT SENSOR#
text_sensor:
  - platform: template
    name: "Uptime"
    lambda: |-
      uint32_t dur = id(uptime_s).state;
      int dys = 0;
      int hrs = 0;
      int mnts = 0;
      if (dur > 86399) {
        dys = trunc(dur / 86400);
        dur = dur - (dys * 86400);
      }
      if (dur > 3599) {
        hrs = trunc(dur / 3600);
        dur = dur - (hrs * 3600);
      }
      if (dur > 59) {
        mnts = trunc(dur / 60);
        dur = dur - (mnts * 60);
      }
      char buffer[17];
      sprintf(buffer, "%ud %02uh %02um %02us", dys, hrs, mnts, dur);
      return {buffer};
    icon: mdi:clock-start
    update_interval: 60s
  
  
  - platform: homeassistant
    entity_id: alarm_control_panel.alarmo
    name: "Alarm State"
    id: alarm_state


## STATUS  LIGHT ##
light:
  - platform: status_led
    name: "Switch state"
    pin: 
      number: 04
      inverted: true

esp32_ble_tracker:
  scan_parameters:
    interval: 1100ms
    window: 1100ms
    active: true

bluetooth_proxy:
  active: true

## FONTS ##

font:
     
  - file: 'slkscr.ttf'
    id: font1
    size: 18

  - file: 'BebasNeue-Regular.ttf'
    id: font2
    size: 18

  - file: 'aerial.ttf'
    id: font3
    size: 8

## DISPLAY  OLED  ##

display:
  - platform: ssd1306_i2c
    model:  "SSD1306 128x64"
    reset_pin: 0
    address: 0x3C
    lambda: |-
       // Print "Alarm State: <state>" in top center
       it.printf(64, 0, id(font1), TextAlign::TOP_CENTER, id(alarm_state).state.c_str());
       // Print time in HH:MM format
       it.strftime(0, 60, id(font2), TextAlign::BASELINE_LEFT, "%Y-%m-%d     %I:%M%p", id(esptime).now());

## WEB SERVER ##
web_server:
     port: 80  
     pasword: !SECRET

## I2C ##
i2c:
     sda: 14
     scl: 15
     scan: true
     id: bus_a



sensor:
  
  - platform: aht10
    address: 0x38
    variant: AHT20
    temperature:
      name: "Temperature"
    humidity:
      name: "Humidity"
    update_interval: 60s
  
  - platform: adc
    pin: 39
    name: "Voltage Sensor"
    unit_of_measurement: "Volts"
    icon: "mdi:car-battery"
    accuracy_decimals: 2
    attenuation: 11dB
    update_interval: 10s 
    filters:
      - multiply:  2.0
      - lambda: |-
         if (x > 1) return x;
         else return {0};


  - platform: pulse_counter
    pin: 5
    name: PWM Fan RPM
    id: fan_pulse
    unit_of_measurement: 'RPM'
    filters:
      - multiply: 0.5
    count_mode:
      rising_edge: INCREMENT
      falling_edge: DISABLE
    update_interval: 3s


  - platform: uptime
    name: "Uptime"
    id: uptime_s
    update_interval: 60s

  - platform: adc
    id: aux_monitor
    name: Aux_Monitor
    attenuation: 11dB
    accuracy_decimals: 2
    update_interval: 10s
    pin: 36
    filters:
      multiply: 5.3

    

## PWM-FAN  12V ##
fan:
  - platform: speed
    output: fanhub_pwm
    name: "PWM Fan"



## MULTIPLEXER MCP23017##
mcp23017:
  - id: 'mcp23017_hub'
    address: 0x20
  - id: 'mcp23017_hub_2'
    address: 0x21
  - id: 'mcp23017_hub_3'
    address: 0x22    



time:
  - platform: homeassistant
    id: esptime

## SWITCH##
switch:
  - platform: gpio
    pin: 12
    name: SIREN
    id: SIREN  
    icon: "mdi:alarm-bell"
    inverted: False
    restore_mode: always_OFF

  - platform: gpio
    pin: 02
    name: AUX1
    id: STROBE1
    icon: "mdi:alarm-bell"
    inverted: False
    restore_mode: always_OFF


  - platform: gpio
    pin: 17
    name: AUX2
    id: TABLET
    icon: "mdi:alarm-bell"
    inverted: False
    restore_mode: always_OFF


  - platform: template
    name: "Door Chime"
    id: door_chime
    turn_on_action:
    - switch.template.publish:
        id: door_chime
        state: ON
    - output.turn_on: output_buzzer
    - output.ledc.set_frequency:
        id: output_buzzer
        frequency: "1000Hz"
    - output.set_level:
        id: output_buzzer
        level: "80%"
    - delay: 100ms
    - output.turn_off: output_buzzer
    - switch.template.publish:
        id: door_chime
        state: OFF


  - platform: template
    name: "Reminder Buzzer"
    id: reminder_buzzer
    turn_on_action:
    - script.stop: reminder_buzzer_script
    - output.turn_off: output_buzzer
    - script.execute: reminder_buzzer_script
    - switch.template.publish:
        id: reminder_buzzer
        state: ON
    turn_off_action:
    - script.stop: reminder_buzzer_script
    - output.turn_off: output_buzzer
    - switch.template.publish:
        id: reminder_buzzer
        state: OFF


  - platform: restart
    name: "Restart Switch"
    id: restart_switch
    internal: True 


output:

  - platform: ledc
    pin: 32
    id: output_buzzer


  
  - platform: ledc
    pin: 33
    frequency: 1000 Hz
    id: fanhub_pwm
  

script:
  - id: reminder_buzzer_script
    then:
    - while:
        condition:
          lambda: return true;
        then:
          - output.turn_on: output_buzzer
          - output.ledc.set_frequency:
              id: output_buzzer
              frequency: "1000Hz"
          - output.set_level:
              id: output_buzzer
              level: "80%"
          - delay: 1s
          - output.turn_off: output_buzzer
          - delay: 10s

## Individual inputs ##

binary_sensor:

  - platform: gpio
    name: "LVROOM MOTION"
    filters:
      - delayed_on: 10ms
    pin:
      mcp23xxx: mcp23017_hub
      # Use pin A7
      number: 07
      # One of INPUT or INPUT_PULLUP
      mode:
        input: true
        pullup: true
      inverted: false
    device_class: MOTION


  - platform: gpio
    name: "MAIN ENTRANCE MOTION"
    filters:
      - delayed_on: 10ms
    pin:
      mcp23xxx: mcp23017_hub
      # Use pin A6
      number: 06
      # One of INPUT or INPUT_PULLUP
      mode:
        input: true
        pullup: true
      inverted: false
    device_class: MOTION
   
   
  - platform: gpio
    name: "MASTER BED GLASS DOOR"
    filters:
      - delayed_on: 10ms
    pin:
      mcp23xxx: mcp23017_hub
      # Use pin A5
      number: 05
      # One of INPUT or INPUT_PULLUP
      mode:
        input: true
        pullup: true
      inverted: false
    device_class: DOOR  
  

  - platform: gpio
    name: "GARAGE SIDE DOOR"
    filters:
      - delayed_on: 10ms
    pin:
      mcp23xxx: mcp23017_hub
      # Use pin A4
      number: 04
      # One of INPUT or INPUT_PULLUP
      mode:
        input: true
        pullup: true
      inverted: false
    device_class: door
  

  - platform: gpio
    name: "LAUNDRY DOOR"
    filters:
      - delayed_on: 10ms
    pin:
      mcp23xxx: mcp23017_hub
      # Use pin A3
      number: 03
      # One of INPUT or INPUT_PULLUP
      mode:
        input: true
        pullup: true
      inverted:  false
    device_class: door    
  

  - platform: gpio
    name: "LVROOM GLASS DOOR"
    filters:
      - delayed_on: 10ms
    pin:
      mcp23xxx: mcp23017_hub
      # Use pin A2
      number: 02
      # One of INPUT or INPUT_PULLUP
      mode:
        input: true
        pullup: true
      inverted: false
    device_class: door
  

  - platform: gpio
    name: "MAIN DOOR"
    filters:
      - delayed_on: 10ms
    pin:
      mcp23xxx: mcp23017_hub
      # Use pin A1
      number: 01
      # One of INPUT or INPUT_PULLUP
      mode:
        input: true
        pullup: True
      inverted: false
    device_class: door


  - platform: gpio
    name: "TEST2"
    filters:
      - delayed_on: 10ms
    pin:
      mcp23xxx: mcp23017_hub
      # Use pin A0
      number: 00
      # One of INPUT or INPUT_PULLUP
      mode:
        input: true
        pullup: True
      inverted: false
    device_class: door 
 
  - platform: status
    name: "System Status"
 

 
 
