# Freenove FNK0077 Raspberry Pi GPIO and bus map

Use this map to identify the signals on the Freenove expansion board when designing the Propeller 2 (P2) interface. Match the board's printed revision first. The table uses Raspberry Pi BCM GPIO numbers and physical positions on the 40-pin header. Assign P2 pin numbers separately. Signal direction is relative to the controller.

## GPIO assignments by PCB revision

| Device signal | Direction | PCB V1.0 GPIO / header pin | PCB V2.0 GPIO / header pin |
| --- | --- | --- | --- |
| Left track forward, M1− | Out | GPIO24 / 18 | GPIO24 / 18 |
| Left track reverse, M1+ | Out | GPIO23 / 16 | GPIO23 / 16 |
| Right track forward, M2− | Out | GPIO5 / 29 | GPIO5 / 29 |
| Right track reverse, M2+ | Out | GPIO6 / 31 | GPIO6 / 31 |
| Servo 0 | Out | GPIO7 / 26 | GPIO12 / 32 |
| Servo 1 | Out | GPIO8 / 24 | GPIO13 / 33 |
| Servo 2, auxiliary channel | Out | GPIO25 / 22 | GPIO19 / 35 suggested by board photo, verify continuity |
| IR01 line sensor | In | GPIO16 / 36 | GPIO16 / 36 |
| IR02 center line sensor | In | GPIO20 / 38 | GPIO26 / 37 |
| IR03 line sensor | In | GPIO21 / 40 | GPIO21 / 40 |
| Ultrasonic TRIG | Out | GPIO27 / 13 | GPIO27 / 13 |
| Ultrasonic ECHO | In | GPIO22 / 15 | GPIO22 / 15 |
| RGB LED chain DIN | Out | GPIO18 / 12 | GPIO10 / 19 (SPI0 MOSI) |

The Freenove README gives the motor circuit labels. The direction shown above comes from the actual `gpiozero.Motor(forward, backward)` argument order and the positive/negative command handling in `motor.py`. For GPIO25, both V1.0 servo implementations define the auxiliary channel, and the selected gpiozero backend instantiates it. The stock startup and servo test exercise channels 0 and 1, and the kit tutorial describes two servos. Verify the GPIO25 board routing and fitted connector before relying on it.

The V2.0 board photo (`Picture/V2.0.jpg`) shows servo columns marked 19, 13, and 12. The reference Python driver controls only 12 and 13. The same photo marks the center IR terminal [20], while the V2.0 README and driver specify GPIO26. Verify both routes on the actual board before wiring a P2 adapter.

## Direct GPIO and PWM

- **Drive motors.** The four header GPIOs carry direction and speed control signals to the expansion board's motor driver circuitry. The motors attach to the expansion board. The driver uses one GPIO pair per track, scales the magnitude of signed commands by 4096, and stops the track at zero. The application supplies no separate enable, encoder, or motor PWM frequency.
- **Servos.** Each channel takes an independent control pulse. The selected V1.0 backend is gpiozero `AngularServo`; V2.0 uses hardware PWM channels 0 and 1 at 50 Hz. Both selected paths target 0.5–2.5 ms over 0–180°. The normal wrapper starts servo 0 at 90° and servo 1 at 140°, and clamps commands for those channels to 90–150°. The file also contains a pigpio implementation for V1.0 pins that the normal wrapper does not select.
- **Line tracker.** The three digital outputs go directly to separate GPIO inputs read by gpiozero `LineSensor`. The combined value puts IR01 in bit 2, IR02 in bit 1, and IR03 in bit 0. The supplied test identifies the center with IR02 and reports the left and right cases from the outer channels. Use the module orientation shown in the tutorial when installing it.
- **Ultrasonic range.** The HC-SR04 plugs into the expansion board with VCC to 5V, TRIG to TRIG, ECHO to ECHO, and GND to GND. The controller emits a trigger pulse and measures the width of the returning ECHO pulse. The Pi 5 path uses `lgpio` with a nominal 10 µs trigger; the other Pi path uses gpiozero `DistanceSensor`. The 5V supply label describes the module supply. Verify the ECHO voltage **at the expansion board's Pi header** and any onboard level conversion before connecting P2.

