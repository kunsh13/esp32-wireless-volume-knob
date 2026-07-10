# Changelog

All notable changes to this project will be documented in this file.

## [2026-06-26 15:45:00 IST]

### Added
- **Swift Pair Name Resolution**: Deep-injected the display name "Esp32 Media Knob" directly into the Microsoft Swift Pair BLE payload. Windows will now correctly identify the device by name in the pop-up notification instead of showing a generic "New Bluetooth Keyboard Found".

### Changed
- **Unpair LED Animations**: The unpair confirmation flash has been updated to a smooth "double green breathe" (identical to the initial boot connection animation) rather than a solid orange block.
- **Unpair State Reset**: The device now explicitly resets its internal mute state to `false` when disconnecting, guaranteeing it starts unmuted when connecting to a new host.
- Disabled the Right Touch Sensor (`ENABLE_TOUCH_RIGHT false`) in the settings block per user request.

- **Encoder Ghost Clicks / Lost Clicks**: Fixed a critical logic flaw where the encoder would permanently drop physical clicks if the device was busy playing an animation (e.g. the 540ms orange flash for touch controls or red disconnect flashes). The encoder now accurately tracks the absolute delta in the background and processes every single physical click when the loop resumes.
- **Disconnected Ghost Click Bomb**: Fixed an issue where spinning the encoder while the device was disconnected would invisibly queue up hundreds of clicks in the background, only to unleash all of them instantly (blasting the volume) the moment a PC connected. The encoder position is now properly zeroed while searching for a host.
- **Touch Sensor Unpair Lockout**: Fixed a bug where the touch sensors would become permanently disabled after successfully unpairing the device using the encoder hold sequence. The internal hold-state flag now properly resets upon releasing the button.
- **System Bluetooth Logs**: The device now clearly prints `[SYSTEM] Bluetooth Connected (MAC: ...)` and `[SYSTEM] Bluetooth Disconnected` in the serial monitor so you can easily track its connection state and the MAC address of the connected PC.
- **Serial Monitor Timestamps**: Added millisecond timestamps `[000000]` to every single line in the serial monitor so you can precisely measure the time delta between actions and events from the moment the ESP32 boots up.
- **Visual "Blank Ring" Bug**: Fixed an edge case where aborting an unpair sequence, finishing a touch double-tap, toggling the BOOT button LEDs, or unmuting the PC would wipe the LEDs blank. The device now implements a robust `restoreLEDState()` function to seamlessly rebuild the visual state (whether in standard mode or Brightness Mode) without waiting for the user to touch the knob again.
- **Brightness Wake-Up Bug**: Fixed an issue where the LED ring would permanently get stuck at 0 brightness if the user woke the device from sleep by turning the encoder. The device now remembers and restores the previous brightness level upon waking.
- **Mute LED Override Bug**: Fixed a bug where the red "Mute" breathing animation would aggressively overwrite the orange flashing animation if the user attempted to unpair the device while muted.
- **Brightness Mute-Desync Bug**: Fixed a logic desync where rotating the encoder to adjust brightness would inadvertently clear the hardware mute flag without actually unmuting the PC audio.
- **Touch Sensor Logs**: Updated the serial monitor debugging statements to accurately report double and triple taps, rather than eagerly logging every physical touch as a single tap.

## [2026-06-26 13:20:00 IST]### Fixed
- **Double-Mute Bug**: Fixed an issue where the second press during the "press-once-then-press-and-hold" disconnect sequence would trigger an unintended secondary audio mute event on the host PC. Muting now strictly only occurs on the first tap.
- **Unpair Button Logic**: Simplified and fixed the internal state machine for the disconnection sequence, preventing the sequence from requiring inhumanly fast clicking speeds and removing buggy sticky state variables. Increased the double-tap window to 800ms.
- **Boot Crash / Bootloop**: Fixed a severe logic flaw where the hardware unpair logic incorrectly nested the entire loop logic. Deleted a corrupted `partitions.csv` file that previously caused the ESP32-C3 to instantly kernel panic upon boot due to attempting to map 16MB of flash partitions on a 4MB hardware chip.

## [2026-06-26 12:08:00 IST]

### Added
- **Encoder Acceleration**: Added dynamic volume adjustment. Spinning the encoder faster increases the volume step size (multiplier) to easily jump from 0 to 100 without spinning endlessly.
- **Double-Tap & Hold Disconnect**: The rotary encoder button now supports a double-tap-and-hold (for 3 seconds) feature to manually unpair the device from the current host. Holding it triggers accelerating red LED flashes and synchronized haptic ticks before forcing a disconnect.
- **Double-Vibration for Mute**: Changed the single-press haptic feedback on the mute button to a distinct double-vibration pulse.

### Changed
- **Bluetooth Advertising Payload (Android Fix)**: Restructured the BLE payload to fix the "App needed to use this device" error on Samsung/Android devices. Moved the "Esp32 Media Knob" name to the secondary Scan Response packet while prioritizing standard HID and Battery Service UUIDs in the primary 31-byte advertisement packet.

## [2026-06-05 20:55:00 IST]

