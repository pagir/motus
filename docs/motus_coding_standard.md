# Motus Coding Standard

Motus uses a MISRA-aware, embedded-focused, and open-source-friendly C style for safety, portability, and global collaboration. 
This document defines conventions for C code, headers, and directory layout used across all Motus modules.

## Purpose and scope

The coding standard exists to keep Motus readable, portable, deterministic, and easy to extend across teams, companies, and MCU families. 
It favors consistency over individual style and treats naming, API shape, module boundaries, and documentation as part of the framework design itself.

## General principles

- Safety first: prefer explicit, deterministic constructs and avoid surprising behavior in control paths.
- Portability first: avoid vendor-specific assumptions in core APIs and reusable modules.
- Readability over cleverness: code should be understandable during review without hidden intent.
- Consistency over preference: project rules take precedence over personal coding habits.
- Small, single-purpose modules are preferred over large, cross-cutting implementations.

## Naming conventions

### Identifier style

- Use `snake_case` for functions, variables, files, and directories .
- Use `UPPER_SNAKE_CASE` for macros, compile-time constants, and configuration switches.
- Avoid identifiers that differ only by case or minor spelling changes because they reduce review clarity.
- Avoid unexplained abbreviations except well-known embedded terms such as `adc`, `pwm`, `gpio`, `foc`, and `pll`.

### Namespacing

- All public functions, public data types, and public interfaces must use the `motus_` namespace prefix.
- All public macros, constants, and enum values must use the `MOTUS_` namespace prefix.
- Internal helpers should remain file-local with `static` whenever possible to reduce symbol leakage.

### Type naming

- Public types use the form `motus_<name>_t`.
- Public enums use the form `motus_<name>_t`.
- Public structures use the form `motus_<name>_t`.
- Structure members use plain `snake_case`.

Example:

```c
typedef enum motus_precharge_state_t
{
    MOTUS_PRECHARGE_STATE_IDLE = 0,
    MOTUS_PRECHARGE_STATE_ACTIVE,
    MOTUS_PRECHARGE_STATE_DONE,
    MOTUS_PRECHARGE_STATE_FAULT
} motus_precharge_state_t;

typedef struct motus_precharge_cfg_t
{
    uint32_t timeout_ms;
    uint32_t interval_ms;
    bool enable_verify_bus_voltage;
} motus_precharge_cfg_t;
```

### Units in identifiers

Physical quantities and timing values should include units in their names to improve correctness and reviewability. 
Preferred suffixes include `_ms`, `_us`, `_mv`, `_ma`, `_rpm`, and `_cdeg`.

Example:

```c
uint32_t timeout_ms;
uint16_t bus_voltage_mv;
int16_t phase_current_ma;
int16_t speed_rpm;
int16_t temp_cdeg;
```

## File and module structure

A consistent directory layout improves portability and makes contribution patterns obvious to global collaborators. 
Public interfaces should remain stable and easy to find, while target-specific code should stay isolated from reusable logic.

Recommended layout:

- `include/motus/` for public headers.
- `core/` for portable logic, math, utilities, and framework primitives.
- `hal/` for abstraction interfaces and target-specific low-level implementations.
- `services/` for reusable application-level services such as precharge or protection managers.
- `profiles/` for MCU, board, or product configuration bindings.
- `tests/` for host and target tests.
- `docs/` for architecture and contribution documents.

File names should use one module per pair of files, such as `motus_precharge.h` and `motus_precharge.c`. 
Private helpers may use `*_internal.h` only when a clear internal API boundary is needed.

## Data types and portability rules

All external interfaces should use fixed-width integer types from `<stdint.h>` for predictable behavior across toolchains and architectures. 
Public APIs should avoid implementation-defined base types such as `int`, `short`, and `long` unless the exact width is irrelevant and documented. 
Boolean intent should use `bool` from `<stdbool.h>`.

Example:

```c
#include <stdint.h>
#include <stdbool.h>

typedef struct motus_precharge_cfg_t
{
    uint32_t timeout_ms;
    uint32_t interval_ms;
    bool enable_verify_bus_voltage;
} motus_precharge_cfg_t;
```

