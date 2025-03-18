# H1V3 M1ND - Smart Home Platform

## Overview

This project is an instance of Home Assistant - Smart Home Management platform. I have implemented advanced home automation systems to enhance convenience, and energy efficiency within my home. dDeveloped custom automations and a variety of sensors to create a seamless and intelligent smart home experience.  

![icon](../img/icons/h1v3-m1nd.png)

## Features


- **Automated Device Configurations**: Modify setting of devices on the network based on other events on other devices such as:
	- Alert owner for low ba
  - Phone ringing
  - User entered an online meeting
  - User changed room
  - Certain media type being played (movie or music)
- **Calendar Event Tracker**: Change device settings on the network based on calendar events, such as:
	- Changing arm mode
	- Run power saving routines
	- Adjust for other users and guests.
	- Notify user for weather info
- **Automated Lights**: Lights at home automatically turns on or off based on:
	- User's current room
	- Weather (changes brightness according to cloud coverage)
	- Before Sunset
- **Power saving Solutions**: Regularly track and power off unnecessary devices  based on usage and home arm mode. Notify user if power off failed or not available.
  
## Technologies Used

- *[Home Assistant](https://www.home-assistant.io/)*, open sourced base of smart home platform 
- *Yaml and Jinja* for defining automations and templating custom sensors
- *Docker* for maintaining service status, updates and backups

## Usage

Automations are developed using the YAML language by defining triggers, conditions, and actions, utilizing built-in variables known as sensors. For more complex sensors or those that depend on multiple devices, Jinja templates can be used to define custom sensors.


### Media Playback Room Transfer - Automation
If user changes room while media is playing, the media playback is transferred to the devices that are available at users current room.

<details>
	<summary>Show YAML code</summary>

```yaml
trigger:
  - platform: state
    entity_id:
  - {{ROOM_LOCATION_SENSOR}}
condition:
  - condition: and
    conditions:
      - condition: state
        entity_id: {{MEDIA_PLAYER_ENTITY}}
        state: playing
action:
  - choose:
	- conditions:
 	 - condition: state
    entity_id: {{ROOM_LOCATION_SENSOR}}
    state: {{DESIRED ROOM}}
sequence:
  - wait_for_trigger:
      {{DEVICE AVAILABILITY}}
    data:
      source: {{DEVICE NAME AT NEW ROOM}}
    target:
      entity_id: {{MEDIA_PLAYER_ENTITY}}
    action: media_player.select_source
    # REPEAT FOR DESIRED ROOMS/DEVICES
```
</details>

### Meeting Mode - Automation

When user enters a meeting, phone is set to meeting mode.

<details>
	<summary>Show YAML code</summary>

```yaml
trigger:
  - platform: state
    entity_id:
      [{MICROPHONE OR CAMERA ENTITIES}]
    to:
      - chrome
      - Zoom
      - Teams
      - {{MEETING PROGRAMS}}
action:
  - metadata: {}
    data:
      message: command_dnd
    action: notify.{{MOBILE_DEVICE}}
```
</details>

### Charge Handler - Automation
Notifies user for low battery devices on the network, starts charging them if connected charger available.

<details>
	<summary>Show YAML code</summary>

```yaml
trigger:
  - platform: numeric_state
    entity_id:
      - {{BATTERY_LEVEL_SENSOR}}
    above: {{UPPER % LIMIT}}
  - platform: numeric_state
    entity_id:
      - {{BATTERY_LEVEL_SENSOR}}
    below: {{LOWER % LIMIT}}
action:
- choose:
    - conditions:
      - condition: numeric_state
        entity_id: {{BATTERY_LEVEL_SENSOR}}
        below: {{LOWER % LIMIT}}
      sequence:
        - metadata: {}
          data: {}
          action: switch.turn_on
          target:
            entity_id: {{CHARGER_DEVICE}}
        action: {{NOTIFICATION_SENSOR}}
        - data: { {{NOTIFICATION DETAILS}} }

      - conditions:
        - condition: numeric_state
              entity_id: {{BATTERY_LEVEL_SENSOR}}
              above: {{UPPER % LIMIT}}
        sequence:
          - metadata: {}
            data: {}
            action: switch.turn_off
            target:
              entity_id: {{CHARGER_DEVICE}}
```
</details>

### Phone Ringing - Automation

When user's phone rings, any playing media is paused. Playback continues when the call's over.

<details>
	<summary>Show YAML code</summary>

```yaml
trigger:
  - platform: state
    entity_id:
      - {{PHONE_RING_ENTITY}}
    from: idle
    to:
      - ringing
      - offhook
action:
  - choose:
      - conditions:
          - condition: state
            entity_id: {{MEDIA_PLAYER_ENTITY}}
            state: playing
        sequence:
          - metadata: {}
            data: {}
            target:
              entity_id: {{MEDIA_PLAYER_ENTITY}}
            action: media_player.media_pause
          - wait_for_trigger:
              - platform: state
                entity_id:
                  - {{PHONE_ENTITY}}
                to: idle
            continue_on_timeout: false
          - action: media_player.media_play
            metadata: {}
            data: {}
            target:
              entity_id: {{MEDIA_PLAYER_ENTITY}}
```
</details>

### Gradually Change Brightness - Script

Dim or brighten up certain smart light over time. Perfect for wake up and going to sleep.

<details>
	<summary>Show YAML code</summary>

```yaml
sequence:
  - metadata: {}
    data:
      brightness_pct: 30
      rgb_color:
        - 255
        - 149
        - 0
    target:
      entity_id: {{LIGHT_ENTITY}}
    action: light.turn_on
  - delay:
      hours: 0
      minutes: 0
      seconds: 5
      milliseconds: 0
  - repeat:
      sequence:
        - delay:
            hours: 0
            minutes: 3
            seconds: 0
            milliseconds: 0
        - metadata: {}
          data:
            brightness_step_pct: -/+2 # - FOR DIM, + FOR BRIGHTEN 
          target:
            entity_id: {{LIGHT_ENTITY}}
          action: light.turn_on
      until:
        - condition: or
          conditions:
            - condition: numeric_state
              entity_id: {{LIGHT_ENTITY}}
              attribute: brightness
              below: 5  # FOR DIM
              above: 95 # FOR BRIGHTEN
  - action: light.turn_off
    metadata: {}
    data: {}
    target:
      entity_id: {{LIGHT_ENTITY}}
```
</details>

### Notify Multiple Devices - Script

Easily notify desired devices with single activity. Keeps recurring notification formats organized.

<details>
	<summary>Show YAML code</summary>

```yaml
sequence:
  - repeat:
      sequence:
        - action: notify.mobile_app_{{ device_attr( repeat.item , 'name') | lower }}
          metadata: {}
          data:
            title: "{{ notification_title }}"
            message: "{{ notification_message }}"
            data: "{{ notification_data }}"
      for_each: "{{ targets }}"
fields:
  targets:
    selector:
      device:
        multiple: true
    name: Targets
    required: true
    description: target notification devices
  notification_message:
    selector:
      text: null
    name: Notification Message
    required: true
  notification_data:
    selector:
      text: null
    name: Notification Data
  notification_title:
    selector:
      text: null
    name: Notification Title
alias: Notify - Targets
```
</details>

### Portable Drive Location - Sensor

Certain portable drive's last plugged device. Can be used for backup automations.

<details>
	<summary>Show Jinja template</summary>

```python
{% set Device1_Drives = states('{{PLUGGED DRIVE LIST AT DEVICE 1}}') %}
{% set Device2_Drives = states('{{PLUGGED DRIVE LIST AT DEVICE 2}}') %}
# REPEAT FOR POSSIBLE DEVICES

{% if '{{DRIVE NAME}}' in Device1_Drives %}
  Device1
{% elif '{{DRIVE NAME}}' in Device2_Drives %}
  Device2
# REPEAT FOR POSSIBLE DEVICES
{% else %}
  {{ this.state }}
{% endif %}
```
</details>

### PC Mode - Sensor

Keep track of a PCs custom use case to trigger automations, and configure arm modes.

<details>
	<summary>Show Jinja template</summary>

```python
{% set user = states('{{PC_USER_SENSOR}}') %}
{% set window = states('{{PC_ACTIVE_WINDOW_SENSOR}}')%}

{% if user == '{{GAMING USERNAME}}' %}
  gaming
  # ADD ELIF FOR POSSIBLE WINDOWS
{% elif user == '{{GUEST USERNAME}}' %}
  guest
  # ADD ELIF FOR POSSIBLE WINDOWS
{% elif user == '{{MAIN USERNAME}}' %}
  {% if 'Visual Studio Code' in window %}
    development
  {% elif 'Studio One' in window %}
    recording
  {% elif 'DaVinci Resolve' in window %}
    video_edit
  {% elif 'company name' in window %}
    working
  # REPEAT FOR POSSIBLE WINDOWS
  {% else %}
    {{ this.state }}
  {% endif %}
{% else %}
    {{ this.state }}
{% endif %}
```
</details>

[Other projects on H1V3](../README.md)
