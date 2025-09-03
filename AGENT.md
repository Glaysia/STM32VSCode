# AGENT.md — Working Guide for the Assistant

Project: 100W High‑Efficiency Resonant DC‑DC Converter on NUCLEO‑H533RE
Owner: harry (undergraduate project: Power Electronics / Microprocessor Lab)
Audience: AI assistant only (internal working notes and scaffolding)

## Purpose
- Establish a crisp, practical plan to get a working 100W, high‑efficiency DC‑DC converter using a resonant topology, controlled by STM32 NUCLEO‑H533RE.
- Balance feasibility for a semester project: start simple and safe, then iterate toward higher performance.
- Keep scope modular so we can pivot (e.g., to a simpler non‑resonant backup) if needed.

## Current Assumptions (to confirm)
- Input: TBD (e.g., 24V or 48V DC). Define exact range (min/nom/max).
- Output: TBD (e.g., 12V @ ≥8A for 100W). Define tolerance and ripple.
- Isolation: Prefer isolated topology; confirm isolation requirement and safety class.
- Topology: Half‑bridge LLC resonant (preferred), with variable frequency control on the primary.
- Switching frequency: Nominal resonance around 100–250 kHz; confirm magnetics feasibility and driver limits.
- Controller: STM32H533RE on NUCLEO‑H533RE using advanced timers (e.g., TIM1) for complementary PWM with dead‑time, and ADC with timer‑triggered sampling; no assumption of HRTIM availability.
- Gate driving: Primary half‑bridge driver IC with proper level shift/bootstrapping; secondary uses diode or dedicated SR controller (MCU may not directly drive secondary SR).
- Sensing: V_in, V_out, I_out (or I_primary), NTC temperature(s). All scaled to MCU ADC range with adequate filtering.

Callouts: Any unknown spec above should be resolved early; decisions affect magnetics, component ratings, firmware timing, and protections.

## High‑Level Approach
- Phase 1 (Foundations): Confirm specs, select LLC half‑bridge, rough power stage sizing, choose key components. Bring up MCU PWM + ADC + UART on bench load (no high power yet).
- Phase 2 (Low‑Power Bring‑Up): Power the primary from a current‑limited DC supply with a small resistive/dummy load through the transformer secondary; verify soft‑start, resonant operation, and protection trips.
- Phase 3 (Regulation & Control): Implement output regulation via frequency control, add basic PI around V_out, validate transient behavior. Add telemetry/logging and fault handling.
- Phase 4 (Optimization): Tune switching frequency band, snubbers, magnetics, and layout to push efficiency and power up toward 100W.

## Control Strategy (LLC, half‑bridge)
- Primary: Complementary gate signals with dead‑time. Variable frequency about LLC resonance to regulate V_out.
- Soft‑start: Start above resonance (lower gain), sweep toward resonance while ramping PWM enable window; enforce current limit.
- Regulation: Outer voltage loop (PI). Command a target switching frequency limited to [f_min, f_max] for safe ZVS region.
- Protections: Fast OCP (peak or average via ADC + comparator if available), OVP, UVLO on V_in, OTP, and fault latching with back‑off.

## Firmware Architecture (STM32H5, no vendor lock)
- BSP: Clock, GPIO, UART, SysTick.
- PWM: TIM1 in center‑aligned mode, complementary outputs on two channels, programmable dead‑time, brake/kill input provision.
- ADC: Timer‑triggered conversions synchronized to PWM (sample currents/voltages at consistent phase), DMA circular buffer for low CPU overhead.
- Control: Cooperative task loop at fixed rate (e.g., 10–20 kHz control tick via timer interrupt). States: INIT → PRECHARGE → SOFT_START → RUN → FAULT.
- Telemetry: UART CLI for logs, parameter tweaks (gain, freq limits, trip thresholds), and fault readback.
- Config: Persistent parameters via a small config block (flash or compile‑time constants for simplicity first).

Suggested modules (folders once we start firmware):
- firmware/drv_pwm/ — timer setup, dead‑time, enable/disable, brake.
- firmware/drv_adc/ — channels, scaling, DMA rings, calibration.
- firmware/ctrl/ — state machine, PI regulator, frequency scheduler.
- firmware/prot/ — OCP/OVP/UVLO/OTP, fault latch, clear.
- firmware/bsp/ — clocks, pins, uart, timebase.
- firmware/app/ — main loop, CLI commands, telemetry framing.

