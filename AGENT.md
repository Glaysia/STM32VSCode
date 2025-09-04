# AGENT.md — Working Guide for the Assistant

Project: 50W Isolated Resonant DC‑DC Converter on NUCLEO‑H533RE
Owner: harry (undergraduate project: Power Electronics / Microprocessor Lab)
Audience: AI assistant only (internal working notes and scaffolding)

## Purpose
- Deliver a practical, working 50W isolated DC‑DC converter using a resonant tank with a high‑frequency transformer, controlled by STM32 NUCLEO‑H533RE.
- Keep it safe and achievable: fixed 10 kHz PWM, duty‑cycle control, simple secondary diode rectification.
- Maintain room to iterate (snubbers, magnetics tuning) without expanding scope.

## Current Assumptions (frozen)
- Input: 30 V DC, CV mode, source current limit about 3 A.
- Output: 12 V DC nominal; up to ~50 W load. Define ripple and regulation targets during test.
- Isolation: Yes, via high‑frequency transformer.
- Power train: Single MOSFET (low‑side) → gate driver → primary → transformer (center‑tap secondary) → two diodes (full‑wave) → output capacitor.
- Resonant tank: Series L–C using transformer leakage inductance as Lr plus a series capacitor Cr (LL/series‑resonant behavior ahead of the transformer).
- Switching: Fixed 10 kHz PWM. Regulation via duty‑cycle (not frequency sweep).
- Controller: STM32H533RE using TIM1 PWM and ADC with timer‑triggered sampling; UART for telemetry.
- Sensing: V_in, V_out, I_out (or primary current proxy), NTC temperature(s) into ADC with proper scaling and filtering.

Callouts: Any unknown spec above should be resolved early; decisions affect magnetics, component ratings, firmware timing, and protections.

## High‑Level Approach
- Phase 1 (Foundations): Lock specs (30 V in, 12 V out, 50 W, 10 kHz). Choose transformer core/turns, select MOSFET, gate driver, diodes, and Cr.
- Phase 2 (Low‑Power Bring‑Up): MCU PWM + ADC + UART on bench; validate 10 kHz gate drive into dummy load and observe resonant tank at low bus voltage (5–12 V).
- Phase 3 (Regulation & Control): Implement duty‑cycle regulation (PI on V_out) at fixed 10 kHz; add soft‑start and protection latching.
- Phase 4 (Optimization): Tune Cr/stray L, snubbers, and layout; scale to ~50 W with thermal/efficiency checks.

## Control Strategy (Fixed‑Freq, Series‑Resonant)
- Primary: Single PWM at 10 kHz into a low‑side MOSFET through a gate driver.
- Soft‑start: Ramp duty from 0% up to a safe limit while monitoring current/voltage; optional pre‑charge of output through a bleed resistor if needed.
- Regulation: Outer voltage PI loop commands duty within [d_min, d_max] at fixed 10 kHz.
- Protections: Fast OCP, OVP, UVLO, OTP with fault latch and back‑off; hardware brake/KILL input to driver.

## Firmware Architecture (STM32H5, no vendor lock)
- BSP: Clock, GPIO, UART, SysTick.
- PWM: TIM1 single‑channel PWM at 10 kHz, adjustable duty, hardware brake input.
- ADC: Timer‑triggered conversions synchronized to PWM; DMA circular buffer; scaling and calibration.
- Control: Timer ISR for a control tick around 5–10 kHz; states: INIT → PRECHARGE → SOFT_START → RUN → FAULT.
- Telemetry: UART CLI for logs, parameter tweaks (gains, duty limits, trip thresholds), and fault readback.
- Config: Compile‑time defaults with runtime overrides; guardrails for duty and protection thresholds.

Suggested modules (folders once we start firmware):
- firmware/drv_pwm/ — timer setup, dead‑time, enable/disable, brake.
- firmware/drv_adc/ — channels, scaling, DMA rings, calibration.
- firmware/ctrl/ — state machine, PI regulator, frequency scheduler.
- firmware/prot/ — OCP/OVP/UVLO/OTP, fault latch, clear.
- firmware/bsp/ — clocks, pins, uart, timebase.
- firmware/app/ — main loop, CLI commands, telemetry framing.

