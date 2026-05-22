# Simulation Tests

## single_phase_diode_half_wave_rectifier.slx

Single-phase diode half-wave rectifier simulation.

It shows how a diode rectifies an AC sine wave into a positive half-wave output on a resistive load.

Observed waveforms:

- Input voltage: full sine wave.
- Load voltage: positive half-wave.
- Load current: positive half-wave, in phase with the load voltage.

## single_phase_scr_half_wave_rectifier.slx

Single-phase SCR half-wave controlled rectifier simulation.

It shows how a thyristor controls the positive half-wave output by delaying the gate trigger angle on an R-L load.

Observed waveforms:

- Input voltage: full sine wave.
- Gate pulse: delayed trigger pulse at the selected firing angle.
- Load current: delayed positive half-wave, extended by the inductive load.
- Mean and RMS current: calculated from the rectified load current.