## Hardware Checklist (LLC Half‑Bridge)
- Magnetics: Transformer with LLC magnetizing inductance; Lr, Lm design; Cr selection; target f0; turns ratio for V_out at nominal.
- Semiconductors: Half‑bridge MOSFETs sized for voltage/current with low Qg/Qoss; diode or SR controller on secondary.
- Gate Driver: Proper high‑side/low‑side driver with dead‑time and UVLO; consider isolated driver if needed.
- Sensing: Scaled dividers and current sense (shunt + amp or CT/HV sense), RC filtering, anti‑aliasing.
- Protections: Input fuse, inrush limiter or pre‑charge, TVS/RC snubbers, primary/secondary OCP, temp sensors.
- Layout: Tight half‑bridge loop, short resonant tank loops, Kelvin sense for shunts, split analog/digital grounds with single‑point connection.

## Safety
- Use an isolated bench supply with current limit for initial tests. No mains input during early bring‑up.
- One‑hand rule, safety glasses, clear lab area, no jewelry, insulated probes. Document emergency stop procedure.
- Add hardware “kill” line to disable gate driver; map to an MCU brake pin and physical switch.

## Bring‑Up Procedure (recommended)
- Dry‑run: Validate PWM waveforms into a resistive dummy at very low bus voltage (5–12 V) and a current‑limited supply.
- Measure: Check gate dead‑time, Vgs ringing, resonant tank voltage/current at low power. Verify zero/near‑zero voltage switching region.
- Protections: Force faults (short load, over‑voltage via supply) in a safe, low‑energy setup to confirm trips and latching.
- Soft‑start: Ramp to modest load (10–30 W) while monitoring temperature and waveforms.
- Scale up: Increase input voltage and load in small steps, logging efficiency and temperatures at each step.

## Efficiency & Validation
- Measure: Input V/I and output V/I with calibrated meters or power analyzer; compute η at several load points (10%, 25%, 50%, 75%, 100%).
- Thermal: Spot‑check MOSFETs, transformer, diodes/SR, driver IC, inductors with IR camera or thermocouples.
- EMI: Quick sanity check with near‑field probe or AM radio; refine snubbers/layout if excessive.

## Repo Conventions
- Top‑level: `AGENT.md` (this), `프로젝트계획/` (Korean plan), `firmware/`, `hardware/`, `docs/`.
- Firmware style: C with HAL or LL drivers; avoid blocking delays inside control ISR. Keep ISR lean.
- Parameterization: `config.h` for limits and default gains; guardrails for freq range and protections.
- Logging: Minimal text to avoid UART overhead; optional binary telemetry if needed later.

## Definition of Done (v1)
- Converts ≥ 80 W continuously, target 100 W peak/continuous per thermal design.
- Meets V_out regulation within spec across intended load range using frequency control.
- Demonstrates protections: OCP, OVP, UVLO, OTP with safe recovery.
- Efficiency ≥ 90% at mid‑to‑high load (update target after magnetics selection).
- Documented schematic, BOM, firmware build/flash steps, and test results.

## Milestones
- M0: Specs frozen (Vin/Vout, isolation, power derating). Risks logged.
- M1: Schematic + preliminary magnetics design; component samples ordered.
- M2: Firmware PWM/ADC/CLI validated on bench (no power stage).
- M3: PCB bring‑up at low voltage, soft‑start verified, basic regulation works.
- M4: 50–70 W operation stable; protections validated; thermal OK.
- M5: 100 W target reached; efficiency tuned; report prepared.

## Risks & Mitigations
- Resonant design complexity: Keep a simpler non‑resonant buck as fallback path for demo, if needed.
- Magnetics lead‑time: Start ordering cores/wires early; prepare alternate core sizes.
- Lack of HRTIM: Use TIM1 advanced features; reduce switching frequency if timer resolution becomes limiting.
- SR complexity: Start with diodes; adopt SR controller later if efficiency needs it.
- EMI/instability: Keep snubbers footprints; use damping resistors; ensure clean analog routing and grounds.

## Immediate Next Actions
- Confirm: Vin range, Vout target/spec, isolation requirement, allowed switching frequency band.
- Decide: Initial LLC design point (f0, Lr, Lm, Cr, turns ratio) and primary MOSFET/driver.
- Start: Firmware skeleton (clock, GPIO, UART, TIM1 PWM, ADC DMA) with a “fake plant” mode for control development without hardware.

---
If you want, I can scaffold the firmware folders and a minimal TIM1 PWM + ADC DMA example next.

