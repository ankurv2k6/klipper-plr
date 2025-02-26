# Power Loss Recovery Plugin Guide for Klipper

This guide provides a comprehensive overview of the Power Loss Recovery plugin for Klipper 3D printers, explaining its core concepts, workflows, and implementation details. While the example configuration is derived from a Voron 2.4 with four Z steppers, the plugin is designed to work with any printer having up to four Z motors.

## Introduction

The Klipper Power Loss Recovery (PLR) plugin offers automated state saving and recovery functionality for 3D printers running Klipper firmware. In the event of unexpected printer shutdowns, the plugin enables users to resume prints from where they left off, minimizing wasted time and materials.

The plugin's Z+ homing capabilities provide accurate Z-axis calibration after unexpected stoppages, particularly beneficial for printers with multiple Z steppers.


## Demo Video

https://youtu.be/3jmUaCYAoxU

## Installation

### Adding the Plugin to an Existing Klipper Installation

1. **Download the Plugin**
   - Connect to your Klipper host via SSH
   - Navigate to the Klipper extras directory:
   ```bash
   cd ~/klipper/klippy/extras
   ```
   - Download the plugin file:
   ```bash
   wget https://raw.githubusercontent.com/ankurv2k6/klipper-plr/main/power_loss_recovery.py
   ```
   - Ensure proper permissions:
   ```bash
   chmod 644 power_loss_recovery.py
   ```

2. **Restart Klipper**
   - After installing the plugin, restart Klipper to load it:
   ```bash
   sudo service klipper restart
   ```

### Creating a Separate Configuration File

Instead of adding all configuration directly to your main `printer.cfg`, you can create a separate file for the Power Loss Recovery settings:

1. **Create the Configuration File**
   ```bash
   cd ~/printer_data/config
   nano power_loss_recovery.cfg
   ```

2. **Add Configuration to the File**
   ```ini
   # Power Loss Recovery Configuration
   [save_variables]
   filename: ~/printer_data/config/variables.cfg

   [power_loss_recovery]
   debug_mode: False
   save_interval: 30
   save_on_layer: True
   history_size: 3
   save_delay: 2
   
   # Z-PLUS HOMING OPTIONS #
   pin_stepper_z: ^PG10
   pin_stepper_z1: ^PG11
   pin_stepper_z2: ^PG12
   pin_stepper_z3: ^PG13
   
   # Add the rest of your PLR configuration here
   ```

3. **Include the File in Your Main Configuration**
   - Add the following line to your `printer.cfg`:
   ```ini
   [include power_loss_recovery.cfg]
   ```

4. **Add Macros to a Separate File (Optional)**
   - For better organization, you can also keep your PLR macros in a separate file:
   ```bash
   nano power_loss_recovery_macros.cfg
   ```
   - Add the PLR macros to this file
   - Include it in your printer.cfg:
   ```ini
   [include power_loss_recovery_macros.cfg]
   ```

5. **Restart Klipper**
   - After creating and including the config files, restart Klipper:
   ```bash
   sudo service klipper restart
   ```

## Hardware Requirements

### Z-Axis Endstops Configuration

To implement the Z+ homing functionality for printers with multiple Z steppers, you'll need to install endstops appropriate to your printer design:

#### For Floating Gantry Printers (e.g., Voron 2.4)

1. **Endstop Placement**
   - Install microswitches on the corners of the gantry facing upward
   - Install corresponding bumpers on the top of the Z rails/frame
   - The gantry moves up, and the microswitches trigger against the bumpers on the fixed frame

2. **Installation Details**
   - Mount microswitches at each corner of the gantry assembly facing upward
   - Install matching bumpers on the top of the Z rails or top frame at positions that align with each switch
   - Use adjustable mounting solutions to fine-tune the trigger point for each endstop
   - Allow approximately 5-10mm clearance from the physical hard stop

3. **Wiring Considerations**
   - Route wires along the gantry to the moving toolhead cable chain
   - Use shielded cables to reduce electrical noise
   - Secure cables with zip ties or cable management solutions
   - Ensure adequate slack for full Z-axis movement

