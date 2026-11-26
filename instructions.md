# Fll-BioGlow robot instructions

This guide describes the supplied `afentiko.py` exactly as written. The robot code has been preserved unchanged. Examples below belong inside a `run_N()` function in that file; they are commands for its custom mission runner, not standalone Python programs.

## Hardware and setup

Use a LEGO SPIKE Prime Hub running compatible Pybricks firmware. This program controls two drive motors and up to two attachment motors; it does not use color or distance sensors.

| Connection | Role | Configured positive motor direction |
| --- | --- | --- |
| B | Left drive motor | Counterclockwise |
| D | Right drive motor | Clockwise |
| A | Attachment A | Clockwise |
| F | Attachment F | Clockwise |

The hub is configured with `top_side=Axis.Z, front_side=Axis.X`. Mount it consistently with that orientation. Wheel diameter is `62.4` mm and axle track is `120.0` mm. Distances assume the configured wheel size and direct wheel drive. Attachment angles are motor degrees; the program does not configure attachment gear ratios or automatically find their starting positions.

1. Open your Pybricks programming environment, connect the hub, and load `afentiko.py` into a program.
2. Check motor ports and hub orientation against the table.
3. Put the robot and attachments at their intended starting positions. Keep the robot still while starting the program so the IMU can become ready.
4. Run the file. Startup waits 500 ms, then waits up to 8 seconds for IMU readiness. It proceeds even if that readiness timeout expires.
5. The display shows the selected run number. The initial selection is run 1.

