# This is a Klipper plugin for a self calibrating Z offset

This is a Klipper plugin to self calibrate the nozzle offset to the print surface on a
Voron V1/V2. There is no need for a manual Z offset or first layer calibration any more.
It is possible to change any variable in the printer from the temperature, the nozzle,
the flex plate, any modding on the print head or bed or even changing the Z endstop
position value of the klipper configuration. Any of these changes or even all of them
together do **not** affect the first layer at all.

> **NEW:** The probing repeatability is now increased dramatically by using the probing
> procedure instead of the homing procedure! But note, the offset will change slightly,
> if Z is homed again or temperatures changes - but this is as intended.

## Why this

- With the Voron V1/V2 z-endstop (the one where the tip of the nozzle clicks on a switch),
  you can exchange nozzles without adapting the offset:
  ![endstop offset](pictures/endstop-offset.png)
- Or, by using a mag-probe (or SuperPinda, but this is not probing the surface directly
  and thus needs an other offset which is not as constant as the one of a switch)
  as a Z endstop, you can exchange the flex plates without adapting the offset:
  ![probe offset](pictures/probe-offset.png)
- But, why can't you get both of it? Or even more.. ?

And this is what I did. I just combined these two probing methods to be completely
independent of any offset calibrations.

## Requirements

- A Z endstop where the tip of the nozzle drives on a switch (like the standard
  Voron V1/V2 enstop)
