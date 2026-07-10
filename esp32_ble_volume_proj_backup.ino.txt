#include "src/HijelHID_BLEKeyboard/HijelHID_BLEKeyboard.h"
#include <Adafruit_NeoPixel.h>
#include <RotaryEncoder.h>

// --- PIN DEFINITIONS ---
// You can adjust these if your wiring is different
#define ENCODER_CLK  5
#define ENCODER_DT   6
#define ENCODER_SW   7
#define NEOPIXEL_PIN 4
#define BOOT_BUTTON_PIN 9 // The ESP32-C3 onboard BOOT button
#define HAPTIC_PIN 10     // Vibration motor pin
#define TOUCH_PIN_LEFT  20   // Multi-tap touch sensor (Media Control)
#define TOUCH_PIN_RIGHT  21   // Brightness mode toggle
#define NUMPIXELS 8

// --- SETTINGS ---
// Set to true if you are mounting the NeoPixel ring upside-down (face down) over the encoder.
// This reverses the LED animation direction so it visually matches your physical rotation.
#define RING_UPSIDE_DOWN true

// Default RGB Color for the rotating LED (0-255)
// Currently set to Fire Red
#define DEFAULT_COLOR_R 255
#define DEFAULT_COLOR_G 35
#define DEFAULT_COLOR_B 0

// Vibration Motor Settings (in milliseconds)
// How long the motor vibrates when turning the knob
#define HAPTIC_ROTATION_MS 80
// How long the motor vibrates when pressing the mute button
#define HAPTIC_MUTE_MS 110

// Keyboard Control Mode
// Set to true to send Left/Right Arrow keyboard keys when turning the knob (great for YouTube scrubbing).
// Set to false to send standard Media Volume Up/Volume Down.
#define USE_KEYBOARD_ARROWS false

// Touch Sensor Modules
// Set to true to enable the individual touch sensor modules.
// Touch Left (Pin 20): Single Tap = Play/Pause | Double Tap = Next Track | Triple Tap = Prev Track
#define ENABLE_TOUCH_LEFT true

// Touch Right (Pin 21): Toggle Brightness Mode
#define ENABLE_TOUCH_RIGHT false

// --- ACCELERATION SETTINGS ---
// Enable to increase volume rate when you spin the knob continuously/faster
const bool ENABLE_ACCELERATION = true;
const unsigned long ACCEL_FAST_MS = 30;   // ms per click threshold for fast spin
const int ACCEL_FAST_MULTIPLIER = 5;      // Sends 5 volume ticks per click
const unsigned long ACCEL_MEDIUM_MS = 60; // ms per click threshold for medium spin
const int ACCEL_MEDIUM_MULTIPLIER = 2;    // Sends 2 volume ticks per click

// --- OBJECTS ---
// Initialize BLE Keyboard (Name, Manufacturer, Initial Battery Level)
HijelHID_BLEKeyboard bleKeyboard("Esp32 Media Knob", "kunsh", 100);

// Initialize NeoPixel ring
Adafruit_NeoPixel pixels(NUMPIXELS, NEOPIXEL_PIN, NEO_GRB + NEO_KHZ800);

// Initialize RotaryEncoder (Matthias Hertel library)
// FOUR3 is the standard 4-step-per-notch mode for most cheap EC11 bare encoders
RotaryEncoder encoder(ENCODER_DT, ENCODER_CLK, RotaryEncoder::LatchMode::FOUR3);

// Interrupt service routine for the encoder
void IRAM_ATTR checkPosition() {
  encoder.tick();
}

// --- STATE VARIABLES ---
int currentPixel = 0;
uint32_t modeColor;
bool wasConnected = false; // Tracks if we have already played the connection animation
bool pairingHapticTriggered = false; // Tracks if we have already vibrated for the pairing request
unsigned long pairingRequestStartTime = 0; // Tracks how long we've been waiting for a pairing handshake
bool isMuted = false;      // Tracks if we are currently muted
bool isIdle = true;        // Tracks if the LEDs are currently sleeping
bool ledsEnabled = true;   // Master toggle for all NeoPixels
unsigned long lastInteractionTime = 0; // Tracks inactivity for auto-off
int currentBrightness = 50; // Master brightness (0-255)
bool isBrightnessMode = false; // Are we currently adjusting brightness?

