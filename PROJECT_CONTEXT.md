```
AQUARIUM AUTOMATION (ESP32 IoT) - FULL PROJECT CONTEXT EXPORT
Exported: 30 Oct 2026
Updated:  30 Oct 2026 (software route, manual button, relay choice decided)
Purpose: hand this to a new Claude workspace so it can continue without re-asking settled questions.

======================================================================
0. HOW TO USE THIS DOCUMENT (instructions for the next Claude)
======================================================================
- This is a DIY ESP32 aquarium controller. Planning is mostly done. No code
  has been written, nothing has been bought, nothing has been wired.
- Section 3 lists DECIDED items. Do not re-open them unless the user does.
- Section 11 lists OPEN items. Ask the user about these; do not assume.
- Prices are snapshots from 29 Oct 2026. Re-check before telling the user
  to order.
- Next deliverable once open items are settled: firmware + a final wiring
  diagram/pin map for the chosen software route (Blynk).

======================================================================
1. ABOUT THE USER / WORKING PREFERENCES
======================================================================
- Lively but accurate communication.
- Never assume or make up information. Ask when something is unclear.
- Don't comment on the user's word choices (including swearing).
- The user is fairly new to this. Plain explanations of "why" land well
  (e.g. they asked "why relay?", "why NC?"). Short, direct messages; they
  often reply with brief questions.
- Located in India (shops at robu.in, Robocraze, Amazon.in). Budget is in
  Indian rupees. Mains voltage not confirmed, but very likely 230V.
- Does NOT have access to 3D printing. Mechanical parts must be buildable
  from household items / hardware-shop parts.
- Has no GitHub CLI installed. A GitHub plugin is configured in Claude but
  was failing to connect (auth header error) at the time of export.
  Claude Code (cloud) can commit and push to the project repo
  sweetysharan29-bit/AutoAquariumMaintainer_cld. The user is happy with
  direct pushes to main (no PRs) for this personal repo.

======================================================================
2. GOAL AND v1 SCOPE
======================================================================
A DIY IoT aquarium controller built on an ESP32. v1 scope:
  1. Filter on/off on a timer, through a single relay
  2. Automatic scheduled feeding
  3. Manual (on-demand) feeding
Feeding must handle TWO food types: flakes, and a fine powder-like food.

The fish and tank (what we know):
- Small fish. They eat very little and can go days without food.
- The tank currently has NO filter and is doing fine. Water is changed
  about once a month.
- Freshwater vs saltwater: UNKNOWN. Tank size: UNKNOWN.

Key consequence: overfeeding, not underfeeding, is the real risk. Uneaten
food rots and produces ammonia. Portions should be small, with strict
limits. A missed feed is harmless.

======================================================================
3. DECISIONS MADE (with reasoning)
======================================================================

3.1 Board: ESP32 NodeMCU 30-pin (ESP32-WROOM-32, CP2102 USB chip)
- Classic ESP32: most tutorials, supported by ESPHome/Arduino/Blynk/
  RainMaker, plenty of spare GPIO for future sensors.
- 30-pin chosen over 38-pin because the 38-pin is too wide for a standard
  breadboard.
- Rejected: ESP32-C3 (too few pins for future add-ons), ESP32-S3 (overkill
  unless adding a camera), ESP8266 (old, few usable pins).

3.2 Feeder: TWO "pill-bottle drum" feeders, each on its own MG90S servo
- Why: no 3D printing available; needs to handle flakes AND powder; fish
  need very little food, so rough powder portions are acceptable.
- How the alternatives were ruled out:
  * Single rotating drum: fine for flakes, poor for powder (dumps it).
  * Auger (screw): great for powder, but flakes jam/shred. Building one
    without a 3D printer is fiddly. A vending-machine spring was
    discussed; the user correctly pointed out its open centre and wide
    pitch would leak powder. A spring glued over a solid rod inside a
    tube (~1 mm clearance) would work but needs trial-and-error sourcing.
    Shelved.
  * Pocket wheel: OK for powder, flakes bridge above it.
  * Compartment carousel (round pill organiser on a stepper): handles
    both foods since each portion is pre-loaded; downside is refilling
    every 7-14 feeds. Kept as the FALLBACK if the powder drum turns out
    too inconsistent.
  * Two hoppers (auger + drum): too much build effort without printing.
- Servo, not stepper: each bottle only needs to move within 180 degrees.
  MUST be the 180-degree MG90S, NOT the "360 degree continuous rotation"
  version (that one can't hold a position).

3.3 Filter switching: mechanical relay, filter wired through NO contact
- Relay chosen over SSR: most SSRs are normally-open only, and SSRs leak
  a little current when off (can make small motors hum).
- Originally NC was proposed (filter keeps running if the controller
  dies). The user changed this to NO because the tank currently runs fine
  with no filter. With NO: relay idle / controller dead / no power to the
  relay = filter OFF, which matches today's normal state.
- MODULE CHOSEN (30 Oct 2026): Robocraze "1CH 5v Relay Board High/Low
  Level Trigger with Optocoupler", INR 72. The user picked this over the
  recommended 3V module (INR 159) to save INR 87, knowing the risk.
  Consequences:
  * Relay module VCC goes to the 5V rail (adapter +5V), NOT the ESP32
    3V3 pin. A 5V coil won't pull in reliably at 3.3V.
  * The ESP32 still drives IN with 3.3V logic. That is the risk: some 5V
    modules don't switch reliably from a 3.3V signal. Must be verified on
    the bench (relay clicks on and off cleanly, every time) before any
    mains wiring.
  * Fallback if it's unreliable: buy the 3V/3.3V-trigger module (INR 159),
    or add a small NPN transistor stage. Decide only if the test fails.
  * Must still be a 10A 250VAC module with COM/NO/NC terminals (confirm
    on arrival).
- Must be set to HIGH-level trigger (H/L jumper on H).
  Active-low would turn the filter ON while the ESP32 boots.
- 10k ohm pull-down resistor from the relay control GPIO to GND so the
  relay stays off during boot/reset before firmware drives the pin.
- Relay coil draws current from the 5V adapter the whole time the filter
  is on; that's normal and well within the 2A adapter.
- Filter behaviour = timer-based on/off windows (the user's original
  request), plus automatic off during feeding.
- User was told (and accepted) the caveat: water sitting in a filter
  while it's off goes stale and is pushed into the tank on restart.
  Several shorter on-windows per day are gentler than one long one;
  rinsing the filter at monthly water changes helps. Low risk here
  because the filter is mainly for circulation, not the tank's
  biological filtration.

3.4 Logic lives on the ESP32
- Whatever app/cloud is used, all schedules, the filter timer, cooldowns,
  daily limits and failsafes run ON THE ESP32. The app is only a remote
  control. Cloud outage, internet loss or a free-plan change must never
  stop the tank from working. The physical button always works.

3.5 Budget target: about INR 1,250 for electronics
- The user wrote "1.25k"; interpreted as INR 1,250 (not objected to).

3.6 Software/control route: BLYNK (decided 30 Oct 2026)
- Arduino firmware + Blynk phone app. Control from anywhere, free push
  notifications. ESP RainMaker remains the backup if Blynk's free-plan
  limits get in the way.
- See section 10 for the datastream plan.

3.7 Manual button: feeds the food type SELECTED IN THE APP (decided 30 Oct 2026)
- The user rejected short/long press. Instead: a "button food" selector
  in the Blynk app (flakes or powder). The physical button always feeds
  whichever food is selected.
- The selection STAYS until the user changes it manually. Store it in the
  ESP32's flash (NVS/Preferences) so it survives reboots and works with
  no Wi-Fi/cloud.
- Reason given by the user: in practice only one food type will be in
  use, so one fixed button choice is simpler.

======================================================================
4. FINAL PARTS LIST (prices checked 29 Oct 2026)
======================================================================

BUY NOW
  1. ESP32 NodeMCU 30-pin (WROOM-32, CP2102) ........ x1  INR 425  Robocraze
  2. 1CH 5V relay board, High/Low level trigger,
     optocoupler (confirm 10A 250VAC, COM/NO/NC) ..... x1  INR 72   Robocraze
  3. TowerPro MG90S servo, 180-degree ................ x2  INR 250  robu.in (125 each, "standard quality")
  4. 5V 2A power adapter ............................. x1  INR 183  Robocraze
  5. PBS-11B 12mm panel push button (momentary) ...... x1  INR 19   Robocraze
  6. 1000uF 25V electrolytic capacitor ............... x2  INR 18   Robocraze
  7. Jumper wires M-F 40pcs + F-F 40pcs .............. x1 each INR 87 Robocraze
     SUBTOTAL ......................................................... INR 1,054
     (shipping excluded; everything except the servos is now Robocraze,
      so at most two shipping charges: Robocraze + robu.in)

ADD LATER (optional, deferred to stay in budget)
  - DS3231 RTC module ............................ INR 234 Robocraze
    (keeps schedules running without Wi-Fi/NTP. Deferred because the fish
     can skip a feed.)
  - LIR2032 rechargeable coin cell, not priced (most DS3231 modules try
    to charge the cell, so a plain CR2032 is not suitable unless the
    charging resistor is removed)
  - 3V/3.3V-trigger relay module, INR 159 Amazon.in: ONLY if the 5V
    module fails the bench test (section 3.3).

STILL NEEDED, NOT PRICED
  - Mains side: 3-pin plug, 3-core cable, a socket for the filter, inline
    fuse holder + fuse (sized from the filter's label), small junction box
  - Enclosure for the electronics (an electrical-shop junction box works)
  - DC barrel jack to screw-terminal adapter (to connect the 5V adapter)
  - Breadboard for testing (if the user doesn't have one)
  - 10k ohm resistor (relay pull-down), a few rupees
  - The FILTER itself (the user doesn't own one yet)

HOUSEHOLD / FREE
  - 2 small screw-cap plastic bottles (pill bottles, film canisters,
    small spice jars)
  - Small funnel or cut-off top of a plastic bottle (food chute)
  - Silica gel sachets
  - Screws / double-sided foam tape, scrap wood/acrylic/plastic for a
    servo bracket

COST VARIANTS
  - Chosen (5V relay, no RTC): INR 1,054
  - Chosen + RTC: INR 1,288
  - If the 5V relay fails and the 3V module is bought as well: +INR 159

PRICE NOTES / THINGS TO AVOID
  - Robocraze's 180-degree MG90S (INR 129) was OUT OF STOCK. Their in-stock
    "Towerpro MG90S 360 degree" (INR 230) is continuous rotation: AVOID.
  - Other prices seen: ESP32 38-pin Robocraze INR 422, robu.in INR 449;
    28BYJ-48 + ULN2003 Robocraze INR 154 (no longer needed); DS3231 RPi
    mini version Robocraze INR 209.
  - Robocraze prices came from live store data. Amazon.in and robu.in
    prices came from search snapshots and may be stale.

======================================================================
5. SYSTEM OVERVIEW
======================================================================

  5V 2A adapter ---> ESP32 VIN
                ---> Servo FLAKES (+1000uF cap)
                ---> Servo POWDER (+1000uF cap)
                ---> Relay module VCC
  ESP32 GPIOs ----> servo signals, relay IN; button into a GPIO

  Wall plug Live -- fuse -- Relay COM -> NO -- filter socket Live
  Wall plug Neutral ------------------------- filter socket Neutral
  Wall plug Earth --------------------------- filter socket Earth
  (Relay NC unused)

  All low-voltage grounds tied together (adapter GND, ESP32 GND, servo
  GNDs, relay GND).

======================================================================
6. PIN MAP (ESP32 NodeMCU 30-pin)
======================================================================
  GPIO 26 -> Relay IN. HIGH = filter on. 10k pull-down to GND.
             (not a strapping pin)
  GPIO 18 -> Servo FLAKES signal (orange wire), 50 Hz PWM
  GPIO 19 -> Servo POWDER signal (orange wire), 50 Hz PWM
  GPIO 27 -> Manual feed button to GND (internal pull-up, software
             debounce). Feeds the food selected in the app (section 3.7).
  GPIO 25 -> Spare button (optional, e.g. filter override)
  GPIO 2  -> Onboard status LED (on most of these boards; verify)
  GPIO 21 -> RTC SDA (reserved for later)
  GPIO 22 -> RTC SCL (reserved for later)
  VIN     -> Adapter +5V
  GND     -> Common ground
  (Relay VCC is fed from the adapter's +5V rail, not from an ESP32 pin.)

  Deliberately avoided: GPIO 0, 2 (as input), 5, 12, 15 (boot strapping),
  1/3 (USB serial), 6-11 (flash), 34-39 (input-only, no pull-ups).

======================================================================
7. WIRING DETAILS
======================================================================

LOW-VOLTAGE SIDE
  - Adapter +5V -> ESP32 VIN, Servo FLAKES red, Servo POWDER red,
    relay VCC
  - Adapter GND -> ESP32 GND, both servo browns, relay GND
  - One 1000uF capacitor across each servo's +5V/GND, close to the servo
    connector (prevents brownout resets when servos start). Polarised:
    stripe side to GND.
  - GPIO18 -> flakes servo orange; GPIO19 -> powder servo orange
  - GPIO26 -> relay IN, plus 10k from GPIO26 to GND
  - Relay H/L jumper on H (high-level trigger)
  - GPIO27 -> button -> GND
  - Servos NEVER powered from the ESP32's 3V3 pin.
  - When uploading over USB, unplug the 5V adapter unless the board is
    confirmed to protect against both being connected. (With the adapter
    unplugged the relay and servos have no power; that's expected.)

MAINS SIDE
  - Switch only the LIVE wire, through relay COM -> NO. Neutral and earth
    go straight through to the filter socket.
  - Fuse on the live wire before the relay.
  - Mains wiring lives inside a closed box, never on a breadboard. Never
    open it while plugged in.
  - Use a GFCI/RCD-protected outlet. Drip loops on every cable.
  - If the user isn't confident with mains wiring, an electrician should
    do that part. Everything else is low-voltage.
  - Fuse rating and relay suitability get confirmed once the filter's
    label (watts/amps, voltage) is known.
  - Do NOT wire mains until the 5V relay has passed the 3.3V-signal bench
    test (section 3.3).

======================================================================
8. FEEDER MECHANICS (no 3D printing)
======================================================================
- Each bottle lies horizontally, its axis in line with the servo shaft.
- The bottle's CAP is glued/screwed to the servo horn. The bottle screws
  into the cap, so refilling = unscrew bottle, fill, screw back. Nothing
  to remount.
- Servo mounted on a simple bracket (wood/acrylic/plastic scrap) on the
  tank rim or lid, a few cm above the water.
- FLAKES bottle: one side slot, start around 8 x 15 mm. At rest the slot
  faces UP. Feed = rotate ~180 degrees (slot down), small wiggle to shake
  loose, rotate back up. Portion set by slot size.
- POWDER bottle: 2-4 small holes about 1-1.5 mm (heated needle). At rest
  holes face UP. Feed = rotate holes down, rock about +/-25 degrees N
  times ("salt shaker"), rotate back up. Portion = number of shakes
  (configurable).
- A funnel/chute under both bottles guides food into one spot, away from
  the filter intake.
- Calibration: run feeds over paper first and tune slot size / shake
  count. Start small.
- Humidity: tight caps; silica gel in the feeder housing, NOT loose in the
  bottles (it could fall into the tank). Keep the lid opening under the
  funnel small to limit rising moisture.
- Electronics box sits beside the tank, not over it.
- Powder portions won't be precise; the user accepted this because the
  fish eat very little and can go days without food.

======================================================================
9. BEHAVIOUR SPEC (proposal; all numbers configurable)
======================================================================

NORMAL STATE
- Filter follows its on/off timer (relay on = filter running).
- Servos at rest, openings facing up.
- LED shows Wi-Fi / time status.

FILTER TIMER
- Daily on/off windows (e.g. 08:00-12:00 and 18:00-22:00; real times TBD).
- Manual on/off override from the app (or spare button), returning to the
  timer after 30 min.
- No valid time (no Wi-Fi/NTP, no RTC) -> filter stays OFF.

FEED SEQUENCE (scheduled or manual)
  1. Check limits (cooldown, daily max). If blocked: do nothing, log it,
     blink LED.
  2. If the filter is running, turn it off. Wait ~3 s for flow to settle.
  3. Dispense flakes (rotate-tip-return) and/or powder (N shakes).
  4. Keep the filter off for 10 min.
  5. Return the filter to whatever the timer says now. Log the feed.

SCHEDULE
- Feed times, each with a food choice (flakes or powder), e.g. 09:00
  flakes, or alternating foods by day.
- Fasting days: selectable weekdays with no scheduled feeding.
- No valid time -> scheduled feeds are SKIPPED. Manual feeding still
  works. Safe because the fish can go days without food.

MANUAL FEED (decided, section 3.7)
- Physical button: one press = feed the food currently selected in the
  app ("button food": flakes or powder). The selection is saved in flash
  and stays until the user changes it in the app.
- Default before the user ever sets it: NOT YET DECIDED (ask; flakes is
  the obvious proposal).
- App feed buttons (flakes, powder) work independently of the selector.
- Counts toward the same cooldown and daily limit.

LIMITS
- Cooldown between feeds: 4 h
- Max feeds per day: 2

FAILSAFES
- NO relay wiring: controller dead/unpowered -> filter off (the tank's
  current normal).
- Boot state: relay off (filter off) and servos to rest with openings up,
  before anything else runs; then the timer takes over.
- 10k pull-down keeps the relay off during boot/reset.
- Watchdog: if firmware hangs, the ESP32 reboots into the safe state.
- Feed limits mean button spam or schedule bugs can't overfeed.
- All logic local to the ESP32; the cloud/app is optional.

OPTIONAL
- Notification on each feed ("Fed flakes at 09:00").
- Feed log with timestamps.
- Calibration mode (dispense without counting toward limits, for testing
  on paper).

======================================================================
10. SOFTWARE / CONTROL APP (DECIDED: Blynk)
======================================================================
Chosen: Arduino firmware + Blynk (decided 30 Oct 2026).
Backup: ESP RainMaker, if Blynk's free-plan limits get in the way.

Options that were discussed:
  a) Arduino + built-in web page: ESP32 serves its own page on home Wi-Fi.
     Free, no server; only works on home Wi-Fi without extra setup.
  b) Arduino + Telegram bot: chat commands, works from anywhere, free
     notifications; settings via commands are clunky.
  c) ESPHome + Home Assistant: best dashboard and expansion, but needs an
     always-on HA server (Raspberry Pi/PC), outside budget unless the
     user already has one (unknown).
  d) Blynk: phone app with custom buttons, control from anywhere, free
     push notifications. CHOSEN.
  e) ESP RainMaker (Espressif): free official Android/iOS app, built-in
     schedule UI, schedules run on the ESP32 itself. BACKUP.
  f) Arduino Cloud: free tier is 2 devices and 5 variables per device.
     Too tight. Rejected.
  g) Adafruit IO: free tier 10 feeds, 5 dashboards, 30 data points/min;
     web dashboard rather than a real phone app. Not preferred.

Blynk free plan (from blynk.io/pricing, checked 29-30 Oct 2026):
  5 devices, 1 user, 100,000 messages/month (resets monthly), 10 product
  templates, 1 multi-device dashboard, 1 week data retention, 5
  events/notifications, 5 cloud automation scenarios, in-app push and
  email notifications included, SMS not included. Datastreams per
  template not listed on the pricing page; Blynk's forum/docs conflict
  (5, 10, 20 have all been cited) and the free plan has changed several
  times. Design to fit about 5 datastreams. Avoid frequent status pushes
  (send on change plus every few minutes) to stay well within the
  message cap.

Datastream plan (5):
  1. Feed flakes (button)
  2. Feed powder (button)
  3. Filter on/off override (switch)
  4. Status text (last feed, next feed, filter state, blocked reasons)
  5. Button food selector (flakes/powder) - uses the former "spare" slot
  Schedule/limit settings that don't fit in Blynk datastreams live in
  firmware config (or a small local web page later). If Blynk turns out
  to allow more datastreams on the free plan, settings can move into the
  app.

======================================================================
11. OPEN DECISIONS (ask the user)
======================================================================
  1. Which filter to buy (sponge / internal / hang-on-back), its
     wattage/amps, and mains voltage (sets fuse rating, confirms relay).
  2. Filter on/off windows (times per day).
  3. Feed times and which food per slot; fasting days.
  4. Freshwater or saltwater? Tank size? (affects enclosure sealing and
     placement; salt creep corrodes electronics)
  5. RTC now or later (currently deferred).
  6. Which parts, if any, the user already owns.
  7. Default "button food" before the user first sets it in the app.

  RESOLVED 30 Oct 2026: software route (Blynk), manual button behaviour
  (app-selected food, persistent), relay module (5V, INR 72).

======================================================================
12. FUTURE IDEAS (out of scope for v1)
======================================================================
Temperature sensor (DS18B20) + heater control, LED lighting schedule with
sunrise/sunset, auto top-off with float switches, leak sensor, TDS/pH
probes (isolated modules), dashboards/graphs, ESP32-CAM, dosing pumps.
The spare GPIOs and reserved I2C pins (21/22) leave room for these.

======================================================================
13. SOURCES USED FOR PRICES / PLATFORM LIMITS
======================================================================
- Robocraze store data (ESP32, relays, servos, stepper, DS3231, power,
  buttons, capacitors, wires): https://robocraze.com
- robu.in TowerPro MG90S 180 (standard quality):
  https://robu.in/product/towerpro-mg90s-mini-digital-servo-motor-180-rotation-standard-quality/
- robu.in ESP32 38-pin:
  https://robu.in/product/esp32-38pin-development-board-wifibluetooth-ultra-low-power-consumption-dual-core/
- Amazon.in 1-channel relay listings:
  https://www.amazon.in/1-channel-relay-module/s?k=1+channel+relay+module
- Blynk pricing: https://blynk.io/pricing
- Blynk forum on free-plan limits:
  https://community.blynk.cc/t/blynk-io-free-limits-confusion-how-many-datastreams-per-template-20-or-5/73829
  https://community.blynk.cc/t/essential-changes-to-free-plan/65289
- Arduino Cloud plans: https://cloud.arduino.cc/plans/
- ESP RainMaker scheduling: https://rainmaker.espressif.com/docs/scheduling
- ESP RainMaker in Arduino: https://developer.espressif.com/blog/esp-rainmaker-now-in-arduino/
- Adafruit IO free vs IO+: https://learn.adafruit.com/welcome-to-adafruit-io/io-plus

======================================================================
14. CURRENT STATUS / NEXT STEPS
======================================================================
- Status: hardware and feature design agreed; software route (Blynk),
  button behaviour and relay module decided; parts list ready; nothing
  purchased; no code.
- Next steps:
  1. Settle the remaining open decisions in section 11.
  2. User buys parts (re-check prices/stock; 180-degree MG90S only).
  3. Deliver Blynk firmware implementing section 9, with all logic on
     the ESP32.
  4. Deliver a final wiring diagram + pin map matching sections 6-7.
  5. Bench test on a breadboard WITHOUT mains: servos, button, relay
     click/LED. Specifically confirm the 5V relay switches reliably from
     the ESP32's 3.3V signal. Calibrate portions over paper.
  6. Mains wiring in a closed box (or by an electrician), then install.
```
