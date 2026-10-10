# Twilight-Networks fork: customizations

This fork tracks upstream **v1.2.0** and re-implements the Twilight-Networks
customizations on top of the upstream architecture (no blind cherry-picks of the
old `feature/twilight-customizations` branch). Package and boot sound are
pulled from this repository (`Twilight-Networks/esphome-waveshare-esp32-s3-audio-va`,
branch `update/v1.2.0-custom`).

Everything below is implemented in `base/core.yaml`; `waveshare-va.yaml` only
points the `packages:` block at the fork.

## 1. What differs from upstream

| Area | Behavior in this fork |
|---|---|
| Package / boot sound | `waveshare-va.yaml` pulls `base/core.yaml` from the Twilight fork (`ref: update/v1.2.0-custom`, switch to the final tag/branch once released). `boot_sound_file` points to `base/sounds/startup.mp3` in the fork. |
| Volume buttons | Key 1 = **Volume Down**, key 3 = **Volume Up** (ids `volume_down_button`, `volume_up_button`). Holding a key auto-repeats: first repeat after 400 ms, then every 150 ms. |
| Play/Pause | Key 2 (`play_pause_button`). Short press = play/pause (fires on **release**). Long press (>= 1.5 s) toggles **Do Not Disturb**. While a timer/alarm is ringing, a press only ends the ringing. |
| Status ring architecture | `status_ring` holds only the base state from Home Assistant / Node-RED. `status_ring_hw` is the physical WS2812 ring. See section 2. |
| Voice assistant color | Blue (`voice_assist_color_r/g/b` = 0.0 / 0.0 / 1.0) instead of upstream violet. Timer colors stay violet. |
| Replying effect | Default of the `replying_effect` select is **Spinner** (applies on a fresh flash only, the select is restored afterwards). |
| Do Not Disturb | New switch `do_not_disturb` (restored, default off): hard quiet mode, see section 5. |
| Ring feedback | Short overlays on the ring: volume (1200 ms, green fast pulse), mic mute (250 ms, solid red), DND on/off acknowledgement (700 ms, 600 ms fade). |
| Night Mode | Unchanged upstream feature (switch + brightness number). Separate from DND and **lower priority** than DND. |

## 2. Architecture decisions

- `status_ring` is a template RGB light on no-op outputs. It stores **only** the
  base state that Home Assistant / Node-RED sets. It never drives LEDs directly.
  `restore_mode: ALWAYS_OFF`: no persistence, Node-RED is the source of truth after a boot.
- `status_ring_hw` is the physical `esp32_rmt_led_strip` light (internal).
- `led_set_effect(effect, r, g, b, scale, transition_ms)` is the **only** writer
  of `status_ring_hw`.
- `control_leds` is the central priority state machine. It reads the base state
  from `status_ring.remote_values` and decides what the hardware shows.
- `show_ring_feedback(type, duration_ms)` (script mode `restart`) runs temporary
  overlays and then lets `control_leds` re-resolve. Overlays never overwrite the
  base state, so the exact previous base state returns afterwards.
- A DND acknowledgement cannot be interrupted by volume/mic feedback (guards live
  in the wrappers, because a `restart` script would kill a running ack before an
  in-script check).

## 3. Old customizations deliberately not adopted

- **`channel_colors` change**: obsolete, upstream v1.2.0 already contains it.
- **Old wake-word model change**: not adopted. The model list is upstream
  (alexa, okay_nabu, hey_jarvis, hey_mycroft).
- **Old special logic for the waiting state**: not adopted. Upstream v1.2.0
  shares the Listening effect between waiting and listening.
- **Direct LED manipulation from old commits**: replaced by the
  `status_ring` / `status_ring_hw` / `led_set_effect` architecture.

## 4. LED ring priority (highest first)

| # | State | Notes |
|---|---|---|
| 1 | DND acknowledgement | brief on/off feedback (types 3/4) |
| 2 | DND hard off | ring fully dark |
| 3 | Temporary feedback | volume (1), mic mute (2) |
| 4 | BLE / Improv | |
| 5 | Boot | |
| 6 | Wi-Fi / API offline | |
| 7 | Timer / alarm ringing | |
| 8 | Voice assistant phases | waiting+listening, thinking, replying, error (red fast pulse), not ready (red slow pulse) |
| 9 | Timer countdown | only if "Timer countdown on ring" is on |
| 10 | Base state | from Home Assistant / Node-RED (`status_ring`) |
| 11 | Off | |

