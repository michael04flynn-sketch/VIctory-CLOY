# Intel Arc B390 and an Intel mobile processor

Target: an integrated Intel Arc B390 (Xe3, Panther Lake class) on a mobile
Intel Core Ultra processor with hybrid cores (performance, efficiency and low-power
efficiency cores). The code was written and measured on one AMD machine (see
[findings.md](findings.md)). This note says what that means for Intel, what
this change does, and what still needs a run on the hardware. Nothing here has
been measured on Intel yet.

## What the code assumes today

| Area | Where | On Intel today |
|---|---|---|
| Indirect dispatch group cap (65,535 vs 2^20) | `indirect.rs` `widest()` | **65,535.** The 2^20 cap is only on for AMD (`VendorId == 0x1002`). See the first item below. |
| RDNA float rules (`VICTORY_RDNA_FLOAT`) | `host.rs` `set_host_rdna` | Off. Only on for waves of 32/64, which is AMD. Intel gets the generic float path, which is the default. |
| Widest wave | `host.rs` `set_host_max_wave` | Taken from the driver. Intel Xe2/Xe3 are expected to report 16 to 32 lanes; console wave64 programs are then translated as two 32-lane halves. Check the `host wave lanes` log line. |
| Fast/efficiency core pinning | `cores.rs` | Generic: uses the system's efficiency classes, so it applies to Intel hybrid cores as it does to Alder Lake. |
| Thread counts | `list/thin.rs` (8), `shadow.rs` `wide_threads` (cores/4, 2 to 6), `pipeline/make.rs` | Fixed or core-count based; not tuned for a mobile package's power limit. |
| Memory model | everywhere | Assumes a discrete card: uploads and copies into the card's memory. On an iGPU that memory is the system's own (see below). |

## What this change does

- **`crates/victory-gpu/src/hardware.rs`** logs the host adapter once at start:
  vendor, device id, name, UMA, tile-based. Look for the `host adapter` line in a
  run's log. On integrated graphics it also sets the memory defaults below.
- **`.cargo/config.toml`** builds Windows code for `x86-64-v3` (AVX2, FMA, BMI2,
  and the rest of that level). Intel Core Ultra and the AMD machine both have it.
  Float results do not change. It is not measured yet.
- `indirect::amd()` now compares against the shared `VENDOR_AMD` constant. Same
  behaviour.
- **Release profile** uses `lto = "thin"` and `codegen-units = 1`. Expected to
  be faster at run time, at the cost of longer release and diagnostics builds.
  Not measured.
- **In-place memory cap** (`VICTORY_INPLACE_MB`) defaults to 4 GiB, or an eighth
  of installed RAM where that is less. A 16 GB laptop now locks at most 2 GiB;
  machines with 32 GB or more are unchanged. Set the variable to override.
- **Shared memory, integrated graphics only** (the adapter reports UMA):
  - `VICTORY_INPLACE` defaults to 1. Windows of the title's private memory are
    placed for the GPU instead of copied, so the bytes are kept once instead of
    twice. A window the driver refuses still falls back to a copy. Upstream
    confirmed the approach with a probe only; it has not been run as a whole
    game on any machine. **This is the change most likely to show up as a
    visual or crash problem.** Revert with `VICTORY_INPLACE=0`.
  - `VICTORY_TEXTURE_ROOM_PERCENT` was 70 instead of 90 here for a day and is
    90 again: on the Arc B390 the card's budget is a reserved 8 GB, not the
    PC's free memory, and at 70 % textures waited with a third of it in use
    (see the run below).
  - Caches that keep memory empty for reuse are sized from installed RAM (32 GB
    as the example). Each keeps the smaller of its old limit and its share:
    - scratch chunks for command lists: 1/128 of RAM, 256 MB (was 1 GB);
    - idle command lists: one per 128 MB of RAM, 256 (was 512).

    The cost is re-creating those objects more often, which shows up as
    stutter when a scene changes quickly. Each limit is a guess and is not
    measured. Revert by setting the limits back in the code, or measure first.
  - Still large and not touched here: upload memory (about 4 GB in the notes)
    and the title's own memory (40 to 46 GB in total). Run with
    `VICTORY_MEMORY_MAP=1` to see the breakdown on the device.
  - The rest of the shared-memory path (one copy instead of two) is staged and
    gated in [shared-memory-plan.md](shared-memory-plan.md). Only stage 1 is done.

