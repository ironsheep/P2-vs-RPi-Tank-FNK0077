# FNK0077 V2.0 Propeller 2 control design

## Scope and ownership

One `TankControl` object owns all P2 I/O for the V2.0 Freenove board. It controls the two tracks, three servo outputs, three line-sensor inputs, ultrasonic ranging, and four RGB pixels. Camera support is out of scope. Driving modes, autonomous behaviors, and any remote protocol belong in callers. The board photo also shows power and load switches and battery indicators. The reference provides no GPIO interface for those circuits.

The API below is written as a Spin2-style contract. A caller supplies a pin map when starting the object. The map uses named signals and accepts noncontiguous P2 pins. Timing implementations, cog allocation, and pin access stay private to the object.

## Proposed public API

| Method | Behavior |
| --- | --- |
| `start(pinMap) → status` | Validate the full map and start timing. Initialize both tracks stopped, TRIG low, pixels off, and all servo pulse trains disabled. |
| `stop()` | Stop both tracks, cancel ranging, disable servos, and clear the pixels before releasing owned resources. |
| `drive(left, right) → status` | Set both track commands together. Accept signed values from −4095 to +4095. Positive moves forward, negative reverses, and zero stops. Reject an out-of-range pair without changing either track. |
| `stopDrive()` | Stop both tracks while keeping servos, LEDs, and sensors available. |
| `setServoAngle(channel, degrees) → status` | Enable or update channel 0, 1, or 2 using its configured pulse and angle limits. Use 90–150° for channels 0 and 1 initially, matching the reference wrapper. Require calibrated limits for channel 2 before enabling it. |
| `disableServo(channel) → status` | Stop pulses on the selected channel. The mechanism can lose holding torque. |
| `readLine() → mask` | Return the current input levels as IR01 in bit 2, IR02 in bit 1, and IR03 in bit 0. |
| `requestRange() → status` | Start one asynchronous ultrasonic measurement. Return busy if a measurement is active or the retrigger interval has yet to elapse. |
| `readRange() → state, millimeters` | Return the current request's state and distance. States are idle, pending, valid, timeout, or fault. Distance is meaningful only when valid. |
| `setPixel(index, red, green, blue) → status` | Stage an RGB value for pixel 0–3. Each component is 0–255. |
| `showPixels() → status` | Submit a snapshot of the four staged pixels for transmission. Return busy while a frame is in flight. |

Use a single application caller for this interface. Publish multi-value updates coherently to the timing code so that a track pair or LED frame cannot be read halfway through an update. Reject invalid arguments before touching outputs. If startup fails, release acquired resources and leave the outputs inactive.

## Suggested contiguous pin layout

Use a block of **13 P2 pins** beginning at a base pin chosen for the target board. Group signals for wiring and review. Keep the explicit map available for adapter revisions.

| P2 offset | Signal | V2.0 BCM GPIO | Pi header pin |
| --- | --- | --- | --- |
| +0 | Left track forward | 24 | 18 |
| +1 | Left track reverse | 23 | 16 |
| +2 | Right track forward | 5 | 29 |
| +3 | Right track reverse | 6 | 31 |
| +4 | Servo 0 | 12 | 32 |
| +5 | Servo 1 | 13 | 33 |
| +6 | Servo 2 | 19, confirm continuity | 35, confirm continuity |
| +7 | RGB pixels DIN | 10 | 19 |
| +8 | Ultrasonic TRIG | 27 | 13 |
| +9 | Ultrasonic ECHO | 22 | 15 |
| +10 | IR01 | 16 | 36 |
| +11 | IR02 | 26, confirm continuity | 37, confirm continuity |
| +12 | IR03 | 21 | 40 |

The Freenove V2.0 photo shows three servo columns marked **19, 13, 12**. The extra servo is the column marked **19**, which suggests BCM GPIO19 at physical pin 35. The Pi 5 overlay also muxes GPIO19 to PWM, while the Python driver only drives GPIO12 and GPIO13. Confirm continuity from the extra servo's signal terminal to pin 35 on the actual board. If the signal is available only at the servo header, route the same P2 output directly to its signal terminal and retain the board's servo power and ground.

The photo marks the center IR terminal **[20]**, while the V2.0 README and active driver specify **GPIO26**. Check continuity to physical pins 37 and 38 before finalizing the harness. Keep the measured result in the pin map so the API stays the same.

