# Rollback netcode: feasibility (round 1, 2026-10-19)

Goal: replace local wireless's lockstep (every frame waits for the other player's input; the delay frames
cover the ping) with rollback (predict the other input, save the state each frame, on a wrong prediction restore
and re-simulate up to now), as Slippi does for Melee. Measured on the competitive branch (light archive, competitive
rules), Mario vs Mario CPU 9 on Battlefield, RTX 2080 SUPER, with the R1 harness (`runtime/r1_state.c`,
`SSBU_R1=N,K,P,C`: save at fight frame N, run K frames, restore, run the same K frames again with the recorded pad
values, every P frames, C cycles).

## What was measured

| Question | Result |
|---|---|
| Whole state (guest heap regions, guest .data/.bss, stacks, TLS, fighter modules' writable sections) | 163 ranges, 1.91 GB; a full copy takes ~365 ms (only as a reference: rollback copies what changes) |
| Memory changed in a fight | 1259-2137 pages (5-8.6 MB) per 8 frames; union over the cycles 9.1 MB |
| Copying that changed set | 1.2-2.4 ms (memcpy of the learned set) |
| Re-run equal to the first pass? (`tests/r1_cmp.py`, fighters' line of every frame) | with `SSBU_DETERMINISTIC=1`: yes, 3 of 4 cycles identical for every frame (the 4th crashed, below). Without it the re-run diverges |
| Pages still different after an identical re-run | 70-614: memory outside the fighters (allocator / cache state) that does not change the fight; to be classified |
| Main thread per fight frame (logic + building the render commands) | ~6 ms (`nvn_perf` cpu by thread), plus ~4 ms on TaskWorker1 in parallel |

## Blockers found

1. **Crash after a few restores**: `c00000ff` (STATUS_BAD_FUNCTION_TABLE) in `ntdll`, always the same address, on the
   host stack of one of the game's task fibers that was switched away at the save, at the 2nd-4th restore. The memory
   state is fine; restoring a fiber's host side (stack, context) is not yet reliable. Must be solved first.
2. **Audio thread**: once `nus3::audio update` crashed reading state that a restore changed under it. Rollback needs
   the audio thread held / its state kept out of the snapshot, and sounds of re-simulated frames muted or de-duplicated.
3. **Re-simulation speed**: at ~6 ms of main-thread time per frame, re-running 3 frames costs ~18 ms if the render
   commands are built for them too: too much inside one 16.7 ms frame. Re-run frames must skip rendering; the logic
   part alone has not been measured yet.

## What already exists (upstream) and helps

- `r1_state.c`: save / restore of every guest region, guest threads held at their save-point waits, fibers saved,
  verify pass, a FAST mode (only the learned set of changed pages; never reached here: it starts after 5 learning
  cycles and the runs crashed before).
- Run-ahead (graphics.ini `runahead = on`): 1 speculative frame offline, built on the same code (needs the script
  modules built with dirty-page marks, `nro_bin_eh_o2r1`, tests/r1_nro_build.py).
- Determinism: two games given the same inputs stay identical frame by frame (local wireless tests through the VPS).

## Verdict

Feasible on the memory side: what a fight changes per frame is a few MB and copies in ~2 ms, and a re-run with the
same inputs gives the same fight. Not yet feasible as is: the fiber restore crashes, the audio thread must be kept out,
and re-simulated frames must not build render commands. Next steps, in order:

1. Fix the fiber restore crash (find what the switched-away fiber's host side needs besides stack and context:
   the fiber's exception / unwind data, `g_catch` frames, TEB fiber fields).
2. Reach FAST mode (learned set) and measure its save / restore per frame.
3. Measure logic-only re-simulation: run K frames without presenting (skip the render job submission).
4. Hold the audio thread across a restore; mute sounds in re-run frames.
5. Then a local prototype: a 2-player local fight where player 2's input arrives through a fake delay and rollback
   corrects it; then the network (UDP through the room server, later direct).

## How the runs were made

`rollback/r1_run.sh <tag> <SSBU_R1> <cycles> [--env ...]` (outside the repository: machine paths): starts the game on
the light build with the R1 harness and `SSBU_DEBUG_MATCH`, walks the menus with an input sequence plus the keyboard
(`SSBU_SEQ_KEYBOARD=1`), starts the fight, waits for the cycles. Fixes made for it in this branch: `debug_mode.c` marks
"fight" for the input sequence at the game's ready-go; `nnhid.c` keeps up to 1024 input marks (128 were filled by the
script modules' `module:` marks before "fight"); `r1_state.c` also saves the script modules (`nro_bin_*`), not only
`fighters_bin`.