// Debouncing timers
unsigned long lastButtonPress = 0;
const unsigned long debounceDelay = 50; // milliseconds
bool lastButtonState = HIGH;

// Boot Button State
bool lastBootButtonState = HIGH;
unsigned long bootButtonDownTime = 0;
bool longPressTriggered = false;

// Mute Button Hold State
unsigned long muteHoldStartTime = 0;
bool muteLongPressTriggered = false;
int encoderTapCount = 0;
unsigned long lastEncoderReleaseTime = 0;
const unsigned long encoderDoubleTapWindow = 400; // ms

// Multi-tap state for Touch Sensor 1
bool lastTouchLeftState = LOW; // Assuming active-high TTP223 touch sensors
int touchLeftTapCount = 0;
unsigned long lastTouchLeftTapTime = 0;
const unsigned long tapTimeoutDelay = 200; // ms to wait before executing taps

// State for Touch Sensor 2
bool lastTouchRightState = LOW;
unsigned long lastTouchRightPress = 0;

void setup() {
  Serial.begin(115200);
  
  // Start NeoPixels FIRST, before any BLE or interrupts to ensure RMT configures correctly
  pixels.begin();
  pixels.setBrightness(currentBrightness); 
  pixels.clear();
  pixels.show();
  
  delay(1000); // Give the LED ring 1 second to power up properly

  // Boot Animation: Fire Orange (255, 69, 0) Breathe ONCE
  // Using a sine wave from 0 to 180 degrees ensures it starts at 0 and fades perfectly back to 0
  for (int j = 0; j <= 180; j += 1) {
    float breath = sin(j * PI / 180.0);
    uint8_t r = 255 * breath;
    uint8_t g = 69 * breath;
    for(int i=0; i<NUMPIXELS; i++) {
      pixels.setPixelColor(i, pixels.Color(r, g, 0));
    }
    pixels.show();
    delay(10);
  }
  
  pixels.clear();
  pixels.show();

  // Setup Encoder pins (MUST use INPUT_PULLUP for bare encoders to prevent floating noise!)
  pinMode(ENCODER_SW, INPUT_PULLUP);
  pinMode(ENCODER_DT, INPUT_PULLUP);
  pinMode(ENCODER_CLK, INPUT_PULLUP);
  pinMode(BOOT_BUTTON_PIN, INPUT_PULLUP);
  
  pinMode(HAPTIC_PIN, OUTPUT);
  digitalWrite(HAPTIC_PIN, LOW); // Ensure motor is off at boot
  
  // Attach interrupts for the rotary encoder to handle bouncing perfectly in software
  attachInterrupt(digitalPinToInterrupt(ENCODER_DT), checkPosition, CHANGE);
  attachInterrupt(digitalPinToInterrupt(ENCODER_CLK), checkPosition, CHANGE);
  
  if (ENABLE_TOUCH_LEFT) {
    pinMode(TOUCH_PIN_LEFT, INPUT);
  }
  if (ENABLE_TOUCH_RIGHT) {
    pinMode(TOUCH_PIN_RIGHT, INPUT);
  }
  
  // Start BLE
  Serial.println("Starting BLE Keyboard...");
  bleKeyboard.begin();
  
  // Set default mode color using user settings
  modeColor = pixels.Color(DEFAULT_COLOR_R, DEFAULT_COLOR_G, DEFAULT_COLOR_B); 
}

// Function to flash a solid color across all LEDs
void flashColor(uint32_t color) {
  for(int i=0; i<NUMPIXELS; i++) {
    pixels.setPixelColor(i, color);
  }
  pixels.show();
  delay(150);
  pixels.clear();
  pixels.show();
}

