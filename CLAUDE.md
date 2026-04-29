# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

This is a ROS 2 colcon workspace. All commands assume you are at the root of your ROS 2 workspace (e.g., `~/ros2_ws/`), not inside this repo's directory.

**Install dependencies:**
```bash
rosdep update
rosdep install --from-paths src --ignore-src -r -y
```

**Standard release build:**
```bash
colcon build --symlink-install --cmake-args=-DCMAKE_BUILD_TYPE=Release --parallel-workers $(nproc)
```

**Debug build (with symbols):**
```bash
colcon build --symlink-install --cmake-args=-DCMAKE_BUILD_TYPE=Debug --parallel-workers $(nproc)
```

**Optimized build with debug symbols (required by ZED SDK for debugging):**
```bash
colcon build --symlink-install --cmake-args=-DCMAKE_BUILD_TYPE=RelWithDebInfo --parallel-workers $(nproc)
```

**Skip the internal debug package:**
```bash
colcon build --symlink-install --cmake-args=-DCMAKE_BUILD_TYPE=Release --parallel-workers $(nproc) --packages-skip zed_debug
```

**Source the workspace after building:**
```bash
source install/local_setup.bash
```

**Clean build artifacts before rebuilding:**
```bash
rm -rf install build log
```

## Running the Node

```bash
ros2 launch zed_wrapper zed_camera.launch.py camera_model:=<model>
```

Valid `camera_model` values: `zed`, `zedm`, `zed2`, `zed2i`, `zedx`, `zedxm`, `zedxhdrmini`, `zedxhdr`, `zedxhdrmax`, `virtual`, `zedxonegs`, `zedxone4k`, `zedxonehdr`

**Simulation mode:**
```bash
ros2 launch zed_wrapper zed_camera.launch.py camera_model:=zedx sim_mode:=true
```

**List all launch parameters:**
```bash
ros2 launch zed_wrapper zed_camera.launch.py -s
```

## Linting / Tests

Tests are ament lint checks (copyright, cppcheck, lint_cmake, pep257, uncrustify, xmllint). Run with:
```bash
colcon test --packages-select zed_components zed_wrapper
colcon test-result --verbose
```

## Debugging

The `zed_debug` package loads components in a single C++ process for debugger attachment.

**With gdbserver (for VSCode remote debugging):**
```bash
ros2 launch zed_debug zed_camera_debug.launch.py camera_model:=<model> cmd_prefix:='gdbserver localhost:3000'
```

**With GDB directly:**
```bash
ros2 launch zed_debug zed_camera_debug.launch.py camera_model:=<model> cmd_prefix:='gdb --args'
```

**With Valgrind:**
```bash
ros2 launch zed_debug zed_camera_debug.launch.py camera_model:=<model> cmd_prefix:='valgrind --leak-check=full --track-origins=yes'
```

VSCode `launch.json` for attaching to gdbserver:
```json
{
    "version": "0.2.0",
    "configurations": [{
        "name": "C++ Debugger",
        "request": "launch",
        "type": "cppdbg",
        "miDebuggerServerAddress": "localhost:3000",
        "cwd": "${workspaceFolder}",
        "program": "install/zed_debug/lib/zed_debug/zed_debug_proc",
        "stopAtEntry": true
    }]
}
```

Note: If `isaac_ros_managed_nitros` is installed, `zed_debug` may fail to start due to an incompatibility with static ROS 2 composition. Uninstall that package and rebuild to work around it.

## Architecture

### Package Overview

| Package | Role |
|---|---|
| `zed_ros2` | Meta-package; aggregates all packages as a single installable unit |
| `zed_components` | Core library; implements the ROS 2 component classes and shared tools |
| `zed_wrapper` | Launch wrapper; loads `zed_components` via manual composition, owns YAML configs and URDF |
| `zed_debug` | Internal dev package; loads components in a single process for debugger attachment |

### Component Classes

`zed_components` exposes two ROS 2 component classes:

- **`ZedCamera`** — stereo camera component for all ZED stereo models (ZED, ZED 2, ZED X, etc.). Handles video/depth, object detection, body tracking, spatial mapping, GNSS fusion, positional tracking, and sensor data.
- **`ZedCameraOne`** — mono camera component for ZED X One models. Handles video and sensor data only (no depth, object detection, body tracking, or spatial mapping).

The implementation of `ZedCamera` is split across multiple `.cpp` files by feature domain:
- `zed_camera_component_main.cpp` — lifecycle, initialization, parameter handling
- `zed_camera_component_video_depth.cpp` — image and depth publishing
- `zed_camera_component_objdet.cpp` — object detection
- `zed_camera_component_bodytrk.cpp` — body tracking

### Configuration

All runtime parameters are YAML files in [zed_wrapper/config/](zed_wrapper/config/):

- `common_stereo.yaml` — shared parameters for all stereo cameras (object detection, body tracking, mapping, GNSS, positional tracking, etc.)
- `common_mono.yaml` — shared parameters for ZED X One cameras
- Per-model files (`zed.yaml`, `zed2.yaml`, `zedx.yaml`, `zedxonegs.yaml`, etc.) — model-specific overrides
- `object_detection.yaml` — object detection module settings
- `custom_object_detection.yaml` — YOLO-like ONNX model configuration for custom inference

Parameters can also be overridden at launch time via CLI arguments.

### Feature Compatibility Matrix

| Feature | ZED X One | First-gen ZED | ZED 2/X/etc. |
|---|---|---|---|
| Depth / Point Cloud | No | Yes | Yes |
| Object Detection | No | No | Yes |
| Body Tracking | No | No | Yes |
| Spatial Mapping | No | Yes | Yes |
| GNSS Fusion | No | Yes | Yes |
| 2D Mode | No | Yes | Yes |
| Simulation Mode | No | No | Yes |

### Key Services

- `start_svo_recording` / `stop_svo_recording` — SVO file recording
- `enable_obj_det` — toggle object detection at runtime
- `enable_mapping` — toggle spatial mapping at runtime
- `toLL` / `fromLL` — convert between Lat/Lon and map frame coordinates (GNSS fusion)

### NVIDIA Isaac ROS / NITROS

The wrapper optionally integrates with NVIDIA Isaac ROS via NITROS for GPU-accelerated ROS graph data transport. This is an optional dependency; the wrapper builds and runs without it.

## Prerequisites

- Ubuntu 20.04 / 22.04 / 24.04
- ZED SDK v5.2
- CUDA
- ROS 2 Humble (recommended) or Jazzy
- `zed_msgs` package (available via `apt` on Humble: `ros-humble-zed-msgs`)
