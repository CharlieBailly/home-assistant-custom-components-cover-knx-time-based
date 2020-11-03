# Cover Time Based Component for KNX

Cover Time Based Component for your [Home-Assistant](http://www.home-assistant.io) based on [nagyrobi's Cover Time Based Component trigger script (RF) version](https://github.com/nagyrobi/home-assistant-custom-components-cover-rf-time-based), modified for covers triggered by KNX payloads that Home-Assistant read on the KNX bus, or any other unidirectional methods.

With this component you can add a time-based cover. You have to set KNX group addresses to open/close and stop the cover. Position is calculated based on the fraction of time spent by the cover travelling up or down. You can set position from within Home Assistant using service calls. When you use this component, you can forget about the cover's original remote controllers or switches, because there's no feedback from the cover about its real state, state is assumed based on the last command sent from Home Assistant. There's a custom service available where you can update the real state of the cover based on external sensors if you want to.

If you want to drive your covers using scripts and not using KNX payloads, just use [nagyrobi's Cover Time Based Component trigger script (RF) version](https://github.com/nagyrobi/home-assistant-custom-components-cover-rf-time-based)

The component adds two services `set_known_position` and `set_known_action` which allow updating HA in response to external stimuli like sensors without really moving the cover.

[Support forum](https://community.home-assistant.io/t/custom-component-cover-time-based/187654/3?u=robi)

## Installation

[![hacs_badge](https://img.shields.io/badge/HACS-Default-orange.svg?style=for-the-badge)](https://github.com/custom-components/hacs)

- ~~Install using HACS~~ (not yet possible with this version, working on it)
- Install manually: copy all files in custom_components/cover_knx_time_based to your config/custom_components/cover_knx_time_based/ directory.
- Restart Home-Assistant.
- Set `fire_event: true` in your configuration.yaml in `knx:`
- Set `fire_event_filter: ["*"]` (or use a custom filter) in your configuration.yaml in `knx:`
- Add the configuration to your configuration.yaml.
- Restart Home-Assistant again.

## Usage

To use this component in your installation, you need a working `knx:` integration (with a KNX/IP Interface so Home-Assistant can read payloads on the KNX bus).  
Then you need to enable knx events (see Installation above) so Home-Assistant can fire an event each time a payload is on the KNX bus (this integration relies on these events)
Finally, add the following to your configuration.yaml file:

## Example configuration.yaml entry

```yaml
cover:
  - platform: cover_knx_time_based
    devices:
      my_room_cover_time_based:
        name: "My Room Cover"
        travelling_time_up: 36
        travelling_time_down: 34
        move_group_address: "2/4/0"
        stop_group_address: "2/4/1"
        watch_move_group_addresses:
          - "2/4/0"
          - "24/4/23"
          - "10/4/226"
          - "24/4/45"
        watch_stop_group_addresses:
          - "2/4/1"
          - "24/4/24"
          - "10/4/225"
          - "24/4/46"
        send_stop_at_ends: False #optional
        aliases: #optional
          - my_room_cover_time_based
```

Mandatory settings that are not obvious :

- `move_group_address` is the KNX group address that Home-Assistant will send a payload to (to actually make the cover move up or down) (it is a KNX DPT 1 payload, which is simply 0 or 1)
- `stop_group_address` is the KNX group address that Home-Assistant will send a payload to (to actually make the cover stop) (it is a KNX DPT 1 payload, which is simply 0 or 1)
- `watch_move_group_addresses` is a list of all the KNX group addresses that Home-Assistant will listen to (to update the cover position without actually make the cover move since it should already be moving (because of a real switch for exemple))
- `watch_stop_group_addresses`: is a list of all the KNX group addresses that Home-Assistant will listen to (to update the cover position without actually make the cover stop since it should already be stopped (because of a real switch for exemple))

- To know the `move_group_address` and other group addresses, you may need to sniff and see payloads that are sent on the KNX bus, if you don't have ETS5 software, i recommand you to use [KNXmap](https://github.com/takeshixx/knxmap) with the following command :  
  `knxmap monitor ip_address_of_your_knx_ip_interface --group-monitor`  
  Then, you may see payloads like this one when you turn on a real KNX switch that control your cover :  
  `[ chan_id: 115, seq_no: 0, message_code: L_Data.ind, source_addr: 0.2.5, dest_addr: 24/4/6, tpci_type: UDP, tpci_seq: 0, apci_type: A_GroupValue_Write, apci_data: 0 ]`  
  So here the `move_group_address` address we may use is "24/4/6" which is the `dest_addr` in the payload, and you also should add in `watch_move_group_addresses` all the addresses that trigger movement of your cover, including the one you put in `move_group_address`, and the same thing apply for `stop_group_address` and `watch_stop_group_addresses`

Optional settings:

- `send_stop_at_ends` defaults to `False`. If set to `True`, the Stop script will be run after the cover reaches to 0 or 100 (closes or opens completely). This is for people who use interlocked relays in the scripts to drive the covers, which need to be released when the covers reach the end positions.
- `aliases`: to override the entity name generated by Home Assistant internally from the (friendly) name.

## Services to set position or action without triggering cover movement.

This component provides 2 services:

1.  `cover_knx_time_based.set_known_position` lets you specify the position of the cover if you have other sources of information, i.e. sensors. It's useful as the cover may have changed position outside of HA's knowledge, and also to allow a confirmed position to make the arrow buttons display more appropriately.
1.  `cover_knx_time_based.set_known_action` is for instances when an action is caught in the real world but not process in HA, .e.g. an RF bridge detects a `stop` action that we want to input into HA without calling the stop command.

### `cover_knx_time_based.set_known_position`

In addition to `entity_id` and `position` takes 2 optional parameters:

- `confident` that affects how the cover is presented in HA. Setting confident to `true` will mean that certain button operations aren't permitted.
- `position_type` allows the setting of either the `target` or `current` posistion.

Following examples to help explain parameters and use cases:

1.  This example automation uses `position_type: current` and `confident: true` when a reed sensor has indicated a garage door is closed when contact is made:

```yaml
- id: "garage_closed"
  alias: "Doors: garage set closed when contact"
  description: ""
  trigger:
    - entity_id: binary_sensor.door_garage_cover
      platform: state
      to: "off"
  condition: []
  action:
    - data:
        confident: true
        entity_id: cover.garage_door
        position: 0
        position_type: current
      service: cover_knx_time_based.set_known_position
```

We have set `confident` to `true` as the sensor has confirmed a final position. The down arrow is now no longer available in default HA frontend when the cover is closed.
`position_type` of `current` means the current position is moved immediately to 0 and stops there (provided cover is not moving, otherwise will contiune moving to original target).

2.  This example uses `position_type: target` (the default) and `confident: false` (also default) where an RF bridge has interecepted an RF command, so we know an external remote has triggered cover opening action:

```yaml
- id: "rf_cover_opening"
  alias: "RF_Cover: set opening when rf received"
  description: ""
  trigger:
    - entity_id: sensor.rf_command
      platform: state
      to: "open"
  condition:
    - condition: state
      entity_id: cover.rf_cover
      state: closed
  action:
    - data:
        entity_id: cover.rf_cover
        position: 100
      service: cover_knx_time_based.set_known_position
```

`confident` is omitted so defaulted to `false` as we're not sure where the movement may end, so all arrows are available.
`position_type` is omitted so defaulted to `target`, meaning cover will transition to `position` without triggering any start or stop actions.

### `cover_knx_time_based.set_known_action`

This service mimics cover movement in Home Assistant without actually sending out commands to the cover. It can be used for example when external RF remote controllers act on the cover directly, but the signals can be captured with an RF brigde and Home Assistant can play the movement in parrallel with the real cover. In addtion to `entity_id` takes parameter `action` that should be one of open, close or stop.

Example:

```yaml
- id: 'rf_cover_stop'
  alias: 'RF_Cover: set stop action from bridge trigger'
  description: ''
  trigger:
  - entity_id: sensor.rf_command
    platform: state
    to: 'stop'
  condition:[]
  action:
  - data:
      entity_id: cover.rf_cover
      action: stop
    service: cover_knx_time_based.set_known_action
```

In this instance we have caught a stop signal from the RF bridge and want to update HA cover without triggering another stop action.

## Icon customization

For proper icon display (opened/moving/closed) customization can be added to `configuration.yaml` based of what type of covers you have, either one by one, or for all covers at once:

```yaml
homeassistant:
  customize_domain: #for all covers
    cover:
      device_class: shutter
  customize: #for each cover separately
    cover.my_room_cover_time_based:
      device_class: shutter
```

More details in [Home Assistant device class docs](https://www.home-assistant.io/docs/configuration/customizing-devices/#device-class).
