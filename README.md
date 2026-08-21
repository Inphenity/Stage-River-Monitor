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

**Contents:** [What you'll need](#what-youll-need) ·
[Step 1: Flash the SD card](#step-1--flash-the-sd-card) ·
[Step 2: Wire up the display](#step-2--wire-up-the-display-and-battery-if-using-one) ·
[Step 3: Find the Pi and SSH in](#step-3--find-the-pi-and-ssh-in) ·
[Step 4: Generate your install script](#step-4--generate-your-install-script) ·
[Step 5: Run the script](#step-5--run-the-script) ·
[You're done](#youre-done) ·
[Changing settings later](#changing-settings-later) ·
[If WiFi ever stops working](#if-wifi-ever-stops-working) ·
[Troubleshooting](#troubleshooting)

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
   - **Operating System**: "Raspberry Pi OS (other)" → **Raspberry Pi OS Lite (32-bit)** — you don't need the desktop version for this. Use 32-bit even if your board supports 64-bit: it runs identically well and works on every Pi this project supports, including the original Pi Zero W/WH, which can't run 64-bit at all (only the Zero 2 W/WH and Pi 3/4/5 can).
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

<details>
<summary><strong>Using Windows? Here's the PowerShell version</strong></summary>

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

</details>

---

## Step 4 — Generate your install script

Open the **<a href="https://inphenity.github.io/Stage-River-Monitor/" target="_blank" rel="noopener">STAGE setup tool</a>**
in a browser on your computer (not the Pi). It's laid out as sections
numbered **00–07** (the tool's own numbering, separate from these
Step 1–5 instructions) — work through them top to bottom:

- **00 — Network (optional)** — skip the WiFi-networks part if the Pi's already on WiFi (it is, from Step 1); add networks here for the Pi to join *in addition* to that one — a work WiFi, a mobile hotspot, a backup network. Also where the **fallback setup hotspot** lives (on by default), which recovers the Pi automatically if it ever loses WiFi — no password to set for it or for the web config page later, both use an on-screen PIN instead. See [Changing settings later](#changing-settings-later) and [If WiFi ever stops working](#if-wifi-ever-stops-working) for how that works.
- **01 — Display hardware** — pick your HAT (2.13" or 2.7"), and for the 2.7" confirm the board revision printed on the back (V1 vs V2) — picking the wrong one leaves the panel blank with no error.
- **02 — Station** (2.13") **/ Gauges & button mapping** (2.7") — find your river's USGS site number at [waterdata.usgs.gov/nwis/rt](https://waterdata.usgs.gov/nwis/rt/) (it's the number in that gauge's URL), then choose discharge or gauge height — check your site's page for which one it actually publishes. Click **Test this site number** after entering it — the tool checks it against USGS live, right in your browser, so a typo or a site that doesn't report your chosen measurement gets caught here instead of after you're already SSH'd into a Pi.
- **03 — Display behavior** — defaults are sensible; adjust as you like.
- **04 — Schedule** — set your refresh interval here. Worth turning on "Offset refresh by +2 minutes" — it runs the fetch 2 minutes after each scheduled time instead of exactly on it, which avoids a race if USGS happens to publish its update right at the top of the interval, so your fetch doesn't just miss it.
- **05 — Power saving** — defaults are sensible; adjust as you like.
- **06 — Battery / button (optional)** — if you're using a PiSugar, pick your model here so the install script also wires up its button.
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
a top corner, with a small WiFi icon under it — use that to find it on
your router's device list, or just check there directly.

Then, from any browser on the same network:

```
http://<the-pi's-ip>:8080
```

The page will ask for a PIN — there's no username or password to
remember. Visiting it makes the Pi show a fresh PIN on its own
display (valid about 10 minutes), so you'll need to be near the
device the first time you log in. Enter that PIN and you'll stay
logged in for about 30 minutes. This is what keeps the page from
being usable by anyone else who happens to be on the same network as
the Pi — they'd need physical access to the display to see the PIN.
After 5 wrong attempts, that source stops being able to try again for
a minute — so mistype it a few times and you'll just need to wait
briefly before the next try.

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
re-flashing required to fix it. **The display itself tells you what to
do** — it swaps the gauge reading for reconnect instructions (the
hotspot's name, and the address to visit) the moment it detects the
fallback hotspot is active, so you don't need to already know this
feature exists to find your way back:

1. On your phone or laptop, connect to the WiFi network named on the
   display (defaults to `<display label>-setup`), using the **WiFi
   PIN** shown right there on the display as the password — it's not
   something you set in the setup tool, the Pi generates it itself and
   rotates it periodically like a 2FA code while nobody's connected.
   Expect a "no internet" warning on this network — that's normal,
   it's local-only.
2. Open the address shown on the display in a browser — normally
   `<hostname>.local:8080`, with `10.42.0.1:8080` underneath as a
   fallback if `.local` doesn't resolve on your device. It'll prompt
   for a PIN — no username. Enter the **Login PIN**, shown right below
   the WiFi PIN on the same screen — the two are labeled separately on
   purpose, since they're different numbers for different steps. No
   need to visit the display twice; both are there from the start.
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

Jump to your issue:
[Display blank/not updating](#display-blank) ·
[Re-install didn't apply new settings](#stale-config) ·
[No data / network error](#no-data) ·
[GPIO busy](#gpio-busy) ·
[Can't SSH in](#cant-ssh) ·
[Buttons don't switch gauges](#buttons-dont-switch) ·
[PiSugar button does nothing](#pisugar-button) ·
[WiFi connected but no internet](#wifi-no-internet) ·
[Config page won't load](#config-page-down) ·
[Login PIN won't show](#pin-not-showing) ·
[Locked out of config page](#too-many-attempts) ·
[Fallback hotspot missing](#hotspot-missing) ·
[Duplicate-connection warning](#duplicate-connection) ·
[Join WiFi doesn't connect](#wifi-join-fails) ·
[Network not in scan list](#network-not-listed)

<a id="display-blank"></a>

**Display stays blank / never updates after the script finishes**
Enabling SPI for the first time needs a reboot (`sudo reboot`) — this
is the most common cause. After rebooting, re-run the test manually to
see any error output directly:
```
cd ~/river_display && python3 river_display.py
```

<a id="stale-config"></a>

**Re-ran the install script with new settings, but the display still shows the old ones**
`config.json` is deliberately left alone by the install script — it's
what lets any live changes you've made through the web config page
survive a re-install (say, one to add a PiSugar or pick up a bug fix).
The trade-off: if a gauge already has a saved entry in there, the
fresh script's new defaults for that same gauge get silently
overridden by whatever's already saved, even though the script itself
ran successfully. If a re-install doesn't seem to have taken effect,
this is almost always why. Clear it and restart to confirm:
```
# Multi-gauge (2.7")
sudo systemctl stop river-display
rm ~/river_display/config.json
sudo systemctl start river-display

# Single-gauge (2.13")
rm ~/river_display/config.json
python3 ~/river_display/river_display.py
```

<a id="no-data"></a>

**"No data yet" / "network error" in the log**
Usually a wrong USGS site number, or a site that doesn't publish the
measurement you picked (discharge vs. gauge height) — double check on
the site's own page at waterdata.usgs.gov.

<a id="gpio-busy"></a>

**"GPIO busy" traceback (2.7" HAT)**
Means something else already has the display's GPIO pins claimed —
almost always the `river-display` service already running in the
background while you (or the install script, on a re-run) try to
launch the display script manually at the same time. The install
script now stops the service before its own test run automatically,
so this shouldn't come up there anymore. If you hit it running
`python3 river_display.py` yourself later, `sudo systemctl stop
river-display` first, then try again.

<a id="cant-ssh"></a>

**Can't SSH in at all**
See the troubleshooting notes at the end of Step 3 — it's almost
always either the Pi not being fully booted yet, or `.local` discovery
not working on your particular network/router.

<a id="buttons-dont-switch"></a>

**2.7" HAT buttons don't switch gauges**
Check `sudo systemctl status river-display` — if the service isn't
running, `journalctl -u river-display -n 50` will show why. Also
confirm you actually filled in a site number for that button's gauge
slot in the setup tool — an empty one is left unmapped on purpose.

<a id="pisugar-button"></a>

**PiSugar button does nothing**
Depends on your model:
- **PiSugar S / S Plus**: check `sudo systemctl status pisugar-button` — if the service isn't running, `journalctl -u pisugar-button -n 50` will show why. Also make sure I2C is disabled in `sudo raspi-config` (Interface Options → I2C) — PiSugar's own docs note this button function conflicts with I2C being enabled.
- **PiSugar 2 / 2 Pro / 3 series**: this depends on `pisugar-server` already being installed (the setup tool's script doesn't install it, only binds to it). Check with `sudo systemctl status pisugar-server` — if it's missing, install it first with `curl https://cdn.pisugar.com/release/pisugar-power-manager.sh | sudo bash`, then re-run `python3 ~/river_display/pisugar_button_setup.py`.

<a id="wifi-no-internet"></a>

**WiFi looks connected in `raspi-config` but the Pi never comes online**
Almost always a 2.4GHz vs. 5GHz mismatch — see the note at the end of
Step 1.

<a id="config-page-down"></a>

**Web config page (`:8080`) won't load, or saving doesn't do anything**
Check `sudo systemctl status river-display-config` — if it's not
running, `journalctl -u river-display-config -n 50` will show why.
On the 2.7" HAT, if saving doesn't restart the display, the sudoers
rule the installer sets up for that one restart command may not have
applied — re-running the install script re-adds it.

<a id="pin-not-showing"></a>

**Can't get the login PIN to show up**
There's no password to look up — visiting `:8080` should always make
the Pi draw a fresh PIN on its own display within a few seconds. If it
doesn't appear, check `sudo systemctl status river-display-config`
(single-gauge) or `sudo systemctl status river-display`
(multi-gauge, since the daemon itself draws the PIN there) — a
`journalctl -u <service> -n 50` on whichever one applies will show
why the draw didn't happen.

<a id="too-many-attempts"></a>

**"Too many attempts" / locked out of the web config page**
After 5 wrong password attempts, that source is locked out for 60
seconds — this resets the moment a correct login goes through, so it's
not a growing penalty, just a brief pause. If it's not you (say,
something else on the network guessing at it), it clears on its own;
no action needed. This only tracks failed logins — normal browsing,
including simply loading the page before you've typed anything, never
counts against it.

<a id="hotspot-missing"></a>

**Fallback hotspot never shows up when WiFi is down**
Confirm you left "Fallback setup hotspot" turned on in the
setup tool's 00 — Network section — it's not added at all if that
was off. If it was on, check `nmcli connection show` on the Pi (over a
wired/direct connection, or once WiFi is working again) for a
connection named `Hotspot`; if it's missing, re-run the install
script. Also allow it a minute or so after WiFi fails — NetworkManager
tries known networks first before falling back.

<a id="duplicate-connection"></a>

**"Warning: There is another connection with the name..." while running the install script**
Harmless, but worth knowing what it means: `nmcli` always creates a
new connection profile rather than updating an existing one, so
re-running the install script (common while testing) adds duplicates
each time instead of replacing them. The script cleans up any
same-named connections before creating a fresh one, so simply
finishing the run — or re-running it again if you're unsure — resolves
it; nothing to fix by hand.

<a id="wifi-join-fails"></a>

**"Join a WiFi network" on the config page doesn't connect**
Double-check the password — this section doesn't validate it before
attempting to connect, so a wrong password just fails silently back to
whatever network was active before. If it was working over the
fallback hotspot specifically, also confirm the sudoers rule applied
correctly (same fix as above: re-run the install script).

<a id="network-not-listed"></a>

**A network I know is nearby doesn't show up in the scan list**
Most likely it's a 5GHz-only network — the Pi Zero W/WH/2W/2WH's radio
can't see 5GHz at all (see the note in Step 1), so if a guest network
(common at businesses, hotels, restaurants) only broadcasts on 5GHz,
no amount of rescanning will make it appear. If you know the exact
name, try typing it into the text field directly instead of picking a
scanned entry — that also covers hidden (non-broadcasting) networks,
which never show up in a scan no matter the band.

---

*USGS Instantaneous Values API · Waveshare e-Paper · No signup, no key required*
