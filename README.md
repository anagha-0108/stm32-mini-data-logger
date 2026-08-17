# STM32 Mini Data Logger — Xalten Systems Take-Home

## Overview
ADC1 samples the internal temperature sensor every 100ms, triggered by TIM2 (no
`HAL_Delay()` polling). Each sample feeds an 8-point circular-buffer moving-average
filter. The filtered value drives PWM on TIM3 (~1kHz), and the filtered value, CCR,
and duty percentage stream over USART2 (115200 baud) each sample. The Nucleo user
button (PC13, EXTI) pauses/resumes the loop with software debounce.

No hardware was available for this submission. Builds clean in CubeIDE 2.1.1,
0 errors / 0 warnings.

## Design decisions

**Analog input:** Used the internal temp sensor (`ADC_CHANNEL_TEMPSENSOR`) since I
had no external pot to wire up. According to the reference manual for STM32F411xC/E (RM0383) and the datasheet, it needs at least
10µs of sampling time — the closest available step in CubeMX is 480 cycles, so
that's what I used.

**Interrupt vs DMA:** Used TIM2->ADC via TRGO with `HAL_ADC_ConvCpltCallback()`
rather than DMA. I've worked with interrupts and timers before, and while DMA cuts
down CPU load, it's really meant for high-rate conversions — a single reading
every 100ms doesn't call for it, so interrupts felt like the right tradeoff here.

**ISR-to-main handoff:** The ADC callback just reads the value, stores it in a
circular buffer, advances the index (wrapping back to 0 and setting `buffer_full`
once it's gone all the way around), and sets a `volatile new_sample_ready` flag —
no math or UART calls inside the ISR itself. The main loop does the actual
averaging/PWM/UART work once that flag is set, and only averages over
`buffer_index` samples until the buffer has filled once, so it's not averaging in
garbage/uninitialized slots early on.

**Debounce:** Timestamp-based, using `HAL_GetTick()` — similar in spirit to
Arduino's `millis()`, which I'd used before. The EXTI callback only toggles the
`paused` flag if more than 200ms has passed since the last accepted press. B1 is
active-low with an external pull-up on the board, so it's configured
`GPIO_MODE_IT_FALLING` with no internal pull needed.

**Clock/timer math:** SYSCLK is 84MHz. APB1's prescaler is /2, but APB1 timer
clocks get automatically doubled by hardware whenever that prescaler isn't 1, so
TIM2 and TIM3 both actually run at 84MHz. TIM2 uses Prescaler=8399, Period=999 for
a 10Hz sample trigger (TRGO on Update Event). TIM3 uses Prescaler=83, Period=999
for a 1kHz PWM.

**PWM output:** The filtered ADC value (0–4095) gets scaled down to the timer's
0–999 count range and written straight into the CCR register with
`__HAL_TIM_SET_COMPARE()` — that's what actually drives PA6. The duty cycle
percentage sent over UART is a separate calculation, `D% = CCR/(ARR+1) × 100`,
just for readability in the log.

## What I learned from scratch
- **CubeMX/HAL toolchain:** HAL's weak-callback override pattern for
  `HAL_ADC_ConvCpltCallback()` and `HAL_GPIO_EXTI_Callback()`; the
  `USER CODE BEGIN/END` marker system (I lost code once by placing it outside a
  protected marker, and the next regeneration wiped it); and the automatic
  doubling of APB1 timer clocks relative to APB1 peripheral clocks, which I had
  to account for to get correct Prescaler/Period values.
- **Internal temperature sensor:** it has to be manually roused via the TSVREFE
  bit in `ADC->CCR`, and needs a minimum 10µs ADC sampling time, which forced me
  to use the highest available SamplingTime step.
- **Circular buffer / moving average:** how a fixed-size window absorbs a new
  value by overwriting the oldest one once it's full, and once the index reaches
  the window size it just resets back to index 0, and averages over those 8
  values with the new value along with the 7 old ones.

## References
- "Hands-On with STM32 Timers: Trigger Periodic ADC Conversions" —
  https://www.youtube.com/watch?v=Yt5cHkmtqlA
- "STM32 Beginners Guide Part3: PWM, TIMERS, Frequency and Duty Cycle. LED Dimming with PWM example." — https://www.youtube.com/watch?v=zHWvFchXhvw
- "STM32 External Interrupt (EXTI) | Button debounce interrupt and LED Toggle | " - https://www.youtube.com/watch?v=dJNHOZxLcOc
- STM32F411 Reference Manual (RM0383) —
  https://www.st.com/resource/en/reference_manual/rm0383-stm32f411xce-advanced-armbased-32bit-mcus-stmicroelectronics.pdf
- STM32F411RE product page / datasheet —
  https://www.st.com/en/microcontrollers-microprocessors/stm32f411re.html
