# 集群调试log

## 雷达倾斜安装

因为是/Odometry -----/vins_to_mavros----> /mavros/vision_pose/pose单线转换,实现了**传感器数据与机器人位姿（Pose）的分离**。

- 这里cloud registered是倾斜的

- 在节点话题图中是看不到** /mavros/vision_pose/pose**和**/mavros/local_position/odom 

   /mavros/local_position/pos**的关系的

  是因为:`/mavros/vision_pose/pose` 是**喂给**飞控的数据（**输入**），而 `/mavros/local_position/odom` 和 `/mavros/local_position/pose` 则是飞控经过内部融合计算后，**反馈给您*的结果（**输出**）。

  这些任务是在单片机上运行的

  > 补充,/mavros话题之间的关系,以及/mavros/vision_pose/pose何去何从,以及飞控板中的内部消息处理逻辑,以及各个mavros话题
  >
  > 之间的关系还有处理逻辑

  > ***回答***见下方正文

  ## mavros逻辑梳理

  好的，这是一个非常深入的问题。要理解您列出的这一百多个 MAVROS 话题，最好的方法不是逐一解释，而是将它们按**功能和数据流向**分成几大类。

  MAVROS 本质上是运行在机载电脑（如树莓派、NVIDIA Jetson）上的 **ROS 系统** 和无人机**飞控板**（如 Pixhawk, a running PX4 or ArduPilot firmware）之间的一座**双向大桥**。它通过 **MAVLink 协议** 来回传递消息。

  

  ### **核心数据流逻辑**

  

  我们可以把整个系统的数据流向简化为下图：

  ```
  +---------------------------------+          +-----------------------------+
  |    机载电脑 (Companion Computer)    |          |      飞控板 (Flight Controller) |
  |            (ROS)                |          |      (e.g., PX4 with EKF2)  |
  +---------------------------------+          +-----------------------------+
                ^   |                                      ^   |
                |   |                                      |   |
    [指令/外部数据]  |   V  [状态/传感器数据]                [指令]   |   V  [内部传感器数据]
                |   |                                      |   |
                |   |        MAVLink 协议通信        |   |
  +---------------------------------+ <=================> +-----------------------------+
  |             MAVROS              |          |      MAVLink Endpoint         |
  +---------------------------------+          +-----------------------------+
  ```

  

  ### **第一类：感知与定位 (飞控 -> ROS)**

  数据流向主要是从**飞控到机载电脑**。

  - **原始传感器数据**:
    - `/mavros/imu/data_raw`: 飞控原始的IMU数据（加速度计、陀螺仪）。
    - `/mavros/global_position/raw/fix`: 原始的GPS定位信息。
    - `/mavros/altitude`: 融合了气压计等信息的高度。
  - **融合后的状态估计 (最常用)**:
    - **`/mavros/local_position/pose`**: 飞控内部EKF（扩展卡尔曼滤波器）融合所有传感器（包括您提供的视觉位姿）后，得出的在**本地坐标系（odom或map）下的位置和姿态**。
    - **`/mavros/local_position/odom`**: 比上面的 `pose` 更完整，除了位姿，还包含了**速度信息**。**这通常是上层应用（如运动规划）最应该使用的位姿来源**。
  - **系统状态**:
    - `/mavros/state`: 飞控的当前状态（例如，`ARMED`, `OFFBOARD`, `MANUAL`）。
    - `/mavros/battery`: 电池电压和电量信息。
    - `/mavros/estimator_status`: 内部状态估计器（EKF）的健康状况。

  ------

  

  ### **第二类：外部数据注入 (ROS -> 飞控)**

  

  。数据流向主要是从**机载电脑到飞控**。

  - **视觉/雷达定位数据**:
    - **`/mavros/vision_pose/pose`**: 您通过VIO或Lidar SLAM计算出的无人机位姿，发送给飞控，用于**修正和融合**。
    - `/mavros/odometry/in`: 另一个可以发送完整里程计信息（包含速度）的接口。通常 `vision_pose` 和 `odometry/in` 只使用其中一个。
  - **动捕系统数据**:
    - `/mavros/mocap/pose`: 用于从外部运动捕捉系统（如Vicon, OptiTrack）注入更高精度的位姿。
  - **外部GPS数据**:
    - `/mavros/gps_input/gps_input`: 可以绕过飞控自己的GPS驱动，直接从ROS注入GPS数据。

  ------

  

  ### **第三类：指令与控制 (ROS -> 飞控)**"/mavros/setpoint_"

  **这是实现自主飞行的核心。**

  - **高级指令 **:
    - `/mavros/setpoint_position/local`: 发送在**本地坐标系**下的期望位置目标点。
    - `/mavros/setpoint_velocity/cmd_vel_unstamped`: 发送期望的速度指令。
  - **低级指令 (需要自己做控制器)**:
    - `/mavros/setpoint_raw/attitude`: 直接发送期望的姿态和油门大小。
    - `/mavros/setpoint_accel/accel`: 发送期望的加速度指令。
  - **轨迹指令**:
    - `/mavros/trajectory/desired`: 发送一个包含位置、速度、加速度的完整轨迹点。

  ------

  ### **MAVROS 话题功能详尽分类表**

  

  | 类别 (Category)                                             | 主要话题 (Key Topic)                          | 数据流向 (Data Flow) | 功能简介 (Brief Description)                                 |
  | ----------------------------------------------------------- | --------------------------------------------- | -------------------- | ------------------------------------------------------------ |
  | **1. 感知、定位与状态**   *(Sensing, State & Localization)* | `/mavros/state`                               | 飞控 -> ROS          | ✅ **核心状态**: 飞行模式、是否解锁、系统ID等。               |
  |                                                             | `/mavros/local_position/odom`                 | 飞控 -> ROS          | **本地里程计**: EKF融合后的高精度位姿+速度，**上层应用首选**。 |
  |                                                             | `/mavros/local_position/pose`                 | 飞控 -> ROS          | 本地坐标系下的位姿（位置+姿态）。                            |
  |                                                             | `/mavros/local_position/velocity_body`        | 飞控 -> ROS          | 机体坐标系下的速度。                                         |
  |                                                             | `/mavros/global_position/global`              | 飞控 -> ROS          | 全球GPS位置（经纬高）。                                      |
  |                                                             | `/mavros/global_position/raw/fix`             | 飞控 -> ROS          | 原始、未融合的GPS定位数据。                                  |
  |                                                             | `/mavros/global_position/compass_hdg`         | 飞控 -> ROS          | 磁罗盘航向角。                                               |
  |                                                             | `/mavros/global_position/rel_alt`             | 飞控 -> ROS          | 相对于起飞点的高度。                                         |
  |                                                             | `/mavros/imu/data`                            | 飞控 -> ROS          | 经过校准和融合的IMU数据（姿态、角速度、加速度）。            |
  |                                                             | `/mavros/imu/data_raw`                        | 飞控 -> ROS          | 原始IMU传感器读数。                                          |
  |                                                             | `/mavros/altitude`                            | 飞控 -> ROS          | 融合后的高度信息（气压计等）。                               |
  |                                                             | `/mavros/battery`                             | 飞控 -> ROS          | 电池电压、电流与电量。                                       |
  |                                                             | `/mavros/estimator_status`                    | 飞控 -> ROS          | 内部状态估计器（EKF）的健康状况。                            |
  |                                                             | `/mavros/extended_state`                      | 飞控 -> ROS          | 扩展的飞控状态（如着陆状态）。                               |
  |                                                             | `/mavros/radio_status`                        | 飞控 -> ROS          | 遥控接收机或数传电台的信号强度(RSSI)。                       |
  |                                                             | `/mavros/time_reference`                      | 飞控 -> ROS          | 飞控与ROS之间的时间参考，用于同步。                          |
  |                                                             | `/mavros/vfr_hud`                             | 飞控 -> ROS          | 用于平视显示器（HUD）的常用数据（空速、地速、高度、油门等）。 |
  | **2. 外部数据注入**   *(External Data Injection)*           | `/mavros/vision_pose/pose`                    | ROS -> 飞控          | 💉 **注入视觉位姿**: 从VIO/SLAM向飞控EKF提供位姿修正。        |
  |                                                             | `/mavros/odometry/in`                         | ROS -> 飞控          | 💉 **注入里程计**: `vision_pose`的更完整版本，包含速度信息。  |
  |                                                             | `/mavros/mocap/pose`                          | ROS -> 飞控          | 💉 **注入动捕位姿**: 从运动捕捉系统（如Vicon）提供高精度位姿。 |
  |                                                             | `/mavros/gps_input/gps_input`                 | ROS -> 飞控          | 💉 **注入GPS**: 从ROS向飞控发送GPS数据，可替代飞控自带的GPS。 |
  |                                                             | `/mavros/landing_target/pose_in`              | ROS -> 飞控          | 🎯 **注入着陆目标**: 发送一个精确的着陆目标位姿（如视觉二维码）。 |
  | **3. 目标设定点与指令**   *(Setpoints & Commands)*          | `/mavros/setpoint_position/local`             | ROS -> 飞控          | 📍 **位置控制**: 发送期望的本地位置目标点。                   |
  |                                                             | `/mavros/setpoint_velocity/cmd_vel_unstamped` | ROS -> 飞控          | 💨 **速度控制**: 发送期望的线速度和角速度。                   |
  |                                                             | `/mavros/setpoint_raw/attitude`               | ROS -> 飞控          | 🔄 **姿态控制**: 直接发送期望的姿态和油门，用于底层控制。     |
  |                                                             | `/mavros/setpoint_accel/accel`                | ROS -> 飞控          | 🚀 **加速度控制**: 发送期望的加速度指令。                     |
  |                                                             | `/mavros/setpoint_trajectory/local`           | ROS -> 飞控          | ✈️ **轨迹控制**: 发送包含位置、速度、加速度的完整轨迹点。     |
  |                                                             | `/mavros/rc/override`                         | ROS -> 飞控          | 🕹️ **覆盖遥控器**: 用程序模拟遥控器输入，实现完全控制。       |
  |                                                             | `/mavros/manual_control/send`                 | ROS -> 飞控          | 发送手动控制指令。                                           |
  |                                                             | `/mavros/play_tune`                           | ROS -> 飞控          | 🎵 控制飞控蜂鸣器播放提示音。                                 |
  | **4. 任务、执行器与反馈**   *(Missions & Actuators)*        | `/mavros/mission/waypoints`                   | ROS <-> 飞控         | 🗺️ **航点任务**: 上传、下载和监控自主飞行航点列表。           |
  |                                                             | `/mavros/mission/reached`                     | 飞控 -> ROS          | ✅ **到达航点**: 飞控发出的到达任务航点的通知。               |
  |                                                             | `/mavros/home_position/home`                  | 飞控 -> ROS          | 🏠 获取当前设置的“家”的位置。                                 |
  |                                                             | `/mavros/rc/in`                               | 飞控 -> ROS          | 📡 **遥控器输入**: 读取人类飞行员的遥控器通道值。             |
  |                                                             | `/mavros/actuator_control`                    | 飞控 -> ROS          | ⚙️ **电机指令**: 查看飞控发送给电机/舵机的最终控制指令。      |
  |                                                             | `/mavros/esc_status` & `/esc_telemetry`       | 飞控 -> ROS          | ⚡ **电调状态**: 读取数字电调的转速、温度、电压等遥测数据。   |
  |                                                             | `/mavros/mount_control/command`               | ROS -> 飞控          | 控制相机云台的指令。                                         |
  | **5. 通用、调试与底层**   *(Utilities & Low-Level)*         | `/mavlink/from` & `/mavlink/to`               | MAVROS <-> 串口      | 🛠️ **底层数据包**: 用于深度调试，直接查看原始的MAVLink通信数据。 |
  |                                                             | `/mavros/param/param_value`                   | ROS <-> 飞控         | 🔧 **参数服务**: 用于读取和设置飞控的所有内部参数（**注意：这是服务，非话题**）。 |
  |                                                             | `/mavros/log_transfer/...`                    | 飞控 -> ROS          | 💾 **飞行日志**: 用于下载飞控的“黑匣子”数据。                 |
  |                                                             | `/mavros/statustext/recv`                     | 飞控 -> ROS          | 💬 **状态文本**: 接收来自飞控的文本消息（如 "Preflight checks passed"）。 |
  |                                                             | `/mavros/debug_value/...`                     | ROS <-> 飞控         | 🐞 **调试值**: 一组通用的调试接口，用于在飞控和ROS间传递命名的变量。 |
  |                                                             | `/mavros/hil/...`                             | ROS <-> 飞控         | 💻 **硬件在环仿真 (HIL)**: 一整套用于HIL仿真的话题，如模拟GPS、IMU等。 |
  |                                                             | `/mavros/px4flow/...`                         | 飞控 -> ROS          | 👁️ **光流传感器**: 用于PX4Flow光流传感器的数据。              |
  |                                                             | `/mavros/adsb/...`                            | ROS <-> 飞控         | ✈️ **ADS-B**: 用于广播式自动相关监视（ADS-B）的数据，避免与其他飞行器相撞。 |
  |                                                             | `/mavros/tunnel/in` & `/out`                  | ROS <-> 飞控         | 🚇 **自定义通道**: 用于在ROS和飞控固件之间传输自定义的MAVLink消息。 |

  ---

  

  

  

  ### **关键点**:

  #### **1. `/mavros/vision_pose/pose` 何去何从？**

  - **从哪来 (Source)**: 它来自于您的外部定位系统。它由 `vins_to_mavros` 节点在对原始VIO/LIO数据进行**坐标校正**后发布出来。
  - **到哪去 (Destination)**:
    1. MAVROS节点订阅这个ROS话题。
    2. MAVROS将收到的 `PoseStamped` 消息**打包**成 MAVLink 协议格式的消息（通常是 `VISION_POSITION_ESTIMATE` 或 `ODOMETRY` 消息）。
    3. MAVROS通过串口或网络将这个MAVLink消息发送给**飞控板**。

  

  #### **2. 飞控板中的内部消息处理逻辑**

  

  1. 飞控板接收到由 `/vision_pose/pose` 转化而来的 MAVLink 消息。
  2. 飞控内部的核心——**EKF (扩展卡尔曼滤波器)**——获取这个外部位姿数据。
  3. EKF开始进行**数据融合**：
     - **预测步 (Prediction)**: EKF根据内部高频的IMU数据（加速度和角速度）来**预测**无人机下一刻的位姿。这个预测会因为IMU的漂移而逐渐不准。
     - **更新/修正步 (Update/Correction)**: 当EKF收到一个外部的、更准确的位姿测量值时（比如您从 `vision_pose` 发送过来的数据，或者GPS数据），它会用这个“外部真值”来**修正**自己的预测。它会计算预测值和测量值之间的差异，并根据传感器的可信度（噪声大小）来调整自己的状态，使其更接近测量值。
  4. 融合后的最佳估计结果，就变成了飞控的“官方”位姿。这个结果随后被MAVROS读取，并发布到 `/mavros/local_position/odom` 等话题上，供ROS系统的其他部分使用。

  