## Function design and API rules

Public APIs should use verb-oriented names that describe action and object clearly, such as `motus_precharge_init()`, `motus_precharge_start()`, and `motus_precharge_process()`.
Functions should do one thing, have a clear contract, and avoid hidden state changes not reflected in the interface.

Recommended practices:

- Keep functions small and single-purpose.
- Pass configuration through explicit structures rather than long argument lists.
- Use `const` on pointer parameters when mutation is not intended.
- Separate initialization, execution, query, and shutdown APIs clearly.

Example:

```c
bool motus_precharge_init(const motus_precharge_cfg_t *cfg);
void motus_precharge_start(void);
void motus_precharge_stop(void);
void motus_precharge_process(uint32_t now_ms);
motus_precharge_state_t motus_precharge_get_state(void);
```

Internal helper functions should be `static` inside the module source file when possible, with descriptive names such as `precharge_state_advance()` and `precharge_check_timeout()`.

## Macros and compile-time configuration

Macros should be reserved for compile-time configuration, constants, header guards, and cases where inline functions are not practical. 
Function-like macros should be avoided where inline functions provide better type safety and debug visibility.

Examples:

```c
#define MOTUS_PRECHARGE_DEFAULT_TIMEOUT_MS   (200U)
#define MOTUS_PRECHARGE_DEFAULT_INTERVAL_MS  (10U)
#define MOTUS_CFG_PRECHARGE_ENABLE           (1U)
```

Magic numbers should not appear in control logic or public APIs; use named constants or configuration fields instead.

## Comments and documentation

Public APIs should have concise documentation that describes purpose, parameters, units, return values, and important side effects. 
Comments should explain intent, assumptions, or safety rationale rather than restating obvious code.

Example:

```c
/**
 * @brief Initialize the Motus precharge service.
 *
 * @param cfg Pointer to configuration structure. Time values are in milliseconds.
 *
 * @return true on success, false if cfg is NULL or invalid.
 */
bool motus_precharge_init(const motus_precharge_cfg_t *cfg);
```

## MISRA-aware rules

Motus follows a MISRA-aware approach to reduce undefined behavior, improve portability, and support safety-oriented review practices. 
The project may not enforce every MISRA rule mechanically in every module, but the design intent should remain aligned with MISRA principles.

Expected practices:

- Avoid undefined and implementation-defined behavior.
- Avoid hidden side effects in expressions and macros.
- Avoid dynamic memory allocation in control-critical paths.
- Avoid recursion and unbounded loops in timing-sensitive code.
- Document justified deviations where practicality requires exceptions.

## Example module template

The following example shows the preferred public API pattern for a Motus service module.

```c
#ifndef MOTUS_PRECHARGE_H
#define MOTUS_PRECHARGE_H

#include <stdint.h>
#include <stdbool.h>

typedef enum motus_precharge_state_t
{
    MOTUS_PRECHARGE_STATE_IDLE = 0,
    MOTUS_PRECHARGE_STATE_ACTIVE,
    MOTUS_PRECHARGE_STATE_DONE,
    MOTUS_PRECHARGE_STATE_FAULT
} motus_precharge_state_t;

typedef struct motus_precharge_cfg_t
{
    uint32_t timeout_ms;
    uint32_t interval_ms;
    bool enable_verify_bus_voltage;
} motus_precharge_cfg_t;

bool motus_precharge_init(const motus_precharge_cfg_t *cfg);
void motus_precharge_start(void);
void motus_precharge_stop(void);
void motus_precharge_process(uint32_t now_ms);
motus_precharge_state_t motus_precharge_get_state(void);

#endif /* MOTUS_PRECHARGE_H */
```

## Adoption notes

This standard should be treated as a living document that evolves with Motus as more modules, targets, and contributors are added. 
Any future additions should preserve the same visible naming pattern, namespace discipline, and portability-first structure so that Motus remains easy to understand across geographies and organizations.
