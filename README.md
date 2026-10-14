# nora-hal-dspic33ck-dma

**NORA-HAL** — *Native On-chip Resource Assistant*

Small, readable low-level DMA HAL for Microchip dsPIC33CK devices — part of
**NORA-HAL**, a HAL family whose public API is namespaced `nora_*` / `NORA_*`.

This repository provides a thin, explicit abstraction over the dsPIC33CK DMA
controller: global setup, per-channel configuration, channel start/stop,
interrupt-flag/status handling, a software trigger, and a few hot-path helpers
for ping-pong audio streaming. It is sized for its real user (a framed-SPI
I2S/TDM audio transport), not as a general DMA framework.

The goal is not a complete DMA framework, but compact, explicit, easy-to-review
DMA building blocks for evaluation, FAE demos, and early software architecture
experiments.

## Naming

The public API is `nora_*` / `NORA_*`.

The chip name survives in exactly two places, both deliberate:

* **Implementation file names** carry a backend tag: `nora_dma_dspic33ck.c` is the
  dsPIC33CK backend of the processor-neutral `nora_dma.h`. A second processor
  would add `nora_dma_<tag>.c` beside it, not a second header.
* **Backend-private identifiers** inside those files (register-layer macros such
  as `NORA_DMA_DSPIC33CK_CON_DMAEN`, the `nora_dma_reg_*` helpers, and statics),
  which no caller of `nora_dma.h` sees.

The tag is `_dspic33ck`, the device family this backend actually drives — a
different silicon family from dsPIC33AK (dsPIC33**C** vs dsPIC33**A**), and never
abbreviated to `_dspic33c`.

### Relationship to the dsPIC33AK HAL of the same name

[nora-hal-dspic33ak-dma](https://github.com/sulaolab/nora-hal-dspic33ak-dma) is
the same contract for the dsPIC33AK family, and the two are deliberately close:
`nora_dma.h` here is written so that a consumer moves between the ports with the
call sites intact. **The two are not claimed to be equivalent**, and the
differences are in the enumerator sets and the validation rules rather than in
the shape of the API:

| | dsPIC33CK (here) | dsPIC33AK |
|---|---|---|
| channels | 4 (`NORA_DMA_CHANNEL_0..3`), 16-bit | 8, 32-bit capable |
| triggers reached | SPI1..SPI3 (+ `NORA_DMA_TRIGGER_NONE`) | SPI1..SPI4 |
| element width | byte or 16-bit word; `NORA_DMA_SIZE_WORD` (32-bit) is **rejected** | byte / halfword / 32-bit word |
| reload controls | one `RELOAD` bit — `reload_count`/`_src`/`_dst` are OR-collapsed onto it | three independent controls |
| DONE interrupt enable | none; DONE is gated only by the CPU IE, so `done_int_en` is accepted for parity and **writes no register bit** | independent |
| SRAM arbitration | `MSTRPR.DMAPR` | `BMXINITPR.DMAPR` |
| address window | `DMAL` / `DMAH`, **mandatory** (see below) | `DMALOW` / `DMAHIGH` |
| count register | 16-bit; `cfg.count` is validated against that limit rather than truncated | wider |

A file this port has and the AK repository does not:
`nora_dma_dspic33ck_reg_assert.h` — static assertions that the installed DFP
still places every SFR bit where this HAL assumes it is (30 assertions, compiled
on every build). The dsPIC33AK and dsPIC33CK fleets are **not** symmetric, and
nothing here should be read as a claim that they are.

## Status

Current validation target:

- Devices: dsPIC33CK64MC105 (EV88G73A Curiosity Nano), dsPIC33CK1024MP710 (EV97U97A)
  and dsPIC33CK256MP508 (DM330030)
- Compiler: XC-DSC v3.31.01
- DFP: dsPIC33CK-MC_DFP 1.10.386 / dsPIC33CK-MP_DFP 1.15.423 (MP710: 1.16.521) or compatible

**How to read the evidence below.** These HALs are built for evaluation, FAE demos and
architecture experiments, so exhaustive per-function coverage was never the goal — there is
no unit-test suite, and what exists is integration testing on real hardware. Three tiers,
used across the seven sibling repositories: **integration-verified** (it ran as part of the
working system, and something observable would have broken if it had not),
**hardware-observed, not a matrix** (it worked in the configuration actually run; other
combinations are untried rather than known-good), and **compiled, not executed**.

What that project exercises, measured 2026-08-11:

- **Executed on hardware (EV88G73A):** a RAM→RAM self-test on channel 3 driven by
  `nora_dma_software_trigger()` — PASS on all five boots of the session — and
  channels armed as the ping-pong substrate of the WM8904 I2S/TDM audio path,
  soaked with `miss = 0` across more than 137,000 audio blocks.
- **Executed on hardware (EV97U97A / CK1024MP710, 2026-10-13):** channels armed as the
  ping-pong substrate of a WM8904 48 kHz TDM8 audio path, `miss = 0` across more than
  202,000 audio blocks. On this part `nora_dma_global_init()` clears `DMAHIGHLOWH` and
  `nora_dma_channel_config()` clears `DMASRCDSTHn`: the HAL keeps its 16-bit, low-RAM
  address contract, and extended-address (EDS) buffers are **not** supported.
- **`NORA_DMA_TRIGGER_PWM_EVENT_A`** (CHSEL `0x19`, from the CK256MC005 trigger table)
  is hardware-observed on CK256MC005 only, as the per-carrier-period duty feed of a PWM
  audio output. The backend maps it on every part; check the selected part's trigger
  table before relying on it elsewhere.
- **Builds clean but unexecuted (DM330030 / CK256MP508):** that configuration is
  compile-only. Its device-specific paths — notably the SPI3
  triggers, which exist on MP508 and are Reserved on MC105 — are therefore
  *compiled* but not *run*. Treat them as reviewed source, not as validated
  silicon behaviour.

This HAL is the low-level building block; the ping-pong / block-streaming policy
lives in the consumer (e.g. the SPI/I2S/TDM HAL), not here.

## Supported Devices

Hardware validated:

- dsPIC33CK64MC105
- dsPIC33CK1024MP710 (channels 0..3, low-RAM addresses only)

Builds clean, unexecuted:

- dsPIC33CK256MP508

The backend does not `#error` on an unlisted dsPIC33CK part: it is written against
the family's DMA SFR layout (`DMACHn` / `DMASRCn` / `DMADSTn` / `DMACNTn` /
`DMAINTn`, `DMACON`, `DMAL` / `DMAH`, `MSTRPR`, and the `_DMAnIF` / `_DMAnIE` /
`_DMAnIP` bit aliases) rather than against a part list, and the DFP bit positions
it relies on are asserted at compile time. What *is* per-device is the trigger
inventory: `nora_dma_channel_config()` rejects a trigger the selected device does
not implement. Review the channel count and the trigger table of any other part
before listing it as hardware-validated.

