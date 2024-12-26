# H1V3 M1ND - Smart Home Platform

## Overview

	Description

![icon](../img/icons/h1v3-m1nd.png)

## Features


- **Automated Device Configurations**: Modify setting of devices on the network based on other events on other devices such as:
	- Device battery low
  - Phone ringing
  - User entered an online meeting
  - User changed room
  - Certain media type being played (movie or music)
- **Calendar Event Tracker**: Change device settings on the network based on calendar events, such as:
	- Changing arm mode
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
				entity_id: {{MEDIA_PLAYER_SENSOR}}
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
              entity_id: {{SPOTIFY_MEDIA_PLAYER_SENSOR}}
            action: media_player.select_source
	# REPEAT FOR DESIRED ROOMS/DEVICES
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
