# STAGE — River Gauge E-Paper Display

Turn a Raspberry Pi and a small e-paper display into a dedicated river
gauge monitor: live discharge or gauge height, a 48-hour trend line, and
(on the bigger display) multiple gauges you can flip between with a
button press. Data comes straight from USGS — no signup, no API key.
Once it's running, you can change the gauge, flip the orientation, or
even join a new WiFi network anytime from a browser — no SSH required —
and the display itself shows the last 3 digits of the Pi's IP so it's
easy to find on your network.

This guide assumes you've never set up a Raspberry Pi before. By the
end you'll have a working display on your desk.

**<a href="https://inphenity.github.io/Stage-River-Monitor/" target="_blank" rel="noopener">→ Open the setup tool</a>**
to generate your install script — you'll come back to that in Step 4.

---

## What you'll need

**Hardware**
- A Raspberry Pi with a 40-pin GPIO header (a **Pi Zero 2 WH** is the
  best pick if you're new to this — the "H" means the header pins come
  pre-soldered, so the e-paper HAT just pushes on with no soldering
  required. The plain Pi Zero 2 W is otherwise identical but needs the
  header soldered on yourself. A Pi 3/4/5 works too, just overkill)
- A micro SD card, 8GB or larger
- One of:
  - [Waveshare 2.13" e-Paper HAT](https://www.waveshare.com/2.13inch-e-paper-hat.htm) — single gauge, simplest setup
  - [Waveshare 2.7" e-Paper HAT](https://www.waveshare.com/2.7inch-e-paper-hat.htm) — up to 4 gauges, switchable with onboard buttons
- *(Optional)* A [PiSugar](https://www.pisugar.com/) battery HAT, if you want it portable/battery-powered
- A computer to flash the SD card and to SSH in from
- A USB power supply (or the PiSugar battery) for the Pi

**You do NOT need:** a monitor, keyboard, or mouse for the Pi itself.
Everything below is done "headless," over the network.

---

## Step 1 — Flash the SD card

1. Download and install [Raspberry Pi Imager](https://www.raspberrypi.com/software/) on your computer.
2. Insert the micro SD card.
3. Open Raspberry Pi Imager:
   - **Device**: pick your Pi model
   - **Operating System**: "Raspberry Pi OS (other)" → **Raspberry Pi OS Lite (64-bit)** — you don't need the desktop version for this
   - **Storage**: select your SD card
4. Click the **gear icon** (⚙) in the bottom right — or press `Ctrl+Shift+X` — to open advanced options. This is the important part: it lets you set everything up before the Pi ever boots, so you never need to plug in a monitor.
   - ✅ Set hostname (e.g. `rivermonitor` — you'll use this to find it later)
   - ✅ Enable SSH → "Use password authentication"
   - ✅ Set username and password (remember these!)
   - ✅ Configure WiFi — enter your network's SSID and password, and set the WiFi country to yours
   - ✅ Set locale/timezone if you'd like correct timestamps
5. Save, then click **Write**. This erases the card and writes the OS — confirm when it asks.
6. When it finishes, move the SD card into the Pi — but hold off on
   powering it on until the display (and battery, if using one) is
   wired up in Step 2.

> **Pi Zero W/WH/2W/2WH note:** these only have a 2.4GHz WiFi radio. If
> your network name doesn't show up in the imager's WiFi list, make
> sure you're entering your router's 2.4GHz network — a lot of routers
> broadcast both bands under the same name, which can cause silent
> connection failures on first boot.

---

## Step 2 — Wire up the display (and battery, if using one)

With the Pi still powered **off**, seat the e-paper HAT onto the
40-pin GPIO header — it only fits one way, flush against all 40 pins.
If you're also using a PiSugar, check its instructions for whether it
sits between the Pi and the HAT or stacks separately — this varies by
PiSugar model.

Nothing else to configure here yet — SPI (which the display needs)
gets enabled by the install script in Step 5.

Once everything is seated, power on the Pi. First boot takes 1-2
minutes.

---

## Step 3 — Find the Pi and SSH in

Give the Pi about a minute after powering on, then from your computer:

```
ssh <username>@<hostname>.local
```

using the username and hostname you set in Step 1 (e.g. `ssh
pat@rivermonitor.local`). Type `yes` if asked about the host's
fingerprint, then enter the password you set.

**If that doesn't connect:**
- Double check the Pi has power and its green LED is flickering (that means it's alive and working)
- Try `ping rivermonitor.local` — if that fails, `.local` (mDNS) discovery may not work on your network; instead check your router's admin page for a device named after your hostname and use its IP directly: `ssh <username>@<that-ip-address>`
- Give it another minute — first boot does some one-time setup

Once connected, you're looking at a normal command-line prompt on the
Pi itself. Everything from here happens in this window.

### Using Windows? Here's the PowerShell version

Modern Windows (10 and 11) ships with SSH built into PowerShell — no
extra software needed.

1. Open **PowerShell** (search for it in the Start menu — the regular blue one, not "PowerShell ISE").
2. Run the same command as above:
   ```
   ssh <username>@<hostname>.local
   ```
3. First connection only: PowerShell will ask something like
   `Are you sure you want to continue connecting (yes/no/[fingerprint])?`
   — type `yes` and press Enter.
4. Enter the password you set in Raspberry Pi Imager. **Nothing will
   appear on screen while you type** — no dots, no cursor movement —
   that's normal password-entry behavior in a terminal, not a stuck
   window. Just type it and press Enter.

**If PowerShell says `ssh` isn't recognized as a command:** you're on
an older Windows build without OpenSSH included. Go to
**Settings → Apps → Optional Features → Add a feature**, search for
**OpenSSH Client**, install it, then close and reopen PowerShell and
try again.

**If `<hostname>.local` doesn't resolve:** Windows' support for `.local`
(mDNS) hostnames can be inconsistent depending on version. Fall back to
finding the Pi's IP address from your router's admin page (look for
the hostname you set) and connect with that instead:
```
ssh <username>@<that-ip-address>
```

---

## Step 4 — Generate your install script

Open the **<a href="https://inphenity.github.io/Stage-River-Monitor/" target="_blank" rel="noopener">STAGE setup tool</a>**
in a browser on your computer (not the Pi). It's laid out as a numbered
sequence of sections — work through them top to bottom:

- **00 — Network** — skip this if the Pi's already on WiFi (it is, from Step 1). Add networks here for the Pi to join *in addition* to that one — a work WiFi, a mobile hotspot, a backup network. Also where the **fallback setup hotspot** lives (on by default) — if the Pi ever can't reach any known WiFi network, it broadcasts its own network so you can fix WiFi from a phone instead of re-flashing the SD card. Give it a name and an 8+ character password here; see [If WiFi ever stops working](#if-wifi-ever-stops-working) below for how it's used.
- **01 — Display hardware** — pick your HAT (2.13" or 2.7"), and for the 2.7" confirm the board revision printed on the back (V1 vs V2) — picking the wrong one leaves the panel blank with no error.
- **02 — Station / Gauges** — find your river's USGS site number at [waterdata.usgs.gov/nwis/rt](https://waterdata.usgs.gov/nwis/rt/) (it's the number in that gauge's URL), then choose discharge or gauge height — check your site's page for which one it actually publishes.
- **03 — Display behavior** — defaults are sensible; adjust as you like.
- **04 — Schedule** — set your refresh interval here. Worth turning on "Offset refresh by +2 minutes" — it runs the fetch 2 minutes after each scheduled time instead of exactly on it, which avoids a race if USGS happens to publish its update right at the top of the interval, so your fetch doesn't just miss it.
- **05 — Power saving** — defaults are sensible; adjust as you like.
- **06 — Battery / button** — if you're using a PiSugar, pick your model here so the install script also wires up its button.
- **07 — Generated install script** — scroll down and click **Copy script**.

---

## Step 5 — Run the script

Back in your SSH window from Step 3, paste the copied script and press
Enter. It will:

- Install everything it needs (this takes a few minutes on a Pi Zero)
- Pull in the Waveshare e-paper library
- Write your display script
- Run a quick test — watch the display, it should update
- Set up either a cron schedule (2.13") or a background service (2.7") so it keeps refreshing on its own
- Start a small web config page on port 8080, so you can change the gauge, flip, or WiFi later without SSHing back in
- Set up the fallback hotspot (unless you turned it off in the setup tool)
- If you picked a PiSugar model, install and start its button watcher too

**If this is the very first time SPI has been enabled on this Pi**,
reboot once so it takes effect, then check the display again:

```
sudo reboot
```

(wait ~30 seconds, then `ssh` back in — no need to re-run the script)

---

## You're done

The display now updates on its own schedule. A few useful commands
going forward, all run over SSH:

```
# Single-gauge (2.13") — check the log
cat ~/river_display/log.txt

# Multi-gauge (2.7") — check service status and recent logs
sudo systemctl status river-display
journalctl -u river-display -f

# PiSugar button (if configured) — confirm it's seeing presses
journalctl -u pisugar-button -f              # PiSugar S / S Plus
journalctl -u pisugar-button-setup           # PiSugar 2/2 Pro/3 series
```

---

## Changing settings later

You don't need to SSH back in or re-run the install script for any of
this. Every display shows the last 3 digits of the Pi's IP address in
the corner (next to the "Updated" timestamp, with a small WiFi icon
under it) — use that to find it on your router's device list, or just
check there directly.

Then, from any browser on the same network:

```
http://<the-pi's-ip>:8080
```

- **2.13" (single-gauge):** change the label, USGS site number,
  measurement, or the 180° flip, and the display refreshes immediately
  after you save.
- **2.7" (multi-gauge):** change any of the 4 buttons, or the 180°
  flip, all at once. Saving restarts the display service to pick up
  the change, so the screen briefly blanks before showing the update —
  that's expected.
- **Either display:** if you turned on the fallback hotspot in the
  setup tool, this same page also has a **"Join a WiFi network"**
  section — scans for nearby networks and lets you switch the Pi to a
  new one without touching a keyboard. See the next section for when
  this actually matters.

This runs as its own `river-display-config` service, separate from the
display itself — `sudo systemctl status river-display-config` if the
page won't load.

---

## If WiFi ever stops working

This only applies if you left **"Fallback setup hotspot"** turned on
in the setup tool's 00 — Network section (it's on by default).

If the Pi ever can't reach any WiFi network it knows about — the
password in Step 1 was wrong, the network got renamed, the router got
replaced, or the Pi moved somewhere new — it automatically broadcasts
its own network instead of just going dark. No monitor, keyboard, or
re-flashing required to fix it:

1. On your phone or laptop, connect to the WiFi network named whatever
   you set as the hotspot SSID in the setup tool (defaults to
   `<display label>-setup`), using the password you set there. Expect
   a "no internet" warning on this network — that's normal, it's
   local-only.
2. Open `http://10.42.0.1:8080` in a browser.
3. Use the **"Join a WiFi network"** section to pick your (real)
   network and enter its password, then submit.
4. The Pi switches over and the hotspot disappears within a few
   seconds. Reconnect your phone/laptop to your normal WiFi, then find
   the Pi there as usual (`ssh` by hostname, or the IP shown on the
   display).

The hotspot only ever activates when normal WiFi is unreachable —
otherwise it stays off and out of the way.

---

## Troubleshooting

**Display stays blank / never updates after the script finishes**
Enabling SPI for the first time needs a reboot (`sudo reboot`) — this
is the most common cause. After rebooting, re-run the test manually to
see any error output directly:
```
cd ~/river_display && python3 river_display.py
```

**"No data yet" / "network error" in the log**
Usually a wrong USGS site number, or a site that doesn't publish the
measurement you picked (discharge vs. gauge height) — double check on
the site's own page at waterdata.usgs.gov.

**Can't SSH in at all**
See the troubleshooting notes at the end of Step 3 — it's almost
always either the Pi not being fully booted yet, or `.local` discovery
not working on your particular network/router.

**2.7" HAT buttons don't switch gauges**
Check `sudo systemctl status river-display` — if the service isn't
running, `journalctl -u river-display -n 50` will show why. Also
confirm you actually filled in a site number for that button's gauge
slot in the setup tool — an empty one is left unmapped on purpose.

**PiSugar button does nothing**
Depends on your model:
- **PiSugar S / S Plus**: check `sudo systemctl status pisugar-button` — if the service isn't running, `journalctl -u pisugar-button -n 50` will show why. Also make sure I2C is disabled in `sudo raspi-config` (Interface Options → I2C) — PiSugar's own docs note this button function conflicts with I2C being enabled.
- **PiSugar 2 / 2 Pro / 3 series**: this depends on `pisugar-server` already being installed (the setup tool's script doesn't install it, only binds to it). Check with `sudo systemctl status pisugar-server` — if it's missing, install it first with `curl https://cdn.pisugar.com/release/pisugar-power-manager.sh | sudo bash`, then re-run `python3 ~/river_display/pisugar_button_setup.py`.

**WiFi looks connected in `raspi-config` but the Pi never comes online**
Almost always a 2.4GHz vs. 5GHz mismatch — see the note at the end of
Step 1.

**Web config page (`:8080`) won't load, or saving doesn't do anything**
Check `sudo systemctl status river-display-config` — if it's not
running, `journalctl -u river-display-config -n 50` will show why.
On the 2.7" HAT, if saving doesn't restart the display, the sudoers
rule the installer sets up for that one restart command may not have
applied — re-running the install script re-adds it.

**Fallback hotspot never shows up when WiFi is down**
Confirm you left "Fallback setup hotspot" turned on in the
setup tool's 00 — Network section — it's not added at all if that
was off. If it was on, check `nmcli connection show` on the Pi (over a
wired/direct connection, or once WiFi is working again) for a
connection named `Hotspot`; if it's missing, re-run the install
script. Also allow it a minute or so after WiFi fails — NetworkManager
tries known networks first before falling back.

**"Join a WiFi network" on the config page doesn't connect**
Double-check the password — this section doesn't validate it before
attempting to connect, so a wrong password just fails silently back to
whatever network was active before. If it was working over the
fallback hotspot specifically, also confirm the sudoers rule applied
correctly (same fix as above: re-run the install script).

---

*USGS Instantaneous Values API · Waveshare e-Paper · No signup, no key required*
