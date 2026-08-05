# STAGE - River Gauge E Paper Display and Configuration Tool

STAGE generates a ready-to-paste install script for turning a Raspberry Pi and a small e-paper display into a dedicated, always-on river gauge monitor. Pick your USGS gauge(s), tune a few display options, and get one script that sets up everything — no coding required.

**Live configurator:** [inphenity.github.io/Stage-River-Monitor](https://inphenity.github.io/Stage-River-Monitor/) — or open `index.html` directly from this repo.

## What it builds

A small standalone device that:

- Pulls live data — discharge (cfs) or gauge height (ft), your choice — from a USGS gauge, using USGS's free public API (no signup or API key needed)
- Displays the current reading, a "last updated" timestamp, a rising/falling trend arrow (comparing against the reading from 12 hours ago, so it catches slow drift rather than just short-term noise), and a 48-hour trend graph
- Refreshes on whatever schedule you set — pick a quick "N times per hour" preset, add fully custom minutes, offset everything by +2 minutes to dodge USGS's publish timing, or skip quiet hours entirely and run 24/7 if it's plugged in rather than on battery
- Can run on battery power for a portable setup, with power-saving options built in
- Optionally joins as many additional WiFi networks as you want (home, work, a mobile hotspot, a backup) — note the Pi's radio is 2.4GHz-only, it can't join 5GHz networks
- **New:** on the 2.7" display, tracks up to 4 gauges at once and switches between them with the onboard buttons

## Two display modes

STAGE now supports two Waveshare e-paper HATs, selected in the configurator:

| Mode | Display | Gauges | How it runs |
| ---- | ------- | ------ | ----------- |
| **Single-gauge (default)** | Waveshare 2.13" e-Paper HAT | 1 | A script invoked on a cron schedule, prints and exits |
| **Multi-gauge** | Waveshare 2.7" e-Paper HAT (264×176, 4 onboard buttons) | Up to 4, one per button | A `systemd` service that runs continuously — refreshing data on a schedule in the background, and switching the displayed gauge instantly when you press a button |

Pick the one that matches the hardware you have. STAGE generates a completely different install script depending on which you choose — the multi-gauge script installs a background service instead of a cron job, since it needs to be listening for button presses at all times, not just waking up periodically.

## Parts list

| Part                                             | Notes                                                                                                                                                                                                    |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Raspberry Pi Zero WH, Zero 2 W, or Zero 2 WH** | The "W" means WiFi built in; the "H" means the 40-pin GPIO header comes pre-soldered, which you need for the e-paper HAT to plug in directly (non-H boards need the header soldered on separately). The original Zero W/WH only supports 32-bit OS (its chip is ARMv6). The Zero 2 W/WH supports both 32-bit and 64-bit, and is otherwise a drop-in upgrade — same header layout, same install script — but its quad-core chip draws more power, so expect shorter runtime on the same battery. |
| **Waveshare 2.13" e-Paper HAT** (single-gauge mode) | Specifically the revision using the `epd2in13_V4` driver. Waveshare has shipped a few hardware revisions (V2/V3/V4) with slightly different driver requirements — check the sticker on the back of your display's PCB against what you actually received. |
| **Waveshare 2.7" e-Paper HAT** (multi-gauge mode) | 264×176, uses the `epd2in7` driver, with 4 onboard buttons (KEY1-KEY4). Button GPIO pins vary slightly by hardware revision — double-check the schematic on the Waveshare wiki against your board before your first run. |
| **MicroSD card**                                 | 8GB or larger. Runs Raspberry Pi OS Lite — 32-bit on an original Zero W/WH, either 32-bit or 64-bit on a Zero 2 W/WH. No desktop environment needed, everything here is headless.                        |
| **Power source**                                 | A standard 5V USB power adapter works fine for a permanent, plugged-in installation.                                                                                                                     |
| **(Optional) Battery pack**                      | A PiSugar-series battery HAT (S, 2, or 3) lets the whole thing run cordless for several hours per charge. The PiSugar S is the simplest and cheapest option, but has no way to report its own battery level back to the display, and its button (if you want to wire one up for manual refresh) requires a small GPIO polling script rather than a built-in config option. PiSugar 2/3 have an onboard RTC and a proper software API for both battery level and button config, at a higher price point. |
| **(Optional) Case**                              | Any case that exposes the e-paper display and leaves the micro-USB/USB-C power port accessible works. Not required for a first build.                                                                    |

## Setting up the Raspberry Pi

If you're starting from a brand-new SD card:

1. Flash **Raspberry Pi OS Lite** onto the microSD card using [Raspberry Pi Imager](https://www.raspberrypi.com/software/) — 32-bit if you're using an original Zero W/WH, either 32-bit or 64-bit if you're using a Zero 2 W/WH. In the Imager's advanced options (gear icon), set a hostname, enable SSH, and set your WiFi network and password so the Pi boots straight onto your network.
2. Insert the SD card, power on the Pi, and give it a minute or two to boot.
3. Find its IP address (check your router's device list, or try `ping raspberrypi.local` if your network supports mDNS) and SSH in: `ssh username@<ip-address>`.
4. **Don't attach the e-paper HAT yet** — install the software first (below), then attach it just before the test run at the end of the install script.

## How to use the configurator

1. Open the configurator page (`index.html` in this repo, or your GitHub Pages link) in any browser.
2. Under **Display hardware**, pick your HAT: 2.13" (single gauge) or 2.7" (multi-gauge, 4 buttons).
3. **If you picked 2.13":** fill in a display label and your USGS gauge's site number — find yours at [waterdata.usgs.gov/nwis/rt](https://waterdata.usgs.gov/nwis/rt); it's the number in the gauge's URL. Pick discharge (cfs) or gauge height (ft) — not every gauge reports both, so check your specific gauge's page first.
4. **If you picked 2.7":** fill in the **Gauges & button mapping** section instead — one row per button, each with its own label, USGS site number, and measurement. Leave a button's site number blank to leave it unused.
5. Adjust the toggles for trend arrow, display orientation, and power-saving options to fit your setup. For the refresh schedule, either click one of the "refreshes per hour" quick-set buttons (which fills in evenly spaced minutes for you) or build your own list by adding individual minutes — each is removable, and there's a "+2 minute" offset toggle if you want refreshes to land a couple minutes after the schedule shown, to dodge USGS's own publish timing. Set active hours, or check **Run 24/7** to skip quiet hours entirely (only worth it if the Pi's plugged into wall power — it'll drain a battery noticeably faster). The live preview panel updates as you go.
6. If needed, use **+ Add WiFi network** to have the script join one or more additional networks (only necessary if the Pi doesn't already have working WiFi, or you want to add a secondary network like a work WiFi or mobile hotspot) — add as many as you want, each with its own SSID and password. Remember the Pi's WiFi radio is 2.4GHz-only, so a 5GHz-only network won't be visible to it.
7. Copy the generated script with the **Copy script** button.
8. Paste it into your SSH session on the Pi and press enter.
   - **2.13" mode:** it installs dependencies, pulls the Waveshare e-paper library, writes the display script, and sets up the cron schedule — all in one go.
   - **2.7" mode:** it does the same, but instead of a cron job it installs and starts a `systemd` service (`river-display.service`) that runs continuously in the background.
9. If this is the first time enabling SPI on this Pi, reboot once (`sudo reboot`) after the script finishes.
10. Attach the e-paper HAT to the Pi's 40-pin header now, if you haven't already — it only fits one way.
11. **2.13" mode:** re-run the test manually to confirm the display updates: `python3 ~/river_display/river_display.py`
    **2.7" mode:** confirm the service is running (`sudo systemctl status river-display`), then press each configured button and confirm the display switches to that gauge.

## Attaching the e-paper HAT

There's no wiring to do — the HAT plugs directly onto the Pi's 40-pin GPIO header and communicates over SPI. Line up the header carefully (it only fits one orientation) and press down evenly on all four corners to make sure every pin makes contact. A HAT that isn't fully seated is one of the most common causes of a "nothing happens" first run.

## Multi-gauge mode (2.7" HAT)

Each of the 4 onboard buttons is wired to its own gauge. A **short press** switches to that gauge and redraws instantly from whatever's already cached — no network call, no delay. **Press and hold** a button (about 1.2 seconds) to force a live USGS fetch for that gauge on demand, in addition to the background service's normal scheduled refresh (same minute/active-hours settings as single-gauge mode). If a forced fetch fails — a dropped connection, a slow USGS response — the display falls back to the last successfully cached reading for that gauge instead of showing an error or blank screen.

The service also forces a fetch and redraw once at startup — but rather than firing immediately (which can race a Pi Zero's WiFi still associating right after boot), it retries with backoff until at least one configured gauge actually succeeds, so power-on reliably shows current data instead of possibly sitting on "No data yet" until the next scheduled refresh. The single-gauge (2.13") install adds an equivalent `@reboot` cron entry with its own short retry loop, since that path is a one-shot script rather than a persistent service.

If you're holding a button repeatedly for testing, keep in mind each hold is its own request to USGS's public API — reasonable for normal use, but avoid scripting rapid repeated holds as a matter of general courtesy to a free public service.

**Button GPIO pins (BCM numbering):**

| Button | GPIO (BCM) |
| ------ | ---------- |
| KEY1   | 5          |
| KEY2   | 6          |
| KEY3   | 13         |
| KEY4   | 19         |

These match Waveshare's own 2.7" demo scripts, but Waveshare has changed pinouts across hardware revisions before — cross-check against the schematic on [waveshare.com/wiki/2.7inch_e-Paper_HAT](https://www.waveshare.com/wiki/2.7inch_e-Paper_HAT) for your specific board before relying on them.

**Why it's a service instead of a cron job:** the single-gauge script only needs to wake up, draw, and exit — cron handles that fine. Multi-gauge mode needs to react to a button press at any moment, which means something has to be running continuously to watch the GPIO pins. STAGE handles this by generating a `systemd` service (`river-display.service`) instead of a cron entry, so it starts on boot and restarts automatically if it ever crashes.

**Button reading uses polling, not interrupts:** newer Raspberry Pi OS releases (Debian 13 "Trixie" and later) have moved away from the legacy GPIO interface that `RPi.GPIO`'s interrupt-based edge detection (`add_event_detect`) depends on — the same underlying issue noted in Troubleshooting below for a single manual-refresh button. The generated multi-gauge script sidesteps this by polling the button pins in a tight loop instead, which is more CPU-active but works reliably across OS versions.

## Troubleshooting

- **Display doesn't update:** check `~/river_display/log.txt` on the Pi first — the script logs any error there instead of failing silently.
- **`ModuleNotFoundError: No module named 'waveshare_epd'`:** the Waveshare library clone didn't complete. Run `rm -rf ~/river_display/e-Paper` and re-run the install script.
- **Ghosting (faint traces of the old reading left on screen):** the display defaults to a full refresh every update to avoid this, at the cost of a brief flash each time. This is a known trait of e-paper, not a bug.
- **`RuntimeError: Failed to add edge detection` (only relevant if you wire up a manual refresh button):** newer Raspberry Pi OS releases (Debian 13 "Trixie" and later) have moved away from the legacy GPIO interface that `RPi.GPIO`'s interrupt-based edge detection depends on. A simple polling loop works around this reliably.
- **Occasional `Temporary failure in name resolution` errors:** this is a DNS/network hiccup, not a USGS problem — usually transient. If it happens right after a scheduled WiFi reconnect, add a few minutes' buffer between when WiFi comes back and when the first refresh of the day is scheduled.
- **The reading only changes once an hour despite refreshing more often:** this is normal for some gauges. USGS gauges often *measure* every 15 minutes but only *transmit* to USGS's servers on an hourly basis — refreshing more frequently than that won't get you anything newer.

**Multi-gauge mode (2.7") specific:**

- **A button press doesn't switch gauges:** first confirm that button's site number was actually filled in during configuration — an unmapped button intentionally does nothing. If it was configured, check `~/river_display/log.txt`, then verify the GPIO pin for that button against your board's schematic — see the pin table above.
- **Display doesn't come back after a reboot:** check the service status with `sudo systemctl status river-display`. If it shows as failed, `journalctl -u river-display -n 50` will show the most recent errors.
- **A button shows "not configured":** that button's site number field was left blank in STAGE. Re-run the configurator with that row filled in and re-paste the install script.
- **Buttons feel unresponsive or laggy:** a short press should feel instant (it's reading straight from cache). A pause is expected on a *hold*, since that's a live network fetch, not a bug. If a hold is consistently slow (multiple seconds) or fails outright, check `~/river_display/log.txt` for `network/API error` entries.

## Notes

- USGS real-time data is **provisional** — it hasn't gone through USGS's quality-review process yet. This is normal for real-time data and applies to every reading this project displays.
- The single-gauge path was built and tested against Raspberry Pi OS Lite (Trixie/Debian 13, 32-bit) on a Pi Zero WH with a Waveshare 2.13" V4 display. A Pi Zero 2 W/WH running the 64-bit build of the same OS should work identically — everything this project uses (Python, PIL, `requests`, `RPi.GPIO`, `spidev`, the Waveshare library) has 64-bit ARM builds available — but this specific combination hasn't been directly verified.
- The multi-gauge path (2.7" HAT, button mapping, systemd service) is new and has not yet been verified on hardware — it's generated from the same tested USGS-fetch and e-paper-drawing logic as the single-gauge path, but the button-polling and service wiring are new additions.
- The configurator runs entirely in your browser — nothing you type into it (site numbers, WiFi passwords, anything) is sent anywhere or stored by the tool itself. If you save a copy of your own generated script somewhere for backup, keep it private rather than committing it to a public repo, since it may contain a real WiFi password in plain text.
