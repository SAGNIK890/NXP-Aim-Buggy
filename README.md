# 🤖 NXP AIM Buggy — Warehouse Navigation & Object Recognition

This project is based on the **NXP AIM India 2025 robotics challenge**, where the goal is to make an autonomous B3RB buggy navigate through a warehouse environment, find shelves, decode QR codes, recognize objects and efficiently complete the given treasure-hunt style task.

I worked with the **ROS 2 + Gazebo simulation environment** provided for the challenge so that the robot's navigation, sensing and decision-making could be tested without needing the physical buggy every time.

The main focus here is on understanding how all the different robotics components come together:

```text
        Gazebo Simulation
               │
               ▼
        B3RB Mobile Robot
               │
       ┌───────┼────────┐
       ▼       ▼        ▼
     LiDAR   Camera    IMU
       │       │        │
       └───────┼────────┘
               ▼
          ROS 2 System
               │
       ┌───────┼───────────┐
       ▼       ▼           ▼
      SLAM    Nav2      Object Detection
       │       │           │
       └───────┼───────────┘
               ▼
        Decision Making
               │
               ▼
       Shelf / QR / Object
          Recognition
```

---

# 🏆 About the Challenge

The project is based on the **NXP AIM India 2025 Warehouse Treasure Hunt & Object Recognition challenge**.

The basic idea of the challenge is to have the B3RB navigate inside a warehouse containing multiple shelves and obstacles.

The robot has to:

* Find shelves
* Navigate to them
* Read QR codes
* Determine the shelf sequence
* Recognize objects placed on the shelves
* Publish the required shelf information
* Move through the warehouse efficiently

The challenge framework provides the basic ROS 2 and simulation infrastructure, while the participant is expected to develop the actual decision-making and challenge-solving logic.

---

# 🤖 Robot

The target robot for the project is the **NXP MR-B3RB**.

The B3RB is treated as a mobile autonomous platform equipped with sensors required for navigation and perception.

The important sensors/components involved in the simulation include:

* LiDAR
* IMU
* Wheel odometry / encoders
* Front-facing camera
* ROS 2 communication
* Nav2 navigation stack
* SLAM

The same software architecture can be developed and tested inside the Gazebo simulation before transferring the solution to the physical robot.

---

# 🌐 Simulation Environment

The project uses **Gazebo** to simulate the warehouse and the B3RB.

This is particularly useful because robotics development gets expensive very quickly when every small change requires testing on the physical robot.

With simulation, I can:

```text
Change algorithm
      ↓
Build ROS workspace
      ↓
Launch Gazebo
      ↓
Run the buggy
      ↓
Observe sensors
      ↓
Debug
      ↓
Repeat
```

The challenge environment provides multiple warehouse configurations for testing.

The official challenge framework includes:

```text
warehouse_1
warehouse_2
warehouse_3
warehouse_4
```

Each environment can have different shelf counts, starting positions and initial navigation parameters.

---

# 🗺️ SLAM

One of the important parts of the system is **SLAM — Simultaneous Localization and Mapping**.

The robot uses its sensor information to build a representation of the environment while simultaneously estimating where it is inside that environment.

The system works with ROS 2 occupancy grids such as:

```text
/map
/global_costmap/costmap
```

The map represents the warehouse as a grid where cells can indicate:

```text
0    → Free space
100  → Occupied space
-1   → Unknown / unexplored
```

This map becomes extremely useful for finding possible shelf locations and planning safe paths around obstacles.

---

# 🧭 Navigation with Nav2

The project uses **ROS 2 Nav2** for autonomous navigation.

Instead of manually controlling every movement of the buggy, a navigation goal can be given to Nav2.

A navigation goal generally contains:

```text
X position
Y position
Yaw / orientation
```

Nav2 then handles the actual path planning and movement of the robot.

This allows the higher-level challenge logic to focus on:

```text
Where should I go?
```

while Nav2 handles much of:

```text
How should I get there?
```

The challenge framework exposes the `NavigateToPose` action for sending navigation goals.

---

# 📷 Camera & QR Detection

The front camera plays a major role in the challenge.

Each shelf contains a QR code that provides information required for navigating the shelf sequence.

The QR information contains:

```text
Shelf ID
Heuristic angle
Secret/random string
```

For example, a QR string can look conceptually like:

```text
2_116.6_HKq3wvCg8DGyflz3oNIj8d
```

Here:

```text
2       → Shelf ID
116.6   → Heuristic angle
...     → Secret code
```

The heuristic provides useful information about the direction of the next shelf.

This is important because the challenge is not simply:

> "Find every shelf."

The idea is to use the information obtained from the current shelf to make the next movement more intelligent.

---

# 🎯 The Treasure Hunt Logic

The overall challenge can be thought of as a loop:

