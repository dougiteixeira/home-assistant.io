---
title: Template
description: Instructions on how to integrate Template Sensors into Home Assistant.
ha_category:
  - Binary sensor
  - Button
  - Helper
  - Image
  - Number
  - Select
  - Sensor
ha_release: 0.12
ha_iot_class: Local Push
ha_quality_scale: internal
ha_codeowners:
  - '@PhracturedBlue'
  - '@tetienne'
  - '@home-assistant/core'
ha_domain: template
ha_platforms:
  - alarm_control_panel
  - binary_sensor
  - button
  - cover
  - fan
  - image
  - light
  - lock
  - number
  - select
  - sensor
  - switch
  - vacuum
  - weather
ha_integration_type: helper
ha_config_flow: true
---

The `template` integration allows creating entities which derive their values from other data. This is done by specifying [templates](/docs/configuration/templating/) for properties of an entity, like the name or the state.

## Platform specific documentation

   | Platform | Config |
   |---|---|
   | [Alarm control panel](/integrations/alarm_control_panel.template/) | YAML |
   | [Binary sensor](/integrations/binary_sensor.template/) | IU or YAML |
   | [Button](/integrations/button.template/) | YAML |
   | [Cover](/integrations/cover.template/) | YAML |
   | [Fan](/integrations/fan.template/) | YAML |
   | [Image](/integrations/image.template/) | YAML |
   | [Light](/integrations/light.template/) | YAML |
   | [Lock](/integrations/lock.template/) | YAML |
   | [Number](/integrations/number.template/) | YAML |
   | [Select](/integrations/select.template/) | YAML |
   | [Sensor](/integrations/sensor.template/) | IU or YAML |
   | [Switch](/integrations/switch.template/) | YAML |
   | [Vacuum](/integrations/vacuum.template/) | YAML |
   | [Weather](/integrations/weather.template/) | YAML |

{% include integrations/config_flow.md %}

{% important %}
To be able to add **{% my helpers title="Helpers" %}** via the user interface, you should have `default_config:` in your {% term "`configuration.yaml`" %}. It should already be there by default unless you removed it.
{% endimportant %}

{% note %}
Configuration using our user interface provides a more limited subset of options, making this integration more accessible while covering most use cases.

If you need more specific features for your use case, the manual YAML-configuration section each platform of this integration might provide them.
{% endnote %}

## Miscellaneous information

This section will present some general information that may apply to different platforms. To apply them, consult the platform's specific documentation.

#### Video tutorial
This video tutorial explains how to set up a Trigger based template that makes use of an action to retrieve the weather forecast (precipitation).

<lite-youtube videoid="zrWqDjaRBf0" videotitle="How to create Action Template Sensors in Home Assistant" posterquality="maxresdefault"></lite-youtube>

### Template and action variables

