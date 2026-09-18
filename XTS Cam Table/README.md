# XTS Cam Table - minimal examples

Mover 1 is the master. Mover 2 follows it through one cam table that covers one full lap
of the 1500 mm track (modulo 1500). The track is two 250 mm straights and two 500 mm
180 degree curve modules (AT2050): straight 0..250, curve 250..750, straight 750..1000,
curve 1000..1500.

`MAIN.eExample` picks which table gets built. Set it in the online view before `bStart`.

## Example 1: `ConstantGap` (9 points)

| Master position (Mover 1) | Section    | Mover 2 is ... behind Mover 1 |
|---------------------------|------------|-------------------------------|
| 0 .. 250                  | straight 1 | 100 mm                        |
| 250 .. 750                | curve 1    | 60 mm                         |
| 750 .. 1000               | straight 2 | 100 mm                        |
| 1000 .. 1500              | curve 2    | 60 mm                         |

The gap cannot change instantly, so the first 100 mm of every section (`TRANSITION` in MAIN)
blends from the old gap to the new one with a 5th-order polynomial. Everywhere else the table
is a straight 1:1 line (`MOTIONFUNCTYPE_POLYNOM1`).

## Example 2: `ClothoidBallGap` (18 points)

Each mover carries a "ball" mounted `BALL_OFFSET` (103 mm) outboard of the track. The two balls
have to stay `BALL_DISTANCE` (115 mm) apart, measured along the path the balls travel.

On a straight that is the same as a fixed 115 mm gap between the movers. In a curve the ball
path is longer than the mover path, so while Mover 1's ball is in the bend and Mover 2's is
not yet, Mover 2 has to run faster than Mover 1 to keep up, and slower again on the way out.
The AT2050 module does not start bending all at once: its curvature ramps up over a short
transition (clothoid-like), holds for the circular middle, and ramps down. That is what the
table has to follow.

`FB_XtsBallCamBuilder` computes the points from that geometry: for a master position it finds
the mover position whose ball is 115 mm back along the ball path. It writes 8 points per curve
(cubic ramp, plateau, cubic ramp, straight line at the entry, mirrored at the exit) plus one
point at each end. The knot positions are tuned for 115 mm / 103 mm and live as defaults in
the FB.

The table's master range is 180..1680 instead of 0..1500 so that no ramp straddles the
1500 -> 0 wrap of the track. The table is periodic, so the NC maps every lap onto it.

## Where things live

- `PLC/POUs/MAIN.TcPOU` - constants, the state machine, and the actions
  `BuildTable_ConstantGap` / `BuildTable_Clothoid`. `BuildCamTable` calls one of them based
  on `eExample` and describes the array to the NC.
- `PLC/POUs/F_CamPoint.TcPOU` - builds one `MC_MotionFunctionPoint` from
  (index, master position, gap, curve type) for the ConstantGap table. Slave position = master - gap.
- `PLC/POUs/FB_XtsBallCamBuilder.TcPOU` - the ClothoidBallGap geometry and point writer.
  Call `Build()` once with a pointer to the point array. `SlavePos(master)` and
  `SpeedRatio(master)` are public if you want to check the table by hand.
- `PLC/DUTs/E_Example.TcDUT` - the two examples.
- `PLC/DUTs/E_State.TcDUT` - the steps of the sequence.
- NC > Tables > `Master 1` > `Slave 1` - cam table ID 1. It is empty in the project;
  the PLC fills it at runtime with `MC_CamTableSelect`.
- `XTS Cam Table.tsproj` - links `MAIN.Mover1` / `MAIN.Mover2` to `Mover Axis 1` / `Mover Axis 2`.

Only Beckhoff standard libraries are used (Tc2_MC2, Tc2_MC2_Camming, Tc2_Standard, Tc2_System).

## Sequence (set the values from the online view)

0. `eExample`: pick the table. Only read at `bStart`.
1. `bStart`: power on both movers, build the table and load it into the NC, ask the NC where
   the table wants Mover 2 for Mover 1's current position (`MC_ReadCamTableSlaveDynamics`),
   move Mover 2 there with a relative move (Mover 1 does not move, so they cannot collide),
   couple with `MC_CamIn`, then run Mover 1 at `MASTER_VELOCITY`.
2. `bStop`: halt Mover 1 (Mover 2 stops with it through the cam), then `MC_CamOut`.
3. `bReset`: from `Error`, resets both axes and goes back to `Idle`.

Watch `fGapActual` while it runs. ConstantGap: 100 on the straights, 60 in the curves, with a
smooth blend at the start of each section. ClothoidBallGap: 115 on the straights, dipping and
rising through the curve transitions.

If the clothoid builder refuses its inputs, `nErrorID` is `16#FFFFFFFF` and `sTableError` says why.

## Notes

- Both tables have a "lift" of 1500 per lap (ConstantGap: slave -60 -> 1440, ClothoidBallGap:
  slave 65 -> 1565). The NC adds 1500 per lap automatically (cyclic cam with lift, see the
  TF5050 docs).
- `MC_CamIn` uses `MC_STARTMODE_ABSOLUTE` with `SlaveScalingMode = MC_CAMSCALING_AUTOOFFSET`,
  so Mover 2 never jumps at coupling. If the pre-positioning was exact, the offset is a
  whole number of laps and the table gaps apply as written.
- Before pre-positioning, MAIN maps Mover 1's track position into the table's master range
  (`fMasterInTable`): a position below the table start is one lap further on in the table.
  For ConstantGap that is a no-op.
- "Behind" means Mover 2 sits at a smaller track position than Mover 1. To have Mover 2
  run ahead instead, change the sign in `F_CamPoint` (slave = master + gap) or set a negative
  `BALL_DISTANCE`.
- To add another example: add a value to `E_Example`, an action `BuildTable_<name>` in MAIN
  that fills `aCamPoints`, `nCamPoints` and `fTableMasterStart`, and a line in `BuildCamTable`.
