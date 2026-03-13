# 🌬️ ESPHome Ecocomfort 2.0

ESPHome package for controlling **Intelliclima Ecocomfort 2.0 Smart** decentralized VMC (Ventilation Mechanical Controlled) units via Bluetooth Low Energy.

Each unit is a wall-mounted heat-recovery ventilator with a ceramic core that alternates between intake and exhaust, recovering up to 90% of heat energy.

## ✨ Features

- 📡 Full bidirectional BLE control (read + write)
- 🏠 Native Home Assistant fan entity with speed and preset modes
- 🌡️ Real-time sensor readback (temperature, humidity, VOC)
- ❄️🔥 Season control (Winter/Summer) with heat recovery management
- 🌙 Free cooling (passive cooling when outdoor air is cooler)
- 📊 Sensor threshold configuration (humidity, VOC, luminosity)
- 🔧 Calibration offset support (temperature, humidity)
- 🕐 Automatic clock sync from Home Assistant
- 🔄 BLE state readback syncs HA entities with actual device state

## 🛠️ Hardware Requirements

- **ESP32** dev board (any, BLE is built-in)
- **Intelliclima Ecocomfort 2.0 Smart** VMC units
- No additional hardware needed (BLE range ~10m through walls)

> 💡 One ESP32 can reliably control 3-4 VMC units. Use multiple ESP32s for larger installations.

## 🚀 Quick Start

1. Create your ESPHome controller config (see [`examples/command-center.yaml`](examples/command-center.yaml) for a complete example)
2. Add the package to your config (see options below)
3. Replace MAC addresses with your VMC units' BLE addresses
4. Flash the ESP32
5. Put each VMC in pairing mode (hold button 5 seconds until LED blinks)
6. Entities appear automatically in Home Assistant

### Option A: Remote Package ⭐ (recommended)

No files to copy — ESPHome pulls the package directly from GitHub:

```yaml
packages:
  vmc_living_room: !include
    url: https://github.com/gledian/esphome-ecocomfort2
    file: packages/ecocomfort2.yaml
    vars:
      mac_address: "AA:BB:CC:DD:EE:FF"
      unit_id: vmc_living_room
      unit_name: "VMC Living Room"
```

### Option B: Local Package

Copy the `packages/` folder to your ESPHome config directory:

```yaml
packages:
  vmc_living_room: !include
    file: packages/ecocomfort2.yaml
    vars:
      mac_address: "AA:BB:CC:DD:EE:FF"
      unit_id: vmc_living_room
      unit_name: "VMC Living Room"
```

> ⚠️ **Required:** A `time` component must be defined in the main config for clock sync:
> ```yaml
> time:
>   - platform: homeassistant
>     id: ha_time
> ```

## 🔍 Finding Your VMC's MAC Address

The VMC advertises via BLE as `Comfort_XXXX`. You can find the MAC address:
- 📱 In the **Intelliclima app** (device info section)
- 📶 Using a BLE scanner app (e.g., nRF Connect)
- 📋 In ESPHome logs during BLE scanning

## 📋 Entities Created Per Unit

### 🎛️ Controllable
| Entity | Description |
|--------|-------------|
| `fan.<unit_id>` | Main control: on/off, speed (25-100%), preset mode |
| `select.<unit_id>_season` | Winter (heat recovery) / Summer (bypass) |
| `select.<unit_id>_free_cooling` | Free cooling intensity: Off/Low/Medium/High |
| `number.<unit_id>_humidity_threshold` | Humidity sensitivity for Sensor mode (0-3) |
| `number.<unit_id>_luminosity_threshold` | Luminosity sensitivity (0-3) |
| `number.<unit_id>_voc_threshold` | VOC sensitivity (0-3) |
| `number.<unit_id>_set_temp_offset` | Temperature calibration offset |
| `number.<unit_id>_set_humidity_offset` | Humidity calibration offset |
| `switch.<unit_id>_humidity_advanced` | Humidity threshold: boost vs +1 step |
| `switch.<unit_id>_voc_advanced` | VOC threshold: boost vs +1 step |
| `button.<unit_id>_pair` | Manual BLE re-pairing |

### 📊 Read-Only
| Entity | Description |
|--------|-------------|
| `binary_sensor.<unit_id>_connected` | BLE connection status |
| `binary_sensor.<unit_id>_boost_active` | Device auto-boost status |
| `sensor.<unit_id>_temperature` | Indoor temperature (°C) |
| `sensor.<unit_id>_humidity` | Indoor humidity (%RH) |
| `sensor.<unit_id>_voc` | Air quality - VOC (ppb) |
| `sensor.<unit_id>_direction` | Current airflow direction |
| `sensor.<unit_id>_actual_mode` | Current operating mode |
| `sensor.<unit_id>_actual_speed` | Current speed level |
| `text_sensor.<unit_id>_firmware` | Device firmware version |
| `sensor.<unit_id>_role` | Master/slave role |
| `sensor.<unit_id>_temp_offset` | Current temp offset on device |
| `sensor.<unit_id>_humidity_offset` | Current humidity offset on device |

## ⚡ Speed Mapping