## Design Policy

This HAL is intentionally small and low-level.

- It knows nothing about SPI, audio, PWM, DSP, `printf`, or application code.
- It contains no ping-pong / block-streaming policy. Ping-pong audio policy
  belongs to the consumer, which configures a channel through this HAL and
  decides what to do with each completed half/buffer.
- The caller owns the DMA buffers; this HAL only takes addresses.
- No callback framework: the DMA ISRs stay in the consumer modules. Each user
  owns its own `_DMAnInterrupt()` handler and calls the hot-path helpers from it.
- No dynamic memory allocation, no RTOS or scheduler dependency.
- No clock-tree or PLL setup (outside this HAL).
- No XC-DSC / DFP bitfield types are exposed in the public API; configuration is
  expressed through plain enums and a config struct.

## Scope

In scope:

- DMA controller global enable, SRAM bus arbitration (`MSTRPR.DMAPR = 1`), and
  the allowed address-limit window (`DMAL` / `DMAH`)
- Per-channel configuration (source/destination/count, address modes, element
  size, transfer/repeat mode, reload, trigger select, IRQ priority/enable)
- Channel start/stop (`CHEN`)
- A software transfer request (`CHREQ`) and its pending query — what makes "is the
  DMA controller itself working?" answerable without any peripheral attached
- Remaining-element count read
- Interrupt-enable control and save/restore for short critical sections
- Status / interrupt-flag clear and read
- Ping-pong half detection from the `HALFIF` / `DONEIF` status flags (pure mechanism)
- An ISR-hot-path snapshot helper (clear flag, read status, clear status)
- Compile-time assertions that the DFP's SFR bit positions still match

Out of scope:

- Scatter-gather, linked descriptors, match mode
- Peripheral-indirect / null-write DMA modes beyond the fields configured here
- Runtime channel allocation
- Ping-pong / double-buffer streaming policy (lives in the consumer)
- OS / scheduler integration

## Files

