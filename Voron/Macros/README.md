# PRINT_START macro

The below macro will perform a G32 gcode (see `[gcode_macro G32]` below for exact tasks) and then print a line at the front of the bed along the X axis. PRIME_LINE command has been commented-out as it already includes a prime-line.

```
[gcode_macro PRINT_START]
#   Use PRINT_START for the slicer starting script - please customise for your slicer of choice
gcode:
    G32                            ; home all axes
    G1 Z20 F3000                   ; move nozzle away from bed 
#    PRIME_LINE
    G90                           ; go to absolute mode. When cancelled prints, next print was failing with "move out of range". Probably print cancelling leaves the printer in relative mode. Issue doesn't appear when printer is freshly booted
    G0 X250 Y5 Z0.28 F9000
    G92 E0                        ; Reset extruder
    G1 E7 F600
    G1 X50 E30 F1500              ; Move to X50 while extruding 30mm of filament with 1500 feedrate
    G92 E0
    G1 Z2.0 F3000                 ; Move Z Axis up
    G1 E-0.5 F3600
```

# PRINT_END macro

```
[gcode_macro PRINT_END]
#   Use PRINT_END for the slicer ending script - please customise for your slicer of choice
gcode:
    M400                           ; wait for buffer to clear
    G92 E0                         ; zero the extruder
    G1 E-10.0 F3600                ; retract filament
    G91                            ; relative positioning
    G0 Z1.00 X20.0 Y20.0 F20000    ; move nozzle to remove stringing
    TURN_OFF_HEATERS
    M107                           ; turn off fan
    G1 Z2 F3000                    ; move nozzle up 2mm
    G90                            ; absolute positioning
    G0  X125 Y250 F3600            ; park nozzle at rear
    #BED_MESH_CLEAR
```

# G32 macro

The below macro performs **conditional** `QUAD_GANTRY_LEVEL` and `BED_MESH_CALIBRATE`. It will first check if they have already been performed. If they have, it skips them. If not, it executes them.

The goal here is to perform QGL and BED_MESH_CALIBRATE only the first time we open the printer and re-use the same settings during the whole session.

In my setup, I do **not** store/load the BED_MESH so when the printer starts, `printer.bed_mesh.profile_name` has no value. When a `BED_MESH_CALIBRATE` is executed, it is stored temporarily usually  with name `default` but it will be deleted on the next printer shutdown. That's what the below code checks to determine whether a bed mesh is in place or not.


```
[gcode_macro G32]
gcode:
    {% if printer.toolhead.homed_axes != "xyz" %}
        #{action_respond_info("printer not home. Homing...")}
        G28
    {% endif %}

    {% if printer.quad_gantry_level.applied == 0 %}
        #{action_respond_info("printer not QGL. QGLing...")}
        QUAD_GANTRY_LEVEL
    {% endif %}

    {% if printer.bed_mesh.profile_name == '' %}
        #{action_respond_info("no bed mesh loaded. Performing BED_MESH_CALIBRATE")}
        BED_MESH_CALIBRATE
    {% endif %}

    G90                            ; Go to absolute mode, otherwise printer tries to move X to 359 (209 from z-endstop + 150 from the below movement meant to move the printhead to the center. Same for Y)
    G0 X150 Y150 Z20 F3600

    #--------------------------------------------------------------------
```

# Force G32

Due to the above conditional G32, we need a different macro for when we need to re-execute `QUAD_GANTRY_LEVEL` and `BED_MESH_CALIBRATE` even if they already have values, without restarting the printer.

```
[gcode_macro ForceG32]
gcode:
    G28
    QUAD_GANTRY_LEVEL
    BED_MESH_CLEAR
    BED_MESH_CALIBRATE
```

# SERVICE macro

The below macro will bring the toolhead at the front of the printer to perform "service" activities, i.e. change nozzle, filament etc.

[gcode_macro SERVICE]
gcode:
    G90                      ; absolute positioning
    G0 X235 Y20 Z35 F3600    ; move to X235 Y20 Z35