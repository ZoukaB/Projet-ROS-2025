# TurtleBot3 Autonomous Challenges — ROS2 Project

- ASMAR Kevin
- BUCLET Zeca

---

A ROS2 package implementing four autonomous robot behaviors on a **TurtleBot3 Burger** simulated in Gazebo and tested on a real life obstacle course. Each challenge is an independent node that can be launched and tested separately.

All code were run using a virtual machine provided by the university with all the necessary system requirements (Ubuntu 22.04 LTS with ROS2 Jazzy).

An explanation of our work with visual examples can be found in `ROS.pdf`.

---

## Package Overview

**Package name:** `projet_ros_2025`  
**ROS2 distribution:** Humble (or compatible)  
**Simulator:** Gazebo (via `ros_gz_sim`)  
**Robot:** TurtleBot3 Burger

---

## Repository Structure

```
projet_ros_2025/
├── projet_ros_2025/
│   ├── __init__.py
│   ├── teleop_camera.py        # Helper script to teleoperate robot back to start position
│   ├── vision.py               # Helper script to activate camera with filters
│   ├── line_following.py       # Challenge 1: Line following
│   ├── object_avoidance.py     # Challenge 2: Obstacle avoidance with line following
│   ├── corridor.py             # Challenge 3: Corridor navigation
│   └── Football.py             # Challenge 4: Football / goal scoring
├── launch/
│   ├── projet_launch.py        # Main launch file (simulation + node)
│   └── spawn_launch.py         # Spawns robot, ball, and goal in Gazebo
├── setup.py
├── setup.cfg
└── README.md
```

---

## Challenges

### Challenge 1 — Line Following

**Node:** `line_following.py`  
**Executable:** `line_following`

The robot follows a two-color track (red line on the right, green line on the left) using computer vision on a compressed camera feed.

**How it works:**
- Converts the camera image to HSV and isolates red and green color masks in the lower portion of the frame.
- Computes the centroid of each line and steers the robot to keep them on either side.
- Handles edge cases (only one line visible) by turning in the appropriate direction.
- Detects a **blue stop zone** to halt the robot at the end of the course.
- Triggers an **emergency stop** via LIDAR if an obstacle is detected closer than 30 cm.
- Supports a `roundabout_direction` parameter (`left` or `right`) to handle roundabout sections.

**Topics used:**
- `/camera/image_raw/compressed` (subscriber)
- `/scan` (subscriber)
- `/cmd_vel` (publisher)

---

### Challenge 2 — Obstacle Avoidance

**Node:** `object_avoidance.py`  
**Executable:** `object_avoidance`

The robot follows the same red/green line track while dynamically avoiding obstacles detected by the LIDAR.

**How it works:**
- Monitors the front-left and front-right LIDAR sectors and overrides line following to steer away if any obstacle is detected within 25 cm.
- Estimates the local direction of each line using **tangent vectors** computed via `cv2.fitLine`, and drives at a speed proportional to how straight the road ahead is.
- Turns in a configurable direction (`turn_direction` parameter) when both lines are detected very close together, indicating a narrow passage or obstacle.
- Throttles camera processing to 10 FPS to reduce computational load.

**Topics used:**
- `/camera/image_raw/compressed` (subscriber)
- `/scan` (subscriber)
- `/cmd_vel` (publisher)

---

### Challenge 3 — Corridor Navigation

**Node:** `corridor.py`  
**Executable:** `corridor`

The robot navigates a corridor using LIDAR-based PID wall-following, then transitions to visual line following upon detecting a colored marker.

**How it works:**
- Maintains a constant distance (setpoint: 20 cm) from the right wall using a proportional controller on the nearest LIDAR readings in the NE sector (`lidar_pid` state).
- Transitions to visual mode when a **blue line** is detected in the lower part of the camera frame.
- Follows a green line on the ground using centroid-based visual steering (`green_approach` state).
- Triggers a full stop when a second **blue zone** is detected in the center-bottom of the frame, after a minimum number of frames has elapsed to avoid false positives (`stop` state).

**Topics used:**
- `/scan` (subscriber)
- `/image_raw` (subscriber)
- `/cmd_vel` (publisher)

---

### Challenge 4 — Football (Goal Scoring)

**Node:** `Football.py`  
**Executable:** `foot`

The robot autonomously finds a yellow ball, positions itself behind it, then drives toward a red goal to score.

**How it works:**
- Searches for the yellow ball by rotating in place, then aligns with it by centering it in the camera frame (`align_ball` state).
- Drives straight toward the ball until it disappears from the frame, indicating the robot is on top of it (`drive_to_ball` state).
- Rotates to search for the red goal posts, identified via a dual-range HSV red mask (`search_goal` state).
- Aligns with the goal by centering the midpoint between the two detected red posts (`align_goal` state).
- Drives forward to push the ball into the goal (`drive_to_goal` state).
- Applies morphological cleaning (`MORPH_OPEN` + `MORPH_CLOSE`) to both color masks to reduce noise.

**Topics used:**
- `/camera/image_raw/compressed` (subscriber)
- `/cmd_vel` (publisher)

---

## Getting Started

### Prerequisites

- ROS2 Humble (or compatible distribution)
- TurtleBot3 packages
- `ros_gz_sim` and the `projet2025` simulation package
- OpenCV (`cv2`), `cv_bridge`, `numpy`

### Build

```bash
cd ~/ros2_ws
colcon build --packages-select projet_ros_2025
source install/setup.bash
```

### Launch the Simulation

The spawn launch file starts Gazebo with the project world and spawns the robot, ball, and goal. The spawn position is configured per challenge:

```bash
ros2 launch projet_ros_2025 spawn_launch.py
```

> Edit `spawn_launch.py` to switch between challenge positions (`challenge_1` through `challenge_4`).

### Run a Challenge Node

```bash
# Challenge 1 – Line Following
ros2 run projet_ros_2025 line_following

# Challenge 2 – Obstacle Avoidance
ros2 run projet_ros_2025 object_avoidance

# Challenge 3 – Corridor Navigation
ros2 run projet_ros_2025 corridor

# Challenge 4 – Football
ros2 run projet_ros_2025 foot
```

### Optional Parameters

For **line following**, you can specify the roundabout direction:
```bash
ros2 run projet_ros_2025 line_following --ros-args -p roundabout_direction:=left
# or
ros2 run projet_ros_2025 line_following --ros-args -p roundabout_direction:=right
```

---

## Debug & Visualization

All nodes display OpenCV debug windows during execution:

- **Line Following / Object Avoidance:** Shows color masks and detected line centroids.
- **Corridor:** Shows a combined green/blue debug mask with state and error overlaid.
- **Football:** Shows an overlay of detected ball and goal with current state label.

Windows are closed automatically on node shutdown.

---

## Notes

- All nodes publish velocity commands to `/cmd_vel` and are designed to run one at a time.
- The `spawn_launch.py` file contains predefined spawn positions for all four challenges — simply update the variable references (`challenge_1` through `challenge_4`) to match the active challenge.
- `teleop_camera` and `vision` are helper scripts for manual control and camera debugging; they are not part of the autonomous challenge pipeline.