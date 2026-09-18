# XTS Cam Table - minimal example

Mover 1 is the master. Mover 2 follows it through one cam table that covers one full lap
of the 1500 mm track (modulo 1500).

| Master position (Mover 1) | Section    | Mover 2 is ... behind Mover 1 |
|---------------------------|------------|-------------------------------|
| 0 .. 250                  | straight 1 | 100 mm                        |
| 250 .. 750                | curve 1    | 60 mm                         |
| 750 .. 1000               | straight 2 | 100 mm                        |
| 1000 .. 1500              | curve 2    | 60 mm                         |

The gap cannot change instantly, so the first 100 mm of every section (`TRANSITION` in MAIN)
blends from the old gap to the new one with a 5th-order polynomial. Everywhere else the table
is a straight 1:1 line (`MOTIONFUNCTYPE_POLYNOM1`).

## Where things live

- `PLC/POUs/MAIN.TcPOU` - constants (gaps, section starts, velocities), the state machine,
  and the action `BuildCamTable` that writes the 9 table points.
- `PLC/POUs/F_CamPoint.TcPOU` - builds one `MC_MotionFunctionPoint` from
  (index, master position, gap, curve type). Slave position = master - gap.
- `PLC/DUTs/E_State.TcDUT` - the steps of the sequence.
- NC > Tables > `Master 1` > `Slave 1` - cam table ID 1. It is empty in the project;
  the PLC fills it at runtime with `MC_CamTableSelect`.
- `XTS Cam Table.tsproj` - links `MAIN.Mover1` / `MAIN.Mover2` to `Mover Axis 1` / `Mover Axis 2`.

## Sequence (set the BOOLs from the online view)

1. `bStart`: power on both movers, load the table into the NC, ask the NC where the table
   wants Mover 2 for Mover 1's current position (`MC_ReadCamTableSlaveDynamics`), move
   Mover 2 there with a relative move (Mover 1 does not move, so they cannot collide),
   couple with `MC_CamIn`, then run Mover 1 at `MASTER_VELOCITY`.
2. `bStop`: halt Mover 1 (Mover 2 stops with it through the cam), then `MC_CamOut`.
3. `bReset`: from `Error`, resets both axes and goes back to `Idle`.

Watch `fGapActual` while it runs: it should sit at 100 on the straights and 60 in the curves,
with a smooth blend at the start of each section.

## Notes

- The table has a "lift": slave goes from -60 at master 0 to 1440 at master 1500.
  The NC adds 1500 per lap automatically (cyclic cam with lift, see the TF5050 docs).
- `MC_CamIn` uses `MC_STARTMODE_ABSOLUTE` with `SlaveScalingMode = MC_CAMSCALING_AUTOOFFSET`,
  so Mover 2 never jumps at coupling. If the pre-positioning was exact, the offset is a
  whole number of laps and the table gaps apply as written.
- "Behind" means Mover 2 sits at a smaller track position than Mover 1. To have Mover 2
  run ahead instead, change the sign in `F_CamPoint` (slave = master + gap).
