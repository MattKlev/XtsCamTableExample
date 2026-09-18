# XTS Cam Table - minimal examples

Mover 1 is the master. Mover 2 follows it through a cam table. The track is 1500 mm long
(modulo 1500): two 250 mm straights and two 500 mm 180 degree curve modules (AT2050):
straight 0..250, curve 250..750, straight 750..1000, curve 1000..1500.

Each example is one self-contained program: its own constants, its own table builder and its own
start/stop sequence. `MAIN` only decides which one runs (`MAIN.eExample`), because they share the
two movers. Pick the example in `MAIN`, then open that program in the online view and set `bStart`.

| Example                 | Program        | NC tables | Points | What it shows                                                            |
|-------------------------|----------------|-----------|--------|--------------------------------------------------------------------------|
| 1 `ConstantGap`         | `PRG_Example1` | 1         | 9      | a fixed gap per section, blended at the section starts                   |
| 2 `ClothoidBallGap`     | `PRG_Example2` | 1         | 18     | a ball clamped between the two movers through the AT2050 clothoid transitions |
| 3 `StitchedSegments`    | `PRG_Example3` | 1         | 18     | the same table, assembled from a track layout and ONE reusable clothoid piece |
| 4 `TwoTableSwitching`   | `PRG_Example4` | 2         | 2 + 10 | a straight table and a clothoid table, the NC swaps them while running   |

## Example 1: `ConstantGap` (9 points)

| Master position (Mover 1) | Section    | Mover 2 is ... behind Mover 1 |
|---------------------------|------------|-------------------------------|
| 0 .. 250                  | straight 1 | 100 mm                        |
| 250 .. 750                | curve 1    | 60 mm                         |
| 750 .. 1000               | straight 2 | 100 mm                        |
| 1000 .. 1500              | curve 2    | 60 mm                         |

The gap cannot change instantly, so the first 100 mm of every section (`TRANSITION` in `PRG_Example1`)
blends from the old gap to the new one with a 5th-order polynomial. Everywhere else the table
is a straight 1:1 line (`MOTIONFUNCTYPE_POLYNOM1`).

## Example 2: `ClothoidBallGap` (18 points)

The two movers work like a pair of tongs: each carries a jaw mounted `BALL_OFFSET` (103 mm)
outboard of the track, and one ball is pinched between the two jaws. To keep the ball clamped,
the jaws have to stay `BALL_DISTANCE` (115 mm) apart, measured along the path the ball travels.

On a straight that is the same as a fixed 115 mm gap between the movers. In a curve the ball
path is longer than the mover path, so while Mover 1's jaw is in the bend and Mover 2's is
not yet, Mover 2 has to run faster than Mover 1 to keep the grip, and slower again on the way
out. The AT2050 module does not start bending all at once: its curvature ramps up over a short
transition (clothoid-like), holds for the circular middle, and ramps down. That is what the
table has to follow.

`FB_XtsBallCamBuilder` computes the points from that geometry: for a master position it finds
the mover position whose jaw is 115 mm back along the ball path. It writes 8 points per curve
(cubic ramp, plateau, cubic ramp, straight line at the entry, mirrored at the exit) plus one
point at each end. The knot positions are tuned for 115 mm / 103 mm and live as defaults in
the FB.

The table's master range is 180..1680 instead of 0..1500 so that no ramp straddles the
1500 -> 0 wrap of the track. The table is periodic, so the NC maps every lap onto it.

## Example 3: `StitchedSegments` (18 points, one table)

Same result as Example 2, different construction. The clothoid is tuned once and lives as ONE
piece that knows nothing about the track; the track is a list of segments.

- `PRG_Example3.BuildCurveModule` runs `FB_XtsBallCamBuilder` for a single module at the origin:
  10 points, master 0..620 measured from the module start (`CURVE_TABLE_LENGTH`), slave measured
  from the same origin with the 115 mm gap included. Point 1 is (0, -115), the 8 clothoid points
  follow, point 10 is (620, 505). Why 620 and not 500: the slave jaw trails by 115 mm, so the
  cam is still busy 103 mm after the module ends (the last exit knot sits at 603).