Night Mode only scales brightness inside levels 3 to 10; it never replaces DND.

## 5. Do Not Disturb

Toggle: HA switch `Do Not Disturb`, or **long press (>= 1.5 s) on Play/Pause**.

While DND is on:

- **Ring**: fully dark. No VA display, no boot/offline/error display, no timer
  display, no night-mode dimming as a substitute. Only the brief on/off
  acknowledgement (700 ms) is shown.
- **Audio**: amplifier forced off, TTS/announcements silent, local sounds
  (`play_sound`, boot chime, timer/alarm) suppressed. Music may continue
  internally but is silent; the boot chime is consumed and never played later.
- **Wake word**: stays active. Voice commands are still processed; there is
  only no audible or visible reply.
- **Enabling DND** ends a running announcement and an already ringing
  timer/alarm immediately (ring script stopped, ducking fully reset).
- **Disabling DND**: the amplifier is switched back on only if playback or an
  announcement is active; `control_leds` then re-resolves the ring.

### Timer and alarm under DND

- A VA timer that finishes during DND: no sound, no ring, no persistent
  `timer_ringing`, no persistent ducking, and it **never rings later** after DND is turned off.
- Local alarm: "Play sound" is fully suppressed; "Sound and event" suppresses
  the sound but still sends the event; "Send event" is unchanged. No late local start.
- DND does not delete countdown timers. A timer that expires **after** DND was
  switched off rings normally.
- Safety net in `timer_ringing.on_turn_on` stops the ringing if DND is on.

### Play/Pause key

- Timer/alarm ringing: a press ends the ringing. No DND toggle, even if the key
  is held for a long time.
- Not ringing: short press = play/pause on release; hold for 1.5 s = toggle DND,
  and no extra play/pause on release.
- Each physical press is evaluated once (press counter + handled flag + 1500 ms
  watch script), so two quick short presses are never a long press and an old
  delayed long-press can never fire after a later press.

## 6. Final test matrix

Mark each line after testing on hardware. "Return" means the exact previous base state is shown again.

| # | Test | Expected result |
|---|---|---|
| 1 | Set base color from HA/Node-RED | Ring shows it |
| 2 | Overlay during base state (volume, mic) | Overlay shows, then base state returns |
| 3 | Volume Up/Down short press | Volume changes, green pulse 1.2 s, base returns |
| 4 | Volume key held | Auto-repeat 400 ms, then 150 ms; ring feedback stays active |
| 5 | Mic mute on/off | Red 250 ms feedback, base returns |
| 6 | Night Mode on/off | Ring dimmed to night level, off restores normal level |
| 7 | DND on (HA) | Short ack, ring dark, amp off |
| 8 | DND off (HA) | Short ack, base state returns |
| 9 | Wake word under DND | Wake word detected, command executed, no sound, ring dark |
| 10 | Music + DND on / off | Silent while on; audible again after off, ring correct |
| 11 | DND activated during TTS | TTS ends immediately, silent |
| 12 | Reboot with DND on | No boot chime (now or later), ring dark, DND still on |
| 13 | Timer expires under DND | No sound, no ring, no ringing afterwards |
| 14 | Timer expires after DND off | Rings normally |
| 15 | DND turned on while timer rings | Ringing ends at once, ring dark, no ducking left |
| 16 | Play/Pause short | Play/pause on release |
| 17 | Play/Pause long (>= 1.5 s) | DND toggles, ack shown, no play/pause on release |
| 18 | Timer ringing + Play/Pause (short and long) | Ringing ends, DND not toggled |
| 19 | Two quick Play/Pause clicks | Two play/pause actions, no DND toggle |
| 20 | Volume auto-repeat then release | Stops cleanly, no extra step |
| 21 | Wi-Fi / API offline | Offline display; hidden under DND; recovers afterwards |
| 22 | Error state (VA error) | Red fast pulse; hidden under DND |
| 23 | Boot behavior | Boot sequence, then dark ring until Node-RED sets the base state |

## 7. Known open polish point

Pulse and fade effects look somewhat choppy. This is not a functional flaw of
the base architecture and will be optimized separately.

Other notes: short Play/Pause fires on release (no debounce filter yet; an
optional `delayed_on_off: 20ms` could be added); the mic-mute flash uses the
current normal/night ring level.
