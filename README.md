# STM32 Motor Drive Monitor — Wokwi Simulation

A Wokwi simulation of a motor drive monitor running STM32 HAL firmware on the **ST Nucleo-C031C6** board. The project demonstrates ADC-controlled PWM, RC servo control via a rotary encoder, GPIO polling with debounce, timer interrupts, and UART serial communication — all implemented using the STM32 HAL library and simulated in Wokwi via a compiled ELF binary.

---

## Repository Contents

| File | Description |
|---|---|
| `diagram.json` | Wokwi circuit schematic — component placement and wiring |
| `wokwi.toml` | Wokwi project config — points to the firmware ELF |
| `plsplspls.elf` | Compiled firmware (STM32CubeIDE Debug build, arm-none-eabi-gcc) |

> The firmware source lives in the companion STM32CubeIDE project (`iluvgod/`). The ELF in this folder is the build output.

---

## How to Run

1. Install the [Wokwi for VS Code](https://marketplace.visualstudio.com/items?itemName=wokwi.wokwi-vscode) extension.
2. Clone or download this repository.
3. Open the `SIM/` folder in VS Code.
4. Press `F1` → **Wokwi: Start Simulator**.
5. The simulation loads automatically from `diagram.json` and `wokwi.toml`.

> A Wokwi account and licence are required for the VS Code extension. A free tier is available at [wokwi.com](https://wokwi.com).

---

## Hardware Overview

**MCU:** STM32C031C6 — ARM Cortex-M0+, 48 MHz (HSI), 32 KB Flash, 12 KB RAM  
**Board:** ST Nucleo-C031C6  
**Framework:** STM32 HAL (STM32Cube_FW_C0)

---

## Circuit

| Component | Model | Connection |
|---|---|---|
| Potentiometer | wokwi-potentiometer | SIG → PA0 (A0), VCC → 5V, GND → GND |
| PWM LED | wokwi-led (green) | Anode → 100Ω → PB0 (D3), Cathode → GND |
| Rotary Encoder | wokwi-ky-040 | CLK → PB10 (D4), DT → PB4 (D5), SW → PA15 (D7), VCC → 5V, GND → GND |
| Servo | wokwi-servo | PWM → PA6 (D9), V+ → 5V, GND → GND |
| Push Button | wokwi-pushbutton-6mm | Pin 1 → PA9 (D8), Pin 2 → GND |
| Onboard LED | (built-in, PA5/D13) | Heartbeat — no external wiring needed |

---

## Pin Mapping

| Arduino Header | STM32 Pad | Peripheral | Direction |
|---|---|---|---|
| A0 | PA0 | ADC1_IN0 — Potentiometer | Analog in |
| D3 | PB0 | TIM3_CH3 AF1 — LED PWM | Output (AF) |
| D4 | PB10 | Encoder CLK (polled) | Digital in (pull-up) |
| D5 | PB4 | Encoder DT (polled) | Digital in (pull-up) |
| D7 | PA15 | Encoder SW (polled) | Digital in (pull-up) |
| D8 | PA9 | Push button (polled) | Digital in (pull-up) |
| D9 | PA6 | TIM3_CH1 AF1 — Servo PWM | Output (AF) |
| D13 | PA5 | Heartbeat LED | Digital out |
| — | PA2 | USART2_TX | Output (AF) |
| — | PA3 | USART2_RX | Input (AF) |

---

## Features and Behaviour

### System States

The system has two states toggled by the push button (PA9) or serial commands:

```
             Button press  /  'R' command
   STOP  ──────────────────────────────►  RUN
         ◄──────────────────────────────
             Button press  /  'S' command
```

| Behaviour | STOP | RUN |
|---|---|---|
| Heartbeat LED (PA5) | Blinks at 2 Hz | Blinks at 2 Hz |
| PWM LED (PB0) | Off | Brightness follows pot |
| Servo (PA6) | Follows encoder | Follows encoder |
| Serial status | Printed every 500 ms | Printed every 500 ms |

### Inputs

- **Potentiometer (PA0):** ADC sampled continuously every main loop iteration. In RUN state, maps 0–4095 → 0–100% LED brightness.
- **Rotary Encoder CLK (PB10):** Polled with 5 ms debounce. Each falling edge steps the servo position ±1°. Direction determined by DT (PB4).
- **Encoder SW (PA15):** Polled with 50 ms debounce. Press resets servo to 90° (centre).
- **Push Button (PA9):** Polled with 50 ms debounce. Toggles RUN/STOP state.

### Serial Commands (115200 baud)

Send single characters via the Wokwi serial monitor:

| Command | Action |
|---|---|
| `R` | Force RUN state |
| `S` | Force STOP state |
| `C` | Centre servo to 90° |

### Serial Output

Status line printed every 500 ms:
```
ADC:2048 | Pot:50% | LED:50% | State:RUN  | Servo:90deg
ADC:2048 | Pot:50% | LED:0%  | State:STOP | Servo:90deg
```

Event messages on input changes:
```
BTN: RUN        BTN: STOP
ENC: 95 deg     ENC SW: centre
CMD: RUN        CMD: STOP       CMD: CENTRE
```

---

## Technical Details

### Clock

HSI oscillator at 48 MHz, no PLL (STM32C031 has no PLL block).  
All peripherals run at PCLK1 = 48 MHz.

### Timer Arithmetic

**TIM3 — Servo (CH1/PA6) and LED (CH3/PB0)**

```
Tick:  48 MHz / (PSC+1) = 48 MHz / 48 = 1 MHz  →  1 µs per tick
PWM:   1 MHz / (ARR+1)  = 1 MHz / 20000 = 50 Hz  →  20 ms period

PSC = 47,  ARR = 19999
```

Servo CCR (RC servo: 1000–2000 µs = 0°–180°):
```
CCR(  0°) = 1000   CCR( 90°) = 1500   CCR(180°) = 2000
Formula:  CCR = 1000 + angle × 1000 / 180
```

LED CCR (full brightness range):
```
CCR = adcValue × 19999 / 4095
CCR = 0  →  0% duty (off)     CCR = 19999  →  100% duty (full brightness)
```

**TIM16 — Heartbeat (2 Hz)**
```
Tick:  48 MHz / (PSC+1) = 48 MHz / 48000 = 1 kHz  →  1 ms per tick
IRQ:   1 kHz / (ARR+1)  = 1 kHz / 500 = 2 Hz

PSC = 47999,  ARR = 499
```

TIM16 interrupt is used **only for the heartbeat LED toggle** (PA5). The ADC is sampled continuously in the main loop for immediate pot response, not gated behind the TIM16 flag.

### ADC

- 12-bit resolution, software trigger, single conversion mode
- Clock: PCLK1/4 = 12 MHz
- Sampling time: 39.5 cycles
- Self-calibration run once at startup via `HAL_ADCEx_Calibration_Start()`
- Sampled every main loop iteration (~4 µs per conversion)

### printf() → UART Redirect

`printf()` is redirected to USART2 by implementing the newlib-nano retarget hook:

```c
int __io_putchar(int ch)
{
    HAL_UART_Transmit(&huart2, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}
```

Chain: `printf` → `_write` → `__io_putchar` → `HAL_UART_Transmit` → PA2 (TX)

### EXTI vs Polling — Wokwi Limitation

The firmware was originally designed with EXTI interrupts for button and encoder inputs. Wokwi's STM32 simulator does not emulate `EXTI->EXTICR[]` port selection registers, so EXTI interrupts never fire. All digital inputs are handled by `HAL_GPIO_ReadPin()` polling in the main loop, which works correctly in simulation.

The HAL EXTI flow for reference:
```
GPIO_MODE_IT_FALLING  →  HAL sets FTSR1 + EXTICR[]
EXTI4_15_IRQHandler   →  HAL_GPIO_EXTI_IRQHandler()  →  clears flag
                      →  HAL_GPIO_EXTI_Callback()     →  user handler
```

---

## Building the Firmware

The source project is in `iluvgod/` (STM32CubeIDE).

1. Open STM32CubeIDE and import the `iluvgod` project.
2. Build → Debug configuration.
3. Output ELF: `iluvgod/Debug/plsplspls.elf`
4. `wokwi.toml` already points to this path: `firmware = "../iluvgod/Debug/plsplspls.elf"`

**Toolchain:** `arm-none-eabi-gcc` (bundled with STM32CubeIDE)  
**Flags:** `-mcpu=cortex-m0plus -mthumb -mfloat-abi=soft --specs=nano.specs`

---

## Known Simulation Constraints

| Issue | Detail |
|---|---|
| EXTI not working | `EXTI->EXTICR[]` not emulated — replaced with polling |
| TIM1 complementary outputs | Not reliably simulated — TIM3 regular channels used instead |
| STM32C031 register differences | No `SYSCFG->EXTICR`, split `FPR1`/`RPR1` pending registers instead of classic `PR1` |

---

## License

This project is for educational and demonstration purposes.  
STM32 HAL drivers are © STMicroelectronics, licensed under BSD-3-Clause.
