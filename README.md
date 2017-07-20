# HomeAssistant Xiaomi Hub Component by Rave (Lazcad)

Credits
---------------
Credits to the following Github project
- https://github.com/fooxy/homeassistant-aqara
- https://github.com/louisZL/lumi-gateway-local-api

Description
---------------
This is an complete Home Assistant component for Xiaomi Gateway. It allows you to integrate the following devices into HA

- Temperature and Humidity Sensor (old and new version)
- Motion Sensor (old and new version)
- Door and Window Sensor (old and new version)
- Button (old and new version)
- Plug aka Socket (ZigBee version, reports power consumed, power load, state and if device in use)
- Wall Plug (reports power consumed, power load and state)
- Aqara Wall Switch (Single)
- Aqara Wall Switch (Double)
- Aqara Wall Switch LN (Single)
- Aqara Wall Switch LN (Double)
- Aqara Wireless Switch (Single)
- Aqara Wireless Switch (Double)
- Cube
- Gas Leak Detector (reports alarm and density)
- Smoke Detector (reports alarm and density)
- Gateway (Light, Illumination Sensor, Ringtone play)
- Intelligent Curtain
- Battery

What's not available?

- Gateway Radio
- Gateway Button
- Water Sensor
- Aqara Air Conditioning Companion
- Aqara Intelligent Air Conditioner Controller Hub
- Decoupled mode of the Aqara Wall Switches (Single & Double)
- Additional alarm events of the Gas and Smoke Detector: Analog alarm, battery fault alarm (smoke detector only), sensitivity fault alarm, I2C communication failure

Installation (Raspberry Pi)
---------------------------

1. First, copy all the files into the Home Assistant location. It can now be installed either to the custom_components folder 
 ```
 /home/homeassistant/.homeassistant/custom_components
 ```
 or the root folder (using virtual environment)
 ```
 /srv/homeassistant/homeassistant_venv/lib/python3.4/site-packages/homeassistant/components
 ```

2. Add the following line to the configuration.yaml. Make sure you're on the latest firmware. You will need to get the Hub's key in order to issue command to the hub like turning on and off plug. Follow the steps here https://github.com/louisZL/lumi-gateway-local-api/blob/master/device_discover.md

 One Gateway
  ```yaml
 # You can leave sid empty if you only have one gateway
 xiaomi:
   gateways:
     - sid:
       key: xxxxxxxxxxxxxxxx
  ```

 Multiple Gateway
  ```yaml
 # 12 characters sid can be obtained from the gateway's MAC address.
 xiaomi:
   gateways:
     - sid: xxxxxxxxxxxx
       key: xxxxxxxxxxxxxxxx
     - sid: xxxxxxxxxxxx
       key: xxxxxxxxxxxxxxxx
  ```

3. Start HA. Pycrypto should install automatically. If not, install pycrypto manually. if you are using virtual environment, remember to install from virtual environment like below
 ```
 (homeassistant_venv) pi@raspberrypi:~ $ pip3 install pycrypto
 ```

4. Add friendly names to the Configuration.yaml like below
  ```yaml
    customize:
      binary_sensor.switch_158d000xxxxxc3:
        friendly_name: Kitchen Switch
      binary_sensor.switch_158d000xxxxxc2:
        friendly_name: Table Switch
      binary_sensor.door_window_sensor_158d000xxxxx7a:
        friendly_name: Door Sensor
  ```

5. Add automation. For the Button and Switch, use the following event. Available click types are 'single', 'double', 'hold', 'long_click_press' and 'long_click_release'. For door window sensor and motion sensor, you can use State as trigger
  ```yaml
  automation:
    # Trigger for the wireless button with different click types

    - alias: Toggle dining light on single press
      trigger:
        platform: event
        event_type: click
        event_data:
          entity_id: binary_sensor.switch_158d000xxxxxc2
          click_type: single
      action:
        service: switch.toggle
        entity_id: switch.wall_switch_left_158d000xxxxx01

    - alias: Toggle couch light on double click
      trigger:
        platform: event
        event_type: click
        event_data:
          entity_id: binary_sensor.switch_158d000xxxxxc2
          click_type: double
      action:
        service: switch.toggle
        entity_id: switch.wall_switch_right_158d000xxxxx01

    - alias: Let a dog bark on long press
      trigger:
        platform: event
        event_type: click
        event_data:
          entity_id: binary_sensor.switch_158d000xxxxxc2
          click_type: long_click_press
      action:
        service: xiaomi.play_ringtone
        data:
          gw_sid: xxxxxxxxxxxx
          ringtone_id: 8
          ringtone_vol: 8
  ```

  ```yaml
    # Trigger for motion sensor

    - alias: If there is motion and its dark turn on the gateway light
      trigger:
        platform: state
        entity_id: binary_sensor.motion_sensor_158d000xxxxxc2
        from: 'off'
        to: 'on'
      condition:
        condition: numeric_state
        entity_id: sensor.illumination_34ce00xxxx11
        below: 300
      action:
        service: light.turn_on
        entity_id: light.gateway_light_34ce00xxxx11
        data:
          brightness: 5

    - alias: If there no motion for 5 minutes turn off the gateway light
      trigger:
        platform: state
        entity_id: binary_sensor.motion_sensor_158d000xxxxxc2
        from: 'on'
        to: 'off'
        for:
          minutes: 5
      action:
        service: light.turn_off
        entity_id: light.gateway_light_34ce00xxxx11
  ```

  ```yaml
    # Trigger for door window sensor

    - alias: If the window is open turn off the radiator
      trigger:
        platform: state
        entity_id: binary_sensor.door_window_sensor_158d000xxxxxc2
        from: 'off'
        to: 'on'
      action:
        service: climate.set_operation_mode
        entity_id: climate.livingroom
        data:
          operation_mode: 'Off'

    - alias: If the window is closed for 5 minutes turn on the radiator again
      trigger:
        platform: state
        entity_id: binary_sensor.door_window_sensor_158d000xxxxxc2
        from: 'on'
        to: 'off'
        for:
          minutes: 5
      action:
        service: climate.set_operation_mode
        entity_id: climate.livingroom
        data:
          operation_mode: 'Smart schedule'
  ```

  ```yaml
    # Trigger for door window sensor

    - alias: Send notification on fire alarm
      trigger:
        platform: state
        entity_id: binary_sensor.smoke_sensor_158d0001574899
        from: 'off'
        to: 'on'
      action:
        - service: notify.html5
          data:
            title: Fire alarm!
            message: Fire/Smoke detected!
        - service: xiaomi.play_ringtone
          data:
            gw_sid: xxxxxxxxxxxx
            ringtone_id: 2
            ringtone_vol: 100
  ```

6. For Cube, use the following trigger. Available actions are flip90, flip180, move, tap_twice, shake_air, swing, alert, free_fall and rotate

 ```yaml
    trigger:
      platform: event
      event_type: cube_action
      event_data:
        entity_id: binary_sensor.cube_158d000xxxxxc2
        action_type: flip90
 ```

7. Important! Only use this if you have have issue with Socket binding or multicast. Using this when you have no problem with socket or mcast will introduce other issues. Add the IP address of the network interface to the config
 
 ```yaml
 xiaomi:
   gateways:
     - sid: xxxxxxxxxxxx
       key: xxxxxxxxxxxxxxxx
     - sid: xxxxxxxxxxxx
       key: xxxxxxxxxxxxxxxx
    interface: xx.xx.xx.xx
 ```
