# PSG
Interrupt-driven sound/music player for **Durango·X** computer with optional PSG (Programmable Sound Generator) SN76489.

## Variants

You may just use the _PSG control_ task alone for automated **envelopes** and sound effects, or include the _score player_ for full **background music** in your software!

For the latter version, don't forget to include `#define SCORE` in your source code, just before `task.s` is included.

## Supplied files

- `task.s`: the interrupt task itself, assumed to be run _inside_ an exisiting ISR.
- `init.s`: _template_ for complete initialisation and setup.
- `sg-test.s`: a simple program that starts playing a simple song.

## Integration with other _Durango·X_ software

### Code generation

1. Insert the contents of `task.s` _inside_ your exisiting ISR, usually via `#include "task.s"` or similar. It will assume a couple of things:
   - All registers already saved -- the player task will affect all of them. You'll need to restore them _before_ exiting the ISR, as usual.
   - Your ISR increments the memory location `ticks` (usually at `$206`) on **every** interrupt.
1. Define the following memory addresses (**not necessarily** on _zeropage_):
   - `psg_if` for the _PSG control_ external interface. Needs **13 consecutive bytes**.
   - `sr_if` for the _score reader_ external interface, if enabled. Needs **12 consecutive bytes**.
   - `sg_local` for local storage of  **28 consecutive bytes**. _Note that if the score reader is not enabled, only the **last 12 bytes** will be used_.
1. This interrupt task only needs **two consecutive _zeropage_ bytes**, nominally at `$FC`, and only for the _score reader_; but if this address is needed elsewhere during operation, you may shift to another address by stating `#define PSG_ZP your-address` just _before_ including `task.s`, where _your-address_ is whatever zeropage location you choose for interrupt tasks.
1. As stated before, if the full _score player_ functionality is to be enabled, add `#define SCORE` _before_ `task.s`

You may need to add a _suitable path_ when referencing your copy of `task.s` for proper assembly, like `#include "../psg/task.s"` (depends on your particular file organisation) 

### Setup

The above steps will allow your code to be assembled together with the **PSG interrupt task**, but won't guarantee _reliable operation_. Thus, the following initialisation is highly recommended:

- Clear the **first 4 bytes** from `psg_if` (defined as `sg_c1l` to `sg_nc` in source code)
- Clear the **first 5 bytes** from `sg_local` (defined as `pr_cnt` to `pr_dly` in source code)
- Clear the **8th byte** from `sr_if` (defined as `sr_ena` in source code, only used for the _score player_)

Alternatively, you may just clear all of the allocated memory -- but make sure you complete the _configuration_ afterwards.

Note: this initialisation is only needed the **first time** before you enable the interrupt task, and not needed any longer _(assuming the allocated memory isn't destroyed...)_

### Configuration values

General operation of the _PSG task_ can be controlled **on-the-fly** via two parameters, but they should be set at least once _before_ the interrupt task is enabled:

- **`sg_envsp`** (located at `psg_if`+8) defines the **envelope speed**, that is, the number of _ticks_ between volume updates. _Recommended value: **16** or lower._
- **`sr_tempo`** (located at `sr_if`+10) is a _divider_ for the **base tempo**, which is _nominally_ **240/(n+1) bpm** _(actual speed is more like 234 bpm, but in most cases the difference will be negligible)_.
- **`sr_turbo`** (located at `sr_if`+11) allows proper score pitch when playing on a _v2 Durango·X in **TURBO** mode_, whenever `bit 7` is **set** (≥128). _Leave as zero (or <128) when playing at non-TURBO or v1 machine_.

## External interface

### _PSG control_

|Label|Offset from `psg_if`|Description|
|-----|--------------------|-----------|
|sg_c1l|0|channel 1 period, low-order 4 bits  `%x000llll`|
|sg_c2l|1|channel 1 period, low-order 4 bits  `%x000llll`|
|sg_c3l|2|channel 1 period, low-order 4 bits  `%x000llll`|
|sg_nc |3|_noise_ channel rate and feedback, `%xxx00frr` |
|sg_c1h|4|channel 1 period, high-order 6 bits `%00hhhhhh`|
|sg_c2h|5|channel 2 period, high-order 6 bits `%00hhhhhh`|
|sg_c3h|6|channel 3 period, high-order 6 bits `%00hhhhhh`|
|sg_nch|7|NOT USED as noise channel has no high-order bits,but _score player_ may write here|
|sg_env | 8|**Envelope speed**, see _Configuration values_ for more info   |
|sg_c1ve| 9|channel 1 **envelope/volume**, see _Score format_ for more info|
|sg_c2ve|10|channel 2 **envelope/volume**, see _Score format_ for more info|
|sg_c3ve|11|channel 3 **envelope/volume**, see _Score format_ for more info|
|sg_nve |12|_noise_ channel **envelope/volume**, see _Score format_ for more info|

### _Score reader_

|Label|Offset from `sr_if`|Description|
|-----|--------------------|-----------|
|sr_c1|0|pointer to channel 1 score|
|sr_c2|2|pointer to channel 2 score|
|sr_c3|4|pointer to channel 3 score|
|sr_nc|6|pointer to _noise_ channel score|
|sr_ena  | 8|pause/mute channels `%n321n321` (high nybble `%0` = **pause**, low nybble `%0` = **mute**)|
|sr_rst  | 9|(re)start selected channels score `%n321xxxx` (`%1` = **start**), automatically reset, affects `sr_ena`|
|sr_tempo|10|**tempo** divider, see _Configuration values_ for more info|
|sr_turbo|11|**TURBO** mode flag in _bit 7_ (`%1` = 3.5 MHz)|

## Score format

Each channel needs a _list_ (whose address should be loaded on `sr_c1/sr_c2/sr_c3/sr_nc`) of _byte-triplets_ for every note, with the following format:

1. **Note pitch**: _chromatic_ scale, values from `1` (for **A2**, 110 Hz) to `63` (for **B7**, 3906 Hz). _Notes above E7 (2604 Hz) may be noticeably out-of-tune_
1. **Note value**:  number of _ticks_ times `sr_tempo`+1, may be scaled.
   - Semibreve/Whole: `0` (nominally _256_, fitted into 8 bits)
   - Minim/Half: `128`
   - Crotchet/Quarter: `64`
   - Quaver/Eighth: `32`
   - Semiquaver/Sixteenth: `16`
   - Demisemiquaver/Thirty-second: `8`
   - Hemidemisemiquaver/Sixty-fourth: `4`
1. **Envelope/Volume**: both values integrated into an 8-bit value.
   - High nybble: _envelope_ as a **signed 4-bit** value:
     - Positive (`1...7`) is _decay_ envelope, like a piano sound.
     - Zero (`0`) is _constant_ volume, like an organ.
     - Negative (`-1...-8` in _2's complement_) is _soft attack_ envelope, a bit like a flute.
   - Low nybble: _volume_ as an **unsigned 4-bit** value (`0`=silent, `$F`=max. volume)

### Special values

**Note pitch:** `0` is _end of score_, `$FF` means _repeat_. With any of these values, the remaining two bytes are not needed.

**Note value:** add _half_ of the value for **dotted** notes. You may use non-power-of-two values as well, to get an intermediate _tempo_ value from a lower one.

**Envelope/Volume:** use `0` for **rests** (note pitch is irrelevant, although you may use `64` for it, as for _PCM playback_ (not currently supported)