```text
src/
  nora_dma.h                          public contract — no device header pulled in
  nora_dma_dspic33ck.c                dsPIC33CK backend
  nora_dma_dspic33ck_fast.h           backend-private ISR fast path (exposes SFRs)
  nora_dma_dspic33ck_reg.h            backend-private register layer
  nora_dma_dspic33ck_reg_assert.h     backend-private DFP bit-position assertions
```

`nora_dma.h` deliberately does **not** `#include <xc.h>`: including the contract
does not drag the device header into a translation unit that only wants the API.
It also holds no `static inline` and no status bit macro. The `_dspic33ck` headers
do expose SFRs and are for the backend and for backend-aware, measured hot paths
only. `nora_dma_dspic33ck_reg_assert.h` is included from `..._reg.h`, so the
assertions compile with the HAL rather than sitting in a file nobody builds.

## Integration

1. Add `src/nora_dma_dspic33ck.c` to the project.
2. Add `src/` to the compiler include path.
3. Include `nora_dma.h`.
4. Call `nora_dma_global_init()` once, **before configuring any channel**.
5. Own each used channel's `_DMAnInterrupt()` vector in the consumer module. Use
   the ordinary calls, or include `nora_dma_dspic33ck_fast.h` and use the `_hot`
   inlines when the ISR is a measured hot path.

There is nothing else to supply: this HAL has no configuration header, no sibling
HAL dependency, and no board hook.

## Basic Usage

Bring up the DMA controller once, then configure and start a channel:

```c
#include "nora_dma.h"

nora_dma_global_init();     /* bus priority + DMAL/DMAH window + DMAEN */

const nora_dma_channel_cfg_t rx_cfg = {
    .src          = (volatile void *)&SPI1BUFL,
    .dst          = rx_buffer,
    .count        = RX_BUFFER_ELEMENTS,       /* whole ping+pong buffer; NOT elements - 1 */
    .src_mode     = NORA_DMA_ADDR_FIXED,
    .dst_mode     = NORA_DMA_ADDR_INCREMENT,
    .size         = NORA_DMA_SIZE_HALFWORD,   /* 16-bit word on this port */
    .tr_mode      = NORA_DMA_TRMODE_REPEAT_ONESHOT,   /* the validated audio mode */
    .reload_count = true,
    .reload_dst   = true,
    .half_int_en  = true,
    .done_int_en  = true,                     /* parity field; see Behavior Notes */
    .trigger      = NORA_DMA_TRIGGER_SPI1_RX,
    .irq_priority_set = true,
    .irq_priority = 4u,
    .irq_enable   = true,
};

if (nora_dma_channel_config(NORA_DMA_CHANNEL_0, &rx_cfg)) {
    nora_dma_channel_enable(NORA_DMA_CHANNEL_0, true);
}
```

`.trigger` names the peripheral event; the backend owns the device trigger-select
code, so the call site does not need the trigger table from the data sheet — and a
trigger the selected device does not implement is refused at config time instead
of silently selecting a Reserved code.

In the consumer's DMA ISR, the hot-path variant folds to direct register accesses
when the channel is a compile-time constant:

```c
#include "nora_dma_dspic33ck_fast.h"   /* backend-private: exposes SFRs */

void __attribute__((__interrupt__, no_auto_psv)) _DMA0Interrupt(void)
{
    /* clear IF, read+clear status */
    nora_dma_status_t stat = nora_dma_isr_snapshot_hot(NORA_DMA_CHANNEL_0);

    switch (nora_dma_half_from_status(stat)) {
    case NORA_DMA_HALF_FIRST:  /* first half ready  */ break;
    case NORA_DMA_HALF_SECOND: /* second half ready */ break;
    default: break;
    }
}
```

A controller-only proof, with no peripheral wired to the channel:

```c
const nora_dma_channel_cfg_t st = { /* ... */ .trigger = NORA_DMA_TRIGGER_NONE };

nora_dma_channel_config(NORA_DMA_CHANNEL_3, &st);
nora_dma_channel_enable(NORA_DMA_CHANNEL_3, true);
nora_dma_software_trigger(NORA_DMA_CHANNEL_3);
while (nora_dma_request_pending(NORA_DMA_CHANNEL_3)) { }
/* single-shot: ask has_completed(), not has_completed_half() */
```

For short critical sections that must not be interrupted by a channel's DMA ISR,
save and restore its interrupt-enable:

```c
bool was = nora_dma_irq_disable_save(NORA_DMA_CHANNEL_0);
/* ... brief critical work ... */
nora_dma_irq_restore(NORA_DMA_CHANNEL_0, was);
```

