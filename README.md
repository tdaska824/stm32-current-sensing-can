# Current Sensing Firmware (STM32F103)

Firmware for a custom STM32F103 board that measures current on three channels using shunt resistors and amplifiers, and sends the readings over CAN bus. The ADC samples the three amplifier outputs, the firmware converts each reading to amperes, and one CAN frame with all three values is sent every 10 ms.

Only the application code is published here (`Core/Src/main.c`). The rest of the project (CubeMX configuration, PCB design files) is not published, but it can be shared on request.

## Features

- 3 current channels measured with ADC1 in scan mode with DMA
- Conversion of the ADC reading to amperes using the shunt resistance and amplifier gain
- CAN 2.0A frames (standard ID `0x322`) at 500 kbit/s
- Non-blocking transmit loop based on the system tick

## Measurement

Each channel is read as a 12 bit value. The firmware removes the amplifier offset, converts the result to volts, and divides by the gain and the shunt resistance:

```
v_out = |adc - OFFSET| * VREF / 4095
I     = v_out / (R_SHUNT * GAIN)
```

| Constant | Value |
|----------|-------|
| `VREF` | 3.3 V (ADC reference) |
| `OFFSET` | 1900 counts (about 1.53 V at zero current) |
| `R_SHUNT` | 0.001 Ω (1 mΩ) |
| `GAIN` | 20 |
| `ADC_RESOLUTION` | 4095 (12 bit) |

With these constants one ADC count corresponds to about 0.04 A. The value is the magnitude of the current, so the direction is not reported.

## CAN frame

| Field | Value |
|-------|-------|
| ID | `0x322` (standard, 11 bit) |
| DLC | 6 |
| Period | 10 ms |

| Byte | Content |
|------|---------|
| 0-1 | Current 1 (PA3 / ADC channel 3), unsigned 16 bit, big-endian |
| 2-3 | Current 2 (PA5 / ADC channel 5) |
| 4-5 | Current 3 (PA7 / ADC channel 7) |

Each value is the current in units of 0.1 A. For example, 12.5 A is sent as `125` (`0x007D`), which appears in the frame as the bytes `00 7D`.

## Peripheral configuration

The code expects a CubeMX project with the following settings:

| Peripheral | Configuration |
|------------|---------------|
| Clock | 8 MHz crystal (HSE), PLL x9, 72 MHz system clock |
| ADC1 | Scan mode, 3 conversions: channel 3 (PA3), channel 5 (PA5), channel 7 (PA7), 71.5 cycles sampling time, ADC clock 12 MHz (PCLK2 / 6), software trigger |
| DMA | DMA1 channel 1 for ADC1, half-word transfers |
| CAN1 | 500 kbit/s: prescaler 18, 1 + 2 + 1 time quanta per bit, 36 MHz APB1 clock; accept-all receive filter |
| GPIO | One output pin labelled `LED` |

## Using the code

1. Create a CubeMX project for the STM32F103 with the configuration above (or open your own `.ioc` for the board).
2. Generate the code and replace the generated `Core/Src/main.c` with the one from this repository, or copy the code between the `USER CODE` markers.

## Author

Theopoula Eirini Daskalou, Computer Science, Aristotle University of Thessaloniki
[LinkedIn](https://www.linkedin.com/in/thei-daskalou-3567a8358) | [GitHub](https://github.com/tdaska824)