## What it does not do, and why

**1. Indirect dispatches above 65,535 groups on Intel (check this first).**
The cap was raised to 2^20 for AMD to fix patches that went blocky, because the
title dispatches more than 65,535 tile groups in one indirect call. On Intel the
cap stays at 65,535, so those dispatches are cut short. That may show as blocky
or missing patches on the Arc. Whether Intel's driver honours a larger count
is not known. Test it with the 2^20 cap on Intel, the same way as on AMD, and
compare a captured frame. Do not change the default until that is measured.

**2. Memory is shared on an iGPU (the largest gain available, not started).**
The adapter reports `UMA = true` on integrated graphics. Then the card and the
processor use the same physical memory, and the copies into card-only memory
(texture uploads, list buffers, descriptor mirrors) are work that is not needed.
Using host-visible heaps for that memory would remove copies and their waits. It
touches most of `victory-gpu`'s memory code, and it cannot be checked without the
machine, so it is left as the next step. The `hardware` log tells you whether
the adapter is UMA.

**3. Power.** On a mobile part the CPU and the Arc share one power budget. Busy
spin loops take power the GPU could use: `shadow.rs` (`spin_loop`), `watch.rs`
(`spin_loop`), and `indirect.rs` (test-only). Measure how often they run on
the laptop before changing them. `VICTORY_PROCESS_PRIORITY=normal` is worth a
test on battery as well as on mains.

**4. Thread counts for a hybrid mobile package.** The fixed defaults (eight list
threads, eight join parts) assume the desktop split of fast and slow cores. Try
`VICTORY_THIN_THREADS` and `VICTORY_JOIN_PARTS` at 4, 6 and 8 on the device.

Where our threads run is settled for such a processor (`cores.rs`,
`helper_sets`): with fewer than eight fast processors, only the threads a frame
waits for one by one are kept on them. The threads that share a frame's work
may run on the next class of cores too. Before, a dozen threads and more were
asked onto four fast cores while twelve others stood by. `VICTORY_FAST_CORES=0`
asks for nothing, to compare. Not measured.

## What this repository's build said on the device (2026-11-26)

From `victory warm` on the Arc B390 PC, the first output of this repository
there:

- `host adapter name=Intel(R) Arc(TM) B390 GPU vendor=0x8086 device_id=0xb080
  uma=true`: the card shares the PC's memory, so the defaults for integrated
  graphics are in force.
- `host wave lanes min=16 max=32`: wave64 programs run as two waves of 32.
- `host view heaps hold 1000000 views`, `raw_twins=false`: the driver caps a
  view heap at a million and refuses the heap of raw twins.
- `10 processes`: two thirds of 16 logical processors.

## The first run on such a PC (2026-11-26)

One run on a PC with 32 GB shared between an integrated Intel card and the
processor, of another fork's build (codec-d/cool_fox), with the Series X
profile: about 2 frames a second. Its log says why:

- `host memory pressure` at the third frame with `physical_free_mib=130`: the
  PC was out of RAM before the world loaded, so Windows was paging.
- `textures picked by index: the card has no room for more; they wait`: the
  card's budget, which is the same RAM, was full too.
- `no wave32 lowering for this wave64 program`: one compute program (hash
  `02384a60fda6f2d0`) did not translate for a card whose widest wave is 32, and
  its draws were dropped. That is the other fork's translator. This one tries
  a wave64 program in several more forms on such a card (`emit/mod.rs`: without
  early-outs, without skips, each host wave as its own), so the program may
  translate here. A run of this repository says: look for `shader does not
  translate` in its log, and run once with `VICTORY_WAVE_WHY=1` if it is there.
- `host view heap capped at 1000000`: the title asks for 1,245,184 views and
  the driver gives a million. Not a fault by itself: cards that cap there
  (NVIDIA) run the same way, the title's highest slot in use is about 720,000,
  and 737,000 title slots are left after the dynamic ones
  (`descriptor/host.rs`, `DYNAMIC`).