#### For Floating Bed Printers (e.g., Voron Trident, VzBot)

1. **Endstop Placement**
   - Install endstop switches on the fixed Z columns/rails
   - Install corresponding bumpers on the bed carriage at each Z motor position
   - The bed moves down to trigger the endstops that are mounted on fixed points

2. **Installation Details**
   - Mount microswitches on the Z columns/rails (typically near the top)
   - Attach bumpers to the bed carriage or frame at positions that align with each endstop
   - Ensure proper alignment for consistent triggering
   - Install with adjustable mounting to allow for fine-tuning

3. **Wiring Considerations**
   - Keep wires organized and routed along the fixed frame
   - Ensure wire paths don't interfere with bed or gantry movement
   - Use proper strain relief to prevent cable damage

### Wiring Configuration

Connect each endstop to its designated pin on your control board:
- First Z endstop connects to `pin_stepper_z`
- Second Z endstop connects to `pin_stepper_z1`
- Third Z endstop connects to `pin_stepper_z2`
- Fourth Z endstop connects to `pin_stepper_z3`

### MCU Configuration

- Wire each endstop to an available pin on your MCU
- Configure each pin with pull-up resistors and appropriate debounce

## Initial Setup and Calibration

Before using the Power Loss Recovery system for the first time, a proper initial calibration is essential. This establishes the baseline reference that the printer will use when resuming prints.

### Initial Calibration Procedure

1. **Prepare the Printer**
   - Ensure all mechanical components are properly tightened and adjusted
   - Clean all endstop surfaces and Z rails
   - Verify that the bed is properly leveled (if applicable)

2. **Baseline Gantry/Bed Leveling**
   - For floating gantry printers (Voron 2.4):
     - Run a Quad Gantry Level (QGL) procedure: `QUAD_GANTRY_LEVEL`
     - Verify the gantry is square and level to the bed

   - For floating bed printers (Voron Trident, VzBot):
     - Run a Z Tilt Adjust procedure: `Z_TILT_ADJUST`
     - Ensure all Z motors are properly synchronized

3. **PLR Z Calibration**
   - After your standard leveling procedure completes, run the PLR Z homing calibration:
     ```
     PLR_Z_HOME MODE=CALIBRATE
     ```
   - This command will:
     - Home each Z stepper individually against its endstop
     - Measure and store the offsets between Z steppers
     - Calculate a reference position for future recovery operations
     - Save these values to the variables file for persistent storage

4. **Verification**
   - After calibration, check the saved offsets:
     ```
     PLR_QUERY_SAVED_STATE
     ```
   - The command should show offset values for each Z stepper
   - These values represent the height differences between Z steppers

5. **Test Prints**
   - Run a small test print (20-30 minutes)
   - Cancel the print intentionally
   - Use `PLR_RESUME_PRINT` to test the recovery process
   - Check that the first layer after resuming matches the previous layer perfectly

### Calibration Schedule

- Perform a full calibration whenever:
  - You make mechanical changes to the printer
  - You adjust endstop positions
  - Z stepper motors or drivers are replaced
  - After firmware updates that affect motion systems

- For regular maintenance:
  - Recalibrate monthly for high-use printers
  - Recalibrate quarterly for occasional-use printers

## Core Features

### 1. Power Loss Recovery

- **Automated state saving** at configurable intervals
- **Layer change detection** for saving state at critical printing stages
- **Historical state tracking** with configurable delay to ensure mechanical stability
- **Debug logging system** for troubleshooting
- **State verification and validation** for data integrity

### 2. Z+ Homing

- Support for **up to four Z steppers** with individual endstops
- **Fast travel capability** with configurable height and speed
- **Individual stepper probing** with sample collection for accuracy
- **Offset calculation and persistent storage**
- **Error detection and recovery**

## How It Works

### State Saving Workflow

1. **Background Monitoring**: The plugin continuously monitors the printer's state during operation.
2. **State Collection**: At configurable intervals or layer changes, the plugin collects comprehensive state data including:
   - Current position (X, Y, Z coordinates)
   - Current layer and layer height
   - File position and progress
   - Temperature settings (hotend and bed)
   - Fan speeds
   - XYZ offsets

