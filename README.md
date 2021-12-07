# home-assistant-config
Any useful Home Assistant Config I can share

## Tado - Regulate temperature using the offset
Like many people who enjoy home automation, Home Assistant and have a Tado system I have been wanting to do a little more to manage my Tado thermostats, especially my TRVs to have them register the real temp in the room. Therefore, I set about adding the ability to get and set the offset for Tado devices in Home Assistant. This was implemented in the 2021.2 release of Home Assistant

Now that we have access to the offset we can use that to keep the device temp in sync with another thermostat in the room, placed in a better location than on the radiator (like the Tado TRVs are)

To take advantage of this I wanted one automation to handle the whole thing and allow me the flexibility to have different settings per device should I choose. I've created said automation and am sharing here for others to use should they wish. Its below for ease or the file can be downloaded from the Tado directory

NB there is a back off setting but that requires a further code update to Tado integration I have written. I'll get a PR in for that, for now it just wont do anything for others (doesnt break anything just wont have any effect)

```yml
# Regulate tado heating equipment based on external sensors.

# For this to work you need to use a standard naming convention for every tado/external thermostat pairing. So each of them must be names the same i.e. 'bedroom'
# Then that should be followed by a consistent name space. I use the standard '_temperature' for tado and '_temperature_measurement' for external thermostat
# NB If you use something else you need to update the name space variables below

# I have made it so you can have a different settings for each room, just create a setting for it on that device (in the settings json variable)
# i.e. you could have the main house thermostat not have battery saver mode on as it doesnt have a motor to worry about so:
#          'house': {'battery_saver': false}
#      this will override the default battery_saver setting for just 'house' object_id
#
# WARNING: It seems every time the offset is updated the device will recalibrate so will open/close each time, this will drain the batteries faster.
#    the default tolerance is set at 0.3 here, which is quite low and will mean more calls to update offset, you may want to allow higher tolerance to help battery life
#    or use the battery_saver option (see conditions for explanation) or a combination of settings that suit your need
#
#   The automation is set as a queued automation with many triggers. It has a delay at the end to allow the Tado device and integration time to sync up
#   the same room could get triggered multiple times, this delay and queue allows time between each run and will prevent multiple offset updates
#   NB queue only queues the action section, hence having some conditions to cancel as soon as triggered if needed and also repeated in action

  - id: automation.regulate_heating
    alias: Regulate Heating
    mode: queued
    max: 25
    # List of all sensors to use to trigger (tado temp sensor and external temp sensor for each room)
    trigger:
    - platform: state
      entity_id:
        - sensor.house_temperature
        - sensor.house_temperature_measurement      
        - sensor.bedroom_temperature
        - sensor.bedroom_temperature_measurement      
        - sensor.guest_bedroom_temperature
        - sensor.guest_bedroom_temperature_measurement      
        - sensor.back_bedroom_temperature
        - sensor.back_bedroom_temperature_measurement      
    # Because I have a condition to only adjust offset when in auto, I also want to trigger when any are set back to auto
    - platform: state
      entity_id:
        - climate.house
        - climate.bedroom
        - climate.guest_bedroom
        - climate.back_bedroom
      to: auto 
    # Because I have a condition to only adjust offset when in home mode, I also want to trigger when I get home to immediately sync temp
    - platform: state
      entity_id: 
        - climate.house
        - climate.bedroom
        - climate.guest_bedroom
        - climate.back_bedroom
      attribute: preset_mode
      to: home
    # With one of the battery saver conditions being to only update offset if the change would impact hvac_action
    #   I want to trigger checking again if hvac_action changes, but only for ones with battery saver mode on
    - platform: state
      entity_id: 
        - climate.bedroom
        - climate.guest_bedroom
        - climate.back_bedroom
      attribute: hvac_action
   # Variables that can be set straight away and can be used in overall condition tests 
    variables:
        # You can configure your settings here, if settings don't exists for the device then 'default' is used
        # If you dont want different settings in any room, then you only need to set default thats it.
        settings: >
            {{ {
            'default': 
              {'tolerance': 0.3, 'battery_saver': true, 'back_off_secs': 900, 'climate_ns': '_temperature', 'sensor_ns': '_temperature_measurement',},
            'house': 
              {'battery_saver': false, 'tolerance': 0.2},
            } }}
        o_id: >
            {% for obj in settings %}
            {% set climate_ns = settings[obj].climate_ns if settings[obj].climate_ns is defined else settings['default'].climate_ns %}
            {% set sensor_ns = settings[obj].sensor_ns if settings[obj].sensor_ns is defined else settings['default'].sensor_ns %}
            {% if trigger.to_state.object_id == obj %}
            {{ trigger.to_state.object_id }}
            {% elif trigger.to_state.object_id == obj + climate_ns or trigger.to_state.object_id == obj + sensor_ns %}
            {{ obj }}
            {% endif %}
            {% endfor %}
        object_id: >
            {% if o_id != '' %}
            {{ o_id }}
            {% elif trigger.to_state.object_id.endswith(settings['default'].climate_ns) %}
            {{ trigger.to_state.object_id[:-(settings['default'].climate_ns|length)] }}
            {% elif trigger.to_state.object_id.endswith(settings['default'].sensor_ns) %}
            {{ trigger.to_state.object_id[:-(settings['default'].sensor_ns|length)] }}
            {% else %}
            {{ trigger.to_state.object_id }}
            {% endif %}
        tolerance: "{{ settings.get(object_id).tolerance if settings.get(object_id).tolerance is defined else settings.get('default').tolerance }}"
        back_off_secs: "{{ settings.get(object_id).back_off_secs if settings.get(object_id).back_off_secs is defined else settings.get('default').back_off_secs }}"
        climate_ns: "{{ settings.get(object_id).climate_ns if settings.get(object_id).climate_ns is defined else settings.get('default').climate_ns }}"
        sensor_ns: "{{ settings.get(object_id).sensor_ns if settings.get(object_id).sensor_ns is defined else settings.get('default').sensor_ns }}"
        battery_saver: "{{ settings.get(object_id).battery_saver if settings.get(object_id).battery_saver is defined else settings.get('default').battery_saver }}"
        climate_id: "{{ 'climate.' + object_id }}"
        climate_sensor_id: "{{ 'sensor.' + object_id + climate_ns }}"
        sensor_id: "{{ 'sensor.' + object_id + sensor_ns }}"
    
    # Conditions to check initially and cancel out the run if needed.
    # NB they will be recheck again in action in case anything changed while waiting to run
    #    so if you decide to remove any, make sure you remove in both
    condition:
    # Only run if temp states are a number i.e. hasn't become 'unavailable'
    - "{{ is_number(states(sensor_id)) }}"
    - "{{ is_number(states(climate_sensor_id)) }}"
    # Only run if the climate device is set to 'auto' and preset is 'home'
    - "{{ states(climate_id) == 'auto' and state_attr(climate_id, 'preset_mode') == 'home' }}"
    # Queuing only queues the action section so some variables need to be created here so caculated at relevant time
    action:
    - variables:
        offset: "{{ ((states(sensor_id)|float - states(climate_sensor_id)|float) + state_attr(climate_id, 'offset_celsius')|float)|round(1) }}"
        new_temp: "{{ state_attr(climate_id, 'current_temperature') - state_attr(climate_id, 'offset_celsius') + offset|float }}"
        last_updated: >
          {{ state_attr(climate_id, 'offset_last_changed') if state_attr(climate_id, 'offset_last_changed') != None else utcnow() - timedelta(seconds=(back_off_secs|int +100)) }}

    # All conditions are optional, so just delete the ones you don't want to use
    - condition: and
      conditions:
        # Only run if temp states are a number (is also check before queuing so if you remove, you need to remove from there too)
        - "{{ is_number(states(sensor_id)) }}"
        - "{{ is_number(states(climate_sensor_id)) }}"
        # Only run if the climate device is set to 'auto' and preset is 'home' (is also check before queuing so if you remove, you need to remove from there too)
        - "{{ states(climate_id) == 'auto' and state_attr(climate_id, 'preset_mode') == 'home' }}"
        # Only run if new_offset is different to current offset
        - "{{ state_attr(climate_id, 'offset_celsius')|float != offset|float }}"
        # Only run if there is a difference in temperature greater than the tolerance
        - "{{ (states(sensor_id)|float - states(climate_sensor_id)|float)|abs > tolerance }}"
        # BATTERY SAVER: The following are if battery saver is set to true for the device
        # Only run if offset hasn't been changed for longer than back_off
        - "{{ (as_timestamp(last_updated) < as_timestamp(utcnow() - timedelta(seconds=back_off_secs|int))) if battery_saver else true }}"
        # Only change the offset if it is going to make a difference to hvac action (so will send above or below target temp based on current hvac action state)
        - "{{ ((state_attr(climate_id, 'hvac_action') == 'heating' and new_temp|float >= state_attr(climate_id, 'temperature')|float) or (state_attr(climate_id, 'hvac_action') == 'idle' and new_temp|float <= state_attr(climate_id, 'temperature')|float)) if battery_saver else true }}"
        
    # Optional add it to logbook
    - service: logbook.log
      data:
          name: "{{ states[climate_id].name }}"
          message: >    
            is currently {{ state_attr(climate_id, 'hvac_action') }} with temp {{ states(climate_sensor_id) }} but room is {{ states(sensor_id) }}
            changing offset from {{ state_attr(climate_id, 'offset_celsius') }} to {{ offset }}
          entity_id: >
            {{ climate_id }}
          domain: climate
    # Set the offset
    - service: tado.set_climate_temperature_offset
      data:
        entity_id: "{{ climate_id}}"
        offset: "{{ offset }}"
    # Stop re triggers for a period allowing offset to have some effect and Tado Integration to refresh, new runs will be queued
    - delay: 60
```
