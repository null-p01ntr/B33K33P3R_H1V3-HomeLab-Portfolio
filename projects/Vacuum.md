# Vacuum - Room-Cleaning Queue Subsystem

## Overview

The robot vacuum ships with its own app-driven cleaning order, but that order isn't
easy to see, change, or hook into other automations. This subsystem takes room order
away from the vacuum's own app and puts it under Home Assistant's control instead: a
text helper holds an ordered room list, one automation drains it one room at a time,
and a handful of other automations feed rooms into it rather than triggering a clean
directly. A second, smaller automation reuses the same room-tracking data to notify
when the vacuum is entering — or already in — whichever room the user is currently in.

Part of the [H1V3 M1ND](H1V3_M1ND.md) smart home platform.

## Features

- **Queue-driven cleaning order**: rooms are queued and drained one at a time by a
  single worker automation, instead of the vacuum picking its own order.
- **Multiple producers, one consumer**: away/empty-house detection, calendar events,
  and return-from-away scheduling all feed the same queue rather than each triggering
  a clean of its own.
- **Cleaning-hours guard**: a permitted time window is enforced independently of what
  triggered the clean, pausing and alerting if a job runs past it.
- **Presence-aware notification**: reuses the queue's own room-tracking data to tell
  the user when the vacuum is about to enter, or already in, their current room.

## Technologies Used

- *Home Assistant helpers* (`input_text`, `input_datetime`, template sensors) for
  queue storage and read-only tracking
- *YAML* scripts and automations for the queue operations and the worker loop
- *Jinja* templates to resolve a room name to the vacuum's own map segment index
- The vacuum integration's own segment-clean service, called directly rather than a
  plain start, so a *specific* room is cleaned rather than a full run

## Usage

### Queue Helper & Tracking Sensors - Helpers

The queue itself is a single comma-separated text helper. Two template sensors mirror
it for dashboards and tracking rather than requiring every consumer to parse the raw
text: one exposes the queue's current contents (and goes `unavailable` once it's
empty), the other exposes specifically the *next* room in line.

<details>
	<summary>Show YAML code</summary>

```yaml
# input_text helper -- the queue itself, "RoomA, RoomB, RoomC"
input_text:
  {{VACUUM_QUEUE_HELPER}}:
    name: Vacuum Clean Queue
    mode: text
    max: 255

# template sensors -- read-only mirrors of the queue, for dashboards/tracking
template:
  - sensor:
      - name: "Vacuum Clean Queue RO"
        availability: "{{ states.input_text.{{VACUUM_QUEUE_HELPER}}.state | count > 0 }}"
        state: "{{ states.input_text.{{VACUUM_QUEUE_HELPER}}.state }}"

      # the room after the one currently running
      - name: "Vacuum Next Room"
        state: "{{ states.input_text.{{VACUUM_QUEUE_HELPER}}.state.split(', ')[1] }}"
```
</details>

### Queue Add / Remove - Scripts

Two scripts are the only things allowed to touch the queue text directly. Adding
appends the given rooms to whatever is already queued; removing always drops the
*head* of the queue, never an arbitrary entry — which is what keeps the worker below
simple, since it only ever has to look at index 0.

<details>
	<summary>Show YAML code</summary>

```yaml
# script.{{VACUUM_QUEUE_ADD_SCRIPT}} -- "Vacuum - Add to Queue"
sequence:
  - variables:
      formatted_list: "{{ states.input_text.{{VACUUM_QUEUE_HELPER}}.state + ', ' + rooms | join(', ') }}"
  - action: input_text.set_value
    target:
      entity_id: input_text.{{VACUUM_QUEUE_HELPER}}
    data:
      value: "{{ formatted_list | regex_replace('^,\\s*', '') }}"
fields:
  rooms:
    name: Rooms
    required: true
    selector:
      select:
        multiple: true
        options: {{ROOM_OPTIONS}}
```

```yaml
# script.{{VACUUM_QUEUE_REMOVE_SCRIPT}} -- "Vacuum - Remove from Queue"
# always pops the head -- there is no "remove this specific room" path
sequence:
  - action: input_text.set_value
    target:
      entity_id: input_text.{{VACUUM_QUEUE_HELPER}}
    data:
      value: "{{ states.input_text.{{VACUUM_QUEUE_HELPER}}.state.split(', ')[1:] | join(', ') }}"
```
</details>

### Clean Room - Scripts

These are the layer that actually talks to the vacuum. The room name is resolved to
the vacuum's own map segment index by matching it against the `select` helpers the
integration exposes per mapped room (`{{VACUUM_ROOM_INDEX_SELECT_PATTERN}}`), then
`{{VACUUM_ENTITY}}` is sent that segment through the vacuum integration's own
segment-clean service rather than a plain start. A single-room and a multi-room
variant exist; the multi-room one is the more complete of the two — it actually
applies the chosen suction level and, when a water level is set, the water volume,
where the single-room version currently only ever sends the segment list.

<details>
	<summary>Show YAML code</summary>