## Operating the devices

Call `start(pinMap)` and check its status before issuing device commands. At startup both tracks are stopped, servo pulses are disabled, and the pixels are off. Use `stopDrive()` to halt movement while continuing to use the accessories. Call `stop()` when the application is finished with the board.

### Tracks

Each track has one two-wire DC motor. The Freenove board's H-bridge supplies motor current and reverses the polarity across those two wires to reverse the track. The two P2 pins per track are logic inputs to that bridge. The selected input sets direction, and PWM duty cycle sets the commanded drive level.

| Track command | Forward input | Reverse input |
| --- | --- | --- |
| Positive | PWM at the requested duty | Low |
| Negative | Low | PWM at the requested duty |
| Zero | Low | Low |

For either direction, the reference scales the command magnitude by 4096 to get duty cycle. A command of 2048 is 50% duty. Never assert both inputs on the same bridge. During reversal, bring both inputs low before enabling the other direction. Set the PWM frequency and reversal interval during bring-up. Verify whether both-low coasts or brakes on the actual board.

Examples use `drive(left, right)`. Use `drive(2048, 2048)` to move forward, `drive(-2048, -2048)` to reverse, and `drive(2048, -2048)` to pivot right. Unequal commands in the same direction steer toward the slower track.

### Servos

Call `setServoAngle(channel, degrees)` to start or update that channel's pulse train. Each output repeats one pulse every 20 ms (50 Hz). The reference scales 0–180° to 0.5–2.5 ms. Channels 0 and 1 initially use the reference command limits of 90–150°. Request `setServoAngle(0, 90)` and `setServoAngle(1, 140)` to reproduce the stock startup targets. Channel 2 uses the same API once its mechanical and pulse limits are calibrated. Call `disableServo(channel)` to stop that pulse train. The servo can lose holding torque when disabled.

### Line sensors

Call `readLine()` for a snapshot of the three digital input levels. For example, `%101` means IR01 and IR03 are high and IR02 is low. The reference test interprets that pattern as centered on black tape. Confirm the polarity and module orientation on the installed board before using the mask for steering.

### Ultrasonic range

Call `requestRange()` once. Poll `readRange()` until it returns valid, timeout, or fault. Use the distance only on a valid result. An accepted request replaces the previous result with pending. A busy response leaves the current measurement unchanged.

The controller sends a nominal 10 µs pulse on TRIG, then measures how long ECHO stays high. Convert the round-trip pulse width to one-way distance using 343 m/s. Apply bounded deadlines to both edges and wait for the configured retrigger interval before starting again.

### RGB pixels

Call `setPixel(index, red, green, blue)` for each change, then `showPixels()` once to transmit the staged frame. For example, `setPixel(0, 255, 0, 0)` followed by `showPixels()` sets the first pixel to red. To clear the chain while the controller is running, stage zeros for all four pixels and submit the frame. If `showPixels()` returns busy, retry after the current transfer ends.

The P2 sends all four pixels in GRB wire order on the single DIN line using the WS281x waveform. Use the reference's nominal 1.25 µs data bit period as the initial timing target, and hold the line low between frames for the latch interval. Keep motor and servo pulse generation and ECHO capture running during LED transfers.

## Adapter bring-up

1. With power removed, verify pin-1 orientation and continuity on the actual V2.0 board, especially the auxiliary servo and center IR routes. Verify signal levels at the Pi connector under power, particularly ECHO and the three IR inputs. Provide level conversion where needed, use a common ground, and supply the board and actuators through their power connections.
2. Check both track direction pairs with the tracks lifted. Test zero, forward, reverse, direction changes, shutdown, and reset before driving on the ground.
3. Scope all three servo outputs and confirm pulse limits before coupling each mechanism. Verify the line-mask bit order independently and test the range API with a known target and an absent echo.
4. Set each pixel to red, green, and blue in turn to verify order and indexing. Refresh the LEDs while motors, servos, and ranging run to verify timing isolation.

See the [GPIO and bus map](FNK0077-GPIO-and-Buses.md) for source paths and the Raspberry Pi implementation. The servo and IR connector markings come from the snapshot's `Picture/V2.0.jpg`.
