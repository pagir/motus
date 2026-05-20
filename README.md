# Motus

**An Open Source Motor Control Framework**  
*by [Pagir](https://github.com/pagir) — Platform for Applied Global Intelligence for Regeneration*

> *Motus* — Latin for motion, movement, impulse.

---

## Vision

Motus is a modular, hardware-agnostic motor control framework inspired by the architectural philosophy of Zephyr RTOS — clean separation of concerns, portability across targets, and production-grade reliability.

Built on years of hands-on experience in power electronics and motor control — from sensorless FOC on Cortex-M0 to resolver-based drives and field weakening — Motus aims to be the open foundation that the motor control community has been missing.

Hardware changes. Algorithms don't. Motus is built on that principle.

Energy efficiency is not a feature. It is the purpose.

---

## Design Philosophy

Motus borrows its hardware abstraction model from Zephyr RTOS:

- **Devicetree (DTS)** — hardware description is separated from firmware logic. Board files describe PWM timers, ADC channels, encoder interfaces, and shunt configurations. Algorithms remain portable and clean.
- **Kconfig** — build-time configuration with zero runtime overhead. Enable FOC, select observers, configure sensor feedback, gate advanced algorithms — all without `#ifdef` pollution in algorithm code.
- **CMake** — structured, scalable build system from single target to multi-platform.

The result: the same FOC pipeline runs unchanged across STM32, TI C2000, NXP S32, Renesas RA, and other targets. Only the board DTS changes.

---

## Architecture

```
motus/
├── dts/
│   ├── bindings/             # Motus-specific DTS bindings
│   │   ├── pagir,motus-pwm.yaml
│   │   ├── pagir,motus-adc.yaml
│   │   ├── pagir,motus-encoder.yaml
│   │   └── pagir,motus-resolver.yaml
│   └── boards/               # Reference board DTS overlays
├── Kconfig                   # Top-level Kconfig
├── CMakeLists.txt
├── hal/                      # DTS-driven Hardware Abstraction Layer
│   ├── pwm/                  # Complementary PWM, dead-time, sync
│   ├── adc/                  # Phase current, DC link, temperature
│   ├── encoder/              # Incremental, Hall, resolver (AD2S1210)
│   └── gpio/                 # Fault inputs, enable signals
├── transforms/               # Pure math — no HAL dependency
│   ├── clarke/               # αβ transform
│   ├── park/                 # dq transform
│   └── inverse/              # Inverse Park, inverse Clarke
├── modulation/               # PWM modulation strategies
│   ├── svpwm/                # Space Vector PWM
│   └── spwm/                 # Sinusoidal PWM
├── observers/                # Rotor position and speed estimation
│   ├── smo/                  # Sliding Mode Observer + PLL
│   ├── luenberger/           # Luenberger state observer
│   └── bemf/                 # BEMF zero-crossing detection
├── controllers/              # Closed loop controllers
│   ├── current/              # PI d-axis and q-axis current loops
│   ├── speed/                # PI speed loop
│   └── position/             # Position control
├── algorithms/               # Advanced control algorithms
│   ├── mtpa/                 # Maximum Torque Per Ampere locus
│   ├── field_weakening/      # Voltage ellipse based field weakening
│   └── mtpv/                 # Maximum Torque Per Volt
├── sensors/                  # Sensor abstraction layer
│   ├── hall/                 # Hall sensor state machine
│   ├── encoder/              # Incremental encoder interface
│   ├── resolver/             # AD2S1210 resolver interface
│   └── sensorless/           # IPD + observer based sensorless
├── protection/               # Fault detection and management
│   ├── overcurrent/
│   ├── overvoltage/
│   ├── thermal/
│   └── fsm/                  # Fault state machine
├── diagnostics/              # Self-commissioning and identification
│   ├── resistance/           # Rs estimation
│   ├── inductance/           # Ld, Lq estimation
│   └── flux/                 # Flux linkage estimation
├── profiles/                 # Target-specific configurations
│   ├── stm32g4/
│   ├── stm32f4/
│   └── nrf5340/
└── docs/                     # Theory, tuning guides, application notes
```

---

## Devicetree Example

```dts
/* motus_board.overlay */

&motus_pwm {
    compatible = "pagir,motus-pwm";
    dead-time-ns = <500>;
    frequency-hz = <20000>;
    complementary;
    status = "okay";
};

&motus_adc {
    compatible = "pagir,motus-adc";
    channels = <PHASE_U PHASE_V DC_LINK TEMPERATURE>;
    shunt-resistance-mohm = <5>;
    status = "okay";
};

&motus_feedback {
    compatible = "pagir,motus-resolver";
    variant = "ad2s1210";
    resolution = <16>;
    status = "okay";
};
```

---

## Kconfig Example

```kconfig
menu "Motus Motor Control"

config MOTUS_FOC
    bool "Field Oriented Control"
    default y
    help
      Enable FOC pipeline — Clarke/Park transforms,
      current loop, SVPWM modulation.

config MOTUS_OBSERVER_SMO
    bool "Sliding Mode Observer"
    depends on MOTUS_FOC
    help
      SMO with PLL for sensorless rotor position estimation.

config MOTUS_OBSERVER_BEMF
    bool "BEMF Zero Crossing Observer"
    depends on MOTUS_FOC

config MOTUS_OBSERVER_LUENBERGER
    bool "Luenberger State Observer"
    depends on MOTUS_FOC

config MOTUS_MTPA
    bool "Maximum Torque Per Ampere"
    depends on MOTUS_FOC
    help
      MTPA locus tracking for IPMSM drives.

config MOTUS_FIELD_WEAKENING
    bool "Field Weakening"
    depends on MOTUS_MTPA
    help
      Voltage ellipse based field weakening for
      above base speed operation.

config MOTUS_MTPV
    bool "Maximum Torque Per Volt"
    depends on MOTUS_FIELD_WEAKENING

config MOTUS_SENSOR_RESOLVER
    bool "Resolver feedback (AD2S1210)"

config MOTUS_SENSOR_HALL
    bool "Hall sensor feedback"

config MOTUS_SENSOR_ENCODER
    bool "Incremental encoder feedback"

config MOTUS_SENSOR_SENSORLESS
    bool "Sensorless operation"
    depends on MOTUS_OBSERVER_SMO || MOTUS_OBSERVER_BEMF

config MOTUS_SELF_COMMISSIONING
    bool "Self commissioning — Rs, Ld, Lq, flux estimation"
    depends on MOTUS_FOC

endmenu
```

---

## Planned Features

- [ ] FOC pipeline — Clarke/Park/SVPWM with configurable ISR structure
- [ ] Observer library — SMO-PLL, Luenberger, BEMF zero-crossing
- [ ] Sensor abstraction — Hall, encoder, resolver (AD2S1210), sensorless IPD
- [ ] MTPA, field weakening, MTPV — Kconfig selectable
- [ ] Fault management FSM with configurable protection thresholds
- [ ] Self-commissioning — Rs, Ld, Lq, flux linkage identification
- [ ] Zephyr RTOS native integration
- [ ] STM32G4 and STM32F4 reference targets
- [ ] Documentation — from theory to tuning

---

## Motivation

Motor drives account for nearly 45% of global electricity consumption. A 10% improvement in drive efficiency at scale translates to enormous energy savings. Open, auditable, high-quality motor control software is a prerequisite for that improvement — especially at the edge, in the hands of engineers who build real products.

Most open source motor control code today is target-specific, monolithic, and difficult to port. Motus is built differently — portable by design, configurable at build time, and grounded in production-grade control theory.

Motus exists to make energy-efficient motor control accessible to every engineer.

---

## Status

🚧 **Initiating** — Architecture and module structure in progress.

---

## License

Apache 2.0 — open for use, modification, and contribution.

---

## Contributing

Contributions welcome. If you work in power electronics, motor control, embedded systems, or related fields — this is your framework too.

Open an issue. Start a discussion. Submit a pull request.

---

*Motus is a project of [Pagir](https://github.com/pagir) — Platform for Applied Global Intelligence for Regeneration.*