```yaml
# script.{{VACUUM_CLEAN_ROOM_SCRIPT}} -- "Vacuum - Clean Room" (single room)
sequence:
  - variables:
      room_indexes: >
        {% set matching = states.select
          | selectattr('entity_id', 'match', '{{VACUUM_ROOM_INDEX_SELECT_PATTERN}}')
          | selectattr('state', 'equalto', room)
          | map(attribute='name') | list %}
        [{{ matching[0].split(" ")[2] | int }}]
  - delay: "{{ delay_amount }}"
  - action: {{VACUUM_DOMAIN}}.vacuum_clean_segment
    target:
      entity_id: {{VACUUM_ENTITY}}
    data:
      segments: "{{ room_indexes }}"
fields:
  room: { name: Room, required: true, selector: { text: {} } }
  fan_speed:
    name: Fan Speed
    required: true
    selector: { select: { options: [Silent, Standard, Strong, Turbo] } }
  water_level:
    name: Water Level
    default: 0
    selector: { number: { min: 0, max: 3, step: 1 } }
  delay_amount:
    name: Delay Amount
    default: "00:00:00"
    selector: { time: {} }
```

```yaml
# script.{{VACUUM_CLEAN_ROOMS_SCRIPT}} -- "Vacuum - Clean Room(s)" (multi-room, applies suction + water)
sequence:
  - variables:
      suction_level: >
        {% set levels = ['Silent','Standard','Strong','Turbo'] %}
        {{ levels.index(fan_speed) }}
  - variables:
      room_indexes: >
        {% set data = namespace(room_ints=[]) %}
        {% for room in rooms %}
          {% set matching = states.select
            | selectattr('entity_id', 'match', '{{VACUUM_ROOM_INDEX_SELECT_PATTERN}}')
            | selectattr('state', 'equalto', room)
            | map(attribute='name') | list %}
          {% set data.room_ints = data.room_ints + [matching[0].split(" ")[2] | int] %}
        {% endfor %}
        {{ data.room_ints }}
  - delay: "{{ delay_amount }}"
  - action: vacuum.set_fan_speed
    target:
      entity_id: {{VACUUM_ENTITY}}
    data:
      fan_speed: "{{ fan_speed }}"
  - if:
      - condition: template
        value_template: "{{ water_level > 0 }}"
    then:
      - action: {{VACUUM_DOMAIN}}.vacuum_clean_segment
        target:
          entity_id: {{VACUUM_ENTITY}}
        data:
          suction_level: "{{ suction_level }}"
          water_volume: "{{ water_level }}"
          segments: "{{ room_indexes }}"
    else:
      - action: {{VACUUM_DOMAIN}}.vacuum_clean_segment
        target:
          entity_id: {{VACUUM_ENTITY}}
        data:
          suction_level: "{{ suction_level }}"
          segments: "{{ room_indexes }}"
fields:
  rooms:
    name: Rooms
    required: true
    selector: { select: { multiple: true, options: {{ROOM_OPTIONS}} } }
  fan_speed: { name: Fan Speed, required: true, selector: { select: { options: [Silent, Standard, Strong, Turbo] } } }
  water_level: { name: Water Level, default: 0, selector: { number: { min: 0, max: 3, step: 1 } } }
  delay_amount: { name: Delay Amount, default: "00:00:00", selector: { time: {} } }
```
</details>

### Queue Worker - Automation

One automation is the only thing that ever starts a room clean from the queue. It
triggers on the queue text changing *or* on the cleaning-hours-start time, holds off
if it's outside the permitted cleaning window (`{{VACUUM_CLEAN_HOURS_START}}` /
`{{VACUUM_CLEAN_HOURS_STOP}}`), takes the head of the queue, runs the single-room
clean script against it, waits for the vacuum to report `returning` (job done for
that room), pops the head, and repeats until the queue is empty. `mode: restart` so a
queue edit while a room is already running re-evaluates from the new head on the next
cycle rather than stacking a second worker.

<details>
	<summary>Show YAML code</summary>