3. **State Validation**: Each collected state is validated for integrity and completeness.
4. **Delayed Storage**: The plugin implements a configurable delay between state collection and storage to account for printer mechanics and ensure a stable state.
5. **Historical Tracking**: Multiple states are maintained in a history queue, allowing for optimal selection based on mechanical conditions.

### Recovery Workflow

1. **File Modification**: When resuming a print, the plugin creates a modified version of the original G-code file, starting from the last saved position.
2. **Z-Axis Calibration**: The plugin performs Z+ homing to ensure proper Z-axis alignment before resumption.
3. **State Restoration**: The plugin restores temperatures, fan speeds, and positional data.
4. **Print Continuation**: The printer resumes printing from the last saved state.

## Configuration Guide

### Basic Setup

Add the following to your `printer.cfg` or to a separate configuration file as described in the installation section:

```ini
[save_variables]
filename: ~/printer_data/config/variables.cfg

[power_loss_recovery]
debug_mode: False                     # Enable detailed logging (default: False)
```

### State Management Options

```ini
#PRINT STATE MANAGEMENT OPTIONS#
save_interval: 30                   # Time between saves in seconds (0-300, default: 30)
save_on_layer: True                 # Whether to save on layer changes (default: True)
history_size: 3                     # Number of states to keep in history (2-20, default: 5)
save_delay: 2                       # States to delay before saving (1-4, default: 2)
```

### Z+ Homing Configuration Example (derived from Voron 2.4)

```ini
# Z-PLUS HOMING OPTIONS #
# Pin configuration (adjust to match your controller board pins)
pin_stepper_z: ^PG10     # First Z stepper endstop pin
pin_stepper_z1: ^PG11    # Second Z stepper endstop pin
pin_stepper_z2: ^PG12    # Third Z stepper endstop pin (if applicable)
pin_stepper_z3: ^PG13    # Fourth Z stepper endstop pin (if applicable)

# Z Stepper Adjustment (tune these values for your specific printer)
stepper_z_adjust_offset: 0.0      # First Z stepper
stepper_z1_adjust_offset: 0.0     # Second Z stepper
stepper_z2_adjust_offset: 0.0     # Third Z stepper (if applicable)
stepper_z3_adjust_offset: 0.0     # Fourth Z stepper (if applicable)

z_height_offset: 0.05       # Additional offset (in mm) to apply to Z height after stepper adjustments

# Probe iteration settings
probe_iteration_count: 1            # Number of times to repeat probing for better accuracy

# Fast Travel Settings
fast_travel_upto_z_height: 250      # Height to fast travel to before homing (adjust for your Z max)
fast_travel_speed: 50               # Speed for fast travel move (mm/s)

# Basic movement settings
z_position: 350.0              # Z position for positive endstop (adjust to your printer's Z height)
fast_move_speed: 5.0           # Speed for initial probing move
slow_homing_speed: 1.0         # Speed for sampling and final homing
max_adjustment: 5.0            # Maximum allowed adjustment distance

# Sampling configuration
sample_size: 3                 # Number of samples to take per stepper
sample_retract_dist: 1.0       # Retraction distance between samples
samples_tolerance: 0.05        # Allowed tolerance between samples
samples_retry_count: 3         # Maximum number of retries for failed probes
samples_tolerance_retries: 3   # Retries allowed for samples outside tolerance
probe_samples_range: 0.07      # Maximum allowed range between highest and lowest samples

retract_dist: 1.0              # Distance to retract after initial probe

# Part Cooling Fan Status
part_cooling_fans: fan
```

### Recovery G-code Options

```ini
#RECOVERY GCODE OPTIONS #
restart_gcode: _PLR_RESUME_PRINT_START   # G-code to add into the modified file to set the printer up correctly to resume printing.

# Pre/Post operation G-code
before_resume_gcode: _PLR_BEFORE_RESUME_GCODE   # G-code to run before resume operations
after_resume_gcode:                # G-code to run after resume operations    
before_calibrate_gcode: G28         # G-code to run before calibration (customize for your printer)
after_calibrate_gcode:             # G-code to run after calibration
```

