# W12 Regional Leveling for Creality K2 / K2 Plus (Klipper)

Adds **regional (area-limited)** bed meshing to the K2/K2 Plus.  
The grid size **auto-selects 5×5 / 7×7 / 9×9** based on part span.  
If no bounds are provided, it **falls back to a full mesh**.

---

## Requirements

- Klipper on Creality **K2 / K2 Plus**
- Stock “box” macros present (used by your original `START_PRINT`)
- A basic Klipper `bed_mesh` setup (the macro calls `BED_MESH_CALIBRATE`)
- In `printer.custom_macro`, the variables exist:  
  `default_bed_temp`, `default_extruder_temp`, `g28_ext_temp`
- (Optional) Filament sensor named exactly `filament_sensor`

---

## 1) Install the file

**Mainsail/Fluidd UI (recommended)**

1. Open **Mainsail/Fluidd → Files → `config/`**.
2. **Drag & drop** `w12_leveling.cfg` into the `config/` folder.

Include it in printer.cfg:

```bash
[include w12_leveling.cfg]


{"code":"key60", "msg":"Internal error on command:BED_MESH_CALIBRATE", "values": ["BED_MESH_CALIBRATE"]}
Fix in printer.cfg, comment out the original probe_count and set it to 5,5

[bed_mesh]
speed: 100
mesh_min: 5,5
mesh_max: 345,345
# probe_count: 9,9    ; original -> COMMENT THIS OUT
probe_count: 5,5       ; NEW VALUE
```

## 2) K2-specific settings (required)

Add/verify in printer.cfg:

[virtual_sdcard]  
path: /mnt/UDISK/printer_data/gcodes  
forced_leveling: false

[prtouch_v3]  
regional_prtouch_switch: True

Restart Klipper after saving.

## 3) OrcaSlicer — Start G-code

Pass first-layer bounds to Klipper, then call your existing START_PRINT:

```bash
; First-layer bounds → Klipper
SET_PRINT_MIN X={first_layer_print_min[0]} Y={first_layer_print_min[1]}
SET_PRINT_MAX X={first_layer_print_max[0]} Y={first_layer_print_max[1]}

; Your original start macro (expects these parameter names)
START_PRINT EXTRUDER_TEMP=[nozzle_temperature_initial_layer] BED_TEMP=[bed_temperature_initial_layer_single]
Keep the rest of your Start G-code (purge line, etc.) unchanged.
```

## 4) Insert the regional mesh into your original macros

A) In START_PRINT — add one line after homing/cleaning
Context (excerpt):

```bash
[gcode_macro START_PRINT]
variable_prepare: 0
gcode:
  BOX_START_PRINT
  G90
  SET_GCODE_OFFSET Z=0
  {% set g28_extruder_temp = printer.custom_macro.g28_ext_temp %}
  {% set bed_temp = printer.custom_macro.default_bed_temp %}
  {% set extruder_temp = printer.custom_macro.default_extruder_temp %}
  {% set BED_TEMP = params.BED_TEMP|default(60)|float %}
  {% set EXTRUDER_TEMP = params.EXTRUDER_TEMP|default(220)|float %}
  # {% set EXTRUDER_WAITTEMP = (EXTRUDER_TEMP/1.5|float)|int %}
  {% set EXTRUDER_WAITTEMP = (140.0|float)|int %}
  # {% set y_park = printer.toolhead.axis_maximum.y/2 %}
  {% if printer['gcode_macro START_PRINT'].prepare|int == 0 %}
    {action_respond_info("print prepared 111")}
    M106 S0  #需要关闭模型风扇
    M140 S{params.BED_TEMP}
    M104 S{EXTRUDER_WAITTEMP}
    # M204 S2000
    # SET_VELOCITY_LIMIT ACCEL_TO_DECEL=2000 #500 20240326
    SET_VELOCITY_LIMIT ACCEL=5000 ACCEL_TO_DECEL=5000
    G28
    # 精擦嘴
    NOZZLE_CLEAR
    # BOX_GO_TO_EXTRUDE_POS#M1500
    M104 S{EXTRUDER_WAITTEMP}
    M190 S{params.BED_TEMP}
    M109 S{EXTRUDER_WAITTEMP}
    BOX_NOZZLE_CLEAN#M1501
    # Z_TILT_ADJUST
    # BOX_GO_TO_EXTRUDE_POS#M1500
    #M190 S{params.BED_TEMP}
    # BOX_NOZZLE_CLEAN#M1501
    # 精归零
    NEXT_HOMEZ_NACCU
    G28 Z
    DO_REGIONAL_MESH  # <- add w12 ------------------------------------------------ w12
    # BED_MESH_CALIBRATE # <- commented out w12 ----------------------------------- w12
    # CXSAVE_CONFIG
  {% else %}
    PRINT_PREPARE_CLEAR
  {% endif %}
  M140 S{params.BED_TEMP}
  M104 S{params.EXTRUDER_TEMP}
  # G1 Z5 F600
  BOX_GO_TO_EXTRUDE_POS#M1500
  M190 S{params.BED_TEMP}
  M109 S{params.EXTRUDER_TEMP} ;wait nozzle heating
  M220 S100 ;Reset Feedrate
  # M221 S100 ;Reset Flowrate
  G21
  SET_VELOCITY_LIMIT SQUARE_CORNER_VELOCITY=10
  M204 S5000
  SET_VELOCITY_LIMIT ACCEL_TO_DECEL=5000
  # M83
  # G1 E120 F600
  # M106 S255
  # G4 P2000
  # M107
  BOX_NOZZLE_CLEAN#M1501
  # G1 X350 Y150 F12000
  # G1 Z0.2 F600
  # G1 X350 Y0 E15 F6000
  # G1 X200 Y0 E15 F6000
  # G1 Z5 F600
  G92 E0 ; Reset Extruder
  SET_PIN PIN=extruder_fan VALUE=1
  SET_TEMPERATURE_FAN_SWITCH TEMPERATURE_FAN=chamber_fan VALUE=1

```

