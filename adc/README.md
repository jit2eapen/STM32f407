# STM32F407 ADC – Reading a Potentiometer on PC1

Bare-metal (register-level) C program for the **STM32F407 Discovery** board.
ADC1 reads a potentiometer on **PC1** (ADC channel 11) and the result is
watched live in the STM32CubeIDE debugger. An optional LED on PC6 turns on
when the reading is above a threshold.

No HAL or CubeMX: every peripheral is set up by writing its registers directly.

## Hardware

| Signal | Pin | Notes |
|---|---|---|
| Potentiometer wiper (POT2) | PC1 | ADC123_IN11 |
| Potentiometer ends | 3V / GND | Do not exceed VDD (3.0 V on the Discovery board) |
| LED (optional) | PC6 | Active-low: wire LED + resistor from 3V to PC6 |

## What the code does

1. Enables the GPIOC clock (`RCC->AHB1ENR` bit 2).
2. Sets PC6 as output (`MODER` = 01) and PC1 as analog (`MODER` = 11).
3. Enables the ADC1 clock (`RCC->APB2ENR` bit 8).
4. Configures ADC1: one conversion (`SQR1` L = 0), channel 11 (`SQR3` SQ1 = 11), then powers it on (`CR2` ADON).
5. Loops forever: starts a conversion (`CR2` SWSTART), waits for `SR` EOC,
   reads `DR` into `result`, and drives PC6 low (LED on) when `result > 100`.

## Expected readings

The Discovery board's VREF is about 3.0 V, so:

`result = Vin / 3.0 × 4095`

| Input | `result` |
|---|---|
| 0 V | 0 |
| 1.5 V | about 2048 |
| 3.0 V | 4095 |

## Build and run

1. Import the project into STM32CubeIDE (**File → Import → Existing Projects into Workspace**).
2. Build with **Ctrl+B**.
3. Connect the board via its ST-LINK USB port and choose **Debug As → STM32 C/C++ Application**.
4. Add `result` in **Window → Show View → Live Expressions**, press **F8**, and turn the pot.

## Notes

- `main.c` must include `<stdint.h>` before `stm32f407.h`, because the header uses `uint32_t` without including it.
- Register details: ST Reference Manual **RM0090** (RCC, GPIO and ADC chapters).
