# LED_BLINK — STM32F407 Bare-Metal LED Blink

A minimal, register-level (bare-metal) firmware that blinks the on-board green LED
(**PD12**) on the **STM32F407G-DISC1 (Discovery)** board. No HAL, no CMSIS device
headers — just direct memory-mapped register access.

---

## Hardware

| Item | Detail |
|------|--------|
| Board | STM32F407G-DISC1 (Discovery) |
| MCU | STM32F407VGTx (ARM Cortex-M4) |
| LED | Green LED, connected to **PD12** |
| Debugger | On-board ST-LINK (SWD) |

> The Discovery board has four user LEDs on GPIOD: PD12 (green), PD13 (orange),
> PD14 (red), PD15 (blue).

---

## How It Works

The program talks directly to three hardware registers using their physical
addresses:

| Register | Address | Purpose |
|----------|---------|---------|
| `RCC_AHB1ENR` | `0x40023830` | Enables the clock for GPIO Port D |
| `GPIOD_MODER` | `0x40020C00` | Sets pin direction (input/output/etc.) |
| `GPIOD_ODR`   | `0x40020C14` | Output data — drives the pin high/low |

Sequence:

1. **Enable the GPIOD clock** — peripherals are clock-gated off at reset to save
   power, so nothing works until the clock is turned on (`RCC_AHB1ENR` bit 3).
2. **Configure PD12 as an output** — `MODER` uses 2 bits per pin; writing `01`
   into bits 24-25 (`12 * 2`) selects general-purpose output mode.
3. **Blink forever** — in an infinite loop, set the ODR bit (LED on), wait, clear
   the ODR bit (LED off), wait.

### Bit-manipulation idioms used

| Operation | Meaning |
|-----------|---------|
| `reg \|= (1 << n)` | **set** bit *n* (leave others untouched) |
| `reg &= ~(1 << n)` | **clear** bit *n* |
| `reg &= ~(3 << s)` | clear a 2-bit field starting at bit *s* |

---

## Source

`Src/main.cpp`:

```cpp
#include <stdint.h>

volatile uint32_t* rcc_ahb1enr = (volatile uint32_t*)0x40023830;
volatile uint32_t* gpiodmoder  = (volatile uint32_t*)0x40020C00;
volatile uint32_t* gpiod_odr   = (volatile uint32_t*)0x40020C14;

int main(void)
{ // PD12 LED

    *rcc_ahb1enr |= (1 << 3);           // PORTD clock enable

    *gpiodmoder &= ~(3 << (12 * 2));    // clear PD12 mode bits
    *gpiodmoder |= (1 << (12 * 2));     // PD12 output mode

    while (1)
    {
        *gpiod_odr |= (1 << 12);        // LED on
        for (volatile int i = 0; i < 10000000; i++);

        *gpiod_odr &= ~(1 << 12);       // LED off
        for (volatile int i = 0; i < 10000000; i++);
    }
}
```

> `volatile` is essential. It stops the compiler from optimizing away the register
> writes and the delay loop.

---

## Build & Flash (STM32CubeIDE)

1. Open the `LED_BLINK` project in **STM32CubeIDE**.
2. **Project → Build Project** (or the hammer icon). A successful build ends with
   `Finished building: default.size.stdout` and produces `Debug/LED_BLINK.elf`.
3. Connect the board via the **ST-LINK USB port** (mini/micro-USB on the short
   edge, *not* the "USB USER" port).
4. **Run → Debug** (or **Run**). Confirm the debugger is
   *STM32Cube: STM32 Launch STLink GDB Server* / *ST-LINK (SWD)*.
5. On first flash, Windows Firewall may ask to allow `st-link_gdbserver` —
   click **Allow**.
6. Look for `Download verified successfully` in the console. The green LED (PD12)
   should now blink.

### Toolchain (command line)

Compiled with the Arm GNU toolchain, e.g.:

```
arm-none-eabi-g++ ../Src/main.cpp -mcpu=cortex-m4 -std=gnu++14 \
  -mfpu=fpv4-sp-d16 -mfloat-abi=hard -mthumb -O0 -g3 \
  --specs=nano.specs -c -o Src/main.o
```

---

## Blink Speed

The delay is a busy-wait loop, so the timing is approximate and depends on the CPU
clock. Rough feel:

| Loop count | Approximate blink |
|------------|-------------------|
| `100000`   | Fast (several Hz) |
| `1000000`  | ~1 Hz-ish |
| `10000000` | Slow (a few seconds per state) |

For a **precise** millisecond delay, use SysTick or a hardware timer instead of a
counting loop.

---

## Notes / Gotchas

- This targets the **F407 Discovery** (LED on PD12). On a **Nucleo** board the user
  LED is **PA5** instead — the port, clock bit, and pin number all change.
- No CMSIS device header (`stm32f4xx.h`) is required because all registers are
  accessed by raw address. If you prefer the `RCC->`, `GPIOD->` style, add ST's
  CMSIS headers and define `STM32F407xx`.

---

## Ideas for Next Steps

- Replace set/clear with a single **toggle**: `*gpiod_odr ^= (1 << 12);`
- Blink all four LEDs (PD12-PD15) in a chase pattern.
- Add a **SysTick**-based `delay_ms()` for accurate timing.
- Print over **USART** to a USB-TTL adapter.
