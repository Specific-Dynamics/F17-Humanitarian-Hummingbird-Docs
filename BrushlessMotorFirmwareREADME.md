*<div align="right">Khai Huynh, Vikram Procter | Jan 13, 2026 </div>*

# BLDC Motor ESC Firmware
### *Specific Dynamics - F17 Humanitarian Hummingbird*

## Overview
Electronic Speed Controller (ESC) for four brushless 3-phase motors built around an STM32F405RGT6 (168MHz, 16MHz HSE). Implements 6-step sensorless BEMF zero-crossing detection with trapezoidal commutation across all four motors simultaneously. Motors are started in a staggered sequence to reduce inrush current and ADC load.

## General Software Architecture

### State Machine
`IDLE → ALIGN → OPENLOOP_RUN → MOTOR_RUN → FAULT`

**IDLE:** Motor inactive. No PWM output. Waiting for a `MOTOR_Start()` call, which is itself gated on receiving a valid PWM throttle signal above the dead zone (>4.8% duty / >960µs pulse).

**ALIGN:** Phase A is energized at `MOTOR_ALIGN_DUTY = 150` for `MOTOR_ALIGN_TIME_US = 100ms` to magnetically align the rotor to a known position before open-loop ramp.

**OPENLOOP_RUN:** Fixed-timing open-loop commutation. The step delay ramps geometrically from `MOTOR_OPENLOOP_START_DELAY_US = 5000µs` down to `MOTOR_OPENLOOP_END_DELAY_US = 240µs` using a factor of `997/1000` per step. Duty simultaneously ramps from 200 to 170 CCR counts. BEMF sampling begins once the step delay reaches the end value. Transition to `MOTOR_RUN` requires: (1) step delay at target, (2) `openloop_hold_us` elapsed (default 400ms, runtime-tunable), and (3) BEMF synced (`zc_sync_count` = 12 consecutive valid zero crossings). A timeout of `MOTOR_OPENLOOP_TIMEOUT_US = 4000ms` triggers `Motor_Recovery()` if these conditions are not met.

**MOTOR_RUN:** BEMF-driven closed-loop commutation. Each zero crossing detected by the ADC ISR calls `MOTOR_ZeroCrossDetected()`, which advances the commutation step immediately and updates `commutation_delay_us` from the measured ZC period. A fallback timer in the main loop commutates the motor if no BEMF event occurs within `commutation_delay_us × (1 + fallback_margin_pct/100)` - the margin widens to 50% for the first 200 BEMF commutations during the closed-loop transition. Duty is controlled by ramping toward the PWM-input target at 1 CCR count per 500µs.

**FAULT:** All PWM outputs disabled. Only an MCU reset recovers. The `global_fault` flag in the main loop immediately asserts all four `nSLEEP` pins low (killing all gate drivers) and enters a fast LED blink loop.

**Recovery:** Before permanently entering `FAULT`, a failing motor attempts `max_restarts` (default 5) automatic restarts via `Motor_Recovery()`, which resets BEMF sync and re-enters `MOTOR_ALIGN`. Each per-motor restart is tracked independently in `restart_per_motor[]`. A restart during open-loop only faults that motor; a stall detected during `MOTOR_RUN` calls `MOTOR_EmergencyShutdown()` and kills all four motors.

### Stall Detection
Stall is detected exclusively by counting fallback commutations. After a grace period of `STALL_GRACE_COMMS = 5000` BEMF-driven commutations, if `fallback_per_motor[i]` exceeds `max_fallback_comms` (default 300) within a rolling 1000-commutation window, a stall is declared. The fallback counter resets every 1000 BEMF commutations, making stall detection rate-based rather than accumulative. `fault_motor` and `fault_cause` (1 = fallback) are set for post-mortem debug inspection.

### Interrupt Structure