B) G29 — pass PROBE_COUNT correctly and call regional mesh
Build get_count (supports 7x7 or 7,7) and call DO_REGIONAL_MESH {get_count}:

```bash
[gcode_macro G29] ;调平2024/03/24  在屏幕设置了温度后不生效，还是默认值 玉山确认
gcode:
  {% if 'PROBE_COUNT' in params|upper %}
    {% set get_count = ('PROBE_COUNT' + params.PROBE_COUNT) %}
  {%else %}
    {% set get_count = "" %}
  {% endif %}

  {% set bed_temp = printer.custom_macro.default_bed_temp %}
  {% set extruder_temp = printer.custom_macro.g28_ext_temp %}
  {% set nozzle_clear_temp = printer.custom_macro.default_extruder_temp %}

  {% if 'BED_TEMP' in params|upper and params.BED_TEMP|default(0)|int >= printer.custom_macro.default_bed_temp %}
    {% set bed_temp = params.BED_TEMP %}
  {% endif %}

  {% if 'EXTRUDER_TEMP' in params|upper %}
    {% set nozzle_clear_temp = params.EXTRUDER_TEMP %}
  {% endif %}
  {% set enabled = printer["filament_switch_sensor filament_sensor"].enabled %}
  {% if enabled %}
    SET_FILAMENT_SENSOR SENSOR=filament_sensor ENABLE=0
  {% endif %}
  M104 S{extruder_temp} #140  #需要跟源码结合
  M140 S{bed_temp}  #50
  {% if "xy" not in printer.toolhead.homed_axes %}
    G28 X Y
  {% endif %}
  # 此处Z归位一定要下降,此次光电找平后,不恢复上次做Z_TILT_ADJUST记录的adjustments值
  # 每次调平在上面的重新做Z_TILT_ADJUST,才能得到新的adjustments调整值
  # 设置标志位 ZDOWN的时候 不恢复上次做Z_TILT_ADJUST记录的adjustments值
  SET_G29_FLAG VALUE=1
  G28 Z
  BED_MESH_CLEAR
  NOZZLE_CLEAR
  M104 S{extruder_temp}
  M190 S{bed_temp}
  M109 S{extruder_temp}
  # 精擦嘴后做一次光电找平获取最大Z轴高度值
  ZDOWN_SWITCH ENABLE=1
  G28 Z
  BED_MESH_CLEAR
  Z_TILT_ADJUST WAITTIME=5
  NEXT_HOMEZ_NACCU
  G28 Z
  SET_G29_FLAG VALUE=0

  M204 S5000
  SET_VELOCITY_LIMIT ACCEL_TO_DECEL=5000
  #BED_MESH_CALIBRATE {get_count} # <- commented out w12 --------------------------------------------- w12
  DO_REGIONAL_MESH {get_count} # <- add w12 ---------------------------------------------------------- w12
  G1 Z175 F1200
  BOX_MOVE_TO_SAFE_POS#M1499
  CXSAVE_CONFIG
  TURN_OFF_HEATERS
  {% if enabled %}
    SET_FILAMENT_SENSOR SENSOR=filament_sensor ENABLE=1
  {% endif %}
```

## 5) Quick test

Start a print and check the Klipper console. You should see:

```bash
Regional mesh: MIN=(...), MAX=(...), span=... mm -> PROBE_COUNT=...
```

The probe points should cover only the printed area.

Public Domain — [The Unlicense](https://unlicense.org).