## RGB LEDs and SPI

**V1.0.** The four-pixel WS281x chain takes a single data signal from GPIO18. The earlier-Pi path configures `rpi_ws281x` for 800 kHz, DMA channel 10, and output channel 0. GPIO18 is the PWM-backed output for this configuration. The reference LED code selects RGB order and disables this path on Pi 5.

**V2.0.** The chain takes a single data signal from GPIO10, the MOSI output of SPI0. The driver opens `/dev/spidev0.0` in mode 0. Its normal transfer requests a 6.4 MHz SPI clock and encodes each WS281x bit as one MOSI byte, `0x80` for zero or `0xF8` for one. That produces a nominal 1.25 µs pixel bit period. The V2.0 driver sends four pixels in GRB order. A P2 implementation can generate the equivalent one-wire signal directly.

| SPI0 signal | BCM GPIO | Physical pin | Role for V2.0 LEDs |
| --- | --- | --- | --- |
| MOSI | 10 | 19 | Pixel data to DIN |
| MISO | 9 | 21 | No pixel connection indicated |
| SCLK | 11 | 23 | Pi peripheral clock used to serialize MOSI |
| CE0 | 8 | 24 | Linux SPI device selection |
| CE1 | 7 | 26 | Outside the LED transfer |

The LED chain has no clock, select, or data-return connection in the documented interface. The driver contains optional SPI bus examples, while `Led` uses SPI0 device 0. SPI0's standard CE pins overlap the two V1.0 servo pins. Keep that overlap in mind when reading the driver's bus diagnostics or configuring a Pi as a test reference.

## Pi overlays, camera, and other interfaces

- The V2.0 servo setup script uses the non-Pi-5 `pwm-2chan` overlay on GPIO12/13. Its Pi 5 overlay source additionally muxes GPIO18/19 into RP1 PWM0. The servo application uses GPIO12/13 only. Both model branches pass `chip=0` in the Python code despite the Pi 5 branch's comments referring to chip 2. These are Linux configuration details rather than additional kit connections.
- The kit camera attaches by ribbon to the Pi's dedicated CSI camera connector. `camera.py` uses Picamera2. The tutorial lists OV5670 or IMX219 modules. No 40-pin GPIOs are assigned by the camera driver. Treat camera support as a separate P2 design task.
- The server opens TCP sockets for commands and video. The protocol PDF lists ports 5003 and 8003 respectively. The application drivers show no 40-pin I²C or UART device, and no ADC or encoder connection. Peripheral capabilities listed by the Pi or by bundled driver examples do not establish wiring on the kit.

## Source trail and hardware checks

Paths below are relative to `REF-NO-COMMIT/Freenove_Tank_Robot_Kit_for_Raspberry_Pi-main/`.

| Source | What it establishes |
| --- | --- |
| `README.md` and `Tutorial.pdf` | PCB revision comparison, 40-pin numbering, kit assembly, camera ribbon and ultrasonic connector |
| `Code/Server/motor.py`, `servo.py`, `infrared.py`, `ultrasonic.py`, and `test.py` | Active pins, direction, timing, and the hardware test entry points |
| `Code/Server/led.py`, `rpi_ledpixel.py`, and `spi_ledpixel.py` | Revision-dependent LED selection, pixel count, color order, and wire encoding |
| `Code/Server/parameter.py`, `Code/setup.py`, and `Code/Libs/pi-hardware-pwm/` | PCB selection, Pi detection, and operating-system PWM setup |
| `Code/Server/camera.py`, `server.py`, `tcp_server.py`, and `Communication Protocol.pdf` | Camera API and network interfaces |

This is a static review of the reference snapshot. Before laying out a P2 adapter, confirm the actual kit revision, header orientation, common ground, and power routing. Check the board's logic levels and pull-ups, especially ECHO and the line-sensor outputs, along with the auxiliary servo routing and any LED data level shifting. The software pin assignments alone do not establish those circuit details.