1. **ADC Conversion Complete ISR — `ADC_IRQHandler` (Medium Priority)**
    - Fires at 52.5kHz triggered by TIM1_TRGO (update event).
    - All three ADCs fire simultaneously from the same trigger.
    - ADC1 JEOC: reads Motor 1 phases (JDR1–3) and Motor 4 phase A (JDR4); sets `adc_new[0]`.
    - ADC3 JEOC: reads Motor 3 phases (JDR1–3) and Motor 4 phase C (JDR4); sets `adc_new[2]`.
    - ADC2 JEOC: reads Motor 2 phases (JDR1–3) and Motor 4 phase B (JDR4); sets `adc_new[1]` and, if Motor 4 is started, `adc_new[3]` (Motor 4 data is only complete after all three ADCs fire).
    - Raw samples are written to `adc_buf[motor][phase]`. BEMF processing is deferred to the main loop via the `adc_new[]` flags to keep the ISR short.

2. **EXTI15_10_IRQHandler — PWM Throttle Input (Priority 3)**
    - Shared ISR for all four PWM receive lines (EXTI10–12, EXTI15).
    - On rising edge: latches `DWT->CYCCNT` timestamp.
    - On falling edge: computes pulse width in µs via `elapsed_cycles / 168`.
    - Valid frames: 500–2500µs. Timeout (no valid frame for 100ms) forces duty to 0 and clears `pwm_rx_valid`.
    - Duty mapping: below 960µs → 0 (motor stop); 960–1100µs → minimum duty 80; 1100–2000µs → linear map 80–780; above 2000µs → capped at 780.

### Main Loop
1. Check `global_fault` — if set, kill all gate drivers and spin in LED blink.
2. Check PWM kill condition — if any running motor receives a valid signal at 0 duty (signal present but below 4.8%), call `MOTOR_EmergencyShutdown()`.
3. Process buffered ADC samples: for each motor, if `adc_new[i]` is set, call `BEMF_Process(i, va, vb, vc)`.
4. Staggered motor startup:
   - **Motor 1**: starts when `pwm_rx_valid[0]` and `pwm_rx_duty[0] > 0`.
   - **Motor 3**: starts 500ms after Motor 1 enters `MOTOR_RUN` and has a valid PWM signal.
   - **Motor 2**: starts 500ms after Motor 3 enters `MOTOR_RUN` and has a valid PWM signal. `Motor2_HW_Init()` deferred-inits TIM1 PWM channels and ADC2 at this point.
   - **Motor 4**: starts 500ms after Motor 2 enters `MOTOR_RUN` and has a valid PWM signal. Uses ADC1/2/3 rank-4 channels (initialized with their respective motors).
5. Call `MOTOR_State_Update(i)` for each started motor — runs the state machine and fallback commutation timer.
6. Ramp each running motor's duty toward the PWM input target (or `motor_run_duty` if no signal, for bench use) at 1 CCR count per 500µs.
7. Compute rolling BEMF-vs-fallback commutation percentage every 1 second into `bemf_percent`, `bemf_recent`, and `fallback_recent` for Live Expressions monitoring.

### Startup Sequence
On power-on: PB10 LED pulses, a 1-second delay allows the DRV8329 to power up, then `nSLEEP_1` is asserted. Motor 1 plays a startup melody (Rick Astley — *Never Gonna Give You Up*) via open-loop commutation before entering the throttle-waiting state. Motors 2–4 gate drivers are enabled lazily during staggered startup.

### Hardware Timers

| Timer | APB | Timer Clock | Mode | ARR | Effective Freq | Role |
|-------|-----|------------|------|-----|----------------|------|
| TIM8  | APB2 | 168MHz | Center-aligned 1 | 800 | ~105kHz | Motor 1 phase B/C (CH1, CH2) |
| TIM12 | APB1 | 84MHz  | Up | 799 | ~105kHz | Motor 1 phase A (CH2) |
| TIM1  | APB2 | 168MHz | Center-aligned 1 | 800 | 52.5kHz | ADC TRGO trigger + Motor 2 CH2/3/4 |
| TIM4  | APB1 | 84MHz  | Up | 799 | ~105kHz | Motor 3 CH2/3/4 (deferred init) |
| TIM2  | APB1 | 84MHz  | — | — | ~105kHz | Motor 4 CH3/4/1 (deferred init) |

TIM1 is the master ADC trigger. Its update event fires the `TIM1_TRGO` signal that simultaneously triggers all three ADCs in injected mode at 52.5kHz. TIM1 also carries Motor 2's PWM channels (CH2, CH3, CH4) which are added to the already-running timer when Motor 2's deferred hardware init fires. All duty values are 0–800 CCR counts across all motors.