## Hardware Checklist (Single‑Ended, Series‑Resonant)
- Magnetics: Transformer for 10 kHz operation; leverage leakage inductance as Lr; select Cr; choose turns ratio for 30 V → 12 V under load.
- Semiconductors: Single low‑side MOSFET sized for bus voltage/current; secondary uses two fast diodes (full‑wave CT) initially.
- Gate Driver: Low‑side driver with strong source/sink, UVLO; include hardware KILL path.
- Sensing: Scaled dividers and current sense (shunt + amp or CT), RC filtering.
- Protections: Input fuse, snubbers/RC damping across MOSFET/primary as needed, OCP/OVP/UVLO/OTP, temp monitoring.
- Layout: Minimize switching loop area; keep tank loop short; Kelvin sense; clean analog ground referencing.

## Safety
- Use an isolated bench supply with current limit for initial tests. No mains input during early bring‑up.
- One‑hand rule, safety glasses, clear lab area, no jewelry, insulated probes. Document emergency stop procedure.
- Add hardware “kill” line to disable gate driver; map to an MCU brake pin and physical switch.

## Bring‑Up Procedure (recommended)
- Dry‑run: Validate 10 kHz PWM and clean Vgs into a dummy load at 5–12 V bus with current limit.
- Measure: Observe resonant tank voltage/current; check duty response of V_out; characterize gain vs duty.
- Protections: Exercise OCP/OVP/UVLO at low energy to validate latching and recovery.
- Soft‑start: Ramp to 10–30 W first; verify temperatures and waveforms; then step toward ~50 W.
- Demo: Power the owner’s monitor from the 12 V output once stability is confirmed.

## Efficiency & Validation
- Measure: Input V/I and output V/I; compute efficiency across load points (10%, 25%, 50%, 75%, 100%).
- Thermal: Check MOSFET, transformer, diodes, and driver temps.
- EMI: Sanity check; add damping/snubbers as needed.

## Repo Conventions
- Top‑level: `AGENT.md` (this), `프로젝트계획/` (Korean plan), `firmware/`, `hardware/`, `docs/`.
- Firmware style: C with HAL or LL drivers; avoid blocking delays inside control ISR. Keep ISR lean.
- Parameterization: `config.h` for limits and default gains; guardrails for freq range and protections.
- Logging: Minimal text to avoid UART overhead; optional binary telemetry if needed later.

## Definition of Done (v1)
- Sustains ~50 W output at 12 V from a 30 V input with stable regulation at 10 kHz.
- Meets basic V_out regulation and ripple targets under typical loads using duty control.
- Demonstrates protections: OCP, OVP, UVLO, OTP with safe recovery.
- Provides documentation: schematic, BOM, firmware build/flash steps, and test results.

## Milestones
- M0: Specs frozen (30 V in CV, 12 V out, 50 W, 10 kHz). Risks logged.
- M1: Schematic + magnetics estimate (turns, L_leak target) + Cr selection; parts ordered.
- M2: PWM/ADC/CLI validated on bench; 10 kHz drive clean; protections stubbed.
- M3: Power stage bring‑up at low voltage; soft‑start and duty regulation working.
- M4: ~50 W operation stable; protections validated; thermal OK; demo monitor powered.
- M5: Documentation finalized; results and waveforms captured.

## Risks & Mitigations
- Low switching frequency (10 kHz) increases transformer size and ripple: choose appropriate core; increase output capacitance and consider small LC output filter.
- Resonant behavior sensitivity to leakage L: be ready to adjust winding layout/gaps or Cr value.
- Drive losses/ringing: ensure strong gate driver, short loop, and adequate snubbing.
- Parts lead‑time: order cores, driver, MOSFETs, diodes, and film capacitors early with alternates.
- Backup plan: simple step‑down (non‑isolated) or hard‑switched isolated variant for minimum demo if needed.

## Immediate Next Actions
- Calculate turns ratio for 30 V → 12 V at 10 kHz and expected duty range.
- Estimate leakage L target and select Cr for desired gain at nominal load.
- Pick MOSFET, gate driver, diodes, and output capacitor bank; draft BOM.
- Scaffold firmware: clock/GPIO/UART/TIM1 PWM at 10 kHz + ADC DMA; add duty PI loop and protection stubs.

---
If you want, I can scaffold the firmware folders and a minimal TIM1 PWM + ADC DMA example next.
