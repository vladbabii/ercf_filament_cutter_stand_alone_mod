# ERCF Filament Cutter Stand Alone Mod

Based on the excellent https://github.com/kevinakasam/ERCF_Filament_Cutter

# Changes
- independent cutter
- option mod for wolfcraft 4196 blades
- requires 2 x F623ZZ flanged bearings ( 4 mm total width, 1 mm flange, 10 mm non flanged OD, 11.5 mm flange OD (doesn't really matter), 3 mm ID)
- compatible with existing cutter arms ( holes relative positions kept )

## Cutter Arm
- using wolfracft 45196 blades
- blade shim that can be printed for accurate positioning of blade

## Servo base
- multiple mounting holes

# Parts from original
Source: https://github.com/kevinakasam/ERCF_Filament_Cutter/tree/main/STL(Beta)
- Servo_Gear.stl or Servo_Gear_M3.stl
- The original arm can also be used


# Configuration of trad-rack with stand alone cutter between printhead and mmu

## Filament path

1. roll of filament
2. some pfte tube
3. trad rack
4. about 5 cm of pfte tube
5. cutter filament sensor - using  https://www.printables.com/model/1051461-ecas-inline-filament-sensor
6. some length of pfte tube
7. belay sensor
8. some length of pfte tube
9. print head

Don't forget to chamfer all the pfte tube end (do this on all ends of the tube where it's appropiate - this helps with pushing filament to hotend and pulling filament from hotend back)
1. cut the tube
2. put something conical to increase entry hole size and give it a conical input
3. cut the exterior of the tube so it fits in the connectors

## Switch to trad rack filament cutter
on klipper host (raspberry pi)
```
cd cd trad_rack_klippy_module/
git checkout git checkout wip-inline-filament-cutter
```

## Restart Klipper
Restart klipper either through ui or command line


## Cutter macros in config
I keep these in separate config file
```
[servo tr_cutter_servo]
pin: tr:P0.1
maximum_servo_angle: 170
minimum_pulse_width: 0.000700
maximum_pulse_width: 0.002200

[gcode_macro CUTTER_SETTINGS]
variable_position_precut: 170
variable_position_cut: 110
variable_position_pass: 25
variable_servo_wait_ms: 1000
gcode:
  G0

[delayed_gcode CUTTER_RESET]
initial_duration: 1
gcode:
  RESPOND PREFIX="info" MSG="Cutter Reset > Moving to pass.."
  CUTTER_PASS WAIT=0

[gcode_macro CUTTER_PRECUT]
gcode:
  RESPOND PREFIX="info" MSG="Cutter > Moving to pre-cut position"
  SET_SERVO SERVO=tr_cutter_servo ANGLE={printer['gcode_macro CUTTER_SETTINGS'].position_precut}
  {% set wait = params.WAIT|default(1)|int %}
  {% if wait == 1 %}
    G4 P{printer['gcode_macro CUTTER_SETTINGS'].servo_wait_ms}
  {% endif %}

[gcode_macro CUTTER_CUT]
gcode:
  RESPOND PREFIX="info" MSG="Cutter > Moving to cut position"
  SET_SERVO SERVO=tr_cutter_servo ANGLE={printer['gcode_macro CUTTER_SETTINGS'].position_cut}
  {% set wait = params.WAIT|default(1)|int %}
  {% if wait == 1 %}
    G4 P{printer['gcode_macro CUTTER_SETTINGS'].servo_wait_ms}
  {% endif %}

[gcode_macro CUTTER_PASS]
gcode:
  RESPOND PREFIX="info" MSG="Cutter > Moving to passthrough position"
  SET_SERVO SERVO=tr_cutter_servo ANGLE={printer['gcode_macro CUTTER_SETTINGS'].position_pass}
  {% set wait = params.WAIT|default(1)|int %}
  {% if wait == 1 %}
    G4 P{printer['gcode_macro CUTTER_SETTINGS'].servo_wait_ms}
  {% endif %}
```

## Configure trad rack

Add in [trad_rack] section the following configuration
```
[trad_rack]
.....
cutter_fil_sensor_pin: !tr:P1.25
enable_cutter: True
cutter_servo_cut_angle: 110   ## overriden below
cutter_servo_reset_angle: 25  ## overriden below
cutter_servo_wait_ms: 1000    ## handled in cutter macros
# cutter_bowden_length:
cutter_retract_length: -10
# cutter_retract_speed: 60.0
# target_cutter_homing_dist: 10.0
cut_override_gcode:
  M400  
  CUTTER_PRECUT
  FORCE_MOVE STEPPER=stepper_tr_fil_driver DISTANCE=50 VELOCITY=<filament_speed> ACCEL=1500  # push filament in
  M400
  CUTTER_PASS NOWAIT=1 ## in this case it includes CUTTER_CUT, to move to passthrough position it goes through cutting and we don't need to wait for it to finish
  M400
```

## Check sensor settings

Run QUERY_ENDSTOPS (from console or UI). If it reports triggered without any filament then remove or add "!" from pin definition then restart klipper.