### ADC / BEMF Sampling

All three ADCs are used in injected mode, triggered simultaneously by TIM1_TRGO. Each ADC has four injected ranks. Motor 4's three phases are distributed one per ADC on rank 4, so Motor 4 data is only marked valid once all three ADCs have fired (gated on ADC2 JEOC).

| ADC  | Rank 1–3 | Rank 4 |
|------|----------|--------|
| ADC1 | Motor 1 (phA, phB, phC) | Motor 4 phA |
| ADC2 | Motor 2 (phA, phB, phC) | Motor 4 phB |
| ADC3 | Motor 3 (phA, phB, phC) | Motor 4 phC |

ADC clock: PCLK2/4 = 21MHz. ADC2 and ADC3 are initialized lazily when Motors 2 and 3 start respectively.

---

## Source File Reference

### `motor.c` / `motor.h`
Core state machine and commutation logic for all four motors.

- `MOTOR_COUNT = 4` defined in `motor.h`.
- `MOTOR_Init()`: zeros all per-motor state and counters.
- `MOTOR_Start(id)`: enters `MOTOR_ALIGN`, resets restart and fallback counters.
- `MOTOR_State_Update(id)`: called every main-loop iteration; runs the state machine and fallback commutation watchdog.
- `MOTOR_ZeroCrossDetected(id)`: called from `BEMF_Process()` on a valid ZC; advances commutation step, sets `bemf_commutated` flag, updates `commutation_delay_us` from ZC period.
- `MOTOR_Commutate(id)`: advances step, calls `PWM_DRIVE_SetCommutationStep()`, updates `last_commutation_us`, notifies BEMF blanking.
- `MOTOR_SetDuty(id, duty)`: clamps to 800 max; only takes effect in `MOTOR_RUN`.
- `MOTOR_EmergencyShutdown()`: disables all motors, sets `global_fault = 1`. Requires MCU reset to recover.
- `Motor_Recovery(id)` (static): per-motor restart attempt up to `max_restarts`; re-enters `MOTOR_ALIGN`. Does not trigger global shutdown.

Key tunable variables (visible in Live Expressions / debugger):
- `motor_run_duty` (default 100): bench-test duty when no PWM signal is present.
- `openloop_hold_us` (default 400ms): minimum time in open loop before closed-loop transition is allowed.
- `stall_timeout_us` (default 200ms): legacy field, currently unused in stall logic.
- `max_restarts` (default 5): per-motor open-loop restart attempts before permanent fault.
- `max_fallback_comms` (default 300): fallback commutations per 1000 BEMF comms before stall declared.
- `fallback_margin_pct` (default 20): percentage of `commutation_delay_us` added as fallback timeout margin.
- `bemf_comms_per_motor[4]`, `fallback_per_motor[4]`: per-motor counters for debug.
- `fault_motor`, `fault_cause`: last-fault diagnostics (cause: 1=fallback).

### `bemf.c` / `bemf.h`
Zero-crossing detection and BEMF synchronization.

- `BEMF_Process(id, va, vb, vc)`: called in the main loop with raw ADC values. Computes virtual neutral `(va+vb+vc)/3`, identifies the floating phase and expected ZC direction from the current step via lookup tables (`floating_phase[]`, `zc_rising[]`), and detects sign changes against a blanking window. Valid ZC periods must be 30–2000µs.
- **Adaptive IIR filter**: in `MOTOR_OPENLOOP_RUN`, uses a 7/8 heavy filter for stable sync detection; in `MOTOR_RUN`, switches to a 3/4 lighter filter for responsive speed tracking.
- **Blanking**: `BEMF_CommutationNotify()` sets a blanking window of `blanking_us` (default 10µs, tunable) after each commutation to suppress PWM switching noise. Previous-sample validity is also cleared on commutation.
- **Sync**: requires `zc_sync_count` (default 12, tunable) consecutive valid ZC events. `BEMF_IsSynced()` must return true before the OL→CL transition.
- **Amplitude guard (MOTOR_RUN only)**: if all three phase voltages fall below `BEMF_STALL_AMPLITUDE_THRESHOLD = 100` ADC counts for 200 consecutive samples, `bemf_lost` is flagged. This is informational; actual shutdown is driven by `fallback_per_motor` in `motor.c`.
- `bemf_debug_motor` (default 255 = all): set to a specific motor index to filter the shared `bemf_debug` struct to that motor's data.

