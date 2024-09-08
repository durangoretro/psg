# PSG
Interrupt-driven sound/music player for **Durango·X** computer with optional PSG (Programmable Sound Generator) SN76489.

## Variants

You may just use the _PSG control_ task alone for automated **envelopes** and sound effects, or include the _score reader_ for full **background music** in your software!

For the latter version, don't forget to include `#define SCORE` in your source code, just before `task.s` is included.

## Supplied files

- `task.s`: the interrupt task itself, assumed to be run _inside_ an exisiting ISR.
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

### Initialisation

