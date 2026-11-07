# Home Assistant Tornado Alert

A tornado warning system for Home Assistant that tells people what they actually need to know: how long the warning lasts, whether a tornado is confirmed or only radar-indicated, where the storm is, which way it's moving, and how many minutes away it is. It stops nagging once everyone is safe, and it gets loud and blunt only when the storm is right on top of you.

Example announcement:

> "Emergency. Tornado warning for our area until 3 PM. Radar shows rotation. No tornado confirmed yet. The storm is 9 miles west, heading toward the house, about 18 minutes away. Go to the lowest interior room of the house now. When everyone is safe, tell the speaker: turn on shelter mode."

## What it does

1. **Tornado warning.** Every speaker announces at the same time and you get a phone push that rings through Do Not Disturb. The announcement repeats every 5 minutes.
2. **Shelter mode.** This is a toggle you can turn on by voice or with a dashboard button that only shows during a warning. It stops the repeats. Real changes still get announced: a new end time, a confirmed tornado, a second warning, or a tornado emergency. It resets itself when the warning ends.
3. **Take shelter now.** This kicks in when the storm is within 3 miles, or is heading at the house and 10 minutes or less away. It uses very different, blunt wording, said twice.
   - All lights go to 100% so people can see.
   - Light automations are paused, but only the ones that were on.
   - When the storm passes, the lights go back exactly how they were, and only those same automations are turned back on.
4. **All clear.** It announces "the threat has passed, the warning officially ends at 3 PM", then sends a final "it's done" once the warning is off the list.
5. **Email (optional).** Every update is easy to read: the status line first, then distance and direction, then the full weather service text.
6. **Watches** get an email only, with no speakers.

## Requirements

- **[NWS Alerts](https://github.com/finity69x2/nws_alerts)** (HACS). Set it up with your **GPS location**, not your county, so a warning means your house is inside the warning area. The code expects the sensor `sensor.nws_alerts_alerts`.
- **US only.** This uses National Weather Service data.
- **TTS on your speakers.** The announcer uses `tts.cloud_say` (Nabu Casa). With another TTS engine, swap that action.
- **Optional:** [Bubble Card](https://github.com/Clooos/Bubble-Card) (HACS) for the dashboard button, and the SMTP integration for email.

## Setup

### 1. Helpers

Create these in Settings → Devices & services → Helpers. The name produces the entity ID the code expects.

| Type | Name | Entity ID |
|---|---|---|
| Toggle | Tornado warning active | `input_boolean.tornado_warning_active` |
| Toggle | Shelter mode | `input_boolean.shelter_mode` |
| Toggle | Tornado take shelter | `input_boolean.tornado_take_shelter` |
| Text (max length **255**) | Tornado paused automations | `input_text.tornado_paused_automations` |
| Text (max length **255**) | Tornado storm info | `input_text.tornado_storm_info` |

### 2. rest_command

Add `config/rest_command.yaml` to your `configuration.yaml` and restart. This lets HA open the full alert, which holds the storm position, speed, direction and tornado detection.

### 3. Automations

Create three automations and paste each file in YAML mode:

- `automations/tornado_alert.yaml` is the main logic.
- `automations/tornado_alert_announcer.yaml` holds your speakers.
- `automations/tornado_alert_clock.yaml` is a once-a-minute tick that only runs during a warning, so it doesn't fill the main automation's traces.

### 4. Edit the placeholders

Edit these in `tornado_alert.yaml`, under `variables:` at the top:

- `email_targets`: your SMTP notify entity, or `[]` for no email
- `safe_room`: where people should go, for example "the downstairs closet"
- `all_lights`: the lights to turn on at take shelter
- `pause_names`: the **names** of your light automations, as shown in the automation list
- `notify.mobile_app_YOUR_PHONE`: search and replace this with your phone

In `tornado_alert_announcer.yaml`, make one step per speaker.

### 5. Voice and dashboard

- Expose `input_boolean.shelter_mode` to your voice assistant so people can say "turn on shelter mode".
- Add `dashboard/shelter_mode_card.yaml` as a card. It only shows during a warning.

## Testing

- **Announcer:** go to Developer Tools → Events, fire `tornado_announce` with `message: test` and `times: 1`.
- **Storm math:** paste `templates/storm_math_test.jinja` into Developer Tools → Template and change the sample numbers.
- **Full run:** this needs a real warning. The weather service doesn't send test tornado warnings to the alerts feed.

## How the storm tracking works

The weather service resends the storm position with each update, about every 15–20 minutes. Between updates the automation moves the storm forward along its track using time, speed and heading. Then it compares the storm against `zone.home`:

- distance and compass direction from the house
- whether the storm is coming toward you, passing to the side (and how close), or moving away
- minutes until it's closest

It uses flat-earth math, which is fine at these distances. The "from" direction becomes a heading (+180°). The automation takes the east/north offset in miles, uses a dot product for "ahead of the storm" and a cross product for "how close the path passes".

To change the thresholds, edit the `imm` line inside the `storm` variable (3 miles / 10 minutes). For the repeat timing, edit the "Every 5th minute" condition.

## Gotchas this handles

- **Re-announcing the ending message.** The weather service sends updates with new IDs that are still labeled "Tornado Warning", including the one that ends it. Its headline says "WILL EXPIRE" or "CANCEL". The automation filters those out, so the end message doesn't re-trigger the warning.
- **Never say "Google" in TTS.** Even without "Hey", it can set off Google speakers. Voice hints say "tell the speaker: ..." instead.
- **One action per speaker, all in parallel,** so one dead speaker doesn't hold up the rest.
- **Power outages.** Saved light scenes (`scene.create`) don't survive an HA restart. The list of paused automations is kept in a text helper, so they still come back. If the saved lights were lost, you get a push instead.
- **Spook** may flag `scene.before_tornado_state` as missing, because it only exists during a take-shelter. The code builds that name with a template to avoid the warning.

## Limits

- The storm position is the weather service's estimate, and storms turn. Treat distance and minutes as a guide, not a promise.
- Most tornado warnings (roughly 7 in 10) never produce a tornado. That's why it says "radar indicated" vs "confirmed" out loud.
- Pausing an automation also cancels anything it was in the middle of. If a motion light was counting down to turn off when take shelter started, that countdown is lost. After the lights are put back, that light stays on until its automation fires again.
- This is a helper, not a replacement for a NOAA weather radio or the alerts on your phone.
