# ESP32-C3 Media Controller Walkthrough

Your custom ESP32-C3 Supermini has been significantly upgraded and optimized!

## Bluetooth Setup
1. **Forget the Old Device**: If you've paired before, you MUST go into your Bluetooth settings and "Forget" the old device (often named "ESP32 Media Controller" or showing up as a generic keyboard).
2. **Scan**: Put the controller into pairing mode (it will breathe Blue slowly when unconnected).
3. **Pair**: Look for **"ESP32 kunsh13 Media Controller"**. It now forces an **Audio/Video Remote icon** instead of a keyboard!

## Hardware Controls

### Rotary Encoder (Knob)
* **Rotate Clockwise:** Volume Up (or Right Arrow)
* **Rotate Counter-Clockwise:** Volume Down (or Left Arrow)
* **Press:** Mute / Unmute
* *Haptics:* Crisp 80ms vibration per tick, 110ms vibration for mute presses.

### Boot Button (Pin 9)
* **Short Press:** Master toggle to instantly turn the entire LED ring ON or OFF.
* **Long Press (Hold 2 seconds):** Force disconnect! The device will drop the Bluetooth connection, flash Red, and immediately re-enter Pairing Mode.

### Touch Sensor 1 (Pin 20)
* **Single Tap:** Play / Pause (Flashes Green)
* **Double Tap:** Next Track (Flashes Orange)
* **Triple Tap:** Previous Track (Flashes Magenta)
* *Haptics:* Vibrates longer based on the number of taps!

### Touch Sensor 2 (Pin 21)
* **Press:** Toggles **Brightness Mode**.
  * The entire LED ring will fade in.
  * While active, turning the knob adjusts the master LED brightness instead of media volume.
  * **Press again** to confirm and exit (the ring will fade back out).

## LED Animations
* **Boot:** Single slow "Fire Orange" breathing animation using sine wave math for a perfectly smooth fade to black.
* **Pairing Mode:** Continuous slow Blue breathing animation.
* **Connected (Idle):** All LEDs smoothly fade OFF after 1.5 seconds of inactivity.
* **Connected (Active):** A single LED moves in the direction of your encoder rotation, leaving a tiny comet tail.
* **Connected (Muted):** Breathes Red continuously. The fade phase-shift is mathematically locked to the exact millisecond you press the button, ensuring it always fades in perfectly from zero brightness without flashing you in the eyes.

## Code Settings
At the very top of `esp32_ble_volume_proj.ino`, you'll find a `--- SETTINGS ---` block. You can change these at any time and re-upload:
* `RING_UPSIDE_DOWN`: Flips the LED rotation direction.
* `DEFAULT_COLOR_R/G/B`: Change your default LED color.
* `HAPTIC_ROTATION_MS` & `HAPTIC_MUTE_MS`: Tune how strong the vibrations feel.
* `USE_KEYBOARD_ARROWS`: Set to `true` to scrub YouTube videos with Left/Right arrows, or `false` for standard Media Volume control.
* `ENABLE_TOUCH_SENSORS`: Set to `false` if you ever want to completely disable the touch pads.