3. fastlio 的 `/cloud_registered` 话题



这个话题是**实时发布，不缓存**的。

让我们详细解释一下：

- **实时发布 (Real-time Publishing):** `fastlio` 在处理完每一帧（或一个小的集合）的LiDAR数据后，就会立即发布一次 `/cloud_registered` 话题。这个话题的内容是**当前帧**的点云，经过运动畸变校正，并且被配准（变换）到**世界坐标系（world frame）**下的结果。
- **不缓存 (Not Cached):** 这个话题本身并不会累积之前的点云。每一次发布的消息都是一个独立的 `sensor_msgs/PointCloud2`，只包含最新处理过的那一帧点云数据。如果你用 `rostopic echo` 去查看这个话题，你会看到消息流持续不断地输出，但每一条消息的大小基本是固定的（取决于LiDAR的扫描线数和点数），并不会随着时间的推移而变大。

**那为什么在 RViz 里看起来是“一直在增长的”呢？**

这是因为可视化工具（如 RViz）的特性。当你把 `/cloud_registered` 话题在 RViz 中显示时，通常会进行如下设置：

1. **Fixed Frame (固定坐标系):** 你会将 RViz 的全局固定坐标系设置为 `world` 或者 `map`。
2. **Decay Time (衰减时间):** 在 PointCloud2 的显示设置里，有一个 "Decay Time" 参数。如果你把它设置为 0 或者一个很大的值，RViz 就会把接收到的**每一帧**点云都保留在画面上，不会清除旧的。

