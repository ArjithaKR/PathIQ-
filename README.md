## Technical Approach

PathIQ is built on a six-stage pipeline that takes raw sensor data 
and converts it into safe, real-time vehicle navigation decisions. 
Every stage is designed specifically for the unpredictability of 
Indian road conditions — not adapted from Western AV stacks, but 
built ground-up for potholes, cattle, lane-less roads, and erratic 
traffic behavior.

---

### Stage 1 — Sensor Input

PathIQ fuses data from four sensor modalities simultaneously:

- **LiDAR (RPLIDAR A1)** — generates a 360° point cloud of the 
  surrounding environment at 10Hz, providing accurate depth and 
  distance measurements independent of lighting conditions
- **Camera (Logitech C920 / equivalent)** — captures RGB frames 
  at 30fps for visual obstacle detection, surface classification, 
  and drivable area identification
- **IMU (ICM-20948)** — measures acceleration, gyroscope, and 
  magnetometer data to track vehicle orientation and motion state
- **GPS Module** — provides global positioning as primary 
  localization source; automatically demoted when signal degrades 
  in tunnels, forests, or urban canyons

Multi-modal redundancy is built in by design — if any single 
sensor degrades due to dust, rain, or hardware failure, the 
remaining sensors compensate without interrupting navigation.

---

### Stage 2 — Sensor Fusion

Raw data from all four sensors is merged using an **Extended 
Kalman Filter (EKF)**:

- Produces a unified, noise-reduced environment representation 
  at every timestep
- Combines depth (LiDAR), visual (Camera), motion (IMU), and 
  position (GPS) data into a single consistent state estimate
- Handles asynchronous sensor update rates gracefully — LiDAR 
  at 10Hz, camera at 30fps, IMU at 100Hz, GPS at 1Hz
- Outputs: fused obstacle map, vehicle pose estimate, velocity 
  vector — all fed forward to the Perception stage

---

### Stage 3 — Perception

PathIQ's perception stack is purpose-built for Indian road 
conditions using the **Indian Driving Dataset (IDD)** from 
IIIT Hyderabad — the only large-scale annotated dataset 
capturing Indian-specific road actors and conditions.

**Obstacle Detection — YOLOv8 (fine-tuned on IDD)**
- Detects 40+ object classes including categories absent from 
  Western datasets: auto-rickshaws, cattle, undefined road edges, 
  roadside vendors, and pothole regions
- Runs at 30fps on NVIDIA Jetson Nano using TensorRT optimization
- Confidence thresholding triggers safe-stop if detection falls 
  below acceptable certainty

**Multi-Object Tracking — Deep SORT**
- Assigns persistent IDs to every detected object across frames
- Maintains position, velocity, and trajectory history for each 
  tracked actor
- Enables handoff to the Prediction stage with continuous 
  object identity

**Drivable Area Detection — CNN**
- Classifies every pixel of the road surface as drivable or 
  non-drivable
- Handles scenarios with no lane markings, broken edges, muddy 
  shoulders, and mixed-use paths
- Critical for pothole avoidance and surface-aware path planning

---

### Stage 4 — Prediction

Most AV systems react to obstacles — PathIQ anticipates them.

**LSTM-Based Behavior Prediction**
- A Long Short-Term Memory (LSTM) network processes the 
  trajectory history of every tracked actor
- Predicts the next 1–3 seconds of movement for pedestrians, 
  vehicles, animals, and cyclists
- Specifically trained on erratic behaviors common in Indian 
  traffic: sudden lane changes, wrong-side driving, unmotored 
  cart drift, and jaywalking
- Outputs a probability distribution over future positions, 
  flagging high-risk actors before collision risk arises

This stage is what separates PathIQ from reactive collision 
avoidance — the planner receives predicted future states, not 
just current positions.

---

### Stage 5 — Path Planning

PathIQ uses a two-layer planning architecture — global route 
planning and local real-time replanning — operating simultaneously:

**Global Planner — Hybrid A***
- Computes the optimal route from current position to destination
- Hybrid A* combines the completeness of A* search with the 
  kinematic feasibility of continuous-space planning
- Accounts for vehicle turning radius, speed limits, and 
  road surface quality in cost computation
- Re-runs automatically when the environment changes 
  significantly (new obstacle, blocked path)

**Local Planner — Dynamic Window Approach (DWA)**
- Handles moment-to-moment obstacle avoidance within a 
  short planning horizon (1–2 seconds ahead)
- Samples velocity commands within the vehicle's dynamic 
  constraints and scores them against obstacle clearance, 
  goal heading, and speed
- Replans at every control cycle (~10Hz) — fast enough for 
  sudden obstacles like animals crossing or potholes appearing