```yaml
trigger:
  - platform: state
    entity_id: input_text.{{VACUUM_QUEUE_HELPER}}
  - platform: time
    at: input_datetime.{{VACUUM_CLEAN_HOURS_START}}
condition:
  - condition: template
    value_template: "{{ states.input_text.{{VACUUM_QUEUE_HELPER}}.state | count > 0 }}"
action:
  - alias: "Wait for cleaning hours"
    if:
      - condition: time
        after: input_datetime.{{VACUUM_CLEAN_HOURS_STOP}}
        before: input_datetime.{{VACUUM_CLEAN_HOURS_START}}
    then:
      - action: persistent_notification.create
        data: { notification_id: waiting, title: "Vacuum - Clean Empty House", message: "Waiting for daytime" }
      - wait_for_trigger:
          - trigger: template
            value_template: >
              {{ now().time() > strptime(states('input_datetime.{{VACUUM_CLEAN_HOURS_START}}'), '%H:%M:%S').time() }}
        continue_on_timeout: false
      - action: persistent_notification.dismiss
        data: { notification_id: waiting }
  - variables:
      queue_head: "{{ states.input_text.{{VACUUM_QUEUE_HELPER}}.state.split(', ')[0] }}"
  - action: script.{{VACUUM_CLEAN_ROOM_SCRIPT}}
    data:
      room: "{{ queue_head }}"
      fan_speed: Standard
      water_level: 1
      delay_amount: "00:00:00"
  - alias: "Wait for cleanup finished"
    wait_for_trigger:
      - trigger: state
        entity_id: {{VACUUM_ENTITY}}
        to: returning
    continue_on_timeout: false
  - action: script.{{VACUUM_QUEUE_REMOVE_SCRIPT}}
  - if:
      - condition: template
        value_template: "{{ states.input_text.{{VACUUM_QUEUE_HELPER}}.state | count == 0 }}"
    then:
      - action: persistent_notification.create
        data: { message: "Cleaning queue finished" }
mode: restart
```
</details>

### Queue Producers - Automations

Nothing else calls the vacuum's clean service directly. Three separate automations
feed rooms into the same queue by calling `{{VACUUM_QUEUE_ADD_SCRIPT}}`, and the
worker above is the single place that turns a queued room into an actual clean — the
same decoupling the notify layer applies to notifications (see the
[H1V3 M1ND](H1V3_M1ND.md) page's state-model notes): *what decides a clean should
happen* stays separate from *how a clean happens*.

- **Away/empty-house cleaning** — gated on `{{HOME_MODE_SENSOR}}`: a sustained switch
  to `{{AWAY_MODE}}` (if enough time has passed since the last clean) or a full day in
  `{{EMPTY_MODE}}` queues the standard room set; on the `{{EMPTY_MODE}}` path it
  instead schedules a calendar reminder for the day before return, rather than
  cleaning immediately.
- **Calendar-scheduled cleaning** — a calendar event whose body names `Cleanup` adds
  either the rooms listed in the event description, or a default room set, to the
  queue when the event starts.
- **Return cleaning** — a time-templated trigger keyed off how long the house has
  been empty adds a fixed room set ahead of the expected return.
- **Cleaning-hours guard** — a separate automation watches
  `{{VACUUM_CLEAN_HOURS_STOP}}`; if the vacuum is still `cleaning` at that time it
  pauses the job and alerts, rather than letting it run into the night.

<details>
	<summary>Show YAML code — a producer feeding the queue (representative pattern)</summary>

```yaml
# shared pattern across all three producers above -- only the trigger and room list differ
trigger:
  - platform: state
    entity_id: {{HOME_MODE_SENSOR}}
    to: {{AWAY_MODE}}
    for: { minutes: 2 }
condition: []
action:
  - if:
      - condition: state
        entity_id: {{HOME_MODE_SENSOR}}
        state: [{{EMPTY_MODE}}, {{AWAY_MODE}}]
    then:
      - action: script.{{VACUUM_QUEUE_ADD_SCRIPT}}
        data:
          rooms: {{ROOM_LIST_EXAMPLE}}
mode: single
```
</details>

### Vacuum Follows You - Automation

Reuses the same room telemetry the queue system already tracks, rather than adding a
new sensor, to notify when the vacuum is entering — or already in — whatever room the
user is currently in. It cross-references two different sources: the vacuum
integration's own *current* room (native telemetry, not queue-derived) and the *next*
room template sensor defined above (which *is* queue-derived — it's just index 1 of
the same text helper the worker drains), matching either one against
`{{ROOM_LOCATION_SENSOR}}`, the room-inference entity documented on the
[H1V3 M1ND](H1V3_M1ND.md) page. It only fires while home alone and the vacuum is
actually mid-job.

<details>
	<summary>Show YAML code</summary>

```yaml
trigger:
  - trigger: template
    value_template: >
      {{ states.sensor.{{VACUUM_CURRENT_ROOM_SENSOR}}.state.replace(' ','_') | lower ==
         states('{{ROOM_LOCATION_SENSOR}}').replace(' ','_') | lower or
         states.sensor.{{VACUUM_NEXT_ROOM_SENSOR}}.state.replace('Master ', '').replace(' ','_') | lower ==
         states('{{ROOM_LOCATION_SENSOR}}').replace(' ','_') | lower }}
condition:
  - condition: state
    entity_id: {{HOME_MODE_SENSOR}}
    state: {{ALONE_MODE}}
  - condition: or
    conditions:
      - condition: state
        entity_id: {{VACUUM_ENTITY}}
        state: cleaning
      - condition: state
        entity_id: {{VACUUM_ENTITY}}
        state: returning
action:
  - action: script.notify_targets
    data:
      targets: {{NOTIFY_TARGETS}}
      notification_message: "Vacuum will clean `{{ states('{{ROOM_LOCATION_SENSOR}}') }}`"
mode: single
```
</details>

[Back to H1V3 M1ND](H1V3_M1ND.md)
