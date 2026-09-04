# Club Facility Automation

A set of connected controllers that run the technical infrastructure of a sports club in
Hamburg: irrigation for hockey and tennis surfaces, court floodlighting, pool filtration,
and a booking display at the padel courts.

All four subsystems are independent devices, but they share one design: a controller in
the field does the real-time work locally, publishes state over MQTT, and accepts commands
over the same broker. Nothing depends on a cloud service, and every controller keeps
operating on its own if the network is down.

<!-- TODO: one photo of a control cabinet or a device in place. A single real photo does
     more for a reader than the whole architecture section below. -->

```
                        ┌──────────────────┐
                        │   MQTT broker    │
                        │   (Mosquitto)    │
                        └────────┬─────────┘
        ┌────────────────┬───────┴────────┬────────────────┐
        │                │                │                │
  HockeyWater      TennisWater       POOLControl      Padel Display
  irrigation       irrigation +      filtration       booking board
                   floodlighting     pumps
        │                │                │                │
   valves, pumps    valves, lighting  Danfoss FC-202   HUB75 LED matrix
   sensors          contactors        VFD (Modbus)     ESP32-S3
```

---

## HockeyWater — irrigation control for hockey pitches

Watering a water-based hockey pitch is not a timer problem. The surface has to be wet for
play, the club pays for every cubic metre, and a valve that fails open is expensive before
anyone notices. HockeyWater runs the irrigation cycles, monitors what actually happened,
and reports rather than assumes.

- Scheduled and manual irrigation cycles per zone, with interlocks so overlapping zones
  cannot draw at the same time
- Pump and valve control with feedback monitoring; a cycle that does not produce the
  expected result raises a fault instead of running silently
- Local operation continues if the network drops; state syncs when it returns
- Web interface for manual control and configuration, MQTT for integration
- Current version: <!-- TODO: v1.6.x --> on ESP32; a port to Arduino Opta is in progress
  for installations that need an industrial DIN-rail form factor and 24 V I/O

<!-- TODO: number of zones, valve type, whether flow metering is installed. Concrete
     numbers are what make this read like a real installation rather than a demo. -->

## TennisWater — irrigation and floodlighting for tennis courts

Clay courts need water on a different logic than a hockey pitch, and the floodlights sit on
the same infrastructure, so both are handled by one controller per court group.

- Zone irrigation matched to clay court requirements, manual and scheduled
- Floodlight control with per-court switching, run-on timers and lamp protection
  (restrike delay after switch-off)
- Runtime accounting per court as a basis for energy cost allocation
- Same MQTT interface as HockeyWater, so both look identical to any client

<!-- TODO: how lighting is actually triggered - key switch, booking system, LoRa button
     node? If the button nodes belong here, describe them: radio, battery life, range. -->

## POOLControl — pool filtration control

The filtration pumps are the largest single electrical consumer on the site. Running them
at fixed speed all day is the default and it is wasteful; running them too little costs
water quality.

- Speed control of the filtration pumps through a Danfoss FC-202 variable frequency drive
  over Modbus RTU
- Scheduled operating profiles with reduced-speed periods
  <!-- TODO: confirm whether backwash handling is in scope -->
- Reads drive status, current and fault codes back from the VFD and publishes them, so a
  drive trip is visible immediately instead of at the next site visit
- Runs on a Waveshare ESP32-S3-POE-ETH board: wired Ethernet with power over the same
  cable, because a plant room is the wrong place to rely on Wi-Fi

## Padel Display — court booking board

The club's booking system is web-only. Players arriving at the courts had no way to see
whether a court was free without pulling out a phone and logging in. This display shows
the same information where the decision is actually made.

- Python service (FastAPI) scrapes the booking system, normalises the schedule and
  publishes it over MQTT, retained
- ESP32-S3 drives a HUB75 LED matrix; the firmware knows nothing about HTTP or
  authentication, it subscribes and renders
- Custom seven-segment renderer for the time fields — the stock font libraries produced
  digits that were unreadable at viewing distance, so digits are drawn as segments with
  explicit control over stroke width and spacing
- Shows a clear "no data" state rather than stale information if the connection drops

---

## Design decisions worth explaining

**Control loops stay on the device.** Every controller runs its schedules and interlocks
locally. The broker carries state and commands, not the logic. A network outage during a
watering cycle is an inconvenience, not a flood.

**MQTT as the only integration surface.** Each subsystem publishes the same shape of
status and accepts the same shape of command. Adding a dashboard, a display or a new
consumer requires no change to any controller.

**Faults are reported, not inferred.** Where feedback is available — flow, drive status,
contactor state — the controller compares intent against reality and raises a fault when
they diverge. Silent failure on unattended equipment is the expensive kind.

**Wired where it matters.** Plant rooms and metal cabinets get Ethernet over PoE; only
devices that genuinely cannot be cabled use Wi-Fi.

## Repository layout

```
hockeywater/     ESP32 firmware, web UI, Arduino Opta port
tenniswater/     ESP32 firmware, irrigation and lighting control
poolcontrol/     ESP32-S3-POE-ETH firmware, Modbus RTU driver for FC-202
padel-display/   Python scraper service + ESP32-S3 HUB75 firmware
docs/            wiring notes, MQTT topic reference, photos
```
<!-- TODO: adjust to the actual structure - one repository with subfolders, or four
     separate repositories with this file as the profile README. -->

## MQTT interface

| Topic pattern | Direction | Payload |
|---|---|---|
| `<site>/<system>/state` | device → broker | retained JSON: mode, active zones, sensor values |
| `<site>/<system>/cmd` | broker → device | JSON command |
| `<site>/<system>/status` | device → broker | online/offline (last will), uptime, link state |

<!-- TODO: paste one real example payload below. Reviewers skim READMEs; a concrete
     payload tells them more about the design than three paragraphs of prose. -->

```json
{
  "updated": "2026-09-04T18:05:00+02:00",
  "mode": "auto",
  "zones": []
}
```

## Getting started

<!-- TODO: build and flash instructions per subsystem, and the broker configuration each
     one expects. Keep credentials out of the repository; .env files and config headers
     holding secrets must be git-ignored. -->

## Status

All four systems are in production use at the club. Built and maintained voluntarily.

## License

<!-- TODO: pick one. MIT is the usual default and keeps things simple. -->
