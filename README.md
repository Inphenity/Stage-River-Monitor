# STAGE - River Gauge E Paper Display and Configuration Tool

STAGE generates a ready-to-paste install script for turning a Raspberry Pi and a small e-paper display into a dedicated, always-on river gauge monitor. Pick your USGS gauge, tune a few display options, and get one script that sets up everything — no coding required.

**Live configurator:** [inphenity.github.io/Stage-River-Monitor](https://inphenity.github.io/Stage-River-Monitor/) — or open `index.html` directly from this repo.

## What it builds

A small standalone device that:

- Pulls live data — discharge (cfs) or gauge height (ft), your choice — from a USGS gauge, using USGS's free public API (no signup or API key needed)
- Displays the current reading, a "last updated" timestamp, a rising/falling trend arrow, and a 48-hour trend graph
- Refreshes automatically on a schedule you set, only during the hours you want it awake
- Can run on battery power for a portable setup, with power-saving options built in
- Optionally joins additional WiFi networks (home, work, a mobile hotspot) so it's not tied to a single network

## Parts list

| Part | Notes |
|---|---|
| **Raspberry Pi Zero WH, Zero 2 W, or Zero 2 WH** | The "W" means WiFi built in; the "H" means the 40-pin GPIO header comes pre-soldered, which you need for the e-paper HAT to plug in directly (non-H boards need the header soldered on separately). The original Zero W/WH only supports 32-bit OS (its chip is ARMv6). The Zero 2 W/WH supports both 32-bit and 64-bit, and is otherwise a drop-in upgrade — same header layout, same install script — but its quad-core chip draws more power, so expect shorter runtime on the same battery. |
| **Waveshare 2.13" e-Paper HAT** | Specifically the revision using the `epd2in13_V4` driver. Waveshare has shipped a few hardware revisions (V2/V3/V4) with slightly different driver requirements — check the sticker on the back of your display's PCB against what you actually received. |
| **MicroSD card** | 8GB or larger. Runs Raspberry Pi OS Lite — 32-bit on an original Zero W/WH, either 32-bit or 64-bit on a Zero 2 W/WH. No desktop environment needed, everything here is headless. |
| **Power source** | A standard 5V USB power adapter works fine for a permanent, plugged-in installation. |
| **(Optional) Battery pack** | A PiSugar-series battery HAT (S, 2, or 3) lets the whole thing run cordless for several hours per charge. The PiSugar S is the simplest and cheapest option, but has no way to report its own battery level back to the display, and its button (if you want to wire one up for manual refresh) requires a small GPIO polling script rather than a built-in config option. PiSugar 2/3 have an onboard RTC and a proper software API for both battery level and button config, at a higher price point. |
| **(Optional) Case** | Any case that exposes the e-paper display and leaves the micro-USB/USB-C power port accessible works. Not required for a first build. |

## Setting up the Raspberry Pi

If you're starting from a brand-new SD card:

1. Flash **Raspberry Pi OS Lite** onto the microSD card using [Raspberry Pi Imager](https://www.raspberrypi.com/software/) — 32-bit if you're using an original Zero W/WH, either 32-bit or 64-bit if you're using a Zero 2 W/WH. In the Imager's advanced options (gear icon), set a hostname, enable SSH, and set your WiFi network and password so the Pi boots straight onto your network.
2. Insert the SD card, power on the Pi, and give it a minute or two to boot.
3. Find its IP address (check your router's device list, or try `ping raspberrypi.local` if your network supports mDNS) and SSH in: `ssh username@<ip-address>`.
4. **Don't attach the e-paper HAT yet** — install the software first (below), then attach it just before the test run at the end of the install script.

## How to use the configurator

1. Open the configurator page (`index.html` in this repo, or your GitHub Pages link) in any browser.
2. Fill in a display label and your USGS gauge's site number — find yours at [waterdata.usgs.gov/nwis/rt](https://waterdata.usgs.gov/nwis/rt); it's the number in the gauge's URL.
3. Pick your measurement type — **discharge (cfs)** or **gauge height (ft)**. Not every gauge reports both, so check your specific gauge's page first.
4. Adjust the toggles for trend arrow, display orientation, refresh schedule, active hours, and power-saving options to fit your setup. The live preview panel updates as you go.
5. If needed, fill in the optional WiFi section to have the script join an additional network (only necessary if the Pi doesn't already have working WiFi, or you want to add a secondary network like a work WiFi or mobile hotspot).
6. Copy the generated script with the **Copy script** button.
7. Paste it into your SSH session on the Pi and press enter. It installs dependencies, pulls the Waveshare e-paper library, writes the display script, and sets up the cron schedule — all in one go.
8. If this is the first time enabling SPI on this Pi, reboot once (`sudo reboot`) after the script finishes.
9. Attach the e-paper HAT to the Pi's 40-pin header now, if you haven't already — it only fits one way.
10. Re-run the test manually to confirm the display updates: `python3 ~/river_display/river_display.py`

## Attaching the e-paper HAT

There's no wiring to do — the HAT plugs directly onto the Pi's 40-pin GPIO header and communicates over SPI. Line up the header carefully (it only fits one orientation) and press down evenly on all four corners to make sure every pin makes contact. A HAT that isn't fully seated is one of the most common causes of a "nothing happens" first run.

## Troubleshooting

- **Display doesn't update:** check `~/river_display/log.txt` on the Pi first — the script logs any error there instead of failing silently.
- **`ModuleNotFoundError: No module named 'waveshare_epd'`:** the Waveshare library clone didn't complete. Run `rm -rf ~/river_display/e-Paper` and re-run the install script.
- **Ghosting (faint traces of the old reading left on screen):** the display defaults to a full refresh every update to avoid this, at the cost of a brief flash each time. This is a known trait of e-paper, not a bug.
- **`RuntimeError: Failed to add edge detection` (only relevant if you wire up a manual refresh button):** newer Raspberry Pi OS releases (Debian 13 "Trixie" and later) have moved away from the legacy GPIO interface that `RPi.GPIO`'s interrupt-based edge detection depends on. A simple polling loop works around this reliably.
- **Occasional `Temporary failure in name resolution` errors:** this is a DNS/network hiccup, not a USGS problem — usually transient. If it happens right after a scheduled WiFi reconnect, add a few minutes' buffer between when WiFi comes back and when the first refresh of the day is scheduled.
- **The reading only changes once an hour despite refreshing more often:** this is normal for some gauges. USGS gauges often *measure* every 15 minutes but only *transmit* to USGS's servers on an hourly basis — refreshing more frequently than that won't get you anything newer.

## Notes

- USGS real-time data is **provisional** — it hasn't gone through USGS's quality-review process yet. This is normal for real-time data and applies to every reading this project displays.
- This was built and tested against Raspberry Pi OS Lite (Trixie/Debian 13, 32-bit) on a Pi Zero WH with a Waveshare 2.13" V4 display. A Pi Zero 2 W/WH running the 64-bit build of the same OS should work identically — everything this project uses (Python, PIL, `requests`, `RPi.GPIO`, `spidev`, the Waveshare library) has 64-bit ARM builds available — but this specific combination hasn't been directly verified.
- The configurator runs entirely in your browser — nothing you type into it (site numbers, WiFi passwords, anything) is sent anywhere or stored by the tool itself. If you save a copy of your own generated script somewhere for backup, keep it private rather than committing it to a public repo, since it may contain a real WiFi password in plain text.