// Function to draw a spinning dot based on rotation direction
void spinAnimation(int dir) {
  pixels.clear();
  
  // If the ring is physically mounted upside down, mirror the animation direction
  if (RING_UPSIDE_DOWN) {
    dir = -dir;
  }
  
  // If we were sleeping, wake up on the exact same LED instead of jumping forward
  if (isIdle) {
    Serial.printf("[%02lu:%02lu:%02lu] [DEBUG] ", (millis()/1000)/3600, ((millis()/1000)%3600)/60, (millis()/1000)%60); Serial.println(" Waking up from Idle State");
    pixels.setBrightness(currentBrightness); // Restore brightness!
  } else {
    if (dir > 0) {
      currentPixel = (currentPixel + 1) % NUMPIXELS;
    } else {
      currentPixel = (currentPixel - 1 + NUMPIXELS) % NUMPIXELS;
    }
  }
  isIdle = false; // We are now awake
  
  // Draw the bright head
  pixels.setPixelColor(currentPixel, modeColor);
  
  // Draw a very short, dim tail so the movement looks like a tiny comet instead of choppy
  uint8_t mr = (modeColor >> 16) & 0xFF;
  uint8_t mg = (modeColor >> 8) & 0xFF;
  uint8_t mb = modeColor & 0xFF;
  int tailPixel = dir > 0 ? (currentPixel - 1 + NUMPIXELS) % NUMPIXELS : (currentPixel + 1) % NUMPIXELS;
  pixels.setPixelColor(tailPixel, pixels.Color(mr/4, mg/4, mb/4));
  
  if (ledsEnabled) {
    pixels.show();
  }
}

// Function to restore the current idle or brightness visual state
void restoreLEDState() {
  if (isBrightnessMode) {
    for (int i = 0; i < NUMPIXELS; i++) pixels.setPixelColor(i, modeColor);
  } else {
    pixels.clear();
    if (!isIdle) {
      pixels.setPixelColor(currentPixel, modeColor);
      // Draw the tail so it doesn't look like a singular LED bug!
      uint8_t mr = (modeColor >> 16) & 0xFF;
      uint8_t mg = (modeColor >> 8) & 0xFF;
      uint8_t mb = modeColor & 0xFF;
      int tailPixel = (currentPixel - 1 + NUMPIXELS) % NUMPIXELS;
      pixels.setPixelColor(tailPixel, pixels.Color(mr/4, mg/4, mb/4));
    }
  }
  if (ledsEnabled) pixels.show();
}

// --- HAPTIC FEEDBACK ---
unsigned long hapticStartTime = 0;
unsigned long hapticDuration = 0;
bool hapticActive = false;
unsigned long lastHapticStopTime = 0; // Tracks when the motor last turned off to ignore ghost inputs

void triggerHaptic(unsigned long durationMs = 30) {
  // If we are already vibrating, don't restart the timer if the new request is the same or shorter.
  // This prevents the motor from being held at 100% continuously if you spin the encoder very fast!
  if (hapticActive && durationMs <= hapticDuration) {
    return;
  }
  digitalWrite(HAPTIC_PIN, HIGH);
  hapticActive = true;
  hapticStartTime = millis();
  hapticDuration = durationMs;
}


