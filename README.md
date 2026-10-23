# Home Assistant METAR backup for the NWS integration

A small, no-custom-code backup for the Home Assistant **National Weather Service (NWS)** integration. It polls your nearest airport's **METAR** report from the NOAA Aviation Weather Center and gives you "selected" temperature, humidity and wind sensors that use NWS, the METAR, or an automatic fallback, whichever you choose.

## The problem

The NWS integration reads each station's recent observations from `api.weather.gov`. That feed sometimes stalls for hours:

- The observations list for a station stops at an old report.
- `/stations/<ID>/observations/latest` keeps publishing current reports.
- Home Assistant logs `Error fetching NWS observation station <ID> data: Observation received with no data.`
- The weather entity loses its temperature, humidity and wind attributes, and cards that depend on them go blank.

This has been reported on the API's own tracker for years (for example [#751](https://github.com/weather-gov/api/discussions/751) and [#361](https://github.com/weather-gov/api/discussions/361)). There was nothing wrong on the Home Assistant side in the case this was written for: polls succeeded and returned in well under a second, and a request that bypassed the CDN cache returned the same stale report.

## What this does

- Polls the Aviation Weather Center METAR API every 10 minutes for a station you choose.
- Creates sensors for METAR temperature, humidity, wind speed, wind gust, pressure and report time.
- Creates three "selected" sensors (temperature, humidity, wind speed) that follow a dropdown:

| Dropdown | Behavior |
|---|---|
| `Auto` | Use the NWS value if the NWS entity has one, otherwise use the METAR |
| `NWS` | Always use the NWS entity |
| `METAR` | Always use the METAR sensor |

Point your cards and automations at the `sensor.selected_weather_*` sensors and they keep working when NWS goes quiet.

## Requirements

- Home Assistant with the NWS integration set up (any weather entity with `temperature`, `humidity` and `wind_speed` attributes will work in its place).
- Your nearest METAR-reporting airport's four-character code (for example `KSFO`).
- A US customary unit system (see [Units](#units) for metric).

Tested on Home Assistant OS.

## Install

### 1. Create two helpers

In **Settings → Devices & services → Helpers → Create helper**:

- **Dropdown**, named `Weather source`, with options `Auto`, `NWS`, `METAR`. This creates `input_select.weather_source`.
- **Text**, named `Weather station`, min length 4 and max length 4, optionally with the pattern `^[A-Za-z0-9]{4}$`. This creates `input_text.weather_station`. After creating it, open the helper and set its value to your airport code. (US airports use `K` plus the FAA identifier, and a few small airports have digits in that identifier, which is why the pattern allows them.)

Prefer YAML helpers? Add this instead (this block is a YAML equivalent of the steps above and wasn't separately tested):

```yaml
input_select:
  weather_source:
    name: Weather source
    options: [Auto, NWS, METAR]
    initial: Auto

input_text:
  weather_station:
    name: Weather station
    min: 4
    max: 4
    initial: "KXXX"   # change to your airport code
```

### 2. Add the sensors

Open `metar_backup.yaml` and copy its contents to the **bottom of `/config/configuration.yaml`**.

1. Back up first. A full backup (Settings → System → Backups) is best, and over SSH you can also run `cp /config/configuration.yaml /config/configuration.yaml.bak`.
2. Check for existing keys: `grep -nE "^(rest|template):" /config/configuration.yaml`. If either exists, add the list items under that key instead of adding it twice. If `template:` points to an include file (for example `template: !include templates.yaml`), put the template sensors in that file.
3. Find your NWS weather entity under **Settings → Devices & services → NWS → Entities**, and replace `weather.YOUR_NWS_ENTITY` (3 places).
4. Edit the file with the File editor add-on, or over SSH with `nano /config/configuration.yaml` (save with Ctrl+O, exit with Ctrl+X). Use spaces, not tabs.

### 3. Check and restart

Run `ha core check` over SSH, or **Developer Tools → YAML → Check configuration**. If it passes, restart Home Assistant.

### 4. Verify

In **Developer Tools → States**, search for `metar`:

- `sensor.metar_temperature` should show a current reading for your station.
- `sensor.metar_report_time` should be a recent time (UTC). METARs are usually issued hourly, often with extra special reports when conditions change, so it can be up to an hour old.
- `sensor.selected_weather_temperature` should show a value.

To revert, restore your `configuration.yaml` backup and restart.

## Sensors

| Entity | Source |
|---|---|
| `sensor.metar_temperature` | METAR (converted to °F) |
| `sensor.metar_humidity` | Calculated from temperature and dew point |
| `sensor.metar_wind_speed` / `sensor.metar_wind_gust` | METAR (knots converted to mph) |
| `sensor.metar_pressure` | METAR altimeter setting (converted to inHg) |
| `sensor.metar_report_time` | METAR observation time (UTC) |
| `sensor.selected_weather_temperature` | Per dropdown |
| `sensor.selected_weather_humidity` | Per dropdown |
| `sensor.selected_weather_wind_speed` | Per dropdown |

## Notes and limitations

- **A backup, not a replacement.** METARs are mostly hourly. NWS observations are usually fresher when the feed is healthy. In the evening, when temperatures are changing, expect the METAR to trail NWS by a degree or two until the next report.
- **`Auto` only falls back when the NWS entity has no value.** If NWS keeps showing an old, stale reading, `Auto` will keep using it. Set the dropdown to `METAR` if you want the METAR sooner.
- **This doesn't create a weather entity.** Cards that need a `weather.*` entity with forecasts still need NWS. A combined entity with forecasts is a bigger job and isn't included.
- **Be polite to the API.** Ten minutes is plenty for hourly reports. The request sets a descriptive `User-Agent`.
- **Humidity is calculated.** METARs report dew point, not relative humidity.

## Units

The sensors assume °F, mph and inHg. For metric, change these lines in `metar_backup.yaml`:

| Sensor | Metric version |
|---|---|
| temperature | `{{ value_json[0].temp \| round(1) }}`, unit `°C` |
| wind speed and gust | multiply knots by `1.852` instead of `1.15078`, unit `km/h` |
| pressure | `{{ value_json[0].altim \| round(1) }}`, unit `hPa` |

Also change the units on the three `Selected weather ...` sensors to match.

## Troubleshooting

- **Sensor is `unknown` or missing:** wait one 10-minute poll, then check **Settings → System → Logs** for `rest`. Make sure you restarted (a reload won't load a new `rest:` block) and that the station helper contains a valid code.
- **Config check fails:** make sure `rest:` and `template:` start at the left edge, use spaces and no tabs, and that you haven't duplicated an existing top-level key.
- **Wrong numbers:** compare `sensor.metar_report_time` with the current time to see how old the METAR is.

## License

MIT License

Copyright (c) 2026 geekmoto13

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