`nora_dma_irq_disable_save_hot()` / `_restore_hot()` in
`nora_dma_dspic33ck_fast.h` are the inline equivalents for a hot path.

## API Summary

Global:

- `nora_dma_global_init()` — give DMA SRAM bus priority over the CPU
  (`MSTRPR.DMAPR = 1`), program the allowed address window (`DMAL` / `DMAH`), and
  enable the controller (`DMACON.DMAEN`). Safe to call more than once.
- `nora_dma_global_is_ready()` — controller on, DMA holds SRAM bus priority, and
  the address window is programmed. Side-effect-free.

Per channel:

- `nora_dma_channel_config()` — configure a channel and leave it disabled.
  Returns false (and writes no register) for a NULL config, invalid channel,
  controller not ready, an out-of-range enum / IRQ priority, `NORA_DMA_SIZE_WORD`,
  a `count` above this port's 16-bit register limit, or a trigger this device does
  not implement. Re-config safe: masks the channel's CPU IRQ and clears stale
  status / pending flag before and after programming.
- `nora_dma_channel_enable()` — set/clear `CHEN` (start/stop the channel).
- `nora_dma_software_trigger()` — request one transfer from software (`CHREQ`),
  exactly as the selected peripheral trigger would.
- `nora_dma_request_pending()` — a requested transfer is still outstanding.
- `nora_dma_read_count()` — elements still to move in the current transfer.
- `nora_dma_irq_enable()` / `nora_dma_irq_is_enabled()` — the channel's CPU
  interrupt enable, independent of `CHEN`.
- `nora_dma_irq_disable_save()` / `nora_dma_irq_restore()` — save/mask and restore
  it around a short critical section.
- `nora_dma_clear_status()` — clear the channel's interrupt-flag bits (trigger
  select and the half-done enable are preserved: on this port they share the
  register).
- `nora_dma_clear_irq_flag()` — clear the channel's CPU interrupt flag.
- `nora_dma_read_status()` — read raw channel status as a `nora_dma_status_t`.
- `nora_dma_read_src()` — read the active source address.

Status predicates (side-effect-free, take the snapshot rather than the channel):

- `nora_dma_status_has_completed_half()` — a ping-pong half boundary was reached
  (`HALFIF` or `DONEIF`).
- `nora_dma_status_has_completed()` — the transfer as a whole completed (`DONEIF`).
  **This is the single-shot question.** `has_completed_half()` is already true at
  the *midpoint* of a single-shot transfer, so spinning on it releases the caller
  while the second half is still being written.
- `nora_dma_status_has_overrun()` — `OVRUNIF`.
- `nora_dma_status_has_half_done_conflict()` — both `HALFIF` and `DONEIF` set in
  one snapshot (a missed-deadline indicator for the consumer).
- `nora_dma_half_from_status()` — interpret a snapshot as a ping-pong half
  indicator (`DONE` takes precedence over `HALF`).
- `nora_dma_isr_snapshot()` — clear the CPU flag, snapshot status, clear status;
  returns the snapshot.

The status bits are read through those accessors rather than through exported bit
masks, so `nora_dma_status_t` stays an opaque carrier.

Hot path (`nora_dma_dspic33ck_fast.h`, backend-private):

Each entry is a `static inline` shadow of the portable call above, named
`<portable stem>_hot`, and the out-of-line version in the backend `.c` is
literally a call to the inline. Call them with a compile-time-constant channel so
the switch folds to a direct SFR access.

- `nora_dma_isr_snapshot_hot()`
- `nora_dma_irq_disable_save_hot()` / `nora_dma_irq_restore_hot()`
- `nora_dma_read_src_hot()`

The `_hot` suffix sits on the portable stem on purpose. An ISR body written
against `_hot` names ports to another backend unchanged — only the `_fast.h` that
supplies the inline differs.

## Behavior Notes

- **The address window is mandatory, not a refinement.** `DMAL` and `DMAH` reset to
  `0x0000`, and a channel enabled against that window arms successfully and then
  faults on its first element. These registers are **not** MPU-style pre-access
  protection: the limit test is explicitly *not* made before the access
  (DS70005399D page 174, `DMAINTn` Note 2), so the first out-of-window access can
  occur and the transaction is terminated after it, setting `HIGHIF` and clearing
  `CHEN`. `nora_dma_global_init()` programs the window, and
  `nora_dma_global_is_ready()` treats `DMAH == 0` as *not ready* so the failure
  surfaces as a refusal at config time rather than as a stream that never starts.