| HA % | Device | Airflow | Power |
|------|--------|---------|-------|
| 25%  | Sleep  | 8.0 m³/h  | 2.0W |
| 50%  | Vel 1  | 20.5 m³/h | 2.5W |
| 75%  | Vel 2  | 35.0 m³/h | 4.0W |
| 100% | Vel 3  | 48.5 m³/h | 6.3W |

## 🔀 Preset Modes

| Mode | Description |
|------|-------------|
| 🟢 In | Fresh air intake only (reverts to Auto after 60 min) |
| 🔴 Out | Exhaust only (reverts to Auto after 60 min) |
| 🔄 In/Out | Alternating intake/exhaust with heat recovery (normal) |
| 📊 Sensor | Auto speed based on humidity/VOC sensor readings |
| 🤖 Auto | Device follows its built-in weekly schedule (configured via the Intelliclima app). Not implemented in this package — use HA automations instead for more flexibility |

## ❄️🔥 Season & Free Cooling

**🔥 Winter mode:** Ceramic core actively recovers heat — warm exhaust air heats the ceramic, which then warms incoming fresh air. Saves heating energy.

**☀️ Summer mode:** Heat recovery is minimized. Enables free cooling feature.

**🌙 Free cooling** (Summer only): When outdoor temperature drops below indoor temperature, the VMC does extended intake-only periods to passively cool the room. The threshold setting controls sensitivity:

| Level | Differential | Best for |
|-------|-------------|----------|
| 🟢 Low | 2°C | Mild nights, gentle cooling |
| 🟡 Medium | 4°C | Balanced cooling/noise |
| 🔴 High | 6°C | Hot days, maximum effect |

## 📊 Sensor Thresholds

Control the "Sensor" preset mode sensitivity. When readings exceed the threshold, the device reacts automatically:

| Level | Humidity | VOC | Luminosity |
|-------|----------|-----|------------|
| 0 | Disabled | Disabled | Disabled |
| 1 (Low) | 55% | 250 ppm | 0.100 lux |
| 2 (Medium) | 60% | 300 ppm | 0.125 lux |
| 3 (High) | 65% | 350 ppm | 0.150 lux |

> 💡 **Advanced mode** (per threshold): Changes reaction from BOOST (max speed) to +1 STEP (gentler). Recommended for bathrooms and kitchens where transient spikes are common.

## 📡 BLE Protocol

All communication uses a single BLE GATT service:

| Characteristic | UUID | Access | Description |
|----------------|------|--------|-------------|
| C_INFO | `f5f56229-dd4f-480f-a829-9189269d8b37` | Read | Firmware version, serial |
| C_STATE | `438d3433-7e5a-459a-a8e4-66343fad2bb0` | Read | Temperature, humidity, VOC, direction |
| C_SETTING_OPER | `b9d6f678-bc0d-4a73-90c8-60b0f07301f1` | R/W | Operating mode (2 bytes) |
| C_CONFIGURATION | `d3dac48e-b4e1-4f3a-8715-326ddf1da89a` | R/W | Device config (12 bytes) |
| C_SETTING_CLOCK | `82788997-49e4-4533-b949-7ed433678044` | Write | Clock sync (8 bytes) |
| C_ADVANCED | `f8b2284e-61dd-44e3-a782-a93c9503ab2d` | R/W | Sensor calibration offsets (4 bytes) |
| C_PROGRAMS | `4a3561b0-...` | R/W | Weekly schedule (168 bytes) |
| C_STATS | `ab53b5e0-...` | Read | Usage statistics |

Service UUID: `f4b827c3-e660-4bc8-bdf6-3c8e9b845e0d`

## 📦 Blueprints (One-Click Install)

Ready-to-use automations — import directly into Home Assistant:

| Blueprint | Description | Import |
|-----------|-------------|--------|
| [Season Auto-Switch](blueprints/vmc_season_auto_switch.yaml) | Switches Winter/Summer based on outdoor temperature with hysteresis | [![Import](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fgledian%2Fesphome-ecocomfort2%2Fblob%2Fmain%2Fblueprints%2Fvmc_season_auto_switch.yaml) |
| [Free Cooling Auto](blueprints/vmc_free_cooling_auto.yaml) | Manages free cooling by season and time of day | [![Import](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fgledian%2Fesphome-ecocomfort2%2Fblob%2Fmain%2Fblueprints%2Fvmc_free_cooling_auto.yaml) |

### How to use

1. Click the **Import Blueprint** badge above (or go to **Settings → Automations → Blueprints → Import Blueprint** and paste the blueprint URL)
2. Click **Create Automation** from the imported blueprint
3. Fill in the inputs:
   - **Season Auto-Switch:** select your outdoor temperature sensor, pick your VMC season entities, and adjust thresholds/duration if needed
   - **Free Cooling Auto:** select your VMC season and free cooling entities, choose a season check entity (any single VMC), and optionally adjust night/day times and levels
4. Save — the automation is ready to go

## 🏡 Home Assistant Automation Examples

### 🔥❄️ Auto season switch based on outdoor temperature

Automatically switches all VMC units between Winter and Summer mode based on
outdoor temperature. Uses a **5°C hysteresis band** (15°C–20°C) and a **4-hour
sustained threshold** to prevent flip-flopping during shoulder seasons.

| Condition | Action |
|-----------|--------|
| 🌡️ Above 20°C for 4 hours | ☀️ Switch to **Summer** (heat recovery bypassed, free cooling available) |
| 🌡️ Below 15°C for 4 hours | ❄️ Switch to **Winter** (heat recovery active, saves heating energy) |

> 📝 Replace `sensor.outdoor_temperature` with your own outdoor temperature sensor
> (e.g., a weather integration or an outdoor sensor).

```yaml
automation:
  - alias: "VMC Season Auto-Switch"
    trigger:
      - trigger: numeric_state
        entity_id: sensor.outdoor_temperature
        above: 20
        for: "04:00:00"
        id: summer
      - trigger: numeric_state
        entity_id: sensor.outdoor_temperature
        below: 15
        for: "04:00:00"
        id: winter
    action:
      - choose:
          - conditions:
              - condition: trigger
                id: summer
            sequence:
              - action: select.select_option
                target:
                  entity_id:
                    - select.vmc_living_room_season
                    - select.vmc_bedroom_season
                data:
                  option: "Summer"
          - conditions:
              - condition: trigger
                id: winter
            sequence:
              - action: select.select_option
                target:
                  entity_id:
                    - select.vmc_living_room_season
                    - select.vmc_bedroom_season
                data:
                  option: "Winter"
```

---

### 🚿 Shower mode (boost ventilation on humidity spike)

Detects a rapid humidity increase (typical of a shower starting) and switches
the bathroom VMC to **full-speed exhaust** to remove moisture quickly.

> ⚠️ Requires a [derivative sensor](https://www.home-assistant.io/integrations/derivative/)
> to calculate the rate of humidity change (%/min) from your humidity sensor:
>
> ```yaml
> sensor:
>   - platform: derivative
>     source: sensor.vmc_bathroom_humidity
>     name: "Bathroom Humidity Rate"
>     unit_time: min
>     time_window: "00:05:00"
> ```

**How it works:** When the rate exceeds **3%/min** (a fast spike typical of shower
steam), the VMC switches to exhaust-only at maximum speed. The "Out" preset
automatically reverts to normal operation after **60 minutes** (device firmware behavior).

```yaml
automation:
  - alias: "VMC Shower Mode"
    trigger:
      - trigger: numeric_state
        entity_id: sensor.bathroom_humidity_rate
        above: 3
    action:
      - action: fan.turn_on
        target:
          entity_id: fan.vmc_bathroom
        data:
          percentage: 100
          preset_mode: "Out"
```

---

### 🌙 Free cooling on summer nights

Takes advantage of **cooler night air** to passively cool the house without
air conditioning. Adjusts free cooling intensity based on time of day.

| Time | Free Cooling | Why |
|------|-------------|-----|
| 🌙 22:00 – 07:00 | Low (2°C delta) | Night air is cool → aggressive intake |
| ☀️ 07:00 – 22:00 | Medium (4°C delta) | Less delta available → avoid marginal cycles |
| ❄️ Winter | Off | Free cooling disabled in heat recovery mode |

```yaml
automation:
  - alias: "VMC Free Cooling Auto"
    trigger:
      - trigger: state
        entity_id:
          - select.vmc_living_room_season
          - select.vmc_bedroom_season
      - trigger: time
        at: "22:00:00"
        id: night
      - trigger: time
        at: "07:00:00"
        id: day
    action:
      - choose:
          # Summer night → Low free cooling (max benefit from cool night air)
          - conditions:
              - condition: state
                entity_id: select.vmc_living_room_season
                state: "Summer"
              - condition: time
                after: "22:00:00"
                before: "07:00:00"
            sequence:
              - action: select.select_option
                target:
                  entity_id:
                    - select.vmc_living_room_free_cooling
                    - select.vmc_bedroom_free_cooling
                data:
                  option: "Low"
          # Summer day → Medium (less aggressive, avoids marginal cycles)
          - conditions:
              - condition: state
                entity_id: select.vmc_living_room_season
                state: "Summer"
            sequence:
              - action: select.select_option
                target:
                  entity_id:
                    - select.vmc_living_room_free_cooling
                    - select.vmc_bedroom_free_cooling
                data:
                  option: "Medium"
        # Winter → disable free cooling
        default:
          - action: select.select_option
            target:
              entity_id:
                - select.vmc_living_room_free_cooling
                - select.vmc_bedroom_free_cooling
            data:
              option: "Off"
```

## 🔧 Troubleshooting

| Problem | Solution |
|---------|----------|
| 🔴 VMC not connecting | Put it in pairing mode (hold button 5 sec), then press the Pair button in HA |
| ⚠️ Entities show unavailable | Check BLE range — ESP32 BLE range is ~10m through walls |
| 📡 Commands not sent | Check `binary_sensor.<unit>_connected` — commands are only sent when connected |
| 🔄 Season/free cooling not changing | Verify the device is in the expected mode via the readback sensors |

## 📄 License

MIT