Key tunable variables:
- `blanking_us` (default 10): post-commutation BEMF blanking window in µs.
- `com_time` (default 0): fractional commutation advance (`zc_period * com_time / 10`). Set to 0 for direct ZC commutation (30° advance is inherent in the ZC event timing).
- `zc_sync_count` (default 12): consecutive ZC events needed to declare sync.
- `bemf_debug_motor`: motor index to isolate debug output; 255 = all motors.

### `pwm_drive.c` / `pwm_drive.h`
Trapezoidal commutation output — high-side PWM via timer CCR, low-side via GPIO.

- Implements DRV8329 **3-PWM mode**: high-side phase uses `INHx = PWM`, `INLx = 1`; low-side phase uses `INHx = 0`, `INLx = 1`; floating phase has both INHx and INLx = 0 (Hi-Z).
- `PWM_DRIVE_SetCommutationStep(id, step, duty)`: turns off all six outputs first (zero CCR + GPIO reset), inserts software dead time via `DWT->CYCCNT` spin (`deadtime_switching` cycles, default 0), then sets the active high and low phases.
- **Maximum duty value is 800 CCR counts** (= ARR), representing true 100% on-time. In practice, `MOTOR_SetDuty()` hard-clamps to 800 and the PWM receive mapping caps full throttle at 780 (97.5%), preserving headroom for the DRV8329 bootstrap capacitor to recharge on the high-side gate driver.
- **Shoot-through prevention**: zeroing all outputs before the next step provides >500ns of dead time in addition to any DRV8329 hardware dead time. The `deadtime_switching` variable (default 0, measured in DWT cycles at 168MHz ≈ 6ns/cycle) can be tuned at runtime.
- Six-step commutation table (high → low): `A→B, A→C, B→C, B→A, C→A, C→B`.
- `PWM_DRIVE_DisableMotor(id)`: zeros all CCR and GPIO outputs immediately.

Pin/timer mapping (all four motors defined; Motors 3 and 4 require deferred `Motor3_HW_Init()` / `Motor4_HW_Init()`):

| Motor | Phase A High | Phase B High | Phase C High |
|-------|-------------|-------------|-------------|
| 1 | PB15 / TIM12_CH2 | PC6 / TIM8_CH1 | PC7 / TIM8_CH2 |
| 2 | PA9 / TIM1_CH2 | PA10 / TIM1_CH3 | PA11 / TIM1_CH4 |
| 3 | PB7 / TIM4_CH2 | PB8 / TIM4_CH3 | PB9 / TIM4_CH4 |
| 4 | PA2 / TIM2_CH3 | PA3 / TIM2_CH4 | PA5 / TIM2_CH1 |

Low-side signals for all motors are separate GPIO outputs (`INLx_N_GPIO_Port` / `INLx_N_Pin`).

### `pwm_receive.c` / `pwm_receive.h`
Standard 50Hz RC servo PWM input decoding for all four throttle channels.

- Uses EXTI on four GPIO pins (PC10–PC12, PA15), shared `EXTI15_10_IRQn` at priority 3.
- Pulse width is measured using `DWT->CYCCNT` timestamps (divide by 168 for µs at 168MHz). This avoids occupying a timer.
- Duty mapping (CCR 0–800): dead zone <960µs → 0; min spin 960–1100µs → 80; linear 1100–2000µs → 80–780; >2000µs → 780.
- Signal loss timeout: if no valid frame is received for 100ms, `pwm_rx_valid[i]` is cleared and `pwm_rx_duty[i]` is forced to 0.
- Exported arrays `pwm_rx_duty[]`, `pwm_rx_pulse_us[]`, `pwm_rx_valid[]` are readable in Live Expressions.

### `melody.c` / `melody.h`
Startup tone sequence played through Motor 1 before the throttle-waiting loop.

