
GlimmerGuard – Step-by-Step Build Guide (Hardware + Software)

This guide walks you from zero → demo for an AI-based automatic headlight dimming prototype
on a Raspberry Pi. Follow the steps in order. Use a BENCH TEST LAMP first (12V LED lamp).
Only after full testing, consider any vehicle wiring with proper fusing and supervision.

──────────────────────────────────────────────────────────────────────────────
PART 0 — WHAT YOU NEED (MINIMUM)
──────────────────────────────────────────────────────────────────────────────
• Raspberry Pi 4 (4GB) + 5V/3A official adapter
• 32GB microSD card (Class 10)
• Camera: Pi Camera v2 (preferred) or USB webcam
• 2‑Channel 5V opto‑isolated relay module
• 12V LED lamp (to simulate headlight low & high beams)
• 12V battery (or 12V adapter) + a buck converter (12V→5V, 3A) if powering Pi from 12V
• Jumper wires, breadboard (optional), wire stripper, small screwdriver
• (Optional) Small automotive inline fuse (2–5A) for safety on the 12V lamp line
• (Optional) Cooling fan/heat sink for the Pi

──────────────────────────────────────────────────────────────────────────────
PART 1 — FLASH OS & FIRST BOOT
──────────────────────────────────────────────────────────────────────────────
1) On your laptop/PC, install “Raspberry Pi Imager”.
2) Flash “Raspberry Pi OS (64-bit)” to the microSD card.
3) First boot the Pi with monitor/keyboard/mouse OR enable SSH if headless.
4) Open a terminal and run:
   sudo apt update && sudo apt upgrade -y
   sudo raspi-config   # Enable camera: Interface Options → Camera → Enable
5) Install Python deps:
   sudo apt install -y python3-pip python3-opencv git
   pip3 install numpy gpiozero

──────────────────────────────────────────────────────────────────────────────
PART 2 — WIRING (BENCH SETUP)
──────────────────────────────────────────────────────────────────────────────
Raspberry Pi pins used (BOARD numbers in brackets):
• 5V Power: Pin 2 (or 4)
• GND: any GND (Pin 6 is convenient)
• GPIO17 → (Pin 11)  → Relay Channel 1 IN1  (LOW BEAM control)
• GPIO27 → (Pin 13)  → Relay Channel 2 IN2  (HIGH BEAM control)

Relay board typical pins:
• VCC → Pi 5V
• GND → Pi GND
• IN1 → Pi GPIO17 (Pin 11)
• IN2 → Pi GPIO27 (Pin 13)

12V lamp wiring (use 2 lamps or a dual-filament lamp if available):
• Battery +12V → both relay COM terminals (one per channel)   [add a 2–5A inline fuse here]
• Relay CH1 NO → Lamp LOW +ve
• Relay CH2 NO → Lamp HIGH +ve
• Lamp –ve → Battery –ve (Ground)
NOTE: Only one channel should be ON at a time (code ensures interlock).

Safety:
• DO NOT connect to a real car circuit until the bench demo works flawlessly.
• Keep common ground: Pi GND ↔ Relay GND ↔ Battery –ve.
• If your relay board has JD-VCC separate header, follow its datasheet. Many hobby boards
  have VCC/GND/INx only and include flyback diodes/optos on board.

──────────────────────────────────────────────────────────────────────────────
PART 3 — QUICK TESTS
──────────────────────────────────────────────────────────────────────────────
A) Test the relays (no camera yet):
   python3 relay_test.py
   → You should see LOW lamp ON for 2s, OFF 1s, then HIGH lamp ON for 2s, OFF 1s, loop…
   Press Ctrl+C to stop.

B) Test the camera:
   python3 camera_test.py
   → A window opens showing the camera. Press ‘q’ to quit.
   (On headless Pi, use HDMI display or run glimmerguard.py which can save frames.)

If either test fails, recheck wiring or camera enable in raspi-config.

──────────────────────────────────────────────────────────────────────────────
PART 4 — RUN GLIMMERGUARD (BASELINE DETECTION)
──────────────────────────────────────────────────────────────────────────────
1) Place the camera looking at a dark corridor/path. Put a bright flashlight or
   a scooter/bike with lights ON at the far end to simulate oncoming vehicle.
2) Run:
   python3 glimmerguard.py
3) By default:
   • When oncoming headlight pair is detected persistently → LOW beam turns ON (dim).
   • When scene clears for a short cooldown → HIGH beam returns.
4) Overlay window shows detections (if a screen is attached). For headless setups,
   set SAVE_DEBUG_FRAMES = True to save images in output_frames/ for tuning.

──────────────────────────────────────────────────────────────────────────────
PART 5 — TUNING CHEATSHEET
──────────────────────────────────────────────────────────────────────────────
In glimmerguard.py (top config section), adjust:
• FRAME_WIDTH/HEIGHT: 640×360 is a good start for speed.
• ROI_Y_START: use ~0.4–0.5 to analyze bottom 60% of frame (road area).
• BRIGHT_PERCENTILE: 96–99; increase if too many bright pixels are detected.
• MIN_BLOB_AREA: raise to ignore small far lights; lower if missing detections.
• PAIR_Y_TOLERANCE: ~20–40 px; how similar in height the pair must be.
• PAIR_GAP_RANGE: (min,max) px; horizontal gap between headlights. Increase max
  for farther vehicles, decrease min to handle close vehicles.
• PERSIST_FRAMES: 5–8; higher = fewer flickers.
• COOLDOWN_SEC: 1.2–2.0; higher = fewer flickers but slower to switch back.

──────────────────────────────────────────────────────────────────────────────
PART 6 — OPTIONAL: AUTO-START ON BOOT
──────────────────────────────────────────────────────────────────────────────
1) Copy glimmerguard.service to /etc/systemd/system/
   sudo cp glimmerguard.service /etc/systemd/system/
2) Then:
   sudo systemctl daemon-reload
   sudo systemctl enable glimmerguard
   sudo systemctl start glimmerguard
3) Check status:
   systemctl status glimmerguard

To stop/disable:
   sudo systemctl stop glimmerguard
   sudo systemctl disable glimmerguard

──────────────────────────────────────────────────────────────────────────────
PART 7 — ROAD TESTING (WITH CAUTION)
──────────────────────────────────────────────────────────────────────────────
• Start in “observation mode” (print/log only; don’t switch lamps) by setting
  OBSERVE_ONLY = True in glimmerguard.py. Confirm logic is correct.
• Choose a safe, empty road. One person drives; one monitors logs.
• Measure false dim rate, miss rate, and reaction time. Tune parameters.
• Only after confidence is high, integrate with a real headlight circuit using
  proper automotive relays/fuses and under supervision.

──────────────────────────────────────────────────────────────────────────────
TROUBLESHOOTING QUICK NOTES
──────────────────────────────────────────────────────────────────────────────
• Relay blinks rapidly → Increase PERSIST_FRAMES and COOLDOWN_SEC. Ensure common ground.
• Camera dark/noisy → Reduce exposure or use a better low-light camera. Add slight blur.
• Picks up streetlights → Ensure ROI is bottom half; require headlight pairing; raise BRIGHT_PERCENTILE.
• FPS too low → Drop resolution to 480p; process every other frame (FRAME_SKIP = 1).

Good luck! – Team GlimmerGuard