**Motion Controller — Model Predictive Control (MPC)**
- Takes the planned path and outputs smooth, physically 
  feasible actuation commands
- Optimizes over a receding time horizon to minimize 
  tracking error while respecting acceleration and 
  steering limits
- Ensures comfortable, jerk-free vehicle motion even 
  during aggressive replanning events

---

### Stage 6 — Control & Actuation

Final outputs from MPC are sent to the vehicle's actuators:

- **Steering** — smooth heading corrections following the 
  planned path
- **Throttle** — speed regulation based on obstacle proximity, 
  road surface quality, and planned route curvature
- **Braking** — emergency stop triggered if perception 
  confidence drops below threshold OR predicted collision 
  time falls below safety margin

**Feedback Loop**
The entire pipeline re-evaluates every **200ms** — approximately 
5 full cycles per second. Sensor data continuously updates the 
fused environment model, and the planner adjusts in real-time. 
This closed-loop architecture means PathIQ never operates on 
stale information.

**Safe-Stop Trigger Conditions:**
- Perception confidence < defined threshold
- Predicted collision time < 1.5 seconds with no viable 
  replanning option
- Sensor fusion failure (loss of 2+ sensor modalities)
- GPS + IMU localization uncertainty exceeds safe bounds

---

### Hardware Stack

| Component | Specification | Role |
|---|---|---|
| NVIDIA Jetson Nano | 4GB, 128-core Maxwell GPU | Main compute — runs all AI inference |
| RPLIDAR A1 | 360°, 10Hz, 12m range | LiDAR depth sensing |
| Camera | 1080p, 30fps | Visual perception |
| ICM-20948 IMU | 9-DOF | Orientation and motion |
| GPS Module | UART, NMEA output | Global localization |
| Raspberry Pi 5 | 8GB | Fallback compute and logging |
| Total hardware cost | ~₹35,000 | 10× cheaper than Western AV kits |

---

### Software Stack

| Layer | Tools |
|---|---|
| Programming | Python 3.10, MATLAB R2024a, Simulink |
| AI / ML | YOLOv8 (Ultralytics), PyTorch, TensorFlow Lite |
| Tracking | Deep SORT |
| Path Planning | Custom Hybrid A*, DWA, MPC (Python + MATLAB) |
| Sensor Fusion | FilterPy (Kalman Filter), NumPy, SciPy |
| Simulation | CARLA 0.9.14, MATLAB Automated Driving Toolbox, ROS 2 |
| Optimization | TensorRT (Jetson inference acceleration) |
| Dataset | IDD — Indian Driving Dataset (IIIT Hyderabad) |

---

### Simulation Environment

Before any physical hardware testing, PathIQ is fully validated 
in simulation:

- **CARLA** — open-source AV simulator used to generate 
  Indian-road-inspired scenarios: uneven surfaces, unmarked 
  roads, mixed traffic, low visibility conditions
- **MATLAB Automated Driving Toolbox** — used for sensor 
  modeling, path planning algorithm validation, and 
  closed-loop simulation with ground truth comparison
- **Simulink RoadRunner** — custom Indian road scene 
  generation with potholes, narrow lanes, and traffic agents

Simulation-first development means zero hardware dependency 
during algorithm development — any team with a laptop can 
run and validate PathIQ's core pipeline.

---

### Key Design Decisions

**Why IDD over COCO/Cityscapes?**
Standard Western datasets do not contain auto-rickshaws, 
cattle, undefined road edges, or the dense mixed-traffic 
scenarios common across Indian cities and highways. Training 
on IDD gives PathIQ perception capabilities that are simply 
not achievable with off-the-shelf model weights.

**Why Hybrid A* over pure A*?**
Pure A* operates on a discrete grid and ignores vehicle 
kinematics — the resulting paths are geometrically correct 
but physically impossible for a real vehicle to follow. 
Hybrid A* plans in continuous space and respects turning 
radius constraints, producing paths a real vehicle can 
actually execute.

**Why DWA for local planning?**
DWA is computationally lightweight and runs at high frequency 
(10Hz+), making it ideal for edge hardware. It natively 
handles the vehicle's dynamic constraints and produces 
smooth velocity commands rather than discrete waypoints.

**Why MPC for control?**
MPC provides predictive, constraint-aware control — it looks 
ahead over a short horizon and optimizes actuation to minimize 
future tracking error. This produces significantly smoother 
motion than reactive PID controllers, especially during 
replanning events.

**Why Jetson Nano?**
At ~₹15,000, the Jetson Nano provides a 128-core GPU capable 
of running TensorRT-optimized YOLOv8 inference at 30fps — 
the minimum viable rate for real-time obstacle detection. 
This makes PathIQ deployable on commercial vehicles, logistics 
fleets, and public transport without specialized hardware.