- Drives Motor 1's commutation steps in a blocking loop at note frequencies derived from step-delay lookup.
- Plays the intro riff and first line of *Never Gonna Give You Up* (Rick Astley, 1987).
- `MELODY_Play(motor_id)`: iterates the note table, calling `PWM_DRIVE_SetCommutationStep()` at the appropriate step-delay with `MELODY_DUTY = 150`. Rests call `PWM_DRIVE_DisableMotor()` + `HAL_Delay()`. This is a **blocking** function and must only be called before the main control loop starts.

### `private_utils.c` / `private_utils.h`
Microsecond timing using the DWT cycle counter.

- `DWT_Init()`: enables DWT via `CoreDebug` and resets `CYCCNT`.
- `UTILS_GetUsTick()`: returns `DWT->CYCCNT / (SystemCoreClock / 1000000)`. Used for all non-blocking timing throughout the firmware. Resolution is 1µs; rollover at ~4295 seconds.
- `UTILS_DelayUs(us)`: busy-wait delay via DWT.

---

## Tuning Notes

- **PWM kill threshold**: adjustable by changing `PWM_DEAD_ZONE_US` (960µs) and `PWM_MIN_SPIN_US` (1100µs) in `pwm_receive.c`.
- **Open-loop ramp rate**: `MOTOR_OPENLOOP_RAMP_NUM/DEN` (997/1000) controls geometric step-delay reduction. Reduce numerator (e.g. 995/1000) for faster ramp; this may destabilize BEMF sync at the end of the ramp window.
- **Closed-loop transition stability**: if Motor 4 or another motor struggles to sync, reduce `zc_sync_count` from 12 to 6 via the debugger.
- **Fallback margin**: `fallback_margin_pct` (default 20%) adds a cushion to the commutation timeout. Increase for slow-ramping or high-inertia loads; decrease if fallback commutations are frequent at steady-state speed.
- **`motor_enable[]`**: set any index to 0 to disable a specific motor without modifying startup logic. Useful for single-motor bench testing.
- **`deadtime_switching`**: additional software dead time in DWT cycles (168 cycles ≈ 1µs). Default 0 relies on the all-phases-off blank pulse inherent in `PWM_DRIVE_SetCommutationStep()`.


---
 
## Glossary
 