void loop() {

  // Update encoder in the main loop as well for safety
  encoder.tick();

  // Handle non-blocking haptic motor turn off (using subtraction prevents millis() rollover bugs)
  if (hapticActive && (millis() - hapticStartTime >= hapticDuration)) {
    digitalWrite(HAPTIC_PIN, LOW);
    hapticActive = false;
    lastHapticStopTime = millis();
  }

  if (bleKeyboard.isPaired()) {
    pairingRequestStartTime = 0; // Reset pairing request timer
    
    // Check if we just connected
    if (!wasConnected) {
      // Just connected! Breathe green twice
      for (int loops = 0; loops < 2; loops++) {
        for (int j = 0; j <= 180; j += 3) { // Faster breathe for the connection success
          float breath = sin(j * PI / 180.0);
          uint8_t g_val = 255 * breath;
          for(int i=0; i<NUMPIXELS; i++) {
            pixels.setPixelColor(i, pixels.Color(0, g_val, 0));
          }
          if (ledsEnabled) pixels.show();
          delay(8);
        }
      }
      pixels.clear();
      if (ledsEnabled) pixels.show();
      wasConnected = true;
      lastInteractionTime = millis();
    }
    
    // --- 1. READ ROTARY ENCODER ---
    int newPos = encoder.getPosition();
    if (newPos != 0) {
      triggerHaptic(HAPTIC_ROTATION_MS);
      
      int delta = abs(newPos);
      
      // Calculate dynamic acceleration rate based on spin velocity
      int multiplier = 1;
      if (ENABLE_ACCELERATION) {
        unsigned long msBetween = encoder.getMillisBetweenRotations();
        if (msBetween > 0) {
          if (msBetween < ACCEL_FAST_MS) {
            multiplier = ACCEL_FAST_MULTIPLIER;
          } else if (msBetween < ACCEL_MEDIUM_MS) {
            multiplier = ACCEL_MEDIUM_MULTIPLIER;
          }
        }
      }

      int totalTaps = delta * multiplier;

      if (encoder.getDirection() == RotaryEncoder::Direction::CLOCKWISE) {
        // Clockwise Rotation
        if (isBrightnessMode) {
          currentBrightness = min(255, currentBrightness + (15 * totalTaps));
          pixels.setBrightness(currentBrightness);
          Serial.printf("[DEBUG] Encoder Rotated CW (Brightness Mode). New Brightness: %d\n", currentBrightness);
          for(int i=0; i<NUMPIXELS; i++) pixels.setPixelColor(i, modeColor);
          if (ledsEnabled) pixels.show();
        } else {
          Serial.printf("[DEBUG] Encoder Rotated CW (Volume Mode). Sending %d Ticks\n", totalTaps);
          for(int t=0; t<totalTaps; t++) {
            if (USE_KEYBOARD_ARROWS) bleKeyboard.tap(KEY_RIGHT);
            else bleKeyboard.tap(MEDIA_VOLUME_UP);
          }
          for(int i=0; i<delta; i++) spinAnimation(1); // Spin forwards
        }
      } else {
        // Counter-clockwise Rotation
        if (isBrightnessMode) {
          currentBrightness = max(5, currentBrightness - (15 * totalTaps));
          pixels.setBrightness(currentBrightness);
          Serial.printf("[DEBUG] Encoder Rotated CCW (Brightness Mode). New Brightness: %d\n", currentBrightness);
          for(int i=0; i<NUMPIXELS; i++) pixels.setPixelColor(i, modeColor);
          if (ledsEnabled) pixels.show();
        } else {
          Serial.printf("[DEBUG] Encoder Rotated CCW (Volume Mode). Sending %d Ticks\n", totalTaps);
          for(int t=0; t<totalTaps; t++) {
            if (USE_KEYBOARD_ARROWS) bleKeyboard.tap(KEY_LEFT);
            else bleKeyboard.tap(MEDIA_VOLUME_DOWN);
          }
          for(int i=0; i<delta; i++) spinAnimation(-1); // Spin backwards
        }
      }
      
      // Reset the position to 0 to act as a relative encoder
      encoder.setPosition(0);
      lastInteractionTime = millis(); // Reset idle timer
      if (!isBrightnessMode) {
        isMuted = false; // Turning volume implicitly unmutes the PC, so we reset our red breathing state too
      }
    }
    
    // Check if we are currently safe from vibration noise (mechanical bouncing / electrical spikes)
    bool isSafeFromVibration = !hapticActive && (millis() - lastHapticStopTime > 50);
    // Stricter safety specifically for the highly sensitive capacitive touch inputs
    bool isTouchSafe = isSafeFromVibration && (encoderTapCount == 0) && !muteLongPressTriggered;

    
    // --- 2. READ ENCODER BUTTON (MUTE & UNPAIR) ---
    // Mask mechanical button reads during haptic vibration to completely eliminate physical contact chatter
    bool rawButtonState = digitalRead(ENCODER_SW);
    bool currentButtonState = isSafeFromVibration ? rawButtonState : lastButtonState;
    
    // Auto-reset tap count if the user hasn't pressed the button within the multi-tap window
    if (currentButtonState == HIGH && (millis() - lastEncoderReleaseTime > 800) && encoderTapCount > 0) {
        Serial.printf("[%02lu:%02lu:%02lu] [DEBUG] ", (millis()/1000)/3600, ((millis()/1000)%3600)/60, (millis()/1000)%60); Serial.println(" Tap window expired, resetting encoderTapCount to 0");
        encoderTapCount = 0;
    }
    
    // Detect button press down
    if (currentButtonState == LOW && lastButtonState == HIGH) { 
      if (millis() - lastButtonPress > debounceDelay) {
        lastButtonPress = millis();
        Serial.printf("[%02lu:%02lu:%02lu] [DEBUG] ", (millis()/1000)/3600, ((millis()/1000)%3600)/60, (millis()/1000)%60); Serial.println(" Button Press Down Detected");
        
        // Tap tracking for multi-press-and-hold (800ms window for second tap)
        if (millis() - lastEncoderReleaseTime < 800) {
            encoderTapCount++;
            Serial.print("[DEBUG] Tap window valid. Tap count: ");
            Serial.println(encoderTapCount);
        } else {
            encoderTapCount = 1;
            Serial.printf("[%02lu:%02lu:%02lu] [DEBUG] ", (millis()/1000)/3600, ((millis()/1000)%3600)/60, (millis()/1000)%60); Serial.println(" New tap sequence. Tap count: 1");
        }
        
        // Mute ONLY on the first press to prevent double-muting during the sequence
        if (encoderTapCount == 1) {
          digitalWrite(HAPTIC_PIN, HIGH);
          delay(60);
          digitalWrite(HAPTIC_PIN, LOW);
          delay(80);
          digitalWrite(HAPTIC_PIN, HIGH);
          delay(60);
          digitalWrite(HAPTIC_PIN, LOW);
          hapticActive = false;
          lastHapticStopTime = millis();
          
          bleKeyboard.tap(MEDIA_MUTE);
          isMuted = !isMuted;
          if (!isMuted) {
             restoreLEDState();
          }
        } else if (encoderTapCount == 2) {
          // Small bump to confirm second tap
          triggerHaptic(50);
        } else if (encoderTapCount == 3) {
          // Acknowledge the start of the hold sequence
          triggerHaptic(100);
        }
        
        lastInteractionTime = millis();
        muteHoldStartTime = millis();
        muteLongPressTriggered = false;
      }
    }
    
    // While button is held down
    if (currentButtonState == LOW && !muteLongPressTriggered && (millis() - lastButtonPress > debounceDelay)) {
      if (encoderTapCount >= 2) { // Unpair on SECOND (or more) press and hold
        unsigned long holdTime = millis() - muteHoldStartTime;
        
        // If held for 3 seconds (3000ms), trigger unpair
        if (holdTime >= 3000) {
          Serial.printf("[%02lu:%02lu:%02lu] [DEBUG] ", (millis()/1000)/3600, ((millis()/1000)%3600)/60, (millis()/1000)%60); Serial.println(" Unpair sequence successfully triggered!");
          muteLongPressTriggered = true;
          triggerHaptic(800); // Stronger vibration for unpair confirmation
          
          bleKeyboard.end();
          NimBLEDevice::deleteAllBonds();
          delay(100);
          bleKeyboard.begin();
          wasConnected = false;
          isMuted = false;
          isBrightnessMode = false;
          
          // Just unpaired! Breathe green twice
          for (int loops = 0; loops < 2; loops++) {
            for (int j = 0; j <= 180; j += 3) {
              float breath = sin(j * PI / 180.0);
              uint8_t g_val = 255 * breath;
              for(int i=0; i<NUMPIXELS; i++) {
                pixels.setPixelColor(i, pixels.Color(0, g_val, 0));
              }
              if (ledsEnabled) pixels.show();
              delay(8);
            }
          }
          pixels.clear();
          if (ledsEnabled) pixels.show();
          
          encoderTapCount = 0; // Reset state
        } else if (holdTime > 500) {
          // Flash fire orange faster and faster while holding
          int currentInterval = map(holdTime, 500, 3000, 400, 40);
          static unsigned long lastFlashTime = 0;
          static bool flashIsOn = false;
          
          if (millis() - lastFlashTime >= currentInterval) {
            flashIsOn = !flashIsOn;
            lastFlashTime = millis();
            
            if (flashIsOn) {
              for(int p=0; p<NUMPIXELS; p++) pixels.setPixelColor(p, pixels.Color(255, 100, 0));
              if (ledsEnabled) pixels.show();
              triggerHaptic(min(80, currentInterval)); // Stronger vibration from the start!
            } else {
              pixels.clear();
              if (ledsEnabled) pixels.show();
            }
          }
        }
      }
    }
    
    // Detect button release
    if (currentButtonState == HIGH && lastButtonState == LOW) {
       lastEncoderReleaseTime = millis();
       Serial.printf("[%02lu:%02lu:%02lu] [DEBUG] ", (millis()/1000)/3600, ((millis()/1000)%3600)/60, (millis()/1000)%60); Serial.println(" Button Release Detected");
       // If this was a multi-press that didn't hold long enough, clear LEDs
       if (encoderTapCount >= 2 && !muteLongPressTriggered && (millis() - muteHoldStartTime > 500)) {
           restoreLEDState();
       }
       muteLongPressTriggered = false; // Fix: Reset the hold state so touch sensors aren't permanently disabled
    }
    
    lastButtonState = currentButtonState;
    
    // --- 3. READ BOOT BUTTON (RGB TOGGLE & DISCONNECT) ---
    // Mask mechanical button reads during haptic vibration
    bool rawBootButtonState = digitalRead(BOOT_BUTTON_PIN);
    bool currentBootButtonState = isSafeFromVibration ? rawBootButtonState : lastBootButtonState;
    
    // Detect button press down
    if (currentBootButtonState == LOW && lastBootButtonState == HIGH) {
      bootButtonDownTime = millis();
      longPressTriggered = false;
    }
    
    // While button is held down
    if (currentBootButtonState == LOW) {
      // If held for 2 seconds (2000ms) and we haven't triggered the long press yet
      if (millis() - bootButtonDownTime > 2000 && !longPressTriggered) {
        longPressTriggered = true;
        triggerHaptic(200); // Long confirmation vibration
        
        // Force disconnect, clear bonds, and restart pairing mode
        bleKeyboard.end();
        NimBLEDevice::deleteAllBonds(); // Clear saved pairings to prevent instant auto-reconnect
        delay(100);
        bleKeyboard.begin();
        wasConnected = false;
        isMuted = false;
        isBrightnessMode = false;
        
        // Flash Red to indicate disconnect
        flashColor(pixels.Color(255, 0, 0));
      }
    }
    
    // Detect button release
    if (currentBootButtonState == HIGH && lastBootButtonState == LOW) {
      // If it was a short press (debounced, but less than 2 seconds)
      if (!longPressTriggered && millis() - bootButtonDownTime > 50) {
        ledsEnabled = !ledsEnabled; // Toggle LED master flag
        if (!ledsEnabled) {
           pixels.clear(); // Immediately turn off LEDs
           pixels.show();
        } else {
           // Immediately wake them up to show current position
           restoreLEDState();
           lastInteractionTime = millis();
        }
      }
    }
    lastBootButtonState = currentBootButtonState;

    // --- 4. READ TOUCH SENSORS (MULTI-TAP LOGIC) ---
    if (ENABLE_TOUCH_LEFT) {
      bool currentTouchLeft = digitalRead(TOUCH_PIN_LEFT);
      
      // Detect rising edge (finger touched)
      if (isTouchSafe && currentTouchLeft == HIGH && lastTouchLeftState == LOW) {
        // Basic debounce: ignore touches that are way too fast
        if (millis() - lastTouchLeftTapTime > 50) { 
          touchLeftTapCount++;
          lastTouchLeftTapTime = millis();
        }
      }
      lastTouchLeftState = currentTouchLeft;

      // Check if tap timeout has expired to execute the command
      if (touchLeftTapCount > 0 && (millis() - lastTouchLeftTapTime > tapTimeoutDelay)) {
        int tapsToExecute = touchLeftTapCount;
        touchLeftTapCount = 0; // Reset tap counter early
        isMuted = false; // Playing media implicitly unmutes!
        
        // 1. Send Bluetooth command INSTANTLY
        if (tapsToExecute == 1) {
          Serial.printf("[%02lu:%02lu:%02lu] [DEBUG] ", (millis()/1000)/3600, ((millis()/1000)%3600)/60, (millis()/1000)%60); Serial.println(" Touch Left: Single Tap Executed (Play/Pause)");
          bleKeyboard.tap(MEDIA_PLAY_PAUSE);
        } else if (tapsToExecute == 2) {
          Serial.printf("[%02lu:%02lu:%02lu] [DEBUG] ", (millis()/1000)/3600, ((millis()/1000)%3600)/60, (millis()/1000)%60); Serial.println(" Touch Left: Double Tap Executed (Next Track)");
          bleKeyboard.tap(MEDIA_NEXT_TRACK);
        } else if (tapsToExecute >= 3) {
          Serial.printf("[DEBUG] Touch Left: %d Taps Executed (Prev Track)\n", tapsToExecute);
          bleKeyboard.tap(MEDIA_PREV_TRACK);
        }
        
        // 2. Play synchronized Fire Orange flashes and Haptic vibrations
        int pulses = min(tapsToExecute, 3);
        for (int i = 0; i < pulses; i++) {
          // Turn ON: Fire Orange and Vibration Motor
          for(int p=0; p<NUMPIXELS; p++) {
            pixels.setPixelColor(p, pixels.Color(255, 35, 0)); // Fire Orange
          }
          if (ledsEnabled) pixels.show();
          digitalWrite(HAPTIC_PIN, HIGH);
          
          delay(80); // Duration of the pulse
          
          // Turn OFF
          pixels.clear();
          if (ledsEnabled) pixels.show();
          digitalWrite(HAPTIC_PIN, LOW);
          
          if (i < pulses - 1) {
            delay(100); // Gap between pulses
          }
        }
        
        // Clean up haptic state to prevent interference with other functions
        hapticActive = false; 
        isIdle = true; // FORCE the ring back to sleep so it doesn't leave a singular LED floating!
        
        restoreLEDState(); // Restore visuals (prevent blank ring if in brightness mode)
        lastInteractionTime = millis();
      }
    }

    // Touch Sensor 2: Brightness Mode Toggle
    if (ENABLE_TOUCH_RIGHT) {
      bool currentTouchRight = digitalRead(TOUCH_PIN_RIGHT);
      if (isTouchSafe && currentTouchRight == HIGH && lastTouchRightState == LOW) {
        if (millis() - lastTouchRightPress > debounceDelay) {
          lastTouchRightPress = millis();
          triggerHaptic(100);
          isBrightnessMode = !isBrightnessMode;
          Serial.printf("[DEBUG] Touch Right Pressed. Toggled Brightness Mode: %s\n", isBrightnessMode ? "ON" : "OFF");
          
          if (isBrightnessMode) {
            // Enter Brightness Mode: Fade up all other LEDs
            bool wasIdle = isIdle;
            isIdle = false; // Ensure we are awake
            for (int b = 0; b <= 255; b += 15) {
              for (int i = 0; i < NUMPIXELS; i++) {
                if (i != currentPixel || wasIdle) {
                  uint8_t mr = (modeColor >> 16) & 0xFF;
                  uint8_t mg = (modeColor >> 8) & 0xFF;
                  uint8_t mb = modeColor & 0xFF;
                  pixels.setPixelColor(i, pixels.Color((mr * b)/255, (mg * b)/255, (mb * b)/255));
                }
              }
              if (ledsEnabled) pixels.show();
              delay(15);
            }
            // Ensure full ring is bright
            for(int i=0; i<NUMPIXELS; i++) pixels.setPixelColor(i, modeColor);
            if (ledsEnabled) pixels.show();
          } else {
            // Exit Brightness Mode: Fade out all other LEDs
            for (int b = 255; b >= 0; b -= 15) {
              for (int i = 0; i < NUMPIXELS; i++) {
                if (i != currentPixel) {
                  uint8_t mr = (modeColor >> 16) & 0xFF;
                  uint8_t mg = (modeColor >> 8) & 0xFF;
                  uint8_t mb = modeColor & 0xFF;
                  pixels.setPixelColor(i, pixels.Color((mr * b)/255, (mg * b)/255, (mb * b)/255));
                }
              }
              if (ledsEnabled) pixels.show();
              delay(15);
            }
            // Ensure only current pixel is on
            pixels.clear();
            pixels.setPixelColor(currentPixel, modeColor);
            if (ledsEnabled) pixels.show();
          }
          
          lastInteractionTime = millis();
        }
      }
      lastTouchRightState = currentTouchRight;
    }

    // --- LED IDLE & MUTE STATE MANAGEMENT ---
    bool isUnpairingHold = (currentButtonState == LOW && encoderTapCount >= 2 && (millis() - muteHoldStartTime > 500));
    
    if (isMuted && !isUnpairingHold && !isBrightnessMode) {
      // Continuous Red Breathing while Muted (Skip if we are currently holding to unpair!)
      // The - (PI/2.0) phase shift combined with the lastButtonPress delta ensures it ALWAYS starts exactly at 0 brightness
      float breath = (exp(sin((millis() - lastButtonPress)/1500.0 * PI - (PI/2.0))) - 0.36787944) * 108.0;
      for(int i=0; i<NUMPIXELS; i++) {
        pixels.setPixelColor(i, pixels.Color(breath, 0, 0));
      }
      if (ledsEnabled) pixels.show();
      delay(10); // Small delay to prevent flickering
    } else if (!isBrightnessMode) {
      // Normal state: Turn off all LEDs if idle for 1.5 seconds (skip if in brightness mode)
      if (millis() - lastInteractionTime > 1500) {
        if (!isIdle) {
          // Smoothly fade out ALL pixels synchronously using global brightness
          int startB = currentBrightness;
          int bStep = max(1, startB / 15); 
          for (int b = startB; b >= 0; b -= bStep) {
            pixels.setBrightness(b);
            if (ledsEnabled) pixels.show();
            delay(15);
            
            // OPTIMIZATION: Break out of the sleep animation instantly if the user touches the knob!
            // This prevents the device from feeling laggy if touched exactly while going to sleep.
            if (encoder.getPosition() != 0 || digitalRead(ENCODER_SW) == LOW || (ENABLE_TOUCH_LEFT && digitalRead(TOUCH_PIN_LEFT) == HIGH) || (ENABLE_TOUCH_RIGHT && digitalRead(TOUCH_PIN_RIGHT) == HIGH)) {
              break;
            }
          }
          pixels.clear();
          pixels.setBrightness(currentBrightness); // Restore global brightness for next wake up
          if (ledsEnabled) pixels.show();
          isIdle = true;
        }
      }
    }
    
  } else if (bleKeyboard.isConnected()) {
    // --- PAIRING REQUEST STATE ---
    encoder.setPosition(0); // Prevent ghost clicks from accumulating!
    
    if (pairingRequestStartTime == 0) {
      pairingRequestStartTime = millis();
    }
    
    // Only vibrate if we have been stuck in the "Connected but not Paired" state for >600ms.
    // This prevents the motor from vibrating during fast auto-reconnections to known devices!
    if (!pairingHapticTriggered && (millis() - pairingRequestStartTime > 600)) {
      triggerHaptic(500); // 500ms strong vibration to alert the user to look at their phone
      pairingHapticTriggered = true;
    }
    
    // FAST breathing blue animation to indicate pairing urgency
    float breath = (exp(sin(millis()/150.0*PI)) - 0.36787944)*108.0; 
    for(int i=0; i<NUMPIXELS; i++) {
      pixels.setPixelColor(i, pixels.Color(0, 0, breath));
    }
    if (ledsEnabled) pixels.show();
    delay(10);
    
  } else {
    // --- DISCONNECTED STATE ---
    encoder.setPosition(0); // Prevent ghost clicks from accumulating!
    wasConnected = false;           // Reset the success connection state
    pairingHapticTriggered = false; // Reset the pairing vibration state
    pairingRequestStartTime = 0;    // Reset the pairing request timer
    
    // SLOW breathing blue animation while passively waiting for a connection
    float breath = (exp(sin(millis()/2000.0*PI)) - 0.36787944)*108.0;
    for(int i=0; i<NUMPIXELS; i++) {
      pixels.setPixelColor(i, pixels.Color(0, 0, breath));
    }
    if (ledsEnabled) pixels.show();
    delay(10);
  }
}