```text
Find Shelf
   ↓
Move Near Shelf
   ↓
Find QR Code
   ↓
Decode QR
   ↓
Understand Next Target
   ↓
Move to Object Viewing Position
   ↓
Recognize Objects
   ↓
Publish Shelf Information
   ↓
Move to Next Shelf
   ↓
Repeat
```

This is where the project becomes more interesting than simply running a navigation stack.

The robot needs a proper **state-management strategy** so that it knows what it is currently doing and what needs to happen next.

---

# 🧩 Shelf Detection

The first major challenge is locating the shelf.

There are two possible approaches provided by the challenge framework:

### Map-based detection

Use the SLAM-generated map to identify shelf footprints and estimate their positions.

Relevant map topics include:

```text
/map
/global_costmap/costmap
```

The robot can process the occupancy grid and estimate useful positions in the world coordinate frame.

### Vision-based detection

Another approach is to use the front camera and process the image to identify the shelf directly.

This makes the camera useful not only for QR recognition but also potentially for shelf localization.

---

# 📦 Object Recognition

Finding a shelf is only part of the task.

The robot also needs to identify the objects placed on it.

The challenge framework provides an external YOLO-based object recognition component.

The detected shelf objects are published through:

```text
/shelf_objects
```

using:

```text
synapse_msgs/WarehouseShelf
```

The participant's logic can subscribe to this topic and associate the detected objects with the current shelf.

---

# 📤 Publishing Shelf Data

After identifying the objects and decoding the QR code, the required information is published through:

```text
/shelf_data
```

using:

```text
synapse_msgs/WarehouseShelf
```

This is an important part of the challenge because the system isn't just expected to navigate around the warehouse.

The robot has to communicate what it has discovered back to the challenge system.

---

# 🧠 Decision Making

The actual intelligence of the system comes from combining all these pieces.

For example:

```text
Current Robot Pose
       +
SLAM Map
       +
QR Information
       +
Object Recognition
       +
Shelf State
       ↓
Decision
       ↓
Navigation Goal
```

The robot therefore needs to maintain information such as:

* Current shelf
* Previously visited shelves
* Decoded QR information
* Next target
* Object detections
* Navigation state
* Recovery state
* Whether shelf information has already been published

This naturally leads toward a state-machine style architecture.

---

# 🔄 Suggested State Flow

A practical way to think about the complete system is:

```text
START
  │
  ▼
EXPLORE
  │
  ▼
FIND SHELF
  │
  ▼
NAVIGATE TO SHELF
  │
  ▼
FIND QR
  │
  ▼
DECODE QR
  │
  ▼
MOVE TO OBJECT VIEW POSITION
  │
  ▼
GET OBJECT DETECTIONS
  │
  ▼
PUBLISH SHELF DATA
  │
  ▼
CALCULATE NEXT TARGET
  │
  ▼
NAVIGATE TO NEXT SHELF
  │
  └───────────────► REPEAT
```

This keeps the system easier to debug than putting the entire challenge logic into one large callback.

---

# 📡 Important ROS 2 Topics

Some of the important topics involved in the challenge are:

| Topic                          | Purpose                        |
| ------------------------------ | ------------------------------ |
| `/pose`                        | Current robot pose             |
| `/map`                         | SLAM map                       |
| `/global_costmap/costmap`      | Global navigation costmap      |
| `/camera/image_raw/compressed` | Camera feed                    |
| `/shelf_objects`               | YOLO object recognition output |
| `/shelf_data`                  | Published shelf information    |
| `/cerebri/out/status`          | Robot/controller status        |
| `/cerebri/in/joy`              | Robot control / mode input     |
| `/debug_images/qr_code`        | Optional QR debugging image    |

These topics form the communication layer between the different parts of the robot software.

---

# 🧱 Main Software Components

The challenge environment is based around the **CogniPilot AIRY** ecosystem and a ROS 2 workspace called `cranium`.

Some of the important packages/components include:

```text
b3rb_ros_aim_india
synapse_msgs
dream_world
b3rb
b3rb_nav2
```

The B3RB-specific software runs within this larger ROS 2 ecosystem.

---

# 🛠️ Technologies Used

```text
ROS 2
Gazebo
Nav2
SLAM Toolbox
Python
rclpy
OpenCV
NumPy
SciPy
YOLO
TensorFlow Lite
pyzbar
Tkinter
Foxglove
CogniPilot AIRY
```

The challenge environment targets **ROS 2 Humble** and Ubuntu 22.04.5 according to the official setup instructions.

---

# 🖥️ System Architecture

The overall software architecture looks roughly like this:

```text
                    ┌──────────────────────┐
                    │   Gazebo Simulation  │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │       B3RB           │
                    │                      │
                    │ LiDAR │ Camera │ IMU │
                    └──────────┬───────────┘
                               │
                               ▼
                         ┌───────────┐
                         │   ROS 2   │
                         └─────┬─────┘
                               │
             ┌─────────────────┼─────────────────┐
             ▼                 ▼                 ▼
          SLAM / Map          Nav2           Camera
             │                 │                 │
             │                 │          ┌──────┴──────┐
             │                 │          ▼             ▼
             │                 │         QR           YOLO
             │                 │       Decoder       Objects
             │                 │          │             │
             └─────────────────┼──────────┴─────────────┘
                               ▼
                       Challenge Logic
                               │
                               ▼
                         Shelf Data
```

---

# 🚗 Warehouse Environments

The official simulation framework provides four warehouse environments:

### Warehouse 1

```text
warehouse_id = 1
shelf_count = 2
initial_angle = 135.0
```

### Warehouse 2

```text
warehouse_id = 2
shelf_count = 4
initial_angle = 040.6
```

### Warehouse 3

```text
warehouse_id = 3
shelf_count = 3
initial_angle = 045.0
```

### Warehouse 4

```text
warehouse_id = 4
shelf_count = 5
initial_angle = 045.0
```

The exact starting position and yaw also vary between the worlds. These parameters are supplied when launching the simulation.

---

# ⚙️ Setting Up the Simulation

The official challenge setup is intended for:

```text
Ubuntu 22.04.5
ROS 2 Humble
```

The complete environment is based on CogniPilot AIRY and the NXP AIM India 2025 packages.

The general setup process is:

```text
Install CogniPilot
       ↓
Build B3RB workspace
       ↓
Install NXP AIM India packages
       ↓
Install simulation dependencies
       ↓
Build Cranium
       ↓
Launch Gazebo
       ↓
Run the robot software
```

---

# 🌍 Launching Gazebo

After the workspace has been built and sourced, a warehouse can be launched using:

```bash
cd ~/cognipilot/cranium/

colcon build

source ~/cognipilot/cranium/install/setup.bash
```

Then launch the required warehouse.

For example:

```bash
ros2 launch b3rb_gz_bringup sil.launch.py \
world:=nxp_aim_india_2025/warehouse_1 \
warehouse_id:=1 \
shelf_count:=2 \
initial_angle:=135.0 \
x:=0.0 \
y:=0.0 \
yaw:=0.0
```

Other warehouse environments can be launched by changing the world and corresponding parameters.

---

# 👀 Foxglove

For debugging and visualization, the environment can also be connected to **Foxglove**.

This is useful for looking at things like:

* Robot pose
* Maps
* Camera feeds
* Navigation information
* QR debugging images
* Object recognition output

The official setup provides an `electrode` workspace for this purpose and exposes debugging topics such as:

```text
/debug_images/object_recog
/debug_images/qr_code
```

A Foxglove WebSocket connection can then be made to the local simulation.

---

# 🧪 Development Workflow

My preferred workflow for this kind of project is basically:

```text
1. Launch simulation
        ↓
2. Observe robot behaviour
        ↓
3. Identify failure
        ↓
4. Modify navigation / perception logic
        ↓
5. colcon build
        ↓
6. Source workspace
        ↓
7. Run simulation again
        ↓
8. Compare results
```

Robotics is mostly:

> change → build → test → robot does something stupid → figure out why → repeat.

And honestly, Gazebo makes that loop a lot less painful.

---

# 🧭 Navigation Tuning

The `b3rb_nav2` package contains several configuration files that can be tuned depending on the navigation strategy.

Important files include:

```text
nav_to_pose_bt.xml
nav_through_poses_bt.xml
nav2.yaml
slam.yaml
```

These can influence things such as:

* Navigation behaviour
* Goal tolerances
* Obstacle inflation
* Recovery behaviour
* Local/global planning
* SLAM parameters

There is always a trade-off between:

```text
Mapping accuracy
CPU usage
Responsiveness
Robustness
Stability
```

So blindly increasing every parameter isn't necessarily going to make the robot magically smarter.

---

# 🚨 Navigation Recovery

One thing that becomes very obvious when working with autonomous robots is that reaching a goal isn't always as simple as sending coordinates.

A navigation goal can fail because of:

* Poor localization
* Tight obstacles
* Incorrect costmap configuration
* Bad goal orientation
* Robot getting too close to an obstacle
* Planner failures

The challenge documentation specifically recommends implementing recovery logic where necessary, including cancelling a problematic navigation goal and moving the robot away before trying again.

A simple recovery concept is:

```text
Navigation Failed
       ↓
Cancel Goal
       ↓
Check Robot / Map
       ↓
Move Away
       ↓
Recalculate Goal
       ↓
Try Again
```

---

# 🧠 What Makes This Project Interesting

For me, the interesting part of this project isn't just making a robot move.

It's getting several independent robotics systems to cooperate:

```text
SLAM
 +
Navigation
 +
Computer Vision
 +
QR Detection
 +
Object Detection
 +
State Management
 +
Decision Making
```

Each individual component can work perfectly and the complete robot can still fail if they aren't synchronized properly.

For example:

```text
QR decoded correctly
        +
Object detected correctly
        +
Navigation works correctly
        =
Still fails
```

if the system publishes the shelf information at the wrong time.

That's the kind of integration problem that makes autonomous robotics interesting.

---

# ⭐ Project USP

The main strength of this project is the **integration of navigation, perception and decision-making inside a simulated autonomous mobile robot**.

It isn't just:

```text
Object Detection
```

and it isn't just:

```text
ROS Navigation
```

The complete challenge involves:

```text
        ┌─────────────┐
        │   Mapping   │
        └──────┬──────┘
               │
        ┌──────▼──────┐
        │ Navigation  │
        └──────┬──────┘
               │
        ┌──────▼──────┐
        │ Perception  │
        └──────┬──────┘
               │
        ┌──────▼──────┐
        │   Decision  │
        └──────┬──────┘
               │
        ┌──────▼──────┐
        │   Action    │
        └─────────────┘
```

That combination makes it a good practical project for understanding autonomous robotics rather than just individual algorithms.

---

# 🔧 Things That Can Be Improved

There is plenty of scope for improvement.

Some areas I would like to explore further include:

### Navigation

* Better shelf approach poses
* More reliable obstacle recovery
* Improved Nav2 tuning
* Smarter waypoint selection
* Faster route planning

### Perception

* More robust shelf detection
* Better QR detection under different viewing angles
* Improved object recognition
* Camera-based localization

### Decision Making

* Proper finite-state machine
* Better handling of failed detections
* Dynamic replanning
* More efficient use of the QR heuristic

### Simulation

* More warehouse layouts
* Randomized obstacle positions
* Sensor noise
* Different lighting conditions
* More realistic camera noise

---

# 🧪 Possible Future Architecture

A more advanced version of the system could look like:

```text
             Sensors
                │
       ┌────────┼────────┐
       ▼        ▼        ▼
     LiDAR    Camera     IMU
       │        │        │
       ▼        ▼        ▼
      SLAM    Vision   Odometry
       │        │        │
       └────────┼────────┘
                ▼
         World Understanding
                │
                ▼
          State Machine
                │
        ┌───────┼────────┐
        ▼       ▼        ▼
     Explore   QR      Objects
        │       │        │
        └───────┼────────┘
                ▼
          Target Selection
                │
                ▼
              Nav2
                │
                ▼
              B3RB
```

This would allow the robot to make more deliberate decisions instead of simply executing a fixed sequence.

---

# 📚 What This Project Helped Me Understand

Working with this type of simulation gives practical exposure to several robotics concepts:

* ROS 2 nodes
* Topics
* Messages
* Action clients
* Gazebo simulation
* SLAM
* Occupancy grids
* Costmaps
* Nav2
* Robot localization
* Computer vision
* QR detection
* Object detection
* Sensor integration
* State machines
* Autonomous navigation
* Recovery behaviour

It also makes one thing painfully clear:

**A robot can have ten different things working correctly and still refuse to go where you told it to.**

That's robotics. 😭

---

# ⚠️ Important Notes

The official NXP challenge environment has specific software and dependency requirements.

The challenge documentation specifies Ubuntu 22.04.5 and ROS 2 Humble, along with a defined set of Python dependencies. Additional Python modules may require permission from the NXP AIM technical team during competition evaluation.

The evaluation environment also restricts what can be installed, so keeping the final solution within the supported environment is important.

---

# 📦 Repository Contents

This repository currently contains the simulation package:

```text
NXP_AIM_INDIA_2025-nxp_aim_india_2025_simulation.zip
```

The repository also contains this README.

The simulation archive is approximately **450 KB** in the GitHub repository.

---

# 🔗 References

### NXP AIM India 2025 Platform

The underlying official challenge framework is maintained by the NXP Robotics organization:

https://github.com/NXP-Robotics/NXP_AIM_INDIA_2025

It contains the challenge description, hardware/software information, ROS 2 architecture, simulation instructions and participant implementation guidelines.

### This Repository

https://github.com/SAGNIK890/NXP-Aim-Buggy

---



GitHub:

https://github.com/SAGNIK890

---

# 🚀 Final Note

This project is essentially my workspace for exploring autonomous mobile robotics through the **NXP AIM India 2025 B3RB warehouse challenge**.

The interesting part is not one particular algorithm.

It is the complete chain:

```text
Sense
  ↓
Understand
  ↓
Plan
  ↓
Navigate
  ↓
Act
  ↓
Verify
  ↓
Repeat
```

That's what makes a robot autonomous.

And that's also the part I'm most interested in exploring further.
