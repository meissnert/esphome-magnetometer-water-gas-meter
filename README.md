# esphome-magnetometer-water-gas-meter [![Made for ESPHome](https://img.shields.io/badge/Made_for-ESPHome-black?logo=esphome)](https://esphome.io)

A single, self-contained [ESPHome](https://esphome.io) configuration for reading your water meter or gas meter using a cheap Chinese MMC5883MA / QMC5883P series magnetometer clone (I2C address `0x2C`) and an ESP32.

This is a fork of [tronikos/esphome-magnetometer-water-gas-meter](https://github.com/tronikos/esphome-magnetometer-water-gas-meter), built specifically to support the MMC5883MA / QMC5883P series clone sensor. It is condensed into **one YAML file** that:

- works with the raw-I2C `0x2C` magnetometer clone (the built-in ESPHome `qmc5883l` component cannot read this sensor, so the config polls the registers directly),
- targets an ESP32 dev board only (no D1 mini / ESP8266),
- needs no remote packages or `!include` chains.

The configuration is entirely contained in [`esphome-magnetometer-esp32.yaml`](esphome-magnetometer-esp32.yaml). Just copy its contents into your device configuration.

<img src="https://github.com/user-attachments/assets/eec2ee3b-b133-458f-bf28-de8dc780e3d4" alt="Water meter in Home Assistant" width=40%>

## Quick start

1. Set up **ESPHome**, either the [Home Assistant ESPHome add-on](https://esphome.io/guides/getting_started_hassio.html) or the standalone dashboard.
2. Create a new ESP32 device (e.g. `water-meter`).
3. Add your WiFi credentials to `secrets.yaml`:

   ```yaml
   wifi_ssid: "YourSSID"
   wifi_password: "YourPassword"
   ```

4. Replace the generated configuration with the **full contents of [`esphome-magnetometer-esp32.yaml`](esphome-magnetometer-esp32.yaml)**. Everything is self-contained — there are no packages, includes, or separate sensor files.
5. Optionally change the API encryption key and OTA password in the config to your own values.
6. Select **Save** and then **Install**.
7. Home Assistant should auto-discover your new device.

## Features

- Raw I2C register polling of the `0x2C` QMC5883P clone at 20 ms (QMC5883P register map: CTRL1=`0x0A`, CTRL2=`0x0B`)
- Freeze watchdog: if the data goes static (the clone stops updating its registers), the config re-arms CTRL1/CTRL2 automatically
- On-device calibration button that captures per-axis min/max, auto-selects the axis with the strongest signal, and writes both `Magnet Span` (adaptive mode) and `Threshold lower`/`Threshold upper` (threshold mode)
- Two detection algorithms selectable in Home Assistant:
  - **Adaptive** (default): a min/max tracker with slow decay and span clamping that follows thermal drift automatically
  - **Threshold**: fixed upper/lower thresholds, for stable-temperature environments
- Half-rotation counting, `Total` volume and `Flow` rate sensors
- Onboard LED (GPIO2) blinks as the magnet rotates
- `set_total` API action to align the `Total` sensor with the physical meter reading
- Web UI at `http://<ip>/` (status, logs, entity states, restart, OTA upload) with HTTP basic auth via `web_username` / `web_password` secrets

## Configuration

All tunable values are at the top of the file in the `substitutions:` section:

| Substitution | Default | Description |
| --- | --- | --- |
| `device_class` | `water` | `water` or `gas` |
| `device_icon` | `mdi:water` | `mdi:water` or `mdi:meter-gas` |
| `volume_unit` | `gal` | One of `CCF`, `ft³`, `gal`, `L`, `m³` (see calibration note below) |
| `gpio_led` | `GPIO2` | LED pin (onboard blue LED of most ESP32 dev boards) |
| `i2c_sda` / `i2c_scl` | `GPIO21` / `GPIO22` | Standard ESP32 hardware I2C pins |
| `i2c_frequency` | `100kHz` | I2C bus speed |
| `sensor_update_interval` | `20ms` | Raw magnetometer polling interval |
| `volume_per_half_rotation_initial_value` | `0.01008156` | See [Volume per half rotation](#volume-per-half-rotation) |
| `calibration_minimal_axis_range_initial_value` | `20` | Minimum µT range an axis must show to be accepted during calibration |
| `flow_update_interval_seconds` | `10` | How often the `Flow` sensor is published |
| `magnetic_field_update_delta` | `3` | Only publish tracker updates when the field changes by this many µT |
| `hide_magnetic_field_strength_sensors` | `true` | Set to `false` only during manual calibration |
| `hide_half_rotations_total_sensor` | `true` | Set to `false` only during manual calibration |
| `tracker_decay_rate` | `0.0005` | Adaptive tracker decay per tick (µT). ~1.5 µT/min. Increase if temperature changes cause false triggers |
| `min_span_multiplier` | `0.8` | Minimum adaptive detection window as a fraction of `Magnet Span` |
| `max_span_multiplier` | `1.2` | Maximum adaptive detection window as a fraction of `Magnet Span` |

## Compatibility

### Water meter

The magnetometer is used to read the rotating magnet inside your water meter.

This should be compatible with all the water meters the Flume water sensor is compatible with, which is [compatible](https://help.flumewater.com/en/articles/1618594-is-the-flume-device-compatible-with-all-water-meters) with about 95% of water meters in the United States.

To verify compatibility follow [this](https://help.flumewater.com/en/articles/1618594-is-the-flume-device-compatible-with-all-water-meters). Alternatively, install the Sensors app on your phone, place your phone next to the meter, and see if the Geomagnetic Field sensors are changing while water is running.

[Video](https://www.youtube.com/watch?v=M9nVkSZ6_H4) showing the internals of a water meter.

See [Figure 1: Nutating disc operation](https://www.instrumart.com/assets/RCDL-manual.pdf)

"The metering principle, known as positive displacement, is based on the continuous filling and discharging of the measuring
chamber. Controlled clearances between the disc and the chamber provide precise measurement of each volume cycle.
As the disc nutates, the center spindle rotates a magnet. The movement of the magnet is sensed through the meter wall
by a follower magnet or by various sensors. Each revolution of the magnet is equivalent to a fixed volume of fluid, which is
converted to any engineering unit of measure for totalization, indication or process control."

#### Magnetometer position for water meter

<img src="https://github.com/tronikos/esphome-magnetometer-water-gas-meter/assets/9987465/130f871c-dfd5-45e2-9837-b23bf8f545e7" alt="water meter sensor position" width=40%>

### Gas meter

> **⚠️ Safety Disclaimer:** Placing DIY electronics near gas infrastructure carries inherent risks. Ensure you are compliant with local regulations and are personally comfortable with the safety implications before proceeding. Use at your own risk.

The magnetometer is used to read the diaphragm that expands and contracts inside your gas meter.

This should be compatible with all diaphragm/bellows meters which are the most common type of gas meter, seen in almost all residential and small commercial installations.

To verify compatibility install the Sensors app on your phone, place your phone next to the meter, and see if the Geomagnetic Field sensors are changing while gas is running.

[Video](https://www.youtube.com/watch?v=WKlVmXe46w8) showing the internals of a gas meter.

#### Magnetometer position for gas meter

<img src="https://github.com/tronikos/esphome-magnetometer-water-gas-meter/assets/9987465/9d5a469f-6b92-442e-b2ec-e0e2b57eead3" alt="gas meter sensor position" width=40%>

## Hardware installation

### Parts

- ESP32 dev board with power adapter
  - I placed mine inside the garage
- MMC5883MA / QMC5883P series magnetometer (the Chinese `0x2C` clone works with this config)
  - I placed mine in the water meter box 20ft away from the garage
- Ethernet cable
  - I used 32.8ft or 10m direct burial CAT6. A user has reported they successfully used 75ft or 22.9m direct burial CAT6.
  - CAT6 is preferred because of its lower capacitance. CAT5 50ft or 15m [should work](https://www.youtube.com/watch?v=6v1KZBRZRCI). For 100ft you will need an active terminator such as [LTC4311](https://www.youtube.com/watch?v=nhWPxO7jx_o).
  - Do not use thermostat wire, bell wire, or any other low voltage wire. You will have communication errors or instability. You really need to be using twisted pair cables with proper shielding and lower capacitance such as CAT6.
- Some way to weather proof the magnetometer. Some options:
  - Adhesive 4:1 heat shrink tubing (this is what I used)
  - Liquid electrical tape
  - Silicone sealant
  - Nail polish
  - Hot glue
- Some way to mount the magnetometer on the meter. Some options:
  - Cable zip tie (this is what I used)
  - Duct tape
- Conduit for the ethernet cable. Can be skipped if using direct burial ethernet cable.

### Wiring

| MMC5883MA / QMC5883P clone | ESP32 |
| --- | --- |
| VCC | 3.3V |
| GND | GND |
| SCL | GPIO22 |
| SDA | GPIO21 |

The config also uses GPIO2 for the onboard LED (used to indicate magnet rotation during testing). If your board has the LED on a different pin, change `gpio_led` in the substitutions.

If your magnetometer module has its own 3.3V regulator you can connect the sensor's VCC to 5V.
The ethernet cable has 4 twisted pairs of wires. Use any solid wire color for the 4 above pins. Tie the 4 white wires together with the GND solid wire. You might need to use a header pin for the GND. If you use a header pin cut the 5 GND wires shorter to avoid the ball of wires I had...

![magnetometer wiring](https://github.com/tronikos/esphome-magnetometer-water-gas-meter/assets/9987465/c7052171-eee1-44cb-90f4-76cad4e46334)
![magnetometer in adhesive heat shrink tubing](https://github.com/tronikos/esphome-magnetometer-water-gas-meter/assets/9987465/0ca8c738-63c2-4d38-ae35-42bb219b88d1)
![driveway](https://github.com/tronikos/esphome-magnetometer-water-gas-meter/assets/9987465/69a47f3e-8d8f-4c2e-aec8-14cb729b48a4)

## Calibration

### Detection Algorithm

Two detection algorithms are available, controlled by the **"Detection algorithm"** select:

| Algorithm | Best For |
| --- | --- |
| **Adaptive** (default) | **Recommended**. Handles temperature drift automatically |
| **Threshold** | Backward compatibility with existing installations |

> **Recommendation:** Use the Adaptive algorithm. Threshold is available for backward compatibility with existing installations.

**Adaptive algorithm**:

- Automatically tracks and adapts to thermal drift
- No recalibration needed when temperature changes
- Uses a "Smart Min/Max Tracker" with window clamping and auto-centering

**Threshold algorithm**:

- Uses fixed upper/lower thresholds set during calibration
- Simple and reliable in stable temperature environments
- May require recalibration if temperature drift moves the baseline outside thresholds

### Running Calibration

1. Run a light stream of water/gas.
2. Press the "**Calibrate axis**" button in Home Assistant.
3. Wait for calibration to complete (default 5 seconds, configurable via the `Calibration time` number).

The system will automatically:

- Detect the axis with the strongest signal
- Set `Threshold lower` and `Threshold upper` (for threshold mode)
- Set `Magnet Span` (for adaptive mode)

You can switch between algorithms at any time without recalibrating.

If calibration fails, check the device logs. You might need to lower "Calibration minimal axis range".

### Volume per half rotation

This depends on your specific water/gas meter model and its size.

You can search for specifications of your specific water/gas meter and its size.

If you have the Flume water sensor you can use its lowest reported value. You can find it with:
`select min(min) from statistics_short_term, statistics_meta where statistics_meta.statistic_id = 'sensor.water_usage_current' and statistics_meta.id = metadata_id and min > 0;`

Alternatively:

1. Temporarily set `hide_half_rotations_total_sensor: 'false'` to show the "Half rotations total" sensor in HA.
2. Write it down and also write down the reading on your water/gas meter.
3. After a few hours or even days of regular water/gas usage, write down both of them again.
4. Set this to the result of: diff of readings in volume_unit divided by diff of half rotations.
5. Set `hide_half_rotations_total_sensor: 'true'`.

For water meters this defaults to `0.01008156 gal` which is for my 3/4" Badge Meter Model 35.
For gas meters this defaults to `0.125 ft³` which seems to be the most common in US.
If you have modified the `volume_unit` you have to manually convert this value.

### Setting Total Volume

If you would like your "Total" reading to match the reading displayed on your physical reading, you can use the `set_total` action.

1. From Home Assistant, navigate to **Settings** > **Developer Tools** > **Actions**.
2. Locate the `esphome.water_meter_set_total` action.
3. Enter the desired total volume you wish your device to report in the `new_total` field.
4. Click **Perform Action**.

### Temperature

Not supported on the QMC5883P — unlike the QMC5883L, the QMC5883P has no temperature
output register (regs `0x07`/`0x08` are reserved), so the config leaves the
`Temperature` sensor unpublished. If your clone ever exposes a real temperature
register, re-enable the read in the interval lambda and use the `Temperature Offset`
number to align it with a reference sensor.

## Home Assistant Integration Examples

> **Disclaimer:** The following are advanced examples. You will need to adapt the entity IDs and thresholds to match your own setup and usage patterns.

### Leak Alert Automation

In `Settings > Devices & services > Helpers` I have created a template sensor: `sensor.water_meter_flow_minus_irrigation` with the following state template: `{{ max(0, states('sensor.water_meter_flow') | float - (0.3 if now().hour in range(7, 10) else 0)) }}`. My irrigation system consumes 0.28 gal/min between 7 to 9 am or 8 to 10 am depending on DST. You will need to adjust this to your irrigation system flow and times. If you don't have irrigation you can skip this and use `sensor.water_meter_flow` below.

In `Settings > Automations` I have created the following automation to get notified if water runs continuously for too long, which could indicate a leak. It has logic to allow for longer run times (like showers) if a bathroom light is on.

```yaml
# This automation is provided as an example.
# You MUST customize the following:
# - entity_id: sensor.water_meter_flow_minus_irrigation (or your main flow sensor)
# - The thresholds for flow rate (e.g., above: 1.7)
# - The durations for each trigger (e.g., for: minutes: 3)
# - The condition for exceptions (e.g., is_state('light.bathroom_upstairs_lights', 'off'))
# - The notification service (e.g., notify.all, notify.nikos_mobile)

alias: "Notify: water meter flow"
description: "Sends critical alerts if water is running for an extended period."
triggers:
  - trigger: numeric_state
    id: high_flow
    entity_id: sensor.water_meter_flow_minus_irrigation
    above: 1.7
    for:
      minutes: 3
  - trigger: numeric_state
    id: high_flow_bath_lights_on
    entity_id: sensor.water_meter_flow_minus_irrigation
    above: 1.7
    for:
      minutes: 8
  - trigger: numeric_state
    id: medium_flow
    entity_id: sensor.water_meter_flow_minus_irrigation
    above: 1
    for:
      minutes: 5
  - trigger: numeric_state
    id: medium_flow_bath_lights_on
    entity_id: sensor.water_meter_flow_minus_irrigation
    above: 1
    for:
      minutes: 10
  - trigger: numeric_state
    id: low_flow
    entity_id: sensor.water_meter_flow_minus_irrigation
    above: 0
    for:
      minutes: 15
  - trigger: numeric_state
    id: low_flow_bath_lights_on
    entity_id: sensor.water_meter_flow_minus_irrigation
    above: 0
    for:
      minutes: 20
actions:
  - variables:
      initial_duration_seconds: "{{ trigger.for.total_seconds() }}"
      alert_start_time: "{{ now() }}"
  - if:
      - condition: template
        value_template: >-
          {{ 'bath_lights_on' in trigger.id or
          is_state('light.bathroom_upstairs_lights', 'off') }}
    then:
      - repeat:
          until:
            - condition: numeric_state
              entity_id: sensor.water_meter_flow_minus_irrigation
              below: 0.001
          sequence:
            - action: notify.all
              data:
                title: "💧 Alert: Water Flow"
                message: >-
                  {% set time_since_alert_started = now() -
                  as_datetime(alert_start_time) %}

                  {% set total_duration_seconds = initial_duration_seconds +
                  time_since_alert_started.total_seconds() %}

                  Water flow is {{ states('sensor.water_meter_flow') | round(1)
                  }} gallons per minute.

                  Water has now been running for {{ (total_duration_seconds /
                  60) | round(0) }} minutes.
                data:
                  tag: water-flow-alert
                  push:
                    sound:
                      name: default
                      critical: 1
                      volume: 1
                  ttl: 0
                  priority: high
                  media_stream: alarm_stream_max
            - action: notify.nikos_mobile
              data:
                message: TTS
                data:
                  ttl: 0
                  priority: high
                  media_stream: alarm_stream_max
                  tts_text: Water flow alert
            - delay:
                seconds: 30
```

The group notifiers are defined in `/homeassistant/configuration.yaml`:

```yaml
notify:
  - platform: group
    name: nikos
    services:
      - service: persistent_notification
      - service: mobile_app_pixel_7a
      - service: mobile_app_le2125
  - platform: group
    name: nikos_mobile
    services:
      - service: mobile_app_pixel_7a
      - service: mobile_app_le2125
  - platform: group
    name: wife
    services:
      - service: mobile_app_wife_iphone
  - platform: group
    name: all
    services:
      - service: nikos
      - service: wife
      - service: google_assistant_sdk
      - service: alexa_media_garage_ecobee_switch
```

To find what thresholds and durations to use for your own water usage patterns, run this SQL query in the **SQLite Web** add-on with different `flow_threshold`:

```sql
-- This query calculates the longest continuous period the water meter was running each day,
-- based on a defined flow rate threshold. It includes special handling for a daily
-- "irrigation" window where the flow rate can be artificially reduced.

WITH variables AS (
  -- This is the main configuration block for the query.
  -- All user-adjustable parameters are defined here for easy modification.
  SELECT
    1.5 AS flow_threshold,          -- The flow rate (e.g., in GPM or L/min) above which the water is considered "running".
    0.3 AS irrigation_flow_reduction, -- The value to subtract from the flow rate during the irrigation window.
    '07:00' AS irrigation_start_time,  -- The start time of the daily irrigation window (HH:MM).
    '10:00' AS irrigation_end_time    -- The end time of the daily irrigation window (HH:MM).
),
all_corrected_states AS (
  -- Step 1: Get all states for the target sensor and create an "effective_state".
  -- This step applies the special logic for the irrigation window.
  SELECT
    state_id,
    old_state_id,
    last_updated_ts,
    CASE
      -- If the state's timestamp falls within the irrigation window, reduce its value.
      WHEN STRFTIME('%H:%M', last_updated_ts, 'unixepoch', 'localtime') BETWEEN (SELECT irrigation_start_time FROM variables) AND (SELECT irrigation_end_time FROM variables)
        THEN MAX(0, CAST(state AS REAL) - (SELECT irrigation_flow_reduction FROM variables)) -- Subtract the reduction, ensuring it doesn't go below zero.
      -- Otherwise, just use the state's normal value.
      ELSE CAST(state AS REAL)
    END AS effective_state
  FROM states
  WHERE
    -- Filter the states table to only include our specific water meter sensor.
    metadata_id = (
      SELECT metadata_id FROM states_meta WHERE entity_id = 'sensor.water_meter_flow'
    )
),
state_pairs AS (
  -- Step 2: Get the current state and the immediately preceding state on the same row.
  -- This is done by joining the table to itself using the old_state_id, which links each state to the previous one.
  SELECT
    current_state.last_updated_ts,
    current_state.effective_state AS effective_current_state,
    prev_state.effective_state AS effective_prev_state
  FROM
    all_corrected_states AS current_state
  JOIN
    all_corrected_states AS prev_state ON current_state.old_state_id = prev_state.state_id
),
run_events AS (
  -- Step 3: Analyze the state pairs to identify the exact moments a "run" starts or stops.
  -- A "run" is defined by the flow rate crossing the 'flow_threshold' defined in the variables.
  SELECT
    last_updated_ts,
    CASE
      -- A "start" event (1) is when the flow crosses *above* the threshold.
      WHEN effective_current_state > (SELECT flow_threshold FROM variables) AND effective_prev_state <= (SELECT flow_threshold FROM variables) THEN 1
      -- A "stop" event (-1) is when the flow crosses *below* or becomes equal to the threshold.
      WHEN effective_current_state <= (SELECT flow_threshold FROM variables) AND effective_prev_state > (SELECT flow_threshold FROM variables) THEN -1
      -- Otherwise, it's not a significant event.
      ELSE 0
    END AS event_type
  FROM state_pairs
),
run_periods AS (
  -- Step 4: Match up each "start" event with its corresponding "stop" event.
  -- This defines a complete, continuous run period.
  SELECT
    last_updated_ts AS start_time,
    -- For every start event, look forward in time to find the timestamp of the very next stop event.
    (
      SELECT MIN(e2.last_updated_ts)
      FROM run_events e2
      WHERE e2.last_updated_ts > e1.last_updated_ts AND e2.event_type = -1
    ) AS end_time
  FROM run_events e1
  -- We only care about the "start" events to begin our periods.
  WHERE e1.event_type = 1
),
daily_ranked_runs AS (
  -- Step 5: Calculate the duration of each run and rank them within each day.
  SELECT
    STRFTIME('%Y-%m-%d', start_time, 'unixepoch', 'localtime') AS run_day,
    (end_time - start_time) AS duration_seconds,
    start_time,
    end_time,
    -- The RANK() window function assigns a rank to each run (1 for the longest)
    -- within each day (PARTITION BY run_day).
    RANK() OVER (
      PARTITION BY STRFTIME('%Y-%m-%d', start_time, 'unixepoch', 'localtime')
      ORDER BY (end_time - start_time) DESC
    ) as rank_num
  FROM run_periods
  -- Ignore any runs that may not have a corresponding stop event (e.g., if the water is still running).
  WHERE end_time IS NOT NULL
)
-- Final Step: Select the longest run for each day and format the output for readability.
SELECT
  run_day,
  duration_seconds / 60 AS duration_minutes,
  DATETIME(start_time, 'unixepoch', 'localtime') AS run_start_time,
  DATETIME(end_time, 'unixepoch', 'localtime') AS run_end_time
-- Filter for only the top-ranked (longest) run for each day.
FROM daily_ranked_runs
WHERE rank_num = 1
-- Order the results with the most recent day first.
ORDER BY run_day DESC;
```

### Slow Leak Detection Automation

This automation runs every 30 minutes and checks if there was any water usage in the last hour without the flow rate ever exceeding a low threshold (e.g., 0.1 gal/min). This is useful for detecting very slow leaks that don't trigger a continuous flow alert but still consume water over time.

```yaml
alias: "Notify: Slow Water Leak"
description: Detects if water volume increased while flow rate remained low.
triggers:
  - trigger: time_pattern
    minutes: /30
actions:
  - action: sql.query
    data:
      query: |-
        SELECT
          (
            SELECT MAX("sum") - MIN("sum")
            FROM statistics_short_term s
            JOIN statistics_meta m ON s.metadata_id = m.id
            WHERE m.statistic_id = 'sensor.water_meter_total'
            AND s.start_ts >= strftime('%s', 'now') - 3600
          ) AS volume_delta,
          (
            SELECT MAX("max")
            FROM statistics_short_term s
            JOIN statistics_meta m ON s.metadata_id = m.id
            WHERE m.statistic_id = 'sensor.water_meter_flow'
            AND s.start_ts >= strftime('%s', 'now') - 3600
          ) AS max_flow_rate
    response_variable: sql_result
  - if:
      - condition: template
        value_template: |-
          {% set row = sql_result['result'][0] %}
          {{
             row['volume_delta'] is not none and
             row['volume_delta'] > 0.015 and
             (row['max_flow_rate'] is none or row['max_flow_rate'] < 0.1)
          }}
    then:
      - action: notify.nikos
        data:
          title: 💧 Slow Leak Detected
          message: >-
            {% set row = sql_result['result'][0] %} Usage in last hour: {{
            row['volume_delta'] | round(3) }} gal. Max Flow Rate observed: {{
            row['max_flow_rate'] | round(3) }} gal/min.

            This indicates a leak of approx {{ (row['volume_delta'] / 60) |
            round(4) }} gal/min.
mode: single
```

### Daily Usage Alert

This automation checks your total consumption at the end of the day and notifies you if it's unusually high (possible leak) or low (possible sensor issue).

First, create a **Utility Meter** helper in Home Assistant (`Settings > Devices & Services > Helpers`) to track the daily total from your `sensor.water_meter_total` entity.

```yaml
# This automation requires a Utility Meter helper, e.g., 'sensor.water_meter_daily_total',
# configured to track your main sensor's total volume with a daily cycle.
# You MUST customize the high/low thresholds and notification service.

alias: "Notify: Daily Water Usage"
description: "Alerts if daily water consumption is abnormally high or low."
triggers:
  - trigger: time
    at: "23:59:00"
conditions: []
actions:
  - if:
      - condition: numeric_state
        entity_id: sensor.water_meter_daily_total
        above: 150 # Adjust this to your typical high usage
    then:
      - action: notify.nikos # Change to your notification service
        data:
          title: High daily water usage
          message: >-
            Consumed {{ states('sensor.water_meter_daily_total') }} gal today.
            Is there a leak?
  - if:
      - condition: numeric_state
        entity_id: sensor.water_meter_daily_total
        below: 10 # Adjust this to your typical low usage
    then:
      - action: notify.nikos # Change to your notification service
        data:
          title: Low daily water usage
          message: >-
            Consumed {{ states('sensor.water_meter_daily_total') }} gal today.
            Do you need to reposition or recalibrate the sensor?
mode: single
```

## Troubleshooting

- **No data from sensors or connection instability:**
  - Double-check your wiring. VCC, GND, SCL, and SDA must be correct.
  - Verify the I2C address of your sensor in the ESPHome logs. This config polls address `0x2C` directly.
  - Your cable might be too long or poor quality. Try a shorter, shielded cable.
  - If you are experiencing instability with a longer cable, you may consider adding `i2c_frequency: 10kHz` to the substitution in the YAML.
  - Additionally, you may wish to consider reducing the SCL/SDA pull-up resistors to 1.2kΩ (many devices ship with 4.7kΩ pre-installed). For 3.3V I2C logic, 1kΩ is the minimum.
- **Data goes static and stops updating:**
  - This config includes a freeze watchdog that re-arms CTRL1/CTRL2 when the values stop changing. If you still see freezes, check the ESPHome logs for "Data static detected" messages.
- **Inaccurate readings:**
  - Recalibrate! Flow rate and totals depend entirely on correct calibration.
  - Ensure the sensor is mounted securely and hasn't shifted.