- A magnetic switch based probe at the print head - instead of the stock inductive probe
  ([e.g. this one from Annex](https://github.com/Annex-Engineering/Annex-Engineering_Other_Printer_Mods/tree/master/VORON_Printers/VORON_V2dot4/Afterburner%2BMagnetic_Probe_X_Carriage_Dual_MGN9))
- Both, the Z endstop and mag-probe are configured properly and homing and QGL are working.
- The `z_calibration.py` file needs to be copied to the `klipper/klippy/extras` folder.
  Klipper will then load this file if it finds the `z_calibration` configuration section.
  It does not interfere with the Moonraker's Klipper update since git ignores unknown
  files.
- It's good practise to use the probe switch as normally closed. Then, macros can detect
  if the probe is attached/released properly. The plugin is also able to detect that
  the mag-probe is attached to the print head - otherwise it will stop.
- (My previous Klipper macro for compensating the temperature based expansion of the
  Z endstop rod is **not** needed anymore.)

> **Note:** After copying the pyhton script, a full Klipper service restart is needed to
> load it!

## What it does

1. Normal homing of all axes using the Z endstop for Z - now we have a zero point in Z.
   (this is not part of this plugin)
2. Determine the height of the nozzle by probing the tip of it on the Z endstop
   (should be mor or less the homed enstop position):
   ![nozzle position](pictures/nozzle-position.png)
3. Determine the height of the mag-probe by probing the body of the switch on the
   z-endstop:
   ![switch position](pictures/switch-position.png)
4. Calculate the offset between the tip of the nozzle and the trigger point of the
   mag-probe:

   `nozzle switch offset = mag probe height - nozzle height + switch offset`

   ![switch offset](pictures/switch-offset.png)
5. Determine the height of the print surface by probing one point with the mag-probe.
6. Now, calculate the final offset:

   `probe offset = probed height - calculated nozzle switch offset`

7. Finally, the calculated offset is applied by using the `SET_GCODE_OFFSET` command
   (a previous offset is resetted before).

### Drawback

The only downside is, that the trigger point of the mag-probe cannot be probed directly.
This is why the body of the switch is clicked on the endstop. This small offset between the
body of the switch and the trigger point can be taken from the datasheet of the switch and
is hardly ever influenced in any way.

### Interference

Temperature or humindity changes are not a big deal since the switch is not affected much
by them and all values are probed in a small time period and only the releations to each
other are used. The nozzle height in step 2 can be determined some time later and even
many celsius higher in the printer, compared to the homing in step 1. That is why the
nozzle is probed again and can vary a little to the first homing position.

### Example

The output of the calibration with all determined positions looks like this
(the offset is the one which is applied as GCode offset):

```
Z-CALIBRATION: ENDSTOP=-0.300 NOZZLE=-0.300 SWITCH=6.208 PROBE=7.013 --> OFFSET=-0.170
```

The endstop value is the homed z position which is the same as the configure
Z endstop position and here even the same as the second nozzle probe.

## How to configure it

The configuration is needed to activate the plugin and to set some needed values.

```
[z_calibration]
probe_nozzle_x:
probe_nozzle_y:
#   The X and Y coordinates (in mm) for clicking the nozzle on the
#   Z endstop.
probe_switch_x:
probe_switch_y:
#   The X and Y coordinates (in mm) for clicking the probe's switch
#   on the Z endstop.
probe_bed_x:
probe_bed_y:
#   The X and Y coordinates (in mm) for probing on the print surface
#   (e.g. the center point) These coordinates will be adapted by the
#   probe's X and Y offsets.
switch_offset:
#   The trigger point offset of the used mag-probe switch.
#   This needs to be fined out manually. More on this later
#   in this section..
max_deviation: 1.0
#   The maximum allowed deviation of the calculated offset.
#   If the offset exceeds this value, it will stop!
#   The default is 1.0 mm.
samples: default from [probe] section
#   The number of times to probe each point. The probed z-values
#   will be averaged. The default is from the probe's configuration.
samples_tolerance: default from [probe] section
#   The maximum Z distance (in mm) that a sample may differ from other
#   samples. The default is from the probe's configuration.
samples_tolerance_retries: default from [probe] section
#   The number of times to retry if a sample is found that exceeds
#   samples_tolerance. The default is from the probe's configuration.
samples_result: default from [probe] section
#   The calculation method when sampling more than once - either
#   "median" or "average". The default is from the probe's configuration.
clearance: 2 * z_offset from the [probe] section
#   The distance in mm to move up before moving to the next
#   position. The default is two times the z_offset from the probe's
#   configuration.
position_min: default from [stepper_z] section.
#   Minimum valid distance (in mm) used for probing move. The
#   default is from the Z rail configuration.
speed: 50
#   The moving speed in X and Y. The default is 50 mm/s.
lift_speed: default from [probe] section
#   Speed (in mm/s) of the Z axis when lifting the probe between
#   samples and clearance moves. The default is from the probe's
#   configuration.
probing_speed: default homing_speed from [stepper_z] section.
#   The fast probing speed (in mm/s) used, when probing_first_fast
#   is activated. The default is from the Z rail configuration.
probing_second_speed: default second_homing_speed from [stepper_z] section.
#   The slower speed (in mm/s) for probing the recorded samples.
#   The default is second_homing_speed of the Z rail configuration.
probing_retract_dist: default homing_retract_dist from [stepper_z] section.
#   Distance to backoff (in mm) before probing the next sample.
#   The default is homing_retract_dist from the Z rail configuration.
probing_first_fast: false
#   If true, the first probing is done faster by the probing speed.
#   This is to get faster down and the result is not recorded as a
#   probing sample. The default is false.
```

**CAUTION: If you use a bed mesh, the coordinates for probing on the print bed must be
exaclty the reference point of the mesh since this is the zero point - otherwise, you would
get into trouble!**

The `switch_offset` is the already mentioned offset from the switch body (which is the
probed position) to the actual trigger point. A starting point can be taken from the
datasheet of the Omron switch (D2F-5: 0.5mm and SSG-5H: 0.7mm). It's good to start with
a little less depending on the squishiness you prefer for the first layer (for me, it's about
0.25 less). So, with a lower offset, the nozzle is more away from the bed!

It even doesn't matter what Z endstop position is configured in Klipper. All positions are
relative to this point - only the absolute values are different. But, it is advisable to
configure a safe value here to not crash the nozzle into the build plate by accident. The
plugin only changes the GCode offset and it's still possible to move the nozzle beyond this
offset.

My experiences about probing speeds: for the `second_homing_speed` on Z endstop, I use
2 mm/s and for the probe, I use 2-5 mm/s. Reducing the acceleration for probing may
be good too.

## How to use it

The calibration is started by using the `CALIBRATE_Z` command. There are no more parameters.
If the probe is not attached to the print head, it will abort the calibration process
(if configured normaly open). So, macros can help here to unpark and park the probe like this:

```
[gcode_macro CALIBRATE_Z]
rename_existing: BASE_CALIBRATE_Z
gcode:
    CG28
    M117 Z-Calibration..
    _SET_LOWER_STEPPER_CURRENT  # I lower the stepper current for homing and probing 
    _GET_PROBE                  # a macro for fetching the probe first
    BASE_CALIBRATE_Z
    _PARK_PROBE                 # and parking it afterwards
    _RESET_STEPPER_CURRENT      # resetting the stepper current
    M117
```

Then the `CALIBRATE_Z` command needs to be added to the `PRINT_START` macro. For this,
just replace the second z homing after QGL and nozzle cleaning with this macro. The
sequence could look like this:

1. Home all axes
2. Heat up the bed and nozzle (and chamber)
3. Get probe, make QGL, park probe
4. Purge and clean the nozzle
5. Get probe, CALIBRATE_Z, park probe
6. (Adjust Z offset if needed somehow)
6. Print intro line
7. Start printing...

I don't know why, but if you still need to adjust the offset from your Slicers start GCode,
then add this to your `PRINT_START` macro **after** the Z calibration:
```
    # Adjust the G-Code Z offset if needed
    SET_GCODE_OFFSET Z_ADJUST={params.Z_ADJUST|default(0.0)|float} MOVE=1
```
This does **not** reset to a fixed offset but adjusts it by the given value.

**NOTE: Do not home Z again after running this calibration or it needs to be executed again!**

> Happy printing with an always perfect first layer - doesn't matter what you just
> modded on your print head/bed or what nozzle and flex plate you like to use for the next
> print. It just stays the same :-)

## Dislaimer

It works perfectly for me. But, at this moment it is not widely tested. And I don't know
much about the Klipper internals. So, I had to figure it out by myself and found this as a
working way for me. If there are better/easier ways to accomplish it, please don't
hesitate to contact me!
