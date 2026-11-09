# How the simulation works

ESP32 Field Lab does not compile C++, emulate an ESP32 CPU, or connect to a physical board. It still looks as if the sketch is running. That's because each lesson has a JavaScript model of what the circuit should do, and three parts of the page read from it:

- the 3D scene
- the "Simulation output" panel
- the code tab

The Arduino sketch itself is only displayed text.

## The output panel is generated text

`logState()` in `dist/app.js` has one template string per lesson. It fills each one from the slider value, the pressed state and a timer.

| Lesson | What gets printed | How it's worked out |
| --- | --- | --- |
| Blink | `GPIO23 → HIGH · LED on` / `LOW · LED off` | Elapsed time ÷ interval, even or odd |
| Button | `GPIO18 = LOW · pressed · LED on` | Whether the button is currently held |
| Potentiometer | `ADC → 2048 / 4095 · PWM → 127 / 255` | Dial position scaled to 0–4095, then to 0–255 |
| Light | `Light 40% · ADC 1638 · threshold 1200 · LED off` | Slider value compared with a fixed threshold |

None of this comes from running `digitalWrite` or `Serial.println`. The panel is styled like a serial monitor (its `aria-label` is "Serial monitor") to make it feel real.

## The 3D LED uses the same kind of formula

The render loop (`tick()` in `dist/scene.js`) sets the LED's brightness separately for each lesson:

- **Blink** turns the LED on and off every interval.
- **Button** lights it while the button is held.
- **Potentiometer** and **Force** set brightness from the control value.
- **Light** turns it on below a fixed threshold of 1200/4095.

Both formulas follow the same timing, so the panel and the LED agree, but nothing links them to the C++.

## The code tab is edited text

The six output lessons each name one constant in `dist/app.js`:

| Lesson | Constant |
| --- | --- |
| Blink | `INTERVAL_MS` |
| RGB LED | `HUE_DEGREES` |
| Piezo sound | `FREQUENCY_HZ` |
| DC motor | `DUTY` |
| Servo | `TARGET_ANGLE` |
| Stepper motor | `TARGET_DEGREES` |

When you move the slider, a regex rewrites that one line in the sketch, for example:

```cpp
const int INTERVAL_MS = 1500;
```

Build mode does the same in `buildSketch()` (`dist/build-projects.js`). The demo and the downloaded sketch match because both read the same slider value, not because the code was compiled.

## Build mode checks the wiring, not the program

`validateCircuit()` in `dist/build-circuit.js` follows the wires through the breadboard to find what is connected. It then checks:

- LED direction
- that a resistor limits the current
- occupied holes
- power shorts

`simulationState()` then sets the LED state from what is connected to what. For example, Blink needs a path from GPIO23 through the resistor and LED to GND before the LED will blink. This is a limited electrical model of three fixed projects (Blink, Button and Potentiometer), not a circuit simulator.

## Sound and motion are hand-made

- The piezo and mini piano lessons play real tones through the browser's audio.
- The servo, DC motor and stepper are animated from the control value.

## Summary

The panel, the 3D scene and the code tab all use the same values, so it looks like the sketch is running. In fact, each lesson's behaviour is written separately in JavaScript. If you edited the sketch, nothing on screen would change except the one constant tied to the slider.
