# Spoolman Updater API

## Overview

The Spoolman Updater API provides endpoints to manage spool updates, including tracking filament usage and material details. This API is designed to work with Home Assistant and other automation systems.

To facilitate API development and testing, the Spoolman Updater API utilizes Swagger for interactive API documentation. You can access the Swagger UI at http://<your-server>:8088/swagger, which allows you to explore and test the available endpoints.

> [!TIP]
> The new UI add abilities to set which spool is in which tray of the AMS. Also there is a scan button (top right) that allows you to scan a barcode/qrcode on a spool and that will lead to a page where you can set in which tray the spool is.

## PLEASE READ ALL THE INSTRUCTIONS, THERE IS A DIFFERENCE BASED ON THE PRINTER YOU HAVE

## Base URL

```
http://<your-server>:8088
```

## Environment Variables

The API requires the following environment variables to be set:

| Variable Name                                | Type          | Example                             | Description                                      |
| -----------------                            | ------------- | ----------------------------------- | ------------------------------------------------ |
| APPLICATION__HOMEASSISTANT__URL              | string        | <https://192.169.1.1:8123>            | The URL to Home Assistant, with portnumber       |
| APPLICATION__HOMEASSISTANT__TOKEN            | string        |                                     | The Home Assistant Long-lived access token [more info](https://community.home-assistant.io/t/how-to-get-long-lived-access-token/162159/5?u=marcokreeft87)  Don't forget the quotes!   |
| APPLICATION__HOMEASSISTANT__AMSENTITIES__0   | string        | X1C_00xxxxxxxxxxxxx_AMS_1           | The Device ID of your AMS, when there are multiples AMS in your configuration just add another var and replace the _0 with_1 and so on       |
| APPLICATION__HOMEASSISTANT__TRAYENTITIES__0   | string        | X1C_00xxxxxxxxxxxxx_AMS_1_tray_1          | The tray sensors of your AMS trays. If you want to use this, remove APPLICATION__HOMEASSISTANT__AMSENTITIES or leave empty. Same as in AMSENTITIES replace __0 with 1 and so on for more tray sensors    |
| APPLICATION__HOMEASSISTANT__EXTERNALSPOOLENTITY | string        | sensor.x1x_externalspool_external_spool | The URL to Home Assistant, with portnumber       |
| APPLICATION__SPOOLMAN__URL                   | string        | <https://192.169.1.1:7912>             | The URL to Spoolman, with portnumber       |

## Running with Docker

### **Run the container**

```
docker run -d -p 8088:8080 \
  -e APPLICATION__HOMEASSISTANT__URL=http://homeassistant.local:8123 \
  -e APPLICATION__HOMEASSISTANT__TOKEN=your-token \
  -e APPLICATION__SPOOLMAN__URL=http://spoolman.local:7912 \
  -e APPLICATION__HOMEASSISTANT__AMSENTITIES__0=x1c_ams_1 \
  -e APPLICATION__HOMEASSISTANT__EXTERNALSPOOLENTITY=sensor.x1c_external_spool \
  --name spoolman-updater spoolman-updater
```

or

```
docker run -d -p 8088:8080 \
  -e APPLICATION__HOMEASSISTANT__URL=http://homeassistant.local:8123 \
  -e APPLICATION__HOMEASSISTANT__TOKEN=your-token \
  -e APPLICATION__SPOOLMAN__URL=http://spoolman.local:7912 \
  -e APPLICATION__HOMEASSISTANT__TRAYENTITIES__0=sensor.x1c_ams_1_tray_1 \
  -e APPLICATION__HOMEASSISTANT__TRAYENTITIES__1=sensor.x1c_ams_1_tray_2 \
  -e APPLICATION__HOMEASSISTANT__TRAYENTITIES__2=sensor.x1c_ams_1_tray_3 \
  -e APPLICATION__HOMEASSISTANT__TRAYENTITIES__3=sensor.x1c_ams_1_tray_4 \
  -e APPLICATION__HOMEASSISTANT__EXTERNALSPOOLENTITY=sensor.x1c_external_spool \
  --name spoolman-updater spoolman-updater
```

## Using with Home Assistant

The Spoolman Updater API can be integrated into Home Assistant automations to track filament usage automatically.

# MAKE SURE YOU RUN AT LEAST BAMBU LAB INTEGRATION 2.1.9 AND ABOVE, OTHERWISE IT WILL NOT WORK. If you encounter issues, please use the community forum or create an issue.


# ***P1S/X1C (maybe the H2D) This will update Spoolman with every tray change during Multi Color Printing**

This counts for the P1S and most likely for the X1C also:

### **1. Define a REST Command and Utility Meter in `configuration.yaml`**

Add the following to your `configuration.yaml` to create a REST command that updates the spool:

```yaml
utility_meter:
  bambulab_filament_usage_meter:
    unique_id: 148d1e2d-87b2-4883-a923-a36a2c9fa0ac
    source: sensor.bambulab_filament_usage
    cycle: weekly

rest_command:
 update_spool:
    url: "http://192.168.1.192:8088/Spools"
    method: POST
    headers:
      Content-Type: "application/json"
    payload: >
      {
        "name": "{{ filament_name }}",
        "material": "{{ filament_material }}",
        "tag_uid": "{{ filament_tag_uid }}",
        "used_weight": {{ filament_used_weight | int }},
        "color": "{{ filament_color }}",
        "active_tray_id": "{{ filament_active_tray_id }}"
      }


```

### **2. Create the sensors. Below example is when you have a dedicated sensors.yaml. If you don't have that make sure you use the correct indentations** 

```yaml
#BambuLab
- platform: template
  sensors:
    bambulab_filament_usage:
      unique_id: b954300e-d3a2-44ab-948f-39c30b2f0c00
      friendly_name: "Bambu Lab Filament Usage"
      value_template: >
        {{ states('sensor.ID_PRINTER_print_weight') | float(0) / 100 *
           states('sensor.ID_PRINTER_print_progress') | float(0) }}
      availability_template: >-
        {% if is_state('sensor.ID_PRINTER_print_weight', 'unknown')
           or is_state('sensor.ID_PRINTER_print_weight', 'unavailable') %}
          false
        {% else %}
          true
        {% endif %}

    bambulab_ams1_tray_index:
      friendly_name: "Bambu Lab AMS1 Tray Index"
      value_template: >-
        {% set trays = [1,2,3,4] %}
        {% for tray in trays %}
          {% set eid = 'sensor.ID_PRINTER_ams_1_tray_' ~ tray %}
          {% if state_attr(eid, 'active') in [true, 'true', 'True'] %}
            {{ tray }}
            {% break %}
          {% endif %}
        {% endfor %}
      availability_template: >-
        {{ expand([
          'sensor.ID_PRINTER_ams_1_tray_1',
          'sensor.ID_PRINTER_ams_1_tray_2',
          'sensor.ID_PRINTER_ams_1_tray_3',
          'sensor.ID_PRINTER_ams_1_tray_4'
        ]) | selectattr('attributes.active','defined') | list | count > 0 }}
```
### *** NOTE: If you have more then one AMS, you need to state them into the tray_index sensor.

### **3. Create an Automation. Below is the automation including log write for easy troubleshooting. Place this automation in your automations.yaml. You can always change the name that is logged into the log, it is now in Dutch :)**

```yaml
alias: Bambulab - Update Spool
description: Update spool
triggers:
  - entity_id: sensor.ID_PRINTER_active_tray_index
    id: tray
    platform: state

  - entity_id: sensor.ID_PRINTER_current_stage
    id: print_end
    platform: state
    to:
      - finished
      - idle

variables:
  old_tray: >-
    {% if trigger.id == 'tray'
       and trigger.from_state is not none
       and trigger.from_state.state not in ['', 'unknown', 'unavailable', None] %}
       {{ trigger.from_state.state | int }}
    {% else %}
       None
    {% endif %}

  tray_number: >-
    {% if trigger.id == 'print_end' %}
      {% set t = states('sensor.ID_PRINTER_active_tray_index') %}
      {% if t not in ['', 'unknown', 'unavailable', None] %}
        {{ t | int }}
      {% else %}
        None
      {% endif %}
    {% else %}
      {{ old_tray }}
    {% endif %}

  tray_sensor: >-
    {% if tray_number and tray_number > 0 %}
      sensor.ID_PRINTER_ams_1_tray_{{ tray_number }}
    {% else %}
      None
    {% endif %}

  tray_weight: "{{ states('sensor.ID_PRINTER_filament_usage_meter') | float(0) | round(2) }}"
  tag_uid: "{{ state_attr(tray_sensor, 'tag_uid') if tray_sensor else None }}"
  material: "{{ state_attr(tray_sensor, 'type') if tray_sensor else None }}"
  name: "{{ state_attr(tray_sensor, 'name') if tray_sensor else None }}"
  color: "{{ state_attr(tray_sensor, 'color') if tray_sensor else None }}"

action:
  - choose:

      - conditions:
          - condition: template
            value_template: "{{ trigger.id == 'tray' and old_tray is not none and old_tray > 0 }}"
        sequence:
          - service: rest_command.update_spool
            data:
              filament_name: "{{ name }}"
              filament_material: "{{ material }}"
              filament_tag_uid: "{{ tag_uid }}"
              filament_used_weight: "{{ tray_weight }}"
              filament_color: "{{ color }}"
              filament_active_tray_id: "{{ tray_sensor | replace('sensor.', '') if tray_sensor else '' }}"

          - service: utility_meter.calibrate
            target:
              entity_id: sensor.ID_PRINTER_filament_usage_meter
            data:
              value: "0"

      - conditions:
          - condition: template
            value_template: >
              {{ trigger.id == 'print_end'
                 and tray_number is not none and tray_number > 0
                 and trigger.from_state is not none
                 and trigger.from_state.state in ['printing','pause'] }}
        sequence:
          - service: rest_command.update_spool
            data:
              filament_name: "{{ name }}"
              filament_material: "{{ material }}"
              filament_tag_uid: "{{ tag_uid }}"
              filament_used_weight: "{{ tray_weight }}"
              filament_color: "{{ color }}"
              filament_active_tray_id: "{{ tray_sensor | replace('sensor.', '') if tray_sensor else '' }}"

          - service: utility_meter.calibrate
            target:
              entity_id: sensor.ID_PRINTER_filament_usage_meter
            data:
              value: "0"

mode: single


```
# MAKE SURE YOU RUN AT LEAST BAMBU LAB INTEGRATION 2.1.9 AND ABOVE, OTHERWISE IT WILL NOT WORK.

# ***FOR Bambu Lab H2S/P2S This will update Spoolman with every tray change during Multi Color Printing**

This counts for the H2S/P2S, but most likely not the H2D.
The H2S/P2S need an adjustment that isn't needed for the P1S. The active AMS tray for the P1S remains active after print. Meaning the automation works without issues.
With the H2S the active tray becomes directly inactive after the print, which prevents the automation to fire. For this to work you need to create a helper and automation
Utility meter, rest command and sensors remain the same as for the P1S. First follow step 1 and 2 for the P1S instructions and then step 3 below.

### **3. Create the helper in the configurations.yaml** 

This is now set to max 5 trays. In case you are working with multiple AMS systems, bump this number up.

```yaml
input_number:
  bambulab_last_tray:
    name: "BambuLab Last Tray"
    min: 0
    max: 5
    step: 1
```

### **4. Create an Automation. Below is the automation including log write for easy troubleshooting. Place this automation in your automations.yaml. You can always change the name that is logged into the log, it is now in Dutch :)** 

```yaml
- id: '1764680179439'
  alias: Bambulab - Update Spool
  description: Update spool
  triggers:
  - entity_id: sensor.bambulab_ams1_tray_index
    id: tray
    trigger: state
  - entity_id: sensor.ID_PRINTER_current_stage
    to:
    - finished
    - idle
    id: print_end
    trigger: state
  actions:
  - choose:
    - conditions:
      - condition: template
        value_template: '{{ trigger.id == ''tray'' and old_tray is not none and old_tray
          > 0 }}'
      sequence:
      - data:
          message: 'Tray wissel: oude tray {{ old_tray }} → {{ name }} ({{ material
            }}), Gewicht: {{ tray_weight }} g, Kleur: {{ color }}, Tag UID: {{ tag_uid
            }}'
          level: info
        action: system_log.write
      - data:
          filament_name: '{{ name }}'
          filament_material: '{{ material }}'
          filament_tag_uid: '{{ tag_uid }}'
          filament_used_weight: '{{ tray_weight }}'
          filament_color: '{{ color }}'
          filament_active_tray_id: '{{ tray_sensor | replace(''sensor.'', '''') }}'
        action: rest_command.update_spool
      - target:
          entity_id: sensor.bambulab_filament_usage_meter
        data:
          value: '0'
        action: utility_meter.calibrate
      - target:
          entity_id: input_number.bambulab_last_tray
        data:
          value: '{{ trigger.to_state.state | int(default=old_tray) }}'
        action: input_number.set_value
      - data:
          message: input_number.bambulab_last_tray → {{ trigger.to_state.state | int(default=old_tray)
            }}
          level: info
        action: system_log.write
    - conditions:
      - condition: template
        value_template: '{{ trigger.id == ''print_end'' and tray_number is not none
          and tray_number > 0 }}'
      sequence:
      - data:
          message: 'Print-einde: Tray {{ tray_number }} → {{ name }} ({{ material
            }}), Gewicht: {{ tray_weight }} g'
          level: info
        action: system_log.write
      - data:
          filament_name: '{{ name }}'
          filament_material: '{{ material }}'
          filament_tag_uid: '{{ tag_uid }}'
          filament_used_weight: '{{ tray_weight }}'
          filament_color: '{{ color }}'
          filament_active_tray_id: '{{ tray_sensor | replace(''sensor.'', '''') }}'
        action: rest_command.update_spool
      - target:
          entity_id: sensor.bambulab_filament_usage_meter
        data:
          value: '0'
        action: utility_meter.calibrate
  variables:
    old_tray: "{% if trigger.id == 'tray'\n      and trigger.from_state.state not
      in [None, '', 'unknown', 'unavailable'] %}\n  {{ trigger.from_state.state |
      int }}\n{% else %}\n  None\n{% endif %}"
    tray_number: "{% if trigger.id == 'print_end' %}\n  {{ states('input_number.bambulab_last_tray')
      | int }}\n{% else %}\n  {{ old_tray }}\n{% endif %}"
    tray_sensor: "{% if tray_number %}\n  sensor.ID_PRINTER_ams_1_tray_{{
      tray_number }}\n{% else %}\n  None\n{% endif %}"
    tray_weight: '{{ states(''sensor.bambulab_filament_usage_meter'') | float(0) |
      round(2) }}'
    tag_uid: '{{ state_attr(tray_sensor, ''tag_uid'') if tray_sensor else None }}'
    material: '{{ state_attr(tray_sensor, ''type'') if tray_sensor else None }}'
    name: '{{ state_attr(tray_sensor, ''name'') if tray_sensor else None }}'
    color: '{{ state_attr(tray_sensor, ''color'') if tray_sensor else None }}'
  mode: single
```



### Setting the active tray in the UI when switching spools

When you switch your spool in the AMS, you will need to tell spoolman which tray the new spool is in. You can do this in the UI of Spoolman updater.
Just go to the base of the URL of the API. So for example if your API url is <http://192.168.1.1:8088/spools> you go to <http://192.168.1.1:8088/>

![alt text](image.png)

Here you can set which spool is in which tray.

## Contributing

Pull requests are welcome! Please follow the standard GitHub workflow:

1. Fork the repository
2. Create a feature branch
3. Submit a pull request

## License

MIT License. See `LICENSE` file for details.