### Example Macros

These macros handle the recovery sequence:

```ini
[gcode_macro _PLR_RESUME_PRINT_START]
gcode:
    {% set resume_info = printer.save_variables.variables.resume_meta_info %}
    _PLR_RESUME_READY      
    PLR_Z_HOME MODE=RESUME
    PLR_ENABLE
    G90             ; absolute positioning
    M107            ; turn fan off
    G92 E0                         ; zero the extruder
    G1 E5.0 F1800                  ; extrude filament
    G1 X{resume_info.position.x} Y{resume_info.position.y} Z{resume_info.position.z} F3000
    G92 E0
    G90
    G21
    M83 ; use relative distances for extrusion
    RESTORE_FAN_SPEEDS

[gcode_macro _PLR_RESUME_READY]
gcode:
    {% set resume_info = printer.save_variables.variables.resume_meta_info %}
    {% set bed_center_x = printer.configfile.config.stepper_x.position_max|float / 2 %}
    {% set bed_center_y = printer.configfile.config.stepper_y.position_max|float / 2 %}
    
    M140 S{resume_info.bed_temp} # start heating the bed to the temp it was before the failure.
    M109 S{resume_info.hotend_temp} # wait for the toolhead to reach the temp it was printing on before failure
    SET_KINEMATIC_POSITION X={bed_center_x} Y={bed_center_y} Z=0 
    G91
    G1 Z5 F1200 #Clear the print by raising the toolhead
    G90
   
[gcode_macro _PLR_BEFORE_RESUME_GCODE]
gcode:
    {% set bed_center_x = printer.configfile.config.stepper_x.position_max|float / 2 %}
    {% set bed_center_y = printer.configfile.config.stepper_y.position_max|float / 2 %}
    
    G28 X Y      ; Home X and Y
    
    G1 X{bed_center_x} Y{bed_center_y} F2000
    SET_KINEMATIC_POSITION Z=0 

[gcode_macro RESTORE_FAN_SPEEDS]
description: Restore fan speeds from saved state in resume_meta_info
gcode:
    {% set svv = printer.save_variables.variables %}
    {% if 'resume_meta_info' not in svv %}
        { action_respond_info("No saved state found in variables") }
    {% else %}
        {% set state = svv.resume_meta_info %}
        {% if 'fan_speeds' not in state %}
            { action_respond_info("No fan speed data found in saved state") }
        {% else %}
            {% set fan_speeds = state.fan_speeds %}
            {% set cur_extruder = printer.toolhead.extruder %}
            
            # Loop through all saved fan speeds
            {% for fan_name, speed in fan_speeds.items() %}
                {% if fan_name == 'fan' or fan_name == cur_extruder + '_fan' %}
                    # This is a parts cooling fan for current extruder - use M106
                    {% set speed_byte = (speed * 255.0 + 0.5)|int %}
                    M106 P0 S{speed_byte} ; Set parts cooling fan
                    { action_respond_info("Setting parts cooling fan %s using M106: S%d (%.1f%%)" % (fan_name, speed_byte, speed * 100.0)) }
                {% else %}
                    # Check if this is a valid generic fan
                    {% if fan_name in printer %}
                        SET_FAN_SPEED FAN={fan_name} SPEED={speed}
                        { action_respond_info("Setting generic fan %s using SET_FAN_SPEED: %.1f%%" % (fan_name, speed * 100.0)) }
                    {% else %}
                        { action_respond_info("Warning: Fan %s not found - skipping" % fan_name) }
                    {% endif %}
                {% endif %}
            {% endfor %}
        {% endif %}
    {% endif %}
```

## Integration with Existing Macros

### With Standard Homing

For printers with multiple Z steppers, integrate with your existing homing routine:

```ini
# Example for printer with multiple Z steppers
[gcode_macro HOME_ALL]
gcode:
    G28                # Home all axes
    # Add this for printers that need additional Z leveling
    PLR_Z_HOME MODE=CALIBRATE  # Calibrate Z offsets
    
# Add this to your before_calibrate_gcode option in [power_loss_recovery]
before_calibrate_gcode: G28
```

