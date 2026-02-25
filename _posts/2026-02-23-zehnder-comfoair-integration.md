---
layout: post
title: "Zehnder ComfoAir E300 HA Integration"
date: 2026-02-23 23:00:00 +0000
description: "A journey from googling to granular control via RS485 and 0-10V."
---

It started with a simple search for Home Assistant integration for my Zehnder ComfoAir E300. I quickly found that most projects were limited to basic 3-way switch overrides. Then I stumbled upon [this thread](https://github.com/wichers/esphome-comfoair/issues/11#issuecomment-3919157398) hinting at a deeper interface.

At the time, I'd lent out my multimeter, so I couldn't do much investigation. I followed the thread and waited. Two weeks ago, [@CodedCactus](https://github.com/CodedCactus) made progress on an ESPHome component, which was the spark I needed.

## The Hardware Hunt

I wasted no time ordering an [RS485 to TTL module](https://amzn.to/4sbdrxK) to pair with an [ESP32](https://amzn.to/4cu7EyL) I had lying around.

Once the module arrived, I did some quick soldering and used a standard network cable to connect the RS485 module to the Zehnder’s C3 port.

![RS485 Soldering]({{ '/assets/images/zehnder_rs485_soldering.jpg' | relative_url }})
*Drafting the physical connection.*

## Discovery and Collaboration

The initial register set was small. Without other scanning tools, I started cloning [@CodedCactus](https://github.com/CodedCactus)'s work to investigate further. I found a minor bug when the outside temperature dropped below zero (which I reported in [PR #1](https://github.com/CodedCactus/zehnder-comfoair/pull/1)) and eventually identified several missing registers while tinkering with the unit's settings ([PR #2](https://github.com/CodedCactus/zehnder-comfoair/pull/2)).

## The Case (Iteration 1)

With things working, I fired up OpenSCAD and designed a 3D-printed enclosure. I was a bit lazy and used the listed dimensions for the components rather than measuring them myself.

When I pulled it off the printer, the realization hit: it didn't fit. The connector stuck out too far.

![Case v1 Fit Issue]({{ '/assets/images/zehnder_case_v1_fit.jpg' | relative_url }})
*The "specs vs reality" moment.*

I grabbed my trusty calipers, measured the actual modules, and printed version 2. It fit perfectly.

## The Pivot to 0-10V

I wanted control, not just data. Unfortunately, it seemed the discovered RS485 registers were read-only. I looked toward the Zehnder's 0-10V interface as an alternative.

I found the [DFRobot GP8211S DAC module](https://www.tinytronics.nl/en/communication-and-signals/signal-generators-and-dacs/dfrobot-gravity-gp8211s-dac-module-1-channel-15bit-i2c-0-5v-10v) and added it to the ESP32.

### ESPHome Config

```yaml
i2c:
  - id: bus_a
    sda: GPIO23
    scl: GPIO22
    scan: true

gp8403:
  id: gp8211
  voltage: 10V
  model: GP8413
  i2c_id: bus_a

output:
  - platform: gp8403
    id: gp8211_output_1
    gp8403_id: gp8211
    channel: 0

number:
  - platform: template
    name: "GP8211 Voltage Control"
    id: gp8211_slider
    min_value: 0
    max_value: 10
    step: 0.1
    unit_of_measurement: "V"
    initial_value: 0
    optimistic: true
    set_action:
      then:
        - output.set_level:
            id: gp8211_output_1
            # We divide by 10 because set_level expects a value from 0.0 to 1.0
            level: !lambda 'return x / 10.0;'
```

## "Winging it" and Final Calibration

Still without my multimeter, I decided to "wing it" and connect the DAC to the unit. It worked, but asking for 10V only registered as 9.2V on the Zehnder. After digging into the unit's settings, I found the **Analoog 0-10V Max. Instelling** and adjusted it to match the actual output.

![Zehnder Max Voltage Setting]({{ '/assets/images/zehnder_settings_max_voltage.png' | relative_url }})
*Adjusting the maximum voltage setting to calibrate the DAC.*

I also changed the **Analoog 0-10V Methode** to **Proportioneel Debiet**. This is crucial as it allows fine-grained ventilation control based on the input voltage, rather than just mapping to the three standard presets.

![Zehnder Method Setting]({{ '/assets/images/zehnder_settings_method.png' | relative_url }})
*Setting the unit to proportional flow mode.*

![Final Internal Assembly]({{ '/assets/images/zehnder_case_v2_dac.jpg' | relative_url }})
*Revision 2 of the OpenSCAD case, now housing both the RS485 and the DAC.*

![Finished Unit]({{ '/assets/images/zehnder_case_v2_lid.jpg' | relative_url }})
*Closed up and in service.*

## The Enclosure (OpenSCAD)

To house the final assembly, I continued refining the previous OpenSCAD design. The benefit of a parametric script meant I could simply expand the dimensions and add cutouts for the new DAC module without starting from scratch.

```scad
// Parameters
part = "assembly"; // [assembly, case, lid]
handle_len = 5.0;
$fn = 50;

// Dimensions (mm)
wall_tickness = 3.0; // Increased for rail strength
lid_thickness = 2.0;
groove_depth = 1.0;
tolerance = 0.5;
internal_height = 22; // Enough clearance for pins/headers + lid clearance
divider_height = 2.0; // Height of the internal walls separating modules

// D1 Mini Dimensions
d1_width = 31.5;
d1_length = 39.0;
d1_height = 13.0; // Approx height with headers
d1_usb_width = 8.0;
d1_usb_height = 3.0;

// RS485 Module Dimensions
rs485_width = 22.0;
rs485_base_length = 53.0; 
rs485_connector_extra = 15.0; // Space for connector
rs485_length = rs485_base_length + rs485_connector_extra; 
rs485_height = 15.0; // Approx max height (usually the terminal block)
rs485_terminal_width = 15.0;
rs485_terminal_height = 10.0;

// Gravity I2C Module Dimensions (Rotated 90 degrees)
// Original Image: Width=35, Height=27. We rotate so it sits narrow.
gravity_width = 27.0; // X dimension in case
gravity_length = 35.0; // Y dimension in case
gravity_height = 13.2; // 11.6 terminal + 1.6 PCB
gravity_terminal_width = 10.0; // Approximate width of the screw terminal
gravity_terminal_height = 11.6;

// Layout
module_spacing = 5.0; // Space between boards

// Calculated Outer Dimensions
case_inner_width = rs485_width + gravity_width + d1_width + (2 * module_spacing) + (2 * tolerance);
case_inner_length = max(d1_length, rs485_length, gravity_length) + (2 * tolerance);

outer_width = case_inner_width + 2 * wall_tickness;
outer_length = case_inner_length + 2 * wall_tickness;
outer_height = internal_height + wall_tickness;

// Modules for Visualization and Cutouts

module d1_mini() {
    color("blue") 
    cube([d1_width, d1_length, d1_height]);
    
    // USB Connector (centered on width, at the 'bottom' end)
    color("silver")
    translate([(d1_width - d1_usb_width)/2, -2, 0])
    cube([d1_usb_width, 5, d1_usb_height]);
}

module rs485_module() {
    color("green")
    cube([rs485_width, rs485_length, 2]); // PCB
    
    // Terminal Block (approximate)
    color("darkgreen")
    translate([0, rs485_length - 8, 2])
    cube([rs485_width, 8, rs485_terminal_height]);
    
    // Chip/Components area (rough volume)
    color("black")
    translate([2, 5, 2])
    cube([rs485_width - 4, rs485_length - 15, 5]);
}

module gravity_module() {
    color("darkorange")
    cube([gravity_width, gravity_length, 1.6]); // PCB

    // VOUT/GND Terminal Block (rotated to face +Y, back wall)
    color("darkgreen")
    translate([(gravity_width-gravity_terminal_width)/2, gravity_length-10, 1.6]) 
    cube([gravity_terminal_width, 10, gravity_terminal_height]);

    // I2C connector (facing -Y, inside case)
    color("white")
    translate([(gravity_width-14)/2, 0, 1.6]) 
    cube([14, 6, 8]);
}

module electronics_assembly() {
    // Center the assembly
    translate([-case_inner_width/2 + tolerance, -case_inner_length/2 + tolerance, wall_tickness]) {
        // Gravity placement (Left side, flush back for terminal)
        translate([0, case_inner_length - gravity_length, 0])
        gravity_module();
        
        // RS485 placement (Middle, flush back for terminal)
        translate([gravity_width + module_spacing, case_inner_length - rs485_length, 0])
        rs485_module();

        // D1 Mini placement (Right side, Flush front for USB)
        translate([gravity_width + rs485_width + (2 * module_spacing), 0, 0])
        d1_mini();
    }
}

module rounded_cube(size, r) {
    x = size[0]; y = size[1]; z = size[2];
    linear_extrude(height=z)
    hull() {
        translate([-x/2 + r, -y/2 + r, 0]) circle(r=r);
        translate([x/2 - r, -y/2 + r, 0]) circle(r=r);
        translate([-x/2 + r, y/2 - r, 0]) circle(r=r);
        translate([x/2 - r, y/2 - r, 0]) circle(r=r);
    }
}

module case_shell() {
    difference() {
        // Main Block (Rounded)
        rounded_cube([outer_width, outer_length, outer_height], 3.0);
        
        // Inner Void
        translate([-case_inner_width/2, -case_inner_length/2, wall_tickness])
        cube([case_inner_width, case_inner_length, internal_height + 1]);
        
        // Sliding Lid Grooves
        // Extended by groove_depth into the front wall to allow the lid to tuck in and sit perfectly flush at the back
        translate([-case_inner_width/2 - groove_depth, -case_inner_length/2 - groove_depth, outer_height - lid_thickness - 2]) 
        cube([case_inner_width + 2*groove_depth, case_inner_length + wall_tickness + 5 + groove_depth, lid_thickness + tolerance]);
        
        // Rear Lid Entry (Back Wall Top Clearance)
        // Added 2mm extra clearance in front of the handle position so it doesn't bottom out early
        translate([-case_inner_width/2 - groove_depth, case_inner_length/2 + wall_tickness - handle_len - 2.0, outer_height - lid_thickness - 1])
        cube([case_inner_width + 2*groove_depth, handle_len + wall_tickness + 3.0, lid_thickness + 5]); 
        
        // USB Cutout (Rounded)
        usb_cutout_width = 14.0;
        usb_cutout_height = d1_usb_height + 2*tolerance + 2.0; // Increased height
        usb_cut_depth = wall_tickness + 2;
        usb_r = 2.0; 
        
        // D1 is on the right
        d1_center_x = -case_inner_width/2 + tolerance + rs485_width + gravity_width + (2*module_spacing) + d1_width/2;
        cut_center_z = wall_tickness + d1_usb_height/2 + 2.5; // Raised by 2.5mm
        cut_y_start = -outer_length/2 - 1; 

        translate([d1_center_x, cut_y_start, cut_center_z])
        rotate([-90, 0, 0])
        rounded_cube([usb_cutout_width, usb_cutout_height, usb_cut_depth], usb_r);
        
        // Gravity Terminal Cutout - Left, Back wall
        gravity_rel_x = -case_inner_width/2 + tolerance;
        gravity_cutout_width = 14.0; // slightly wider than the 10mm terminal 
        translate([gravity_rel_x + (gravity_width - gravity_cutout_width)/2, outer_length/2 - wall_tickness - 1, wall_tickness + 2])
        cube([gravity_cutout_width, wall_tickness + 2, outer_height]); // Drop in cutout

        // RS485 Terminal Cutout - Middle, Back wall
        rs485_rel_x = gravity_rel_x + gravity_width + module_spacing;
        translate([rs485_rel_x, outer_length/2 - wall_tickness - 1, wall_tickness + 2])
        cube([rs485_width, wall_tickness + 2, outer_height]); // Cuts all the way to top
    }
    
    // Internal Retention Walls
    pcb_retention_walls();
}

module pcb_retention_walls() {
    // Gravity is at Left
    gravity_x_start = -case_inner_width/2 + tolerance;
    gravity_x_end = gravity_x_start + gravity_width;
    
    // RS485 is in Middle
    rs485_x_start = gravity_x_end + module_spacing;
    rs485_x_end = rs485_x_start + rs485_width;

    // Divider 1 (between Gravity and RS485)
    divider_width = module_spacing - 0.2; 
    divider1_x = gravity_x_end + (module_spacing - divider_width)/2;
    
    translate([divider1_x, -case_inner_length/2, wall_tickness])
    cube([divider_width, case_inner_length, divider_height]);

    // Divider 2 (between RS485 and D1)
    divider2_x = rs485_x_end + (module_spacing - divider_width)/2;
    translate([divider2_x, -case_inner_length/2, wall_tickness])
    cube([divider_width, case_inner_length, divider_height]);
}

module lid() {
    // Top Plate Runner (Sides ride in grooves)
    runner_width = case_inner_width + 2 * groove_depth - 2 * tolerance;
    runner_length = case_inner_length + wall_tickness ; // Fits inside the case (Inner Front to Inner Back)
    
    // Position Lid to cover case
    translate([0, 0, 0]) {
        // Main Runner Plate
        translate([-runner_width/2, -case_inner_length/2, 0])
        cube([runner_width, runner_length, lid_thickness]);
        
        // Grip Handle
        handle_height = 1.0;
        translate([-runner_width/2, case_inner_length /2 + wall_tickness - handle_len, lid_thickness])
        cube([runner_width, handle_len, handle_height]);
    }
}


if (part == "assembly") {
    // Case
    color("orange")
    case_shell();

    // Electronics (ghost)
    %electronics_assembly();

    // Lid 
    color("cyan")
    translate([0, 30, internal_height + wall_tickness - lid_thickness - 1]) 
    lid();
} else if (part == "case") {
    case_shell();
} else if (part == "lid") {
    lid();
}
```

## The Result

I now have a complete view of the sensor data and, more importantly, the ability to ramp up ventilation automatically when we're showering or when CO2 levels rise. A rewarding project that went from a "maybe one day" thread to a fully custom control module.
## Join the Effort

The work is far from finished. We are still actively mapping more registers to unlock even more data and control possibilities.

If you own a Zehnder ComfoAir unit and have discovered something new, or if you're interested in testing, your input would be invaluable. Check out the [ESPHome component repository](https://github.com/CodedCactus/zehnder-comfoair) and help us make this integration even better for everyone.