- `FB_XtsCamStitcher` gets the layout `Straight 70 | Curve 500 | Straight 250 | Curve 500 | Straight 180`
  (one lap starting at 180) and walks it with a running master position. At every `Curve` it
  pastes the 8 inner points, shifted by the module start. A `Straight` adds no point at all: the
  table simply continues 1:1 from the previous point. One point at each end completes the table.
- Changing a straight means changing one length in the list. The clothoid piece is untouched.
- Rules: the layout starts and ends on a straight, and every straight after a curve is longer
  than the 103 mm overhang. The stitcher reports an error otherwise ("master positions not
  increasing").

`PRG_Example3.BuildTable` also builds the Example 2 table next to it and stores the largest
difference in `fStitchMaxDeviation`. Expect about 1e-9 mm (-1 means the comparison could not be
made).

## Example 4: `TwoTableSwitching` (2 + 10 points, two tables)

Two NC tables, and the NC swaps them while the movers run:

| Table                          | NC                          | Content                                                      |
|--------------------------------|-----------------------------|--------------------------------------------------------------|
| straight, `CAM_TABLE_ID` = 1   | Tables > Master 1 > Slave 1 | 2 points, slave = master - 115, periodic over 1500           |
| curve, `CURVE_TABLE_ID` = 2    | Tables > Master 2 > Slave 1 | the 10-point clothoid piece from `PRG_Example4.BuildCurveModule`, periodic over 620 |

Neither table knows where the curves are. That is in `aSwitch`:

| Mover 1 passes                            | table that takes over |
|-------------------------------------------|-----------------------|
| 250 (curve 1 start)                       | curve                 |
| 870 (= 250 + 620)                         | straight              |
| 1000 (curve 2 start)                      | curve                 |
| 120 (= 1000 + 620 - 1500, after the wrap) | straight              |

The sequence starts as before with the straight table (build, load, read, pre-position,
`MC_CamIn`), then the state `BuildCurveTable` loads table 2. In `Run`, the action `SwitchTables`
keeps exactly one switch queued in the NC: `MC_CamIn` on an already coupled slave changes the
table, and with `Options.ActivationMode = MC_CAMACTIVATION_ATMASTERAXISPOS` the NC performs the
change when Mover 1 passes `Options.ActivationPosition`. The PLC only has to be early, not
exact. Once the NC reports `InSync`, the next switch is queued.

Why nothing jumps: both tables are 1:1 with a 115 mm gap at every switch position, and the
switch uses `MC_STARTMODE_RELATIVE` (table master 0 = Mover 1 position at the switch = the
module start) with `SlaveScalingMode = MC_CAMSCALING_AUTOOFFSET` (the NC picks the slave offset
so that Mover 2's position is continuous). Velocity is continuous to within 1 %: the piece's
16 mm lead-in line (0..16, before the first ramp knot) has a slope of 1.0106, so switching into
the curve table steps Mover 2's speed by about 3 mm/s at 300 mm/s. In the single-lap tables the
same line is 86 mm long (180..266) and its slope is 1.002.

If Mover 1 gets more than `SWITCH_MISS_TOLERANCE` (50 mm) past a queued switch position
without the NC switching, `sTableError` becomes "cam table switch missed" and the sequence goes
to `Error` rather than running the wrong table.

Watch `nActiveTableID` (1/2) and `nSwitchCount` (+4 per lap).

Two NC details were taken from the Tc2_MC2_Camming 3.4.8 interface, not from a running system.
If the first run misbehaves, check these first:

- `ActivationPosition` is given as the non-modulo axis position (`NcToPlc.SetPos` plus the
  distance to go). If the NC wants the modulo position instead, the switches fire on lap 1 only.
- `MC_STARTMODE_RELATIVE` is assumed to take the master position at activation, not at the
  command. If the curve table is visibly shifted (a speed step at 250), use
  `MC_STARTMODE_ABSOLUTE` with `MasterOffset` set to the curve start (sign to be tried).

## One table or two? Pros and cons

The question behind examples 3 and 4: if the straights change, do we want to re-make the table?

**One stitched table (Example 3)**

- Pro: the NC sees one periodic table, coupled once. Nothing to sequence at runtime, no timing
  to get right.
- Pro: short straights are no problem. If they get shorter than the 103 mm overhang the stitcher
  refuses, but `FB_XtsBallCamBuilder` over the whole lap still works because it sums the
  curvatures of overlapping modules.
- Pro: the clothoid is still tuned in isolation (one module-relative piece) and the track is
  still just a list.
- Pro: easy to check. The finished table can be compared point by point (`fStitchMaxDeviation`).
- Con: a layout change means rebuilding and reloading the table, which needs the slave decoupled.
- Con: the table grows with the number of modules (8 points per curve).

**Two tables switched online (Example 4)**

- Pro: one table per geometric feature. A straight length is a switch position, not table content.
- Pro: the point count does not depend on the track length. More module types = more tables,
  not bigger tables.
- Pro: the layout could in principle change while running (new switch positions).
- Con: the PLC has to queue every switch in time, four per lap. A missed switch means Mover 2
  runs the wrong table (the `SWITCH_MISS_TOLERANCE` check turns that into an error stop).
- Con: continuity at every boundary is your responsibility. Both tables must agree on position,
  ratio and gap at each switch. Here that holds by construction; with a different straight gap
  it would not.
- Con: the pieces must not overlap. Straights shorter than about 120 mm cannot be done this way.
- Con: the curve table is not module-aligned (0..620 for a 500 mm module) because the slave jaw trails.
- Con: more NC semantics to get right (activation position, start mode, auto offset at an online
  change), and harder to debug: "which table was active when" is one more thing to log.

Recommendation: use the stitched single table unless the track layout genuinely changes at
runtime. Example 4 is here to show how the switching works and what it costs.

## Where things live

- `PLC/POUs/MAIN.TcPOU` - the selector. Calls the program picked by `eExample`; a new pick takes
  effect once the running program is back in `Idle`.
- `PLC/POUs/PRG_Example1.TcPOU` - example 1. Constants, the state machine, and the action
  `BuildTable` (9 points by hand with `F_CamPoint`).
- `PLC/POUs/PRG_Example2.TcPOU` - example 2. Same state machine, `BuildTable` runs
  `FB_XtsBallCamBuilder` over the whole lap.
- `PLC/POUs/PRG_Example3.TcPOU` - example 3. `BuildCurveModule` makes the module-relative clothoid
  piece (`aCurvePoints`), `BuildTable` stitches it into the lap with `FB_XtsCamStitcher` and
  compares the result to the example 2 table.
- `PLC/POUs/PRG_Example4.TcPOU` - example 4. `BuildCurveModule` as in example 3, `BuildTable`
  fills the straight table, describes both tables and the switch positions, `SwitchTables` queues
  the table switches while running. Its state machine has the extra step `BuildCurveTable`.
- `PLC/GVLs/GVL_Axes.TcGVL` - `Mover1` / `Mover2`, the two `AXIS_REF`s shared by all examples.
- `PLC/POUs/F_CamPoint.TcPOU` - builds one `MC_MotionFunctionPoint` from
  (index, master position, gap, curve type). Slave position = master - gap.
- `PLC/POUs/FB_XtsBallCamBuilder.TcPOU` - the ClothoidBallGap geometry and point writer.
  Call `Build()` once with a pointer to the point array. `SlavePos(master)` and
  `SpeedRatio(master)` are public if you want to check the table by hand.
- `PLC/POUs/FB_XtsCamStitcher.TcPOU` - lays straight and curve segments end to end into one
  lap table (example 3).
- `PLC/DUTs/E_Example.TcDUT` - the four examples (what `MAIN.eExample` selects).
- `PLC/DUTs/E_State.TcDUT` - the steps of the sequence, shared by the four programs.
- `PLC/DUTs/E_SegmentKind.TcDUT`, `ST_CamSegment.TcDUT` - the layout entries for the stitcher.
- `PLC/DUTs/ST_CamSwitch.TcDUT` - one table switch (track position, table ID) for example 4.
- NC > Tables > `Master 1` > `Slave 1` - cam table ID 1. NC > Tables > `Master 2` > `Slave 1` -
  cam table ID 2 (example 4 only). Both are empty in the project; the PLC fills them at runtime
  with `MC_CamTableSelect`.
- `_Config/PLC/PLC Instance.xti` - links `GVL_Axes.Mover1` / `GVL_Axes.Mover2` to `Mover Axis 1` /
  `Mover Axis 2`.

Only Beckhoff standard libraries are used (Tc2_MC2, Tc2_MC2_Camming, Tc2_Standard, Tc2_System).

## Sequence (set the values from the online view)

0. `MAIN.eExample`: pick the example. `MAIN` switches to that program as soon as the current one
   is in `Idle`. Everything below happens in the selected `PRG_ExampleN`.
1. `bStart`: power on both movers, build the table and load it into the NC (example 4: both
   tables), ask the NC where the table wants Mover 2 for Mover 1's current position
   (`MC_ReadCamTableSlaveDynamics`), move Mover 2 there with a relative move (Mover 1 does not
   move, so they cannot collide), couple with `MC_CamIn`, then run Mover 1 at `MASTER_VELOCITY`.
2. `bStop`: halt Mover 1 (Mover 2 stops with it through the cam), then `MC_CamOut`.
3. `bReset`: from `Error`, resets both axes and goes back to `Idle`.

Watch `fGapActual` while it runs. ConstantGap: 100 on the straights, 60 in the curves, with a
smooth blend at the start of each section. ClothoidBallGap, StitchedSegments and
TwoTableSwitching: 115 on the straights, dipping and rising through the curve transitions.

If the PLC side refuses (clothoid builder, stitcher, or a missed table switch), `nErrorID` is
`16#FFFFFFFF` (`PLC_ERROR_ID`) and `sTableError` says why.

## Notes

- All single-lap tables have a "lift" of 1500 per lap (ConstantGap: slave -60 -> 1440,
  ClothoidBallGap / StitchedSegments: slave 65 -> 1565, the straight table of example 4:
  -115 -> 1385). The NC adds 1500 per lap automatically (cyclic cam with lift, see the TF5050
  docs). The curve table of example 4 has a lift of 620 over its 620 mm master range.
- `MC_CamIn` uses `MC_STARTMODE_ABSOLUTE` with `SlaveScalingMode = MC_CAMSCALING_AUTOOFFSET`,
  so Mover 2 never jumps at coupling. If the pre-positioning was exact, the offset is a
  whole number of laps and the table gaps apply as written.
- Before pre-positioning, examples 2 and 3 map Mover 1's track position into the table's master
  range (`fMasterInTable`): a position below the table start (180) is one lap further on in the
  table. The tables of examples 1 and 4 start at 0, so those programs pass the track position as it is.
- Example 4: if Mover 1 stands inside a curve window at `bStart`, the movers are coupled 115 mm
  apart on the track (straight table) and the ball distance is only correct from the first
  switch on. Harmless, nothing collides.
- "Behind" means Mover 2 sits at a smaller track position than Mover 1. To have Mover 2
  run ahead instead, change the sign in `F_CamPoint` (slave = master + gap) or set a negative
  `BALL_DISTANCE`.
- To add another example: copy the program closest to it (`PRG_Example1` for a hand-written
  table, `PRG_Example3` if you want the clothoid piece), change its `BuildTable`, add a value to
  `E_Example` and a line to the `CASE` in `MAIN`. Each program owns its constants, so nothing
  else has to change.