This requires Pybricks on the hub; desktop Python cannot operate the robot by executing this file alone. Refer to the official [PrimeHub documentation](https://docs.pybricks.com/en/latest/hubs/primehub.html) and [Motor documentation](https://docs.pybricks.com/en/latest/pupdevices/motor.html) for the underlying APIs.

## Buttons, light, and completion

| Control or indication | Behavior in this file |
| --- | --- |
| Left button | Previous run; wraps from 0 to 7. |
| Right button | Next run; wraps from 7 to 0. |
| Center button in the menu | Resets drive distance/heading, then starts the selected run after button release. |
| Center button during a mission | Requests an abort when the runner next checks buttons; stops/coasts all motors and returns to the menu. |
| Bluetooth button | Configured as the whole-program stop button if the firmware accepts that setting. |
| Orange light | A selected run is executing. |
| Green menu light | Battery was at least 7200 mV when startup checked it. |
| Red menu light | Battery was below 7200 mV when startup checked it. Red also appears during startup. |
| Completion beep | 1000 Hz for 300 ms, followed by advancing the selection. |

The ready light is computed once at startup, not a continuous battery monitor. After a successful run the selection increments; after run 7 it returns to `first_run` (1). Run 0 completes by selecting 1. An aborted run keeps the same selection.

The script attempts to replace the default center stop button with Bluetooth, but silently ignores failure. If unsupported, firmware stop-button behavior may still apply. Button checks are cooperative, so abort is not guaranteed during every blocking firmware call or startup wait. On ordinary exit, the outer `finally` stops/coasts all motors and turns off the light; verify stop behavior on your actual firmware.

## Command reference

### Move forward or backward

```python
move(mm, speed=None, accel=None, curve=0, stop=True)
```

| Argument | Meaning |
| --- | --- |
| `mm` | Signed distance in millimeters. Positive goes forward with the intended motor mounting; negative reverses. |
| `speed` | Travel speed setting using the scale explained below, or an absolute mm/s value when its magnitude exceeds 1. |
| `accel` | Travel acceleration setting, or an absolute mm/s² value when its magnitude exceeds 1. |
| `curve` | Signed heading change in degrees distributed over the travel. Zero holds the current target heading. |
| `stop` | `True` decelerates and holds the drive motors at the end. `False` leaves them running for the next command. |

```python
move(300)                            # Forward 300 mm with defaults.
move(-150, speed=-0.6)                # Reverse 150 mm at 20% of max_speed.
move(400, speed=250, accel=600)        # Absolute 250 mm/s and 600 mm/s².
move(300, speed=0, curve=45)           # Travel with a +45-degree heading curve.
move(200, stop=False)                 # Continue into the next move.
move(200, stop=True)                  # End by holding the wheels.
```

Movement uses average wheel encoder distance and IMU heading feedback. Distances and heading targets accumulate across commands, so a small residual error can carry into the next command. `curve` follows a heading profile over wheel travel, not an exact geometric endpoint planner. Small moves within the `dist_tol` threshold may finish without moving.

`stop=False` is intended for immediately connected moves. A following `wait()` does not stop the robot; it can continue driving during that wait. An opposite-direction move or a turn has its own transition behavior. At normal mission completion, all motors are stopped/coasted even if the last command used `stop=False`.

### Turn

```python
turn(deg, rate=None, accel=None)
```

- `deg`: signed relative heading change in degrees. With the intended mounting, positive is clockwise/right and negative is counterclockwise/left.
- `rate`: turn speed setting, or absolute degrees/second when its magnitude exceeds 1.
- `accel`: turn acceleration setting, or absolute degrees/second² when its magnitude exceeds 1.

```python
turn(90)                             # Right 90 degrees with defaults.
turn(-90)                            # Left 90 degrees.
turn(45, rate=100, accel=400)          # Absolute turn rate and acceleration.
turn(90, rate=-0.3923, accel=-0.5)     # Current commonly used mission settings.
```

Turns stop/hold the wheels when finished. If the robot was driving, the turn reduces forward velocity while turning. It finishes after the heading remains inside tolerance for `hold_checks` consecutive checks, or after its timeout. The timeout does not raise an error or retry; the next mission command runs.

### Wait

```python
wait(ms)
```

Time is in milliseconds: `wait(500)` is half a second and `wait(1000)` is one second. This yields through the mission scheduler so button checks and grouped commands can continue. It does not issue a motor stop. Motors already holding keep holding, and nonblocking tools or an unfinished `stop=False` drive may continue moving.

```python
move(200)                            # Stops and holds when complete.
wait(500)                            # Stay there for half a second.
tool("A", 90)
```

### Attachments / tools

```python
tool(name, angle, speed=None, accel=None, hold=True, block=True)
```

| Argument | Meaning |
| --- | --- |
| `name` | `"A"` or `"F"`, case-insensitive. Unknown names and motors missing at startup are silently skipped. |
| `angle` | Relative motor rotation in degrees. Positive follows configured clockwise motor direction; negative reverses. |
| `speed` | Speed setting, or absolute motor degrees/second when its magnitude exceeds 1. |
| `accel` | Acceleration setting, or absolute motor degrees/second² when its magnitude exceeds 1. |
| `hold` | `True` holds the final position; `False` coasts after the commanded rotation. |
| `block` | `True` waits for the tool to finish. `False` starts it and allows the runner to continue. |

```python
tool("A", 90)                        # Relative +90 motor degrees.
tool("F", -180, speed=0, accel=-0.2)  # Reverse at 50% speed, 40% acceleration.
tool("A", 90, speed=200, accel=1000)  # Absolute motor speed and acceleration.
tool("F", 180, hold=False)           # Coast at the end.
tool("A", 180, block=False)          # Start without waiting.
move(300)                            # Can overlap that nonblocking attachment.
```

Standalone blocking tool commands hold moving drive motors before starting. Grouped commands suppress that automatic drive halt. Attachment angles are relative to their position when the motor action starts, not absolute targets. Neither `reset()` nor calibration homes attachments.

Nonblocking tools are not waited for at the end of a run: final cleanup can stop them before their angle completes. The tool jam watchdog only runs while `block=True`. Starting another command on the same attachment can replace the previous motor action; avoid overlapping commands to one motor.

## How speed and acceleration settings work

Every speed, rate, and acceleration argument passes through `frac()`:

```text
If the argument is None: use that command's configured default.
If abs(value) <= 1: result = limit * max(0.03, (value + 1) / 2).
If abs(value) > 1: result = min(abs(value), limit).
```

This is a -1 to +1 slider, not a direct percentage. A negative speed setting selects a slower positive magnitude; direction comes from `mm`, `deg`, or tool `angle`.

| Slider value | Percentage of the configured limit |
| --- | --- |
| `-1` | 3% minimum |
| `-0.8` | 10% |
| `-0.6` | 20% |
| `-0.5` | 25% |
| `-0.4` | 30% |
| `0` | 50% |
| `0.5` | 75% |
| `1` | 100% |

For a desired fraction `p`, use slider `2*p - 1` (subject to the 3% minimum). Example: 40% speed is `-0.2`; 80% is `0.6`. Values just above 1 switch to absolute units: `1` means 100%, whereas `2` means 2 units/second before other command-specific floors. `-0` equals `0`, so it means 50%, not zero. Do not use a speed argument as a stop command.

Configured limits and defaults:

| Setting | Constant | Current value | Default argument | Default requested magnitude |
| --- | --- | --- | --- | --- |
| Drive speed | `max_speed` | 545 mm/s | `default_speed=1` | 545 mm/s |
| Drive acceleration | `max_accel` | 2000 mm/s² | `default_accel=-0.4` | 600 mm/s² |
| Turn rate | `max_rate` | 520 deg/s | `default_rate=-0.3923` | 158.002 deg/s |
| Turn acceleration | `max_turn_accel` | 2000 deg/s² | `default_turn_accel=-0.5` | 500 deg/s² |
| Tool speed | `top` | Initially 1000 motor deg/s | `tool_speed=0` | Initially 500 motor deg/s |
| Tool acceleration | `max_tool_accel` | 5000 motor deg/s² | `tool_accel=-0.2` | 2000 motor deg/s² |

These are requested magnitudes. Actual drive speed is also capped by `top * mm_per_deg`; initially this is about 544.54 mm/s. Steering can scale both drive motors down when either would exceed `top`. Acceleration, braking, minimum motor speed, battery, load, and short distances also affect achieved speed. Tool speed has an additional 10 deg/s lower clamp. Calibration can change `top`, affecting drive limits and tool speeds together.

To change defaults, edit the constants near the top of `afentiko.py`. Explicit values already written in a mission override defaults. To change one movement only, use its keyword arguments.

```python
default_speed = -0.2                 # 40% of max_speed for moves omitting speed.
default_accel = -0.6                 # 20% of max_accel for moves omitting accel.
tool_speed = -0.4                    # 30% of top for tools omitting speed.
```

Changing these constants in a mission body is not the supported way to tune defaults; change their existing definitions at the top of the file.

## Sequential and simultaneous commands

Commands on separate source lines normally run in sequence:

```python
move(300)
tool("A", 90)
```

Commands on the same source line are intended to run together:

```python
move(300), tool("A", 90)
turn(45), tool("F", -180)
tool("A", 90), tool("F", -90)
```

When source-line tracing is available, the builder groups consecutive recorded commands from the same source line. The runner advances each generator in the group cooperatively and waits until all blocking commands in that group finish. They are started in order during the first scheduler pass, not at exactly the same hardware instant.

If `print_exception` or traceback line extraction is unavailable, same-line grouping falls back to separate sequential commands. Confirm it on the actual Pybricks firmware. To overlap one tool without relying on source-line grouping, use `block=False` and leave enough later mission time for completion.

Use at most one drive command (`move` or `turn`) and one command per attachment motor in a group. Concurrent drive commands share control state and write to the same motors. A same-line `wait()` runs alongside the other commands, not before or after them; it does not delay their start.

## Calibration: run 0

Select 0 with the menu, then press center. `calibrate()` performs a physical calibration routine:

1. Holds the drive motors and waits 300 ms.
2. Samples heading for 1500 ms and sets turn tolerance to 1.5 times measured variation, clamped to 0.3–1.5 degrees.
3. Tries increasing opposite wheel speeds in 5 deg/s increments, up to 200 deg/s, to estimate a minimum useful motor speed.
4. Commands opposite wheel speeds of 1000 deg/s for 600 ms, measuring a candidate maximum motor speed during the final 300 ms.
5. Updates `min_dps` to the measured estimate, clamped to 10–60 deg/s, and updates `top` if the measured candidate is at least 300 deg/s.
6. Prints `tol`, `min_dps`, and `top`, then resets drive encoder and heading targets.

This routine turns the robot; use a clear area. Results remain only in RAM until the program restarts. It does not calibrate wheel diameter, axle track, gyro scale, attachment angles, or physical starting placement. Normal runs do not automatically execute run 0 first.

## Editing and adding missions

Existing selectable programs are `run_0()` through `run_7()`. Their exact command sequences are listed at the end of this guide.

| Run | Description supported by the source |
| --- | --- |
| 0 | Calibration only. |
| 1 | Long route with comments marking the first, second, and third mission sections; uses A and F. |
| 2 | Forward route with repeated F attachment actions and reverse return. |
| 3 | Turn/approach route with A actions and reverse return. |
| 4 | Multi-step approach, A actions, and short waits. |
| 5 | Long route with turns and both attachments. See the preserved four-argument move noted below. |
| 6 | A attachment actions with forward travel, turns, and reverse travel. |
| 7 | A attachment actions, a short wait, and reverse moves. |

The source does not identify field models or physical mission names, so those are not inferred here.

1. Edit the command lines inside the chosen `run_N()` function.
2. Keep the definitions above the `programs={...}` construction.
3. For a new run, define `run_8()` and add `8:build(run_8)` to the existing `programs` dictionary.
4. Keep program numbers contiguous starting at 0; menu navigation increments/decrements integers and does not skip missing keys.
5. Keep `first_run` set to a valid key. It is assigned twice in this file, including near the bottom; change both existing assignments when changing the initial selection.
6. Reload/restart the whole file after editing so `programs` is rebuilt.

The builder executes a mission function repeatedly while recording commands, using an internal `Probe` exception. Keep mission definitions as simple command sequences. Ordinary side effects may run multiple times during building. Do not catch broad exceptions around mission command calls, depend on live sensor/motor outcomes inside mission-building conditionals, or put arbitrary hardware operations in mission bodies expecting them to execute only at runtime.

Existing line in `run_5()`:

```python
move(175, -0.6, -0,2), tool("A",200, 0, 0)
```

Python reads `-0,2` as two arguments, so this means `speed=-0.6`, `accel=0`, and `curve=2`. It is valid syntax, not decimal `-0.2`. The submitted code is unchanged. If the intended acceleration was `-0.2` with no curve, the corresponding replacement would be:

```python
move(175, speed=-0.6, accel=-0.2), tool("A", 200, speed=0, accel=0)
```

That replacement is explanatory only and has not been applied.

## Tuning reference

| Constant | Current value | Purpose |
| --- | --- | --- |
| `wheel_diameter` | 62.4 mm | Converts motor angle to travel distance. |
| `axle_track` | 120.0 mm | Wheel spacing used for steering conversion. |
| `gyro_scale` | 1.0 | Multiplier applied to measured heading. |
| `drive_pid` | `(5.0, 6.0, 0.15)` | Heading correction gains while moving. |
| `turn_pid` | `(8.0, 3.0, 0.3)` | Heading correction gains while turning. |
| `i_zone` | 5 degrees | Integral accumulation is limited to errors inside this zone. |
| `i_limit` | 10.0 | Absolute cap on accumulated integral error. |
| `tol` | 0.4 degrees initially | Turn completion tolerance; calibration can overwrite it. |
| `min_dps` | 14 motor deg/s initially | Low-speed floor used by drive/turn control; calibration can overwrite it. |
| `dist_tol` | 1.2 mm | Remaining-distance threshold for completing a move. |
| `hold_checks` | 3 | Consecutive in-tolerance checks required for turning. |
| `stall_ms` | 600 ms | Move no-progress watchdog, using encoder movement over 2 mm to reset it. |
| `tool_stuck_ms` | 2000 ms | Blocking-tool no-progress watchdog, using motor movement over 2 degrees to reset it. |
| `loop_ms` | 5 ms | Requested scheduler wait interval; actual control period includes work time. |
| `low_voltage_mv` | 7200 mV | Startup battery indicator threshold. |
| `first_run` | 1 | Initial menu selection and selection after the last run; defined twice. |

Tune on the actual robot and surface. This upload does not retune the controller or claim measured speed, accuracy, or mission success.

## Stops, faults, and limitations

- A move that makes insufficient encoder progress for `stall_ms` resets the accumulated distance target to the observed position, holds the wheels, and continues to the next command. It does not report mission failure. Slow moves can also trigger this watchdog if they do not exceed its progress threshold in time.
- A turn timeout holds the wheels and continues; it does not reset the requested heading target, so a later command can still try to correct remaining heading error.
- A blocking attachment that makes insufficient progress for `tool_stuck_ms` is held, then the mission continues. That watchdog does not run for `block=False`.
- Missing A/F motors and unrecognized tool names are silently skipped. B/D drive motor initialization is mandatory and can fail before the menu starts.
- Some motor control configuration calls ignore exceptions and keep existing firmware settings.
- A normal run ending or center abort calls `halt(False)`, which coasts/stops drive and attachment motors. It does not keep a tool holding after the run ends, even if its command used `hold=True`.
- Unexpected runtime errors leave the menu and reach the outer cleanup instead of being converted into a recoverable menu message. Initialization errors before the final `try` are outside that cleanup block.
- There is no obstacle avoidance, field recognition, attachment homing, live battery lockout, or persistent calibration storage in this file.

## Verification status

The uploaded Python source is preserved from the supplied text. Desktop syntax/recording checks, when reported with the upload, do not prove Pybricks firmware compatibility, motor direction, traceback grouping, stop-button behavior, calibration accuracy, or mission success. Those require a real hub and robot run.

## Exact existing mission command sequences

The following section is extracted directly from the supplied mission definitions. Distances are mm, turns and attachment rotations are degrees, and waits are ms. Use the reference above to interpret optional positional arguments.

### Run 0

```python
def run_0():
    calibrate()
```

### Run 1

```python
def run_1():
    move(420,.1009)
    move(-200,1)
    ## end of first mission
    turn(90,-.3923,-.5)
    move(200,.4679)
    turn(-92,-.3923,-.5)
    move(440,.8349), tool("F",-400,0,-.2)
    turn(-33,-.3923,-.5)
    move(200,-.633)
    tool("F",270,0,-.2)
    move(-190,.8349)
    turn(37,-.3923,-.5)
    ## end of second mission
    tool("A",-220,-.8,-.2)
    move(117,-.633)
    tool("A",90,-.8,-.2)
    move(170,1)
    tool("A",-15,-.8,-.2)
    move(-200,1)
    ##turn(3,-.3923,-.5)
    tool("A",-50,0,-.2)
    move(90,-.633)
    tool("A",90,-.8,-.2)
    move(120,1)
    tool("A",-20,-.8,-.2)
    wait(500)
    move(10,1)
    ## end of third mission
    move(-100,1,1)
    tool("A",90,1,1)
    move(-835,1,1)
```

### Run 2

```python
def run_2():
    move(710,.5)
    tool("F",170,-.6,-.2)
    tool("F",-100, 1, 1)
    tool("F",100, 1, 1)
    tool("F",-90)
    move(-100,1, 1)
    move(-600,1, 1), tool("F",250,1,1)
```

### Run 3

```python
def run_3():
    move(110,1)
    turn(50,-.3923,-.5), tool("A",310,0,-.2)
    move(315,-.2661)
    tool("A",-80, 0.2)
    move(-10,1)
    tool("A",-60, 1)
    wait(20)
    move(-400,1)
```

### Run 4

```python
def run_4():
    move(620,-.2661)
    turn(90,-.3923,-.5)
    move(275,1)
    tool("A",290,0,-.2)
    turn(-94,-.3923,-.5)
    move(130,.1009)
    wait(30)
    tool("A",-60,0,-.2)
    move(120,.1009)
    move(-80,1)
    move(100,1)
    move(-20,1)
    tool("A",-80,0,-.2)
    move(-80,-.633)
    turn(10,-.3923,-.5)
    tool("A",-70,0,-.2)
    wait(30)
    move(-25,1)
    wait(30)
    move(-120,.1009)
    tool("A",-90,0,-.2)
    move(-23,1)
    move(120,1)
```

### Run 5

```python
def run_5():
    move(260)
    turn(90,-.1,-.1)
    move(450)
    turn(-20,-.1,-.1)
    move(100)
    turn(20,-.1,-.1)
    move(235)
    tool("F", 400, 0.9, 0.9)
    tool("F", -380)
    move(95)
    turn(-70)
    move(400)
    turn(20)
    move(175, -0.6, -0,2), tool("A",200, 0, 0)
    tool("A",-200, 1, 1)
    move(110, -0.6, -0.2)
    move(-90)
    turn(-96), tool("F",280)
    move(-400, 1, 1)
    turn(45, 1, 1)
    move(-500, 1, 1)
```

### Run 6

```python
def run_6():
    move(610), tool("A", 400)
    turn(31)
    tool("A",-90)
    move(10)
    turn(3)
    tool("A", -160)
    turn(-30, 100)
    move(-500), tool("A", 300)
```

### Run 7

```python
def run_7():
    move(420), tool("A", -35)
    tool("A", -120)
    wait(50)
    move(-200, .1, .1)
    turn(5)
    tool("A", 70)
    move(-300), tool("A", -100)
```