- **`count` semantics.** `nora_dma_channel_cfg_t.count` is the number of elements
  (of the configured `size`) to move per transfer/repeat — it is **not** an
  "elements − 1" register. For a ping-pong buffer, pass the element count of the
  **whole ping+pong buffer**, not of one half: `HALFIF` marks the midpoint of this
  count and `DONEIF` its end, which is what makes `NORA_DMA_HALF_FIRST` /
  `_SECOND` name the halves they do. The contract is `uint32_t` in every family;
  this port's register is 16-bit, so an over-range value is *rejected*, never
  truncated.
- **`RELOAD` and the Repeated modes.** `reload_count` / `_src` / `_dst` are
  OR-collapsed onto the single `RELOAD` bit, but in the Repeated transfer modes
  `DMACNTn` is reloaded regardless of that bit (DS70005399D page 173, `DMACHn`
  Note 2). So the collapse effectively decides the **address** reload, which is
  what a ping-pong consumer needs; the count reload it also asks for is already
  hardware behaviour there.
- **`done_int_en` writes no register bit.** This port has no independent DONE
  interrupt enable — DONE always contributes to the channel interrupt when the CPU
  IE is set. The field is accepted for source parity with the dsPIC33AK port; it
  cannot mask DONE independently of HALF.
- **`NORA_DMA_SIZE_WORD` is rejected.** The 16-bit core has no 32-bit element. The
  enumerator is kept (and keeps its AK value) so shared code compiles; passing it
  makes `nora_dma_channel_config()` return false.
- **Invalid-channel convention** (channel index outside the device's inventory):
  - `config` / `enable` return `false` and write nothing.
  - `void` IRQ/status helpers silently ignore the call.
  - read helpers return `0`.
- **ISR snapshot ordering — ordered, not race-free.** `nora_dma_isr_snapshot()`
  (and its `_hot` inline) performs an *ordered* sequence (clear CPU flag, read
  status, clear status), not a single atomic instruction. The CPU flag is cleared
  first so that an event raised anywhere later in the sequence still re-asserts it
  and the ISR runs again. That is all the ordering buys: the status clear is a
  read-modify-write of the register that also holds `CHSEL` / `HALFEN`, so an event
  landing between the snapshot and the clear loses its `DMAINTn` flag too. The
  re-entered ISR therefore sees `NORA_DMA_HALF_NONE`, which the consumer must
  reject rather than map to a buffer half. Do not rely on the second event staying
  latched in the status word — only the CPU flag survives.
- **Global arbitration policy.** `nora_dma_global_init()` sets `MSTRPR.DMAPR = 1`
  for the entire device. This prioritizes DMA over CPU SRAM accesses while leaving
  SFR arbitration unchanged. An audio SPI issues a DMA request per element with
  only one pending-request slot, so a CPU-delayed grant becomes `OVRUNIF` — a
  dropped sample. It is not a blanket guarantee against every overrun cause;
  integrators must include every DMA consumer and CPU/DSP timing path in
  system-level regression tests.

## Notes

- This repository does not include Microchip DFP header files.
- This HAL is the DMA layer only; clock setup and peripheral (SPI/...) setup belong
  to the board/application and the consuming HALs.
- This is the canonical dsPIC33CK DMA HAL used by
  [nora-hal-dspic33ck-spi-i2s-tdm](https://github.com/sulaolab/nora-hal-dspic33ck-spi-i2s-tdm),
  whose public headers include `nora_dma.h`.

* Sibling repositories for this family:
  [timer](https://github.com/sulaolab/nora-hal-dspic33ck-timer) ·
  [gpio](https://github.com/sulaolab/nora-hal-dspic33ck-gpio) ·
  [clock](https://github.com/sulaolab/nora-hal-dspic33ck-clock) ·
  [i2c](https://github.com/sulaolab/nora-hal-dspic33ck-i2c) ·
  [uart](https://github.com/sulaolab/nora-hal-dspic33ck-uart) ·
  [spi-i2s-tdm](https://github.com/sulaolab/nora-hal-dspic33ck-spi-i2s-tdm)

## Source snapshot

`src/` is byte-identical to the private development tree at `0ff5854`. Re-publish from that revision or later, not from an earlier snapshot of this repository.

## License

MIT No Attribution License (MIT-0). See [LICENSE](LICENSE).

Attribution is appreciated but not required.