---



### 节点分析

#### 1. /offset_odom_pub

简单来说，`/offset_odom_pub` 节点的作用是**校正里程计 (odometry) 的初始位置**，并将校正后的结果发布到 `/odom2` 话题上。

##### `/offset_odom_pub` 节点的作用

这个节点通常是一个辅助性的工具，它的核心功能是给一个已经存在的里程计数据流**添加一个固定的偏移量（offset）**。

**工作流程通常是这样的：**

1. **订阅原始里程计：** 这个节点会订阅一个原始的里程计话题，比如来自 VIO (视觉惯性里程计) 或 LIO (激光雷达惯性里程计) 算法输出的话题（例如 `/vins_fusion/odometry` 或 `/fast_lio/odometry`）。
2. **计算偏移量：** 当它收到**第一帧**里程计数据时，它会记下这个初始的位姿（Position 和 Orientation）。这个初始位姿就是原始算法的“世界坐标系”原点。
3. **应用偏移：** 对于后续接收到的每一帧里程计数据，它都会应用一个变换，这个变换的作用是把第一帧的初始位姿“挪”回到真正的世界原点 (0, 0, 0)。
4. **发布新里程计：** 最后，它会将这个经过“原点重置”的、新的里程计数据发布出去。

你可以把它想象成一个“**归零校准器**”。无论你的无人机或机器人在哪里启动SLAM算法，这个节点都能确保最终输出的轨迹图的起点始终在 (0, 0, 0) 位置。

