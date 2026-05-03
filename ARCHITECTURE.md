# BlenderMoco Architecture

## Overview

BlenderMoco is a single-file Blender add-on (`__init__.py`) designed to export motion control movement paths from Blender to Dragonframe. It provides a UI in the 3D Viewport to manage up to 36 custom "axes" of motion, where each axis is tied to a specific object's location or rotation component (X, Y, or Z).

## Core Components

The add-on relies on standard Blender Python API structures (`bpy.types.Panel`, `bpy.types.Operator`, `bpy.props`, and `bpy.app.handlers`).

### UI Panel (`View3dPanel`)
The `View3dPanel` class defines the UI that appears in the 3D Viewport tool panel (Object Mode). It dynamically renders properties for the active axes, including inputs for positions, objects, components, and buttons to reorganize or remove axes. 

### Scene Properties (Data Model)
Instead of creating a custom `PropertyGroup` collection for axes, the add-on dynamically generates flat Scene properties (`bpy.types.Scene.*`) during registration for the maximum allowed axes (`maxAxisCount = 36`).

For each axis index `i`, it stores:
- `moco_axis_label_{i}`: User-friendly name.
- `moco_axis_component_{i}`: Enum for the component type (LX, LY, LZ, RX, RY, RZ).
- `moco_axis_object_{i}`: String pointer to the target Blender Object.
- `moco_axis_setlength_{i}`: FloatProperty for position (length units).
- `moco_axis_setrot_{i}`: FloatProperty for rotation (rotation units).

It also defines global export properties:
- `moco_num_axis`: Current number of active axes.
- `camera_path_file_name`: Name for the exported file.
- `moco_export_type`: Export format (Raw Move or Arc Move).

### Operators
- **`ExportMovement` (`moco.exportmovement`)**: Handles the export process. For "Raw Move" (frame-by-frame), it acts as a modal operator, stepping through the timeline frame-by-frame and accumulating positions. For "Arc Move", it executes immediately, extracting F-Curve keyframes and writing them to XML.
- **`AddAxis` (`moco.addmovementaxis`)**: Increments the active axis count.
- **`RemoveAxis`, `MoveAxisUp`, `MoveAxisDown`**: Dynamically generated classes (one per potential axis index) that handle reordering of the flat properties and swapping F-Curve data paths.

### Animation and Synchronization
The add-on expects the user to animate the custom axis properties (`moco_axis_setlength_*` or `moco_axis_setrot_*`), *not* the objects directly.
- An `update` callback (`updateObjectPositions`) on the custom properties applies the custom axis values to the actual referenced Blender objects.
- `bpy.app.handlers.frame_change_post` is utilized via the `animationUpdate` function to ensure that whenever the frame changes during playback, the actual object positions are updated to reflect the keyframed custom axis properties.

## Export Formats

1. **Raw Move (`.txt`)**: 
   A simple text format where each line represents a frame, and columns represent the axis positions. Rotations are converted to degrees. Handled via the modal timer to step through each frame physically in the scene.
2. **Arc Move (`.arcm`)**: 
   An XML-based format (`http://caliri.com/motion/scene`) parsing the F-Curve control points directly. This format exports the keyframes and their bezier handles, allowing further interpolation editing in Dragonframe. Lengths are converted according to the scene's unit settings.
