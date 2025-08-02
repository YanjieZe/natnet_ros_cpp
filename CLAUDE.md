# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a ROS (Robot Operating System) C++ package that provides a driver for OptiTrack's NatNet protocol version 4.0. It streams motion capture data (rigid bodies, markers, skeletons) from OptiTrack Motive software to ROS topics. The package supports both multicast and unicast communication modes.

## Build System & Commands

**Building the package:**
```bash
cd ~/catkin_ws/src
git clone <repo-url>
cd ..
catkin build  # OR catkin_make
. devel/setup.bash
```

**Prerequisites:**
```bash
sudo apt install -y ros-$ROS_DISTRO-tf2* wget
```

**Launch commands:**
- GUI interface: `roslaunch natnet_ros_cpp gui_natnet_ros.launch`
- Command line: `roslaunch natnet_ros_cpp natnet_ros.launch`

**Key launch parameters:**
- `serverIP`: OptiTrack host PC IP (default: 192.168.0.100)
- `clientIP`: ROS client PC IP (default: 192.168.0.103)  
- `serverType`: "multicast" or "unicast" (default: multicast)
- `pub_rigid_body`: Enable rigid body publishing (default: true)
- `pub_individual_marker`: Enable individual marker tracking (default: false)
- `pub_skeleton`: Enable skeleton data publishing (default: false)
- `conf_file`: Configuration file name (default: initiate.yaml)

## Architecture

### Core Components

1. **Main Node (`natnet_ros.cpp`)**: Entry point that initializes NatNet client, establishes connection to OptiTrack server, and sets up frame callback handling.

2. **Internal Class (`internal.h/cpp`)**: Core logic handler containing:
   - Connection management to NatNet server
   - Data processing and ROS message publishing
   - Parameter management from ROS param server
   - Asset tracking (rigid bodies, skeletons, force plates, devices)

3. **Nearest Neighbor Filter (`nn_filter.h/cpp`)**: Implements tracking algorithm for individual markers using iterative closest point matching.

4. **Marker Poses Server (`marker_poses_server.cpp`)**: ROS service node for querying marker positions.

### Key Data Flow

1. NatNet client receives frames from OptiTrack → `FrameCallback()` → `Internal::DataHandler()`
2. Data is parsed and published to ROS topics:
   - Rigid bodies: `/natnet_ros/<body-name>/pose` (PoseStamped)
   - Markers: `/natnet_ros/<body-name>/marker#/pose` (PointStamped)
   - Individual markers: `/natnet_ros/<config-name>/pose` (PoseStamped)
   - Skeleton bones: `/natnet_ros/<skeleton-name>/<bone-name>/pose` (PoseStamped)
   - Point clouds: sensor_msgs/PointCloud

### Configuration System

- **Main config**: `launch/natnet_ros.launch` - Connection and publishing parameters
- **Individual markers**: `config/initiate.yaml` - Defines trackable individual markers with initial positions
- **Service definitions**: `srv/MarkerPoses.srv` - Custom service for marker queries

### Dependencies

- **External**: NatNet SDK (auto-downloaded during build via `install_sdk.sh`)
- **ROS**: geometry_msgs, tf2, tf2_ros, sensor_msgs, visualization_msgs
- **System**: Boost (≥1.65), Eigen3, Qt5 (for GUI)

### Publishing Modes

The system can publish data in multiple modes controlled by launch parameters:
- Rigid body poses with TF broadcasting
- Individual marker points from rigid bodies  
- Tracked individual markers using nearest neighbor algorithm
- All unlabeled markers as point cloud data
- Skeleton bone poses with TF broadcasting (each bone published as separate PoseStamped)

### Skeleton Publishing Details

When `pub_skeleton:=true`, the system:
1. **Discovery**: Detects skeletons during initialization via `Descriptor_Skeleton`
2. **Publisher Setup**: Creates ROS publishers for each bone: `/natnet_ros/<skeleton_name>/<bone_name>/pose`
3. **Name Sanitization**: Replaces spaces and hyphens with underscores in skeleton/bone names for ROS compatibility
4. **Runtime Publishing**: Maps bone IDs to bone names, publishes `geometry_msgs::PoseStamped` for each bone
5. **TF Broadcasting**: Broadcasts bone transforms as `<skeleton_name>_<bone_name>`
6. **Error Handling**: Includes null-pointer checks, missing bone warnings, and skeleton validation

### SDK Integration

The build system automatically downloads the NatNet SDK to `deps/NatNetSDK/` and links against `libNatNet.so`. The SDK provides the core NatNet client functionality for communicating with OptiTrack systems.