One more thing the smaller picture avoids: the title's tile lists are indirect
dispatches of one group per 8x4 tile, and on a card that is not AMD they are cut
at 65,535 groups (`indirect.rs`, `widest`). At 2560x1440 that is 115,200 tiles
and the lists are cut; at 1280x720 it is 28,800 and they are not.

What this repository does about the first two, by default on a PC with 32 GiB
of RAM or less:

- The console is Series S unless `--console` says otherwise (less title
  memory, 1080p output).
- The memory budgets of a smaller PC are on (`residency.rs`;
  `VICTORY_RESIDENCY=0` turns them off).

**Keep 1920x1080 output when using bundled FidelityFX on Series S.** Its tables
expect that output size. The title normally renders at 1280x720 internally.
The loader skips other requested output sizes unless you select the title's
own upscaler with `VICTORY_UPSCALER=6`. Series X also accepts the other
supported sizes, at the cost of that profile's memory.

With the Series S profile and `VICTORY_OUTPUT=1280x720`, the reporter in
[#307](https://github.com/avatarmanz/VIctory-7900XTX/issues/307) switched to
`VICTORY_UPSCALER=6`, reached the cell and opened the pause menu on an RX 7900 XT
with RADV 26.3.1 and Wine 11.22. The Windows commenter reproduced the loading
exit but did not test mode 6 at 720p.

`-Ports` turns on the five renderer patches of the README together (57-75 ms to
about 45 ms a frame where they were measured, an AMD card). They are off by
default while they are tested, and none has run on an Intel card. Run
without them first, then add `-Ports`, so a fault can be told apart.

The README's own figures say the Series S profile still held about 30 GB of
private memory and 10.5 GB of card memory on a 64 GB PC with a card of its
own. On 32 GB shared that is still more than there is. Expect paging until the
figures of a run of this repository say otherwise.

## The first run of this repository on the Arc B390 (2026-11-26)

A `diag` build from before the core change above, Series S profile chosen by
memory, started with `-Output 1280x720`:

- Menus and loading held 30.0 frames a second (the title's own cap) with the
  card 2 to 5 % busy.
- **It crashed on entering the world**: divide by zero at title+0x28f03e8,
  after the title said `FSR4 configuration mismatch: maxUpscaleSize must be
  1920x1080; received 1280x720`. The loader's `output_fits` check now skips
  that 720p request when using bundled FidelityFX on Series S.
- `textures picked by index: the card has no room for more; they wait` with
  2,636 MB of the card's 7,424 MB budget in use. The 70 % texture share and the
  smaller count of empty texture blocks kept, both set for integrated cards a
  day earlier, are taken back. Whether the wait is gone is not measured; the
  3,072 MB kept for render targets (`targets_room_kept`) is the next suspect.
- The PC shows 23.4 GiB of RAM, because 8 GB is reserved for the card. It had
  about 1 GB free and the title's process held 8.2 GB at the crash.
- `host pipeline refused (0x80070057)` for the compute program
  `02384a60fda6f2d0`: it translates here, but the driver refuses the pipeline
  and its dispatches are dropped. Open. A run with `VICTORY_WAVE_WHY=1` says
  more.
- `new card memory waited for ... the longest 858 ms`: a hitch each time a new
  block of card memory is mapped.

No frame time in the world is known yet.

## The second run: the world, about 3 frames a second, then the card went (2026-11-26)

The same PC, Series S profile, no `-Output`. The title reached the world.

- **About 3 frames a second** (301 to 319 ms a frame) through the first
  minute of the world, with the card 14 to 42 % busy. The pipeline cache was
  cold: every frame used its whole budget of waiting for pipelines
  (`500.8 ms waiting for pipelines`, `VICTORY_PIPELINE_WAIT_MS`, 500), 700 to
  2,800 draws a frame were dropped as `being made`, and the process ran 3.5 to
  3.9 s of CPU time a frame on all threads. Single compute pipelines took 8 to
  12 s to make (the title's FSR4 passes among them), ten at once. What the
  frame is with the pipelines made is not known. (This said `victory warm
  --used` makes them ahead. It could not: see the third run.)
- **The card stopped answering after about a minute** (`0x887A0006`, device
  hung), while the submitting thread had been preparing one submission for
  4.4 s. The first command not completed is command 13 of a list that used
  `[compute 2c3592e9bbad543f]`. Open: a run with `VICTORY_D3D12_DEBUG=1` names
  the command.
- Three compute pipelines are refused by the driver now (`02384a60fda6f2d0`,
  `2da4beb386509b77`, `369cbe906ea72764`), one dispatch a frame each dropped.
- `the card has no room for more; they wait` came back at 5.2 GB of the 7.4 GB
  budget. The cause is the 3,072 MB kept for render targets, a Series X
  figure: this run had 1,123 MB of them. The Series S profile keeps 1,536 MB
  now (`VICTORY_CARD_ROOM_FOR_TARGETS_MB`).
- Memory: 16.5 GB private, 2 to 3.5 GB of the PC's RAM free, 348 MB of title
  memory in place (73 windows, none refused by the driver).

## How to measure on the device

Build as the README says (`cargo build --locked --profile diag -p victory --features diagnostics`),
then run once and read the log:

```powershell
scripts\play.ps1 -Exe <your copy's executable>
```

From the log, check:

- `host adapter`: `intel = true`, `uma = true`, the device name.
- `host wave lanes`: the min and max the driver reports.
- `the thread is kept on the fast cores`: appears when the processor has two kinds of core.

Then compare the frame line (see [findings.md](findings.md)) at the same place in
the prologue: the default, `VICTORY_INDIRECT_WIDE=0`, and `VICTORY_THIN=1` (the
switch is off unless set to 1; `play.ps1` sets it unless `-Safe`). Report the
numbers with the adapter's name and driver version.

## Status

- Compiles for the Windows target (`cargo check` and clippy for `victory-gpu`).
  Clippy with `-D warnings` already fails on code this change does not touch, on
  the unmodified tree too; that is outside this change.
- Unit tests for `hardware.rs` are written; they have not been run, because they
  need Windows and the device.
- Not run on Intel hardware. None of the numbers above are Intel numbers.

## The third run: the card went three frames into the world (2026-11-26)

The same PC and settings, ten minutes later. Not fixed: what follows is what
the log shows and what was changed for the next run.

- **The same device removal** (`0x887A0006`, device hung) at about frame
  1068, three frames after the world began. The first command not completed
  is again an op 8 directly after an op 15: a texture copy after a barrier
  (command 142 of 198 of `victory host list 43`; in the second run command 13
  of list 961). Nothing of ours was being submitted for 4.7 s before. Which
  texture is not known. The report names the operations now; the copy itself
  needs a run with the debug layer: `play.cmd -Set VICTORY_D3D12_DEBUG=1`.
- **The PC was out of memory**: `the PC has 98 MB free`, and of the process's
  14,420 MB private only 6,341 MB were in memory, on a PC that shows 23.4 GiB.
  Frames took 1.7 s, the title waited 3.1 s at presents, and the pipeline
  library's mapped file read as zeros in 7 pages. Who held the other 17 GB is
  not in the log. Below 1 GiB free the memory line now names the eight
  processes holding the most. A card that waits for memory to be paged in for
  longer than Windows' 2 s (`TdrDelay`) is reset, which reads exactly like
  this removal; that this is the cause is a guess, not a finding.
- **`victory warm --used` found nothing** (`0 pipelines known`) after two
  runs that drew the world. A run writes its pipelines down only with
  `VICTORY_PIPELINE_BOOK=1`, and the launcher did not set it. It does now; the
  message of `warm` says so.
- **`play.cmd -More @{...}` cannot work**: `play.cmd` starts the script with
  `powershell -File`, where a table arrives as text. `-Set NAME=VALUE,...` is
  the form for it.
- **Waiting for pipelines**: a pipeline is waited for during one budget
  (500 ms) from its first use on, no longer in every frame until it is made
  (`VICTORY_PIPELINE_WAIT_ONCE=0` for the old way). The compute pipelines that
  take 8 to 17 s here took the whole budget of every frame of those seconds.
  Not measured on this PC yet.
