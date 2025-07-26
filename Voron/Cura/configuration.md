# Machine settings
## Printer
### Printer Settings
|name       |   value   |
|-----------|-----------|
| X (Width) |   300.0mm |
| Y (Depth) |   300.0mm |
| Z (Height) |   300.0mm |
| Build plate shape|Rectangular|
|Origin at center|False|
|Heated bed|True|
|Heated build volume|False|
|G-code flavor|Marlin|

### Printhead Settings
|name       |   value   |
|-----------|-----------|
|X min|-35mm|
|Y min ('-' towards back)|-50mm|
|X max|35mm|
Y max ('+' towards front)|65mm|
|Gantry Height|30.0mm|
|Number of Extruders|1|
|Apply Extruder offests to GCode|True|
|Start GCode must be first|False|

### Start G-code
`print_start EXTRUDER={material_print_temperature_layer_0} BED={material_bed_temperature_layer_0} CHAMBER={build_volume_temperature}`

### End G-code
`print_end`

## Toolhead
|name       |   value   |
|-----------|-----------|
|Compatible material diameter|1.75mm|
all other values: 0
textboxes: empty