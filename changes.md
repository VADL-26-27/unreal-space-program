# Rocket launch fixes

Changes made on the `test/udp-connection` branch to get the rocket launching properly.

**Status:** working. The rocket now launches and flies on screen, confirmed in Play. The logs show burnout, apogee at about 3,920 ft and the drogue deploying. Main parachute and landing have not been seen in the logs yet.

## Main problem: JSBSim headers did not match the library

The rocket stayed on the pad while the logs showed it climbing.

`JSBSim.dll` and `JSBSim.lib` had been rebuilt on Oct 13 from `C:\dev\jsbsim`, but the plugin was still compiling against older headers. The new library adds a field (`bool enabled`) to the `FGModel` base class, so the plugin read values stored after it one slot off. The sim itself calculated the flight correctly, but the plugin received wrong values:

| Symptom | Cause |
|---|---|
| Rocket stuck on the pad | The position was misread as about 10,000 km away, so the rail code pulled the rocket back to the pad every frame |
| Vertical speed far too low | The plugin appears to have been reading east-west speed instead |
| Fuel Freeze on by itself, so the motor never burned out | The plugin most likely read a neighbouring value and sent it back to the sim |
| Editor crashed when Play was stopped (for example with Esc) | Probably the same mismatch. Windows logged a `0xc0000409` crash in `ucrtbase.dll` on the four Nov 5 runs before the fix, and the editor survived both stops after it. The exact mechanism was not traced. |

**Fix:** copied the 22 out-of-date headers from `C:\dev\jsbsim\src` to `Plugins/JSBSimFlightDynamicsModel/Source/ThirdParty/JSBSim/Include`.

**Avoiding a repeat:** the library files are ignored by git but the headers are tracked, so the two can drift apart without any warning. Whenever JSBSim is rebuilt, update the headers at the same time.

## Other fixes

### `JSBSimMovementComponent` (`.cpp` and `.h`)

- **Crash check:** now uses height above ground only. It used to treat a below-sea-level altitude as a crash, which could trigger on the first frame.
- **Ground check (`GetAGLevel`):**
  - Traces down from just above the rocket first, so surfaces overhead are not mistaken for ground.
  - Accepts a hit from higher up only if it is within `MaxGroundRiseMeters` (new setting, default 1 m) of the last good ground height.
  - Falls back to the last good ground height when nothing is found, instead of going straight to sea level.
  - Rejects invalid (NaN) query positions, and logs a warning when a hit is ignored or the ray finds nothing.
- **Pre-ignition hold:** new `bHoldSimulation` flag stops the sim from stepping until ignition.
- **NaN protection:** the sim now checks for NaN/Inf values after each step. If it finds any, it logs the details and stops, which leaves the actor at its last good position. The cause of the earlier NaN errors is still unknown, and the header mismatch may turn out to be responsible.

### `RocketFlightController.cpp`

- Holds the sim at `BeginPlay` and on `ResetFlight`, and releases it at ignition.

### `RocketPawn` (`.cpp` and `.h`)

- `AGLThresholdMeters` is now set to 2 m.
- `SampleAirflow` skips NaN air velocity values.

### `Map_Rocket.umap`

Edited in the Unreal editor:

- GeoReferencingSystem **Origin Altitude** set to about 138.29, because an earlier altitude fix had been lost.
- **Fuel Freeze** unticked.
- Ignition delay is about 10 seconds.

## Test results and open items

Confirmed after the fix:

- The actor follows the sim and comes off the rail (`actorZ=68433`, `offRail=1`).
- Vertical speed matches the altitude change, and liftoff is detected 0.35 s after ignition.
- The motor burns out, apogee is detected at 3,921.6 ft, and the rocket descends under drogue at about 70 ft/s.
- No NaN errors or false crashes appeared in these two runs.
- The editor did not crash when Play was stopped, on both runs.

Still open:

- **Starting height:** still wrong, and not caused by the header mismatch. `PrepareJSBSim` starts the sim before the CG position is known, so the sim begins about 1.5 m too high. The rail hold hides this until ignition, when the rocket should visibly jump up.