##### `/odom2` 话题

这个话题就是 `/offset_odom_pub` 节点**发布校正后里程计数据的地方**。

它的名字 `/odom2` 只是一个约定俗成的名称，意思是“第二个”或者“处理过的”里程计数据。它与原始的 `/odom` 话题相比，具有相同的消息类型 (`nav_msgs/Odometry`)，但其数据经过了原点对齐。

------

#### 2. /drone2_ego_planner_node

`/drone2_ego_planner_node` 是 **EGO-Planner** 框架中，负责**无人机2号（drone2）\**进行\**自主路径规划**的ROS节点。

简单来说，它就是无人机2号在未知复杂环境中，能实时、快速、安全地规划自己飞行轨迹的“**动态导航大脑**”。

#####  主要功能

这个节点的核心是实现一个高效的运动规划器，其特点可以概括为 "EGO"，即 **E**xploration **G**uided **O**ptimization (探索引导的优化)。

1. **🚀 轨迹生成与优化 (Trajectory Generation & Optimization):**
   - 该节点不只是简单地找一条能走的路，而是生成一条**平滑**且**动态可行**的飞行轨迹。
   - 它会综合考虑无人机的速度、加速度等限制，确保生成的轨迹是无人机在物理上能够执行的。
   - 优化目标通常是找到时间最短、最平滑、最安全的路径。