### With START_PRINT Macro

Integrate with your standard START_PRINT macro:

```ini
# Add the following to your START_PRINT macro
[gcode_macro START_PRINT]
gcode:
    # Your existing start print commands
    ...
    
    # Add PLR placeholders
    ;;;;; PLR_RESUME - INITIAL PRINTER SETUP STARTS ;;;;;
    # Your normal printer setup code here
    ;;;;; PLR_RESUME - PRINT GCODE STARTS ;;;;;
    
    # Enable PLR for this print
    PLR_ENABLE
    
    # Rest of your start print code
    ...
```

## Usage Guide

### Normal Operation

During normal printing, the Power Loss Recovery plugin automatically saves the printer state based on your configuration settings. No user intervention is required during this process.

### G-code Commands

The plugin provides several G-code commands for manual control:

#### State Management

- `PLR_ENABLE`: Enable power loss recovery state saving
- `PLR_DISABLE`: Disable power loss recovery state saving
- `PLR_SAVE_PRINT_STATE`: Manually save current printer state
- `PLR_QUERY_SAVED_STATE`: Query the current status of the state saver
- `PLR_RESET_PRINT_DATA`: Clear all saved state data

#### Z+ Homing

- `PLR_Z_HOME MODE=CALIBRATE`: Perform Z+ homing and save offsets
- `PLR_Z_HOME MODE=RESUME`: Perform Z+ homing using saved offsets for resume

#### Recovery

- `PLR_RESUME_PRINT`: Create a modified gcode file for power loss recovery resume

#### Bed Mesh Management

- `PLR_SAVE_MESH`: Save the currently active bed mesh profile name
- `PLR_LOAD_MESH`: Load the previously saved bed mesh profile

### Recovering After Unexpected Stoppage

1. After the printer is restarted, run `PLR_QUERY_SAVED_STATE` to confirm saved data
2. Issue `PLR_RESUME_PRINT` to start the recovery process
3. The plugin will:
   - Create a modified G-code file starting from the saved position
   - Perform Z+ homing to ensure proper Z alignment
   - Restore temperatures, fan speeds, and other settings
   - Resume printing from the last saved position

## Best Practices

### Z+ Homing Optimization

For printers with multiple Z steppers:

1. **Initial Calibration**
   - Run `PLR_Z_HOME MODE=CALIBRATE` after ensuring your basic homing is working correctly
   - If your build platform isn't perfectly level after the process, adjust the `stepper_z_adjust_offset` values
   - Typical values range between -0.3 and +0.3 mm

2. **Verification**
   - Run a test print with a large surface area (like a calibration plate)
   - Check first layer consistency across the build platform
   - If any areas show issues, adjust the corresponding Z offset

### Configuration Optimization

1. **Save Interval**: Balance between safety and performance
   - 30-60 seconds works well for most prints
   - For extremely long prints, consider 120 seconds to reduce overhead

2. **History Size and Delay**: Recommendations by printer type
   - For printers with a rigid frame (like Voron), `history_size: 3` and `save_delay: 2` works well
   - For printers with a moving bed (like Prusa style), consider `history_size: 5` and `save_delay: 3`
   - For lightweight direct drive systems, you can reduce `save_delay` to 1

3. **Z+ Homing Settings**: Optimize for your endstop type
   - `sample_size: 3` provides good balance between speed and accuracy
   - `samples_tolerance: 0.05` works well with standard microswitch endstops
   - For inductive probes, increase to `samples_tolerance: 0.075`

### Slicer Integration

For optimal operation, modify your slicer's start and end G-code to include the necessary placeholders:

1. **Start G-code Modification**
   - Add the PLR placeholders after your START_PRINT macro call:
   ```
   START_PRINT EXTRUDER={first_layer_temperature[initial_extruder]} BED={first_layer_bed_temperature}
   ;;;;; PLR_RESUME - INITIAL PRINTER SETUP STARTS ;;;;;
   ;;;;; PLR_RESUME - PRINT GCODE STARTS ;;;;;
   ```

