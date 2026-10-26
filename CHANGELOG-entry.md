## [Patch] - 2026-10-26

### Fixed
- **Startup error spam from the "Selected weather" template sensors.** When neither the NWS entity nor the METAR sensors had a value yet (typically right after a restart), the sensors output the text `unknown`, and Home Assistant logged `Received invalid sensor state: unknown ... expected a number`. Each selected-weather sensor now has an `availability` template, so "no data yet" shows as `unavailable` instead.
- **Template errors when aviationweather.gov returns an error object.** An error response is a dictionary, not a list, so `value_json[0]` raised `dict object has no element 0`. Every METAR REST sensor now checks `value_json is not mapping` before indexing.

### Behavior change
- If the selected source has no data, the selected-weather sensors now report `unavailable` instead of `unknown`. Automations or cards that test for `unknown` specifically should also handle `unavailable`.
- In `METAR` or `NWS` mode there is no fallback to the other source (unchanged). Only `Auto` falls back.

### Upgrading
1. Back up your current config.
2. Replace the `rest:` and `template:` blocks with the ones in `metar_backup.yaml`, or apply `fix-availability-and-api-guard.patch`.
3. Keep your own NWS weather entity ID in place of `weather.YOUR_NWS_ENTITY`.
4. Restart Home Assistant. A Template reload does not reload `rest:` sensors.