2. **🌲 实时避障 (Real-time Obstacle Avoidance):**
   - 它会接收来自传感器（如深度相机、激光雷达）的环境感知数据，构建一个局部地图。
   - 当规划路径时，它能主动避开地图中的障碍物，保证飞行安全。
3. **🐝 支持无人机集群 (Swarm Support):**
   - EGO-Planner 框架是为**无人机集群**设计的。`/drone2_ego_planner_node` 中的 `drone2` 这个命名空间（namespace）明确表示，它是在一个多无人机系统中，专门服务于2号无人机的规划节点。
   - 在集群中，它不仅要躲避静态障碍物，还要通过节点间的通信，与其他无人机（如 `/drone1_ego_planner_node`）协同，避免相互碰撞。这是一个**去中心化**的系统，意味着每架无人机都独立运行自己的规划器，然后相互协调。
4. **异步与去中心化 (Asynchronous & Decentralized):**
   - 每架无人机的规划节点都是独立工作的，不需要一个中央服务器来统一指挥。这大大提高了系统的鲁棒性，即使一架无人机出现故障，也不会影响其他无人机的正常运行。

#####  工作流程简述

1. **接收目标点:** 节点从外部（如地面站或更高层的任务规划器）接收一个目标点。
2. **感知环境:** 节点订阅来自 `/drone2` 的传感器数据（如点云或深度图），了解周围环境。
3. **全局路径搜索:** 使用A*等算法快速搜索一条到达目标点的粗略路径。
4. **轨迹优化:** 将粗略路径作为输入，通过优化算法（如 MINCO）生成一条光滑、安全的轨迹。
5. **发布指令:** 将最终生成的轨迹点作为指令，发布给无人机的底层控制器去执行。