### Added
- **Microsoft Swift Pair Integration:** 
  - Added Microsoft Vendor ID (`0x0006`) payload to the BLE advertisement data.
  - Windows 10/11 devices will now display a native "New Keyboard Found" notification popup immediately when the ESP32 enters pairing mode.

### Changed
- Migrated the global `HijelHID_BLEKeyboard` library to a local `src/` folder inside the sketch directory to prevent the Swift Pair hack from polluting other Arduino projects.
- Added a 600ms filter delay to the 500ms pairing request haptic vibration. This prevents the motor from vibrating erroneously during standard, fast auto-reconnections to already paired devices.

## [2026-06-04 21:30:00 IST]

### Added
- Multi-tap functionality for Touch Sensor 1 (Pin 20):
  - Single tap: Play/Pause
  - Double tap: Next Track
  - Triple tap: Previous Track
- Brightness adjustment mode via Touch Sensor 2 (Pin 21):
  - Tapping Touch Sensor 2 fades the entire LED ring in/out to indicate the mode.
  - While active, the rotary encoder adjusts the master LED brightness instead of media volume.
- Long-press disconnect via BOOT button (Pin 9):
  - Holding the button for 2 seconds will force the ESP32 to drop the current Bluetooth connection, flash the LEDs red, and immediately re-enter pairing mode.

### Changed
- Re-assigned Touch Sensor pins from 8/9 to 20/21 to avoid ESP32-C3 strapping pin conflicts which were preventing boot.

## [2026-06-04 21:12:00 IST]

### Added
- Integrated a haptic vibration motor on Pin 10.
  - Vibrates for a crisp 80ms duration upon turning the encoder.
  - Vibrates for 110ms upon pressing the mute button.
  - Added software logic to prevent 100% duty cycle lockup if the encoder is rotated extremely rapidly.
- Created a `USE_KEYBOARD_ARROWS` macro in the Settings block for easy toggling between Media Volume and Keyboard Arrows mode.
- Patched the underlying `HijelHID_BLEKeyboard` library to hide the "Keyboard" usage from the HID descriptor and explicitly set the Appearance to `0x0181` (Audio/Video Remote Control), forcing phones to display a remote icon instead of a keyboard.

### Changed
- Customized the BLE Manufacturer to "kunsh13" and the Device Name to "ESP32 kunsh13 Media Controller".
- Optimized LED animations:
  - Red breathing animation on mute now computes its phase shift based on the timestamp of the button press, ensuring it ALWAYS starts exactly at 0 brightness without any harsh flash.
  - The 1.5s sleep fade animation is now fully interruptible, allowing the device to wake up instantly if the encoder is touched mid-fade.
- Re-mapped the Boot button (Pin 9) to act as a master toggle to turn all LEDs completely on or off.

## [2026-06-04 19:51:00 IST]

### Added
- Implemented a complete LED state flow:
  - **Boot:** Single slow "Fire Orange" breathing animation (uses sine wave math for a perfectly smooth fade to black).
  - **Pairing Mode:** Continuous slow Blue breathing animation while unconnected.
  - **Connected (Idle):** All LEDs turn OFF after 1.5 seconds of inactivity.
  - **Connected (Active):** A single LED moves in the direction of encoder rotation.
  - **Connected (Muted):** Breathes Red continuously. Unmutes when button is pressed again or when volume is changed.

## [2026-06-04 19:46:00 IST]

### Changed
- Replaced the simple polling/software reading of the rotary encoder with the `RotaryEncoder` library by Matthias Hertel. 
- *(Note: Initially attempted to use `ESP32Encoder` for hardware PCNT support, but discovered the ESP32-C3 chip physically lacks the PCNT peripheral. `RotaryEncoder` uses efficient software interrupts instead).*
- Enabled internal pull-up resistors in the code to support bare rotary encoders (e.g., EC11) that lack PCB-mounted pull-ups.

## [2026-06-04 19:26:00 IST]

### Changed
- Switched BLE keyboard backend library from `BleKeyboard` to `HijelHID_BLEKeyboard` to fix compatibility issues with ESP32 Core v3.x.
- Refactored `.write()` logic to use `.tap()` per the new library's API.

## [2026-06-04 19:22:00 IST]

### Changed
- Reassigned GPIO pins based on hardware requirements:
  - NeoPixel to Pin 4
  - Rotary Encoder to Pins 5, 6, 7
  - Touch Sensors to Pins 8, 9

## [2026-06-04 19:14:00 IST]

### Added
- Initial project setup for ESP32-C3 Supermini.
- BLE Keyboard emulation using `ESP32-BLE-Keyboard` library.
- Rotary encoder support for Volume Up/Down and YouTube seeking (Left/Right Arrow).
- Hardware interrupt-based rotary encoder handling.
- Push button support on rotary encoder for Mute functionality.
- Touch Sensor 1 integration for Play/Pause media control.
- Touch Sensor 2 integration to toggle rotary encoder functionality (Volume mode vs. Seek mode).
- 8-LED NeoPixel ring integration (`Adafruit_NeoPixel`) for dynamic animations and mode indication.
- Software debouncing for touch sensors and encoder push button.