State-based and trigger-based template entities have the special template variable `this` available in their templates and actions. The `this` variable is the [state object](/docs/configuration/state_object) of the entity and aids [self-referencing](#self-referencing) of an entity's state and attribute in templates and actions. Trigger-based entities also provide [the trigger data](/docs/automation/templating/). 

### Rate limiting updates

When there are entities present in the template and no triggers are defined, the template will be re-rendered when one of the entities changes states. To avoid this taking up too many resources in Home Assistant, rate limiting will be automatically applied if too many states are observed.

{% tip %}
<a href='#trigger-based-template-sensors'>Define a trigger</a> to avoid a rate limit and get more control over entity updates.
{% endtip %}

When `states` is used in a template by itself to iterate all states on the system, the template is re-rendered each
time any state changed event happens if any part of the state is accessed. When merely counting states, the template
is only re-rendered when a state is added or removed from the system. On busy systems with many entities or hundreds of
thousands state changed events per day, templates may re-render more than desirable.

In the below example, re-renders are limited to once per minute because we iterate over all available entities:

{% raw %}

```yaml
template:
  - binary_sensor:
      - name: "Has Unavailable States"
        state: "{{ states | selectattr('state', 'in', ['unavailable', 'unknown', 'none']) | list | count }}"
```

{% endraw %}

In the below example, re-renders are limited to once per second because we iterate over all entities in a single domain (sensor):

{% raw %}

```yaml
template:
  - binary_sensor:
      - name: "Has Unavailable States"
        state: "{{ states.sensor | selectattr('state', 'in', ['unavailable', 'unknown', 'none']) | list | count }}"
```

{% endraw %}

If the template accesses every state on the system, a rate limit of one update per minute is applied. If the template accesses all states under a specific domain, a rate limit of one update per second is applied. If the template only accesses specific states, receives update events for specifically referenced entities, or the `homeassistant.update_entity` service is used, no rate limit is applied.

### Considerations

#### Startup

If you are using the state of a platform that might not be available during startup, the Template Sensor may get an `unknown` state. To avoid this, use the `states()` function in your template. For example, you should replace {% raw %}`{{ states.sensor.moon.state }}`{% endraw %} with this equivalent that returns the state and never results in `unknown`: {% raw %}`{{ states('sensor.moon') }}` {% endraw %}. 

The same would apply to the `is_state()` function. You should replace {% raw %}`{{ states.switch.source.state == 'on' }}`{% endraw %} with this equivalent that returns `true`/`false` and never gives an `unknown` result:

{% raw %}

```yaml
{{ is_state('switch.source', 'on') }}
```

{% endraw %}

## Examples

In this section, you find some real-life examples of how to use template sensors.

### Trigger based sensor and binary sensor storing webhook information

Template entities can be triggered using any automation trigger, including webhook triggers. Use a trigger-based template entity to store this information in template entities.

{% raw %}

```yaml
template:
  - trigger:
      - platform: webhook
        webhook_id: my-super-secret-webhook-id
    sensor:
      - name: "Webhook Temperature"
        state: "{{ trigger.json.temperature }}"
        unit_of_measurement: °C

      - name: "Webhook Humidity"
        state: "{{ trigger.json.humidity }}"
        unit_of_measurement: %

    binary_sensor:
      - name: "Motion"
        state: "{{ trigger.json.motion }}"
        device_class: motion
```

{% endraw %}

You can test this trigger entity with the following CURL command:

```bash
curl --header "Content-Type: application/json" \
  --request POST \
  --data '{"temperature": 5, "humidity": 34, "motion": true}' \
  http://homeassistant.local:8123/api/webhook/my-super-secret-webhook-id
```

### Turning an event into a trigger based binary sensor

You can use a trigger-based template entity to convert any event or other automation trigger into a binary sensor. The below configuration will turn on a binary sensor for 5 seconds when the automation trigger triggers.

```yaml
template:
  - trigger:
      platform: event
      event_type: my_event
    binary_sensor:
      - name: Event recently fired
        auto_off: 5
        state: "true"
```

### State based sensor exposing sun angle

This example shows the sun angle in the frontend.

{% raw %}

```yaml
template:
  - sensor:
      - name: Sun Angle
        unit_of_measurement: "°"
        state: "{{ '%+.1f'|format(state_attr('sun.sun', 'elevation')) }}"
```

{% endraw %}

### State based sensor modifying another sensor's output

If you don't like the wording of a sensor output, then the Template Sensor can help too. Let's rename the output of the [Sun integration](/integrations/sun/) as a simple example:

{% raw %}

```yaml
template:
  - sensor:
      - name: "Sun State"
        state: >
          {% if is_state('sun.sun', 'above_horizon') %}
            up
          {% else %}
            down
          {% endif %}
```

{% endraw %}

### State based sensor with multiline template with an `if` test

This example shows a multiple line template with an `if` test. It looks at a sensing switch and shows `on`/`off` in the frontend, and shows 'standby' if the power use is less than 1000 watts.

{% raw %}

```yaml
template:
  - sensor:
      - name: "Kettle"
        state: >
          {% if is_state('switch.kettle', 'off') %}
            off
          {% elif state_attr('switch.kettle', 'W')|float < 1000 %}
            standby
          {% elif is_state('switch.kettle', 'on') %}
            on
          {% else %}
            failed
          {% endif %}
```

{% endraw %}

### State based sensor changing the unit of measurement of another sensor

With a Template Sensor, it's easy to convert given values into others if the unit of measurement doesn't fit your needs.
Because the sensors do math on the source sensor's state and need to render to a numeric value, an availability template is used
to suppress rendering of the state template if the source sensor does not have a valid numeric state.

{% raw %}

```yaml
template:
  - sensor:
      - name: "Transmission Down Speed"
        unit_of_measurement: "kB/s"
        state: "{{ states('sensor.transmission_down_speed')|float * 1024 }}"
        availability: "{{ is_number(states('sensor.transmission_down_speed')) }}"

      - name: "Transmission Up Speed"
        unit_of_measurement: "kB/s"
        state: "{{ states('sensor.transmission_up_speed')|float * 1024 }}"
        availability: "{{ is_number(states('sensor.transmission_up_speed')) }}"
```

{% endraw %}

### State based binary sensor - Washing Machine Running

This example creates a washing machine "load running" sensor by monitoring an
energy meter connected to the washer. During the washer's operation, the energy meter will fluctuate wildly, hitting zero frequently even before the load is finished. By utilizing `delay_off`, we can have this sensor only turn off if there has been no washer activity for 5 minutes.

{% raw %}

```yaml
# Determine when the washing machine has a load running.
template:
  - binary_sensor:
      - name: "Washing Machine"
        delay_off:
          minutes: 5
        state: >
          {{ states('sensor.washing_machine_power')|float > 0 }}
```

{% endraw %}

### State based binary sensor - Is Anyone Home

This example is determining if anyone is home based on the combination of device tracking and motion sensors. It's extremely useful if you have kids/baby sitter/grand parents who might still be in your house that aren't represented by a trackable device in Home Assistant. This is providing a composite of Wi-Fi based device tracking and Z-Wave multisensor presence sensors.

{% raw %}

```yaml
template:
  - binary_sensor:
      - name: People home
        state: >
          {{ is_state('device_tracker.sean', 'home')
             or is_state('device_tracker.susan', 'home')
             or is_state('binary_sensor.office_124', 'on')
             or is_state('binary_sensor.hallway_134', 'on')
             or is_state('binary_sensor.living_room_139', 'on')
             or is_state('binary_sensor.porch_ms6_1_129', 'on')
             or is_state('binary_sensor.family_room_144', 'on') }}
```

{% endraw %}

### State based binary sensor - device tracker sensor with latitude and longitude attributes

This example shows how to combine a non-GPS (e.g., NMAP) and GPS device tracker while still including latitude and longitude attributes

{% raw %}

```yaml
template:
  - binary_sensor:
      - name: My Device
        state: >
          {{ is_state('device_tracker.my_device_nmap', 'home') or is_state('device_tracker.my_device_gps', 'home') }}
        device_class: "presence"
        attributes:
          latitude: >
            {% if is_state('device_tracker.my_device_nmap', 'home') %}
              {{ state_attr('zone.home', 'latitude') }}
            {% else %}
              {{ state_attr('device_tracker.my_device_gps', 'latitude') }}
            {% endif %}
          longitude: >
            {% if is_state('device_tracker.my_device_nmap', 'home') %}
              {{ state_attr('zone.home', 'longitude') }}
            {% else %}
              {{ state_attr('device_tracker.my_device_gps', 'longitude') }}
            {% endif %}
```

{% endraw %}

### State based binary sensor - Change the icon when a state changes

This example demonstrates how to use template to change the icon as its state changes. This icon is referencing its own state.

{% raw %}

```yaml
template:
  - binary_sensor:
      - name: Sun Up
        state: >
          {{ is_state("sun.sun", "above_horizon") }}
        icon: >
          {% if is_state("binary_sensor.sun_up", "on") %}
            mdi:weather-sunset-up
          {% else %}
            mdi:weather-sunset-down
          {% endif %}
```

{% endraw %}

A more advanced use case could be to set the icon based on the sensor's own state like above, but when triggered by an event. This example demonstrates a binary sensor that turns on momentarily, such as when a doorbell button is pressed. 

The binary sensor turns on and sets the matching icon when the appropriate event is received. After 5 seconds, the binary sensor turns off automatically. To ensure the icon gets updated, there must be a trigger for when the state changes to off. 

{% raw %}

```yaml
template:
  - trigger:
      - platform: event
        event_type: YOUR_EVENT
      - platform: state
        entity_id: binary_sensor.doorbell_rang
        to: "off"
    binary_sensor:
      name: doorbell_rang
      icon: "{{ (trigger.platform == 'event') | iif('mdi:bell-ring-outline', 'mdi:bell-outline') }}"
      state: "{{ trigger.platform == 'event' }}"
      auto_off:
        seconds: 5
```

{% endraw %}

### State based select - Control Day/Night mode of a camera

This show how a state based template select can be used to call a service.

{% raw %}


```yaml
template:
  select:
    - name: "Porch Camera Day-Night Mode"
      unique_id: porch_camera_day_night_mode
      state: "{{ state_attr('camera.porch_camera_sd', 'day_night_mode') }}"
      options: "{{ ['off', 'on', 'auto'] }}"
      select_option:
        - service: tapo_control.set_day_night_mode
          data:
            day_night_mode: "{{ option }}"
          target:
            entity_id: camera.porch_camera_sd
```

{% endraw %}

### Self referencing

This example demonstrates how the `this` variable can be used in templates for self-referencing.

{% raw %}

```yaml
template:
  - sensor:
      - name: test
        state: "{{ this.attributes.test | default('Value when missing') }}"
        # not: "{{ state_attr('sensor.test', 'test') }}"
        attributes:
          test: "{{ now() }}"
```

{% endraw %}

### Trigger based handling of service response data

This example demonstrates how to use an `action` to call a [service with response data](/docs/scripts/service-calls/#use-templates-to-handle-response-data)
and use the response in a template.

{% raw %}

```yaml
template:
  - trigger:
      - platform: time_pattern
        hours: /1
    action:
      - service: weather.get_forecasts
        data:
          type: hourly
        target:
          entity_id: weather.home
        response_variable: hourly
    sensor:
      - name: Weather Forecast Hourly
        unique_id: weather_forecast_hourly
        state: "{{ now().isoformat() }}"
        attributes:
          forecast: "{{ hourly['weather.home'].forecast }}"
```

{% endraw %}

## Event `event_template_reloaded`

Event `event_template_reloaded` is fired when Template entities have been reloaded and entities thus might have changed.

This event has no additional data.
