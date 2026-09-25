# testbed_navigation

Manual Nav2 navigation workflow for the Testbed-T1.0.0 robot (ROS 2 Humble +
Gazebo Classic 11), built as an internship assignment. Rather than using
`nav2_bringup`, each stage is its own launch file wired to the individual
Nav2 plugins (`map_server`, `amcl`, planner/controller/BT/behavior servers),
so each piece can be started, tested, and understood independently.

## Approach

- **`map_loader.launch.py`** — starts `map_server`, which is then manually
  taken through its lifecycle (configure → activate) to publish `/map`.
- **`localisation.launch.py`** — starts `amcl`, which consumes `/map` and
  `/scan` and publishes the `map → odom` transform once an initial pose is
  set.
- **`navigation.launch.py`** — starts `planner_server`, `controller_server`,
  `bt_navigator`, `behavior_server`, and `lifecycle_manager_navigation`,
  which together handle `/navigate_to_pose` goals.

Each stage was brought up and verified in isolation (via `ros2 lifecycle
get/set`) before chaining them together, rather than debugging the whole
stack at once.

## How to Run

Run each command in its own terminal, in this order:

1. `ros2 launch testbed_navigation map_loader.launch.py`
2. `ros2 launch testbed_navigation localisation.launch.py`
3. `ros2 lifecycle set /map_server activate`
4. `ros2 launch testbed_bringup testbed_full_bringup.launch.py` (Gazebo + RViz)
5. `ros2 launch testbed_navigation navigation.launch.py`

**In RViz**, once it's up:
- Set **Fixed Frame** to `map`.
- Add a **Map** display, set its topic to `/map`, and set its QoS
  reliability/durability to **Transient Local** so it picks up the
  already-published map.
- Set an initial pose with **2D Pose Estimate** matching the robot's real
  spawn point, then send goals with **2D Goal Pose**.

## Challenges Faced

The two hardest issues, in short (full diagnosis and fixes in `BUGS.txt`):

- **`behavior_server` wouldn't start** — crashed with a segfault, then with a
  fatal plugin-load error. Caused by wrong Humble parameter names
  (`local_costmap_topic` → should be `costmap_topic`) and a wrong plugin
  type separator (`nav2_behaviors::Spin` → should be `nav2_behaviors/Spin`).
- **Robot failed to reliably pass through narrow doorways** — traced to AMCL
  confidently localizing to the *wrong* spot in symmetric, corridor-heavy
  parts of the map (confirmed via `/amcl_pose` covariance: huge uncertainty
  along the corridor axis, tight across it — classic corridor degeneracy).
  Fixed by lowering AMCL's `alpha1`–`alpha4` odometry-noise parameters so
  wheel odometry is trusted more than ambiguous single-scan laser matches.

Other issues (missing `bt_navigator` BT plugins, a Gazebo lidar sensor
dropping nearly all scan points due to a stray `min_intensity` filter,
environment/Docker setup problems) are all itemized with root cause and fix
in `BUGS.txt`.

## Verification

Tested across multiple goal poses and repeated runs, including through
narrow doorways, with RViz's localized robot position matching Gazebo's
ground-truth position throughout.

## Demo & Supporting Material

- **Video + error screenshots:** [Google Drive folder](https://drive.google.com/drive/folders/13fkjQlbFo67lAZ1HU6wucGyMxR66U0f_?usp=sharing)
  — includes a run showing successful localization/navigation and screenshots
  of the errors encountered along the way.
- **`BUGS.txt`** (repository root, outside this package) — full itemized
  list of starter-repo bugs vs. implementation issues, with root cause and
  fix for each.
- **`docs/ros2_assignment_problem_summary.pdf`** (repository root) —
  detailed write-up of the debugging process, including the AMCL covariance
  diagnosis behind the narrow-doorway fix.