#### 3. ego swarm

##### 1. 核心规划模块

这是整个流程的大脑。

- **`/drone2_ego_planner_node`**:
  - **角色**: 这是 **EGO-Planner 的核心规划节点**，专门负责 `drone2`。它接收目标点和传感器数据（图中未画出，但必然存在），然后实时计算出一条从当前位置到目标点的、光滑且无碰撞的飞行轨迹。
  - **输入**:
    - 它会订阅一个里程计话题（从左侧未完全显示的 `/offset_odom_pub` 节点传来），用于了解自身的实时位置。
    - 它订阅 `/broadcast_bspline` 话题，这是用于**无人机集群协同避障**的。它通过这个话题接收来自**其他无人机**的规划轨迹，从而在规划自己路径时能主动避开队友。
  - **输出**:
    - **`/drone2_planning/bspline`**: 这是最重要的输出。它将计算出的最优轨迹以 **B样条曲线 (B-spline)** 的形式发布到这个话题上。B样条是一种数学表示，可以紧凑地描述一条平滑的曲线。
    - **`/drone2_ego_planner_node/...` (框内的话题)**: 这些是用于**可视化和调试**的话题。在RViz等工具中订阅它们，就可以实时看到：
      - `/grid_map` 和 `/grid_map/occupancy`: 规划器内部用于避障的局部**栅格地图**。
      - `/optimal_list`: 规划出的最优路径点。



##### 2. 轨迹执行模块

这部分负责将规划好的“蓝图”变成实际的飞行动作。

- **`/drone2_traj_server`**:
  - **角色**: 这是一个**轨迹服务器 (Trajectory Server)** 或轨迹跟随者。它的任务是接收来自规划器的完整轨迹（那条B样条曲线）。
  - **工作方式**: 它不会一次性把整个轨迹都发给飞控。而是根据当前时间，从B样条曲线上**高频采样**出一个个具体的目标点（包含位置、速度、加速度等信息）。
- **`/position_cmd`**:
  - **角色**: 这是**最终的控制指令话题**。
  - **数据流**: `/drone2_traj_server` 将实时采样出的指令点发布到这个话题。无人机的底层飞控（例如通过 MAVROS）会订阅这个话题，并精确地控制电机，让无人机严格按照这些指令点飞行。

------



##### 3. 通信与监控模块



这部分负责与地面站（GCS）通信，并可能融合其他数据。

- **`/gcs_bridge_node`**:
  - **角色**: 一个**桥接节点**，用于在无人机上的 ROS 系统和**地面控制站 (Ground Control Station)** 之间建立通信。
  - **工作方式**:
    - 它**订阅** `/drone2_ego_planner_node` 的规划结果和 `/odom_new`（可能是最新的里程计信息）。
    - 然后将这些信息打包，通过网络（如UDP或MAVLink）发送给地面站的软件，这样操作员就能在电脑上实时看到无人机的飞行轨迹、规划路径和当前位置。
- **`/odom_new`**:
  - **角色**: 一个里程计话题。从图中看，它被 `/gcs_bridge_node` 订阅。这很可能是无人机最新的、或者经过融合处理后的位姿估计，用于发送给地面站进行监控。

##### 总结流程

1. **规划**: `/drone2_ego_planner_node` 根据自身位置和集群信息，规划出一条 B 样条轨迹。
2. **发布**: 规划结果发布到 `/drone2_planning/bspline`。
3. **解析**: `/drone2_traj_server` 订阅这条轨迹。
4. **执行**: `/drone2_traj_server` 将轨迹解析成密集的控制指令点，发布到 `/position_cmd`，驱动无人机飞行。
5. **监控**: 同时，`/gcs_bridge_node` 将规划和定位信息发送给地面站，供用户监控。