2. **Layer Change G-code**
   - In your slicer's layer change G-code, add:
   ```
   ;LAYER_CHANGE
   ;Z:[layer_z]
   PLR_SAVE_PRINT_STATE_WITH_LAYER LAYER=[layer_num] LAYER_HEIGHT=[layer_z]
   ```

## Troubleshooting

### Common Issues

1. **Z Homing Problems**
   - **Uneven Homing**: Check mechanical components for each Z stepper
   - **Inconsistent Z Heights**: Verify Z rails or rods are parallel and smooth
   - **Binding During Homing**: Check for debris in Z endstops or mechanical systems

2. **Recovery Misalignment**
   - **X/Y Shift**: Re-check your belts and pulley set screws
   - **Z Layer Shifts**: Possible Z drive system issues, check for mechanical problems
   - **Inconsistent First Layer After Resume**: Adjust Z offsets or run bed leveling before resuming

3. **Endstop Reliability**
   - **False Triggers**: Check for EMI, ensure proper shielding from stepper motors
   - **No Trigger**: Verify endstop wiring and mounting
   - **Inconsistent Triggers**: Clean endstop contact surfaces

### Debug Mode

Enable debug mode for detailed logging:

```ini
[power_loss_recovery]
debug_mode: True
```

Debug logs will be output to both the Klipper log and the printer console.

## Advanced Features

### Optimization Techniques

The plugin implements several optimization techniques:

1. **Adaptive Save Interval**: The plugin can dynamically adjust save intervals based on printing conditions
2. **MCU Load Monitoring**: Checks MCU load before saving state to prevent overloading
3. **Motion Planning Verification**: Ensures mechanical stability before state saving

### Custom Recovery Workflows

You can customize the recovery process for your specific printer by modifying the recovery G-code macros:

```ini
[gcode_macro _PLR_CUSTOM_RESUME]
gcode:
    {% set resume_info = printer.save_variables.variables.resume_meta_info %}
    
    # Printer-specific preparation
    G28                  # Home all axes
    BED_MESH_PROFILE LOAD=default  # Load your default bed mesh
    
    # Heat up with "smart" waiting to reduce oozing
    M104 S{resume_info.hotend_temp}  # Start heating hotend
    M140 S{resume_info.bed_temp}     # Start heating bed
    TEMPERATURE_WAIT SENSOR=heater_bed MINIMUM={resume_info.bed_temp * 0.95}  # Wait for 95% of bed temp
    M109 S{resume_info.hotend_temp}  # Wait for hotend to reach full temp
    
    # Continue with normal resume
    _PLR_RESUME_PRINT_START
```

## Technical Details

### State Data Structure

The saved state contains comprehensive information:

```json
{
  "position": {"x": 150.0, "y": 150.0, "z": 0.2},
  "xyz_offsets": {"x": 0.0, "y": 0.0, "z": 0.1},
  "fan_speeds": {"fan": 0.5},
  "layer": 1,
  "layer_height": 0.2,
  "file_progress": {
    "position": 10000,
    "total_size": 100000,
    "progress_pct": 10.0
  },
  "active_extruder": "extruder",
  "hotend_temp": 240.0,
  "bed_temp": 110.0,
  "current_file": "benchy.gcode"
}
```

### Z+ Homing Process

The Z+ homing process for multiple Z steppers follows these steps:

1. **Initial Probe**: First Z stepper (stepper_z) is used as reference
2. **Individual Probing**: Each Z stepper is probed independently
3. **Offset Calculation**: Offsets are calculated relative to the reference stepper
4. **Offset Application**: Offsets are applied to ensure a level build platform
5. **Final Home**: A final homing move ensures calibration accuracy

## Conclusion

The Klipper Power Loss Recovery plugin, when properly configured for your specific printer, provides a robust solution for recovering from unexpected print interruptions. The plugin's Z+ homing capabilities are particularly valuable for printers with multiple Z steppers, ensuring a perfectly level build platform upon recovery.

By integrating this plugin with your printer's existing macros and carefully tuning the Z stepper offsets, you can significantly improve the reliability and usability of your 3D printer.

For further assistance, refer to the full source code documentation or submit issues through the repository's issue tracker.