| Term | Definition |
|------|------------|
| **ADC** | Analog-to-Digital Converter. Converts an analog voltage (e.g. a BEMF phase voltage) into a digital number the MCU can process. |
| **APB** | Advanced Peripheral Bus. The internal bus on STM32 devices that clocks peripherals such as timers and ADCs. APB1 and APB2 run at different frequencies derived from the main system clock. |
| **ARR** | Auto-Reload Register. Sets the period (top count value) of a hardware timer. Together with the prescaler and clock source, it determines the timer overflow frequency and therefore the PWM frequency. |
| **BEMF** | Back Electromotive Force. The voltage generated by a spinning motor's own magnetic field in the undriven (floating) phase. Used here to detect rotor position without hall-effect sensors. |
| **BLDC** | Brushless DC (motor). A 3-phase synchronous motor that uses electronic commutation rather than physical brushes to switch current through the windings. |
| **CCR** | Capture/Compare Register. The timer register that sets the PWM pulse width. A CCR value of 400 on a timer with ARR = 800 produces a 50% duty cycle. |
| **CH** | Channel. A numbered output of a hardware timer (e.g. TIM1_CH2). Each channel can be configured for PWM output on a specific GPIO pin. |
| **CL** | Closed Loop. Operating mode in which the motor's commutation timing is driven by real-time BEMF feedback, as opposed to open-loop fixed-time stepping. |
| **COTS** | Commercial Off-The-Shelf. Refers to components purchased as finished products rather than designed in-house. |
| **DRV8329** | Texas Instruments gate driver IC used on this ESC. Drives the high- and low-side MOSFETs for one 3-phase motor in 3-PWM mode. One DRV8329 per motor. |
| **DWT** | Data Watchpoint and Trace unit. A Cortex-M4 debug peripheral containing the `CYCCNT` free-running cycle counter, used here as a high-resolution (6ns at 168MHz) µs clock without consuming a hardware timer. |
| **ESC** | Electronic Speed Controller. The circuit and firmware that converts a throttle command into the timed phase switching required to spin a brushless motor at a desired speed. |
| **EXTI** | External Interrupt. An STM32 peripheral that generates an interrupt on a GPIO edge (rising, falling, or both). Used here to time incoming RC PWM throttle pulses on four input pins. |
| **FET / MOSFET** | Field-Effect Transistor / Metal-Oxide-Semiconductor FET. The high-current switches that connect motor phases to supply or ground. Six FETs (three half-bridges) per motor. |
| **GPIO** | General-Purpose Input/Output. A configurable digital pin on the MCU, used here for low-side gate drive signals (INLx), nSLEEP control, and the status LED. |
| **GPIO_PIN_RESET / SET** | HAL constants for driving a GPIO low (RESET) or high (SET). |
| **HAL** | Hardware Abstraction Layer. ST's firmware library (STM32 HAL) that wraps register-level peripheral access into portable C functions. |
| **Hi-Z** | High Impedance. A state in which a pin or phase is disconnected from both supply and ground, allowing it to float. The floating phase is held Hi-Z during each commutation step so its BEMF voltage can be measured. |
| **HSE** | High-Speed External (oscillator). The 16MHz crystal on this board that feeds the PLL to produce the 168MHz system clock. |
| **IIR** | Infinite Impulse Response. A type of digital filter that weights new samples against a running average. Used here to smooth the measured BEMF zero-crossing period: `filtered = (filtered * N + new) / (N+1)`. |
| **INHx / INLx** | DRV8329 input pins for the high-side (INH) and low-side (INL) gate drivers of each phase. In 3-PWM mode, INHx carries the PWM signal and INLx is a static GPIO enable. |
| **IOC** | STM32CubeMX project file (`.ioc`). Contains the complete pin, clock, and peripheral configuration for the STM32CubeMX code-generation tool. |
| **ISR** | Interrupt Service Routine. A function that executes in response to a hardware event (ADC conversion complete, GPIO edge, etc.), pre-empting the main loop. |
| **JEOC** | Injected End-Of-Conversion. The ADC status flag (and optional interrupt) that fires when all ranks of an injected conversion group have completed. |
| **JDR** | Injected Data Register. The ADC result register for injected conversions. JDR1–JDR4 hold the results for injected ranks 1–4 respectively. |
| **MCU** | Microcontroller Unit. Here, the STM32F405RGT6 Arm Cortex-M4 running at 168MHz. |
| **nSLEEP** | Active-low sleep control pin on the DRV8329. Must be held high for normal operation; pulling it low disables the gate driver and places it in a low-power sleep state. One pin per motor driver. |
| **NVIC** | Nested Vectored Interrupt Controller. The Cortex-M4 hardware that manages interrupt priorities and preemption. |
| **OL** | Open Loop. Operating mode in which commutation steps are fired at fixed time intervals determined by a ramp schedule rather than rotor position feedback. Used during startup before BEMF sync is established. |
| **PCLK2** | Peripheral Clock 2. The APB2 bus clock, running at 84MHz on this board. The ADC clock is derived from PCLK2 (PCLK2/4 = 21MHz here). |
| **PDOM** | Parser DOM (index file used by Eclipse CDT for code indexing). Not a firmware concept. |
| **PLL** | Phase-Locked Loop. An on-chip frequency multiplier used to generate the 168MHz system clock from the 16MHz HSE crystal (PLLM=16, PLLN=336, PLLP=2). |
| **PWM** | Pulse Width Modulation. A signal that switches between high and low at a fixed frequency; the ratio of high time to total period (duty cycle) controls the average power delivered to the motor windings. |
| **RC** | Radio Control. Refers to the standard servo PWM signal format used by RC receivers and flight controllers: 50Hz frame rate, 1000–2000µs pulse width encodes 0–100% throttle. |
| **TRGO** | Trigger Output. A signal generated by a timer that can be routed internally to trigger other peripherals. TIM1_TRGO (update event) is used here to simultaneously trigger all three ADCs. |
| **ZC** | Zero Crossing. The moment when the floating phase BEMF voltage crosses the virtual neutral voltage (average of all three phases). Detecting this event gives the rotor's electrical position for commutation timing. |