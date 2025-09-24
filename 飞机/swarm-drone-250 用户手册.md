# **swarm-drone-250 用户手册 **



**警告：** 操作不当可能导致设备损坏、财产损失甚至严重人身伤害。请在操作前务必仔细阅读并完全理解本手册。操作期间，请始终将安全置于首位。

------



### **一、 启动前测试**



在每次飞行前，必须严格执行以下测试与校准流程，以确保系统处于最佳状态。

1. **PX4 飞控基础校准**

   - **遥控器校准**：进入 QGroundControl 地面站，在校准页面重新校准遥控器的各个通道，确保摇杆行程、中位、方向均准确无误。
   - **机身与传感器校准 (IMU, 罗盘等)**：将飞行器放置于水平平面，在地面站中执行完整的传感器校准流程，包括加速度计、陀螺仪和磁罗盘。

2. **机载设备连接与配置**

   - **修改 Mid360 激光雷达 IP**：根据您的网络规划，确保 Mid360 激光雷达的 IP 地址已正确配置，并与机载计算机处于同一网段。
   - **检查网络连接**：确认机载计算机与雷达、飞控等所有外部设备的物理连接牢固，网络通信通畅。

3. **启动定位与状态估计系统**

   - 运行系统脚本：在机载计算机的终端中，执行以下脚本以启动底层驱动和定位算法。

     Bash

     ```
     cd ~/swarm250_ws/src/swarm-drone-250/src/scripts
     ./position_tmux.sh
     ```

4. **检验 EKF2 融合状态**

   - 在脚本运行后，打开一个新的终端，使用以下命令监控由飞控 EKF2 融合视觉（激光雷达）里程计后输出的位姿估计：

     Bash

     ```
     rostopic echo /mavros/vision_pose/pose
     ```

   - **收敛性验证**：手持飞行器，在安全环境下缓慢移动，并观察 `pose.position` 数据的变化：

     - 向前移动，`x` 值应稳定增加。
     - 向左移动，`y` 值应稳定增加。
     - 向上移动，`z` 值应稳定增加。

   - **警告：** 如果数据跳变、不稳定，或移动方向与数据变化不符，说明状态估计存在异常。**严禁在此状态下尝试任何飞行**。

5. **Position 模式悬停测试**

   - **切换模式**：将遥控器正面右侧第二个三段式开关拨至最下方，将飞控切换至 `Position`（定点）模式。
   - **解锁与起飞**：在确认安全后，解锁飞行器并缓慢推动油门杆超过中点 (50%)。
   - **观察状态**：待飞行器平稳爬升至 0.5 米以上后，将油门杆回中。此时，飞行器应能稳定地保持在当前位置悬停。
   - **警告：** **请勿在 Position 模式下强行解锁 (Arm)**。只有在系统状态正常、定位信号稳定时，此模式才能安全工作。

------



### **二、 运行 Swarm 自主飞行**



在完成所有启动前测试且飞行器在 `Position` 模式下悬停稳定后，方可执行自主飞行任务。

1. **启动 Swarm 规划器脚本**：执行 `swarm_tmux.sh` 脚本。

   Bash

   ```
   cd ~/swarm250_ws/src/swarm-drone-250/src/scripts
   ./swarm_tmux.sh
   ```

2. **等待系统初始化**：脚本执行后，等待约 60 秒，让所有模块完成初始化。

3. **激活自主任务**：将遥控器正面最右侧的开关拨动一下，飞行器将开始执行自主任务。

------



### **三、 修改与二次开发（开发者深度参考）**



本章节完整包含了所有用于修改、二次开发和深度调试的必要信息。



#### **3.1 基本参数修改**



- **修改向前探索距离**：
  - 文件路径：`src/planner/plan_manage/launch/swarm.launch`。
  - 修改 `<arg name="forward_dist" value="3.0"/>` 中的 `value` 值。
- **修改默认飞行/巡航高度**：
  - 文件路径：`src/planner/plan_manage/src/main_ctrl.cpp`。
  - 修改 `#define HEIGHT 1.5` 宏定义的值。



#### **3.2 系统控制架构概览**



本系统采用分层决策与规划架构：

- **FSM (Finite State Machine - 有限状态机)**：最高层的**决策大脑**。
- **Replan (Re-planning - 重规划)**：中间层的**路径生成**。
- **EGO (ESDF-free Gradient-based Optimization - 无需ESDF的梯度优化)**：底层的**轨迹精细优化**。



#### **3.3 传感器外参标定 (mid360)**

以下为 `mid360` 外参标定方法。

- **使用内置 IMU**
  - **方法一 (理论计算)**：`FAST_LIO` 的坐标系以IMU为输出原点。`livox mid360` 内部 IMU 和激光传感器有固定偏移。`fastlio` 预设了垂直安装的 `extrinsic_T` 和 `extrinsic_R`。当倾斜安装时，可以通过数学方式计算出新的旋转矩阵并填入 `fastlio` 配置中。
  - **方法二 (Odom 修正)**：完全不改变外参，此时倾斜安装输出的 `odom` 坐标系也会倾斜，但地图构建应是收敛的。然后可以对 Odom 进行反方向的坐标转换来修正坐标。通过竖直上下移动雷达，若仅 Z 轴大量改变而 XY 无改变则说明转换正常。
- **使用外置 IMU**
  - 建议使用专业的开源标定工具。推荐的仓库有：
    - `https://github.com/hku-mars/LiDAR_IMU_Init`
    - `https://github.com/APRIL-ZJU/lidar_IMU_calib`
  - 如果内置 IMU 的标定效果不佳，也可考虑使用此类工具。



#### **3.4 集群系统深度解析 (调试日志)**



本节内容完全来自集群调试日志，用于深度分析。

- **3.4.1 雷达倾斜安装**

  - 因为是 `/Odometry` -----/vins_to_mavros----> `/mavros/vision_pose/pose` 单线转换,实现了**传感器数据与机器人位姿（Pose）的分离**。
  - 在这种情况下，`/cloud_registered` 话题中的点云是倾斜的。
  - 在节点话题图中是看不到 `/mavros/vision_pose/pose` 和 `/mavros/local_position/odom` / `/mavros/local_position/pose` 的关系的。
  - 这是因为：`/mavros/vision_pose/pose` 是**喂给**飞控的数据（**输入**），而 `/mavros/local_position/odom` 和 `/mavros/local_position/pose` 则是飞控经过内部融合计算后，**反馈**的结果（**输出**）。这些融合任务是在飞控的单片机上运行的。

- **3.4.2 mavros逻辑梳理**

  - MAVROS 本质上是运行在机载电脑上的 **ROS 系统** 和无人机**飞控板**之间的一座**双向大桥**，通过 **MAVLink 协议** 来回传递消息。

  - **核心数据流逻辑**

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

  - **第一类：感知与定位 (飞控 -> ROS)**

    - **原始传感器数据**:
      - `/mavros/imu/data_raw`: 飞控原始的IMU数据（加速度计、陀螺仪）。
      - `/mavros/global_position/raw/fix`: 原始的GPS定位信息。
      - `/mavros/altitude`: 融合了气压计等信息的高度。
    - **融合后的状态估计 (最常用)**:
      - `/mavros/local_position/pose`: 飞控内部EKF融合所有传感器后，得出的在**本地坐标系（odom或map）下的位置和姿态**。
      - `/mavros/local_position/odom`: 比上面的 `pose` 更完整，还包含了**速度信息**。**是上层应用（如运动规划）最应该使用的位姿来源**。
    - **系统状态**:
      - `/mavros/state`: 飞控的当前状态（例如，`ARMED`, `OFFBOARD`, `MANUAL`）。
      - `/mavros/battery`: 电池电压和电量信息。
      - `/mavros/estimator_status`: 内部状态估计器（EKF）的健康状况。

  - **第二类：外部数据注入 (ROS -> 飞控)**

    - **视觉/雷达定位数据**:
      - `/mavros/vision_pose/pose`: 通过VIO或Lidar SLAM计算出的无人机位姿，发送给飞控，用于**修正和融合**。
      - `/mavros/odometry/in`: 另一个可以发送完整里程计信息（包含速度）的接口。通常 `vision_pose` 和 `odometry/in` 只使用其中一个。
    - **动捕系统数据**:
      - `/mavros/mocap/pose`: 用于从外部运动捕捉系统（如Vicon, OptiTrack）注入更高精度的位姿。
    - **外部GPS数据**:
      - `/mavros/gps_input/gps_input`: 可以绕过飞控自己的GPS驱动，直接从ROS注入GPS数据。

  - **第三类：指令与控制 (ROS -> 飞控)**

    - **高级指令**:
      - `/mavros/setpoint_position/local`: 发送在**本地坐标系**下的期望位置目标点。
      - `/mavros/setpoint_velocity/cmd_vel_unstamped`: 发送期望的速度指令。
    - **低级指令 (需要自己做控制器)**:
      - `/mavros/setpoint_raw/attitude`: 直接发送期望的姿态和油门大小。
      - `/mavros/setpoint_accel/accel`: 发送期望的加速度指令。
    - **轨迹指令**:
      - `/mavros/trajectory/desired`: 发送一个包含位置、速度、加速度的完整轨迹点。

- **3.4.3 MAVROS 话题功能详尽分类表**

| 类别 (Category)                                           | 主要话题 (Key Topic)                          | 数据流向 (Data Flow) | 功能简介 (Brief Description)                                 |
| --------------------------------------------------------- | --------------------------------------------- | -------------------- | ------------------------------------------------------------ |
| **1. 感知、定位与状态** *(Sensing, State & Localization)* | `/mavros/state`                               | 飞控 -> ROS          | ✅ **核心状态**: 飞行模式、是否解锁、系统ID等。               |
|                                                           | `/mavros/local_position/odom`                 | 飞控 -> ROS          | **本地里程计**: EKF融合后的高精度位姿+速度，**上层应用首选**。 |
|                                                           | `/mavros/local_position/pose`                 | 飞控 -> ROS          | 本地坐标系下的位姿（位置+姿态）。                            |
|                                                           | `/mavros/local_position/velocity_body`        | 飞控 -> ROS          | 机体坐标系下的速度。                                         |
|                                                           | `/mavros/global_position/global`              | 飞控 -> ROS          | 全球GPS位置（经纬高）。                                      |
|                                                           | `/mavros/global_position/raw/fix`             | 飞控 -> ROS          | 原始、未融合的GPS定位数据。                                  |
|                                                           | `/mavros/global_position/compass_hdg`         | 飞控 -> ROS          | 磁罗盘航向角。                                               |
|                                                           | `/mavros/global_position/rel_alt`             | 飞控 -> ROS          | 相对于起飞点的高度。                                         |
|                                                           | `/mavros/imu/data`                            | 飞控 -> ROS          | 经过校准和融合的IMU数据（姿态、角速度、加速度）。            |
|                                                           | `/mavros/imu/data_raw`                        | 飞控 -> ROS          | 原始IMU传感器读数。                                          |
|                                                           | `/mavros/altitude`                            | 飞控 -> ROS          | 融合后的高度信息（气压计等）。                               |
|                                                           | `/mavros/battery`                             | 飞控 -> ROS          | 电池电压、电流与电量。                                       |
|                                                           | `/mavros/estimator_status`                    | 飞控 -> ROS          | 内部状态估计器（EKF）的健康状况。                            |
|                                                           | `/mavros/extended_state`                      | 飞控 -> ROS          | 扩展的飞控状态（如着陆状态）。                               |
|                                                           | `/mavros/radio_status`                        | 飞控 -> ROS          | 遥控接收机或数传电台的信号强度(RSSI)。                       |
|                                                           | `/mavros/time_reference`                      | 飞控 -> ROS          | 飞控与ROS之间的时间参考，用于同步。                          |
|                                                           | `/mavros/vfr_hud`                             | 飞控 -> ROS          | 用于平视显示器（HUD）的常用数据（空速、地速、高度、油门等）。 |
| **2. 外部数据注入** *(External Data Injection)*           | `/mavros/vision_pose/pose`                    | ROS -> 飞控          | 💉 **注入视觉位姿**: 从VIO/SLAM向飞控EKF提供位姿修正。        |
|                                                           | `/mavros/odometry/in`                         | ROS -> 飞控          | 💉 **注入里程计**: `vision_pose`的更完整版本，包含速度信息。  |
|                                                           | `/mavros/mocap/pose`                          | ROS -> 飞控          | 💉 **注入动捕位姿**: 从运动捕捉系统（如Vicon）提供高精度位姿。 |
|                                                           | `/mavros/gps_input/gps_input`                 | ROS -> 飞控          | 💉 **注入GPS**: 从ROS向飞控发送GPS数据，可替代飞控自带的GPS。 |
|                                                           | `/mavros/landing_target/pose_in`              | ROS -> 飞控          | 🎯 **注入着陆目标**: 发送一个精确的着陆目标位姿（如视觉二维码）。 |
| **3. 目标设定点与指令** *(Setpoints & Commands)*          | `/mavros/setpoint_position/local`             | ROS -> 飞控          | 📍 **位置控制**: 发送期望的本地位置目标点。                   |
|                                                           | `/mavros/setpoint_velocity/cmd_vel_unstamped` | ROS -> 飞控          | 💨 **速度控制**: 发送期望的线速度和角速度。                   |
|                                                           | `/mavros/setpoint_raw/attitude`               | ROS -> 飞控          | 🔄 **姿态控制**: 直接发送期望的姿态和油门，用于底层控制。     |
|                                                           | `/mavros/setpoint_accel/accel`                | ROS -> 飞控          | 🚀 **加速度控制**: 发送期望的加速度指令。                     |
|                                                           | `/mavros/setpoint_trajectory/local`           | ROS -> 飞控          | ✈️ **轨迹控制**: 发送包含位置、速度、加速度的完整轨迹点。     |
|                                                           | `/mavros/rc/override`                         | ROS -> 飞控          | 🕹️ **覆盖遥控器**: 用程序模拟遥控器输入，实现完全控制。       |
|                                                           | `/mavros/manual_control/send`                 | ROS -> 飞控          | 发送手动控制指令。                                           |
|                                                           | `/mavros/play_tune`                           | ROS -> 飞控          | 🎵 控制飞控蜂鸣器播放提示音。                                 |
| **4. 任务、执行器与反馈** *(Missions & Actuators)*        | `/mavros/mission/waypoints`                   | ROS <-> 飞控         | 🗺️ **航点任务**: 上传、下载和监控自主飞行航点列表。           |
|                                                           | `/mavros/mission/reached`                     | 飞控 -> ROS          | ✅ **到达航点**: 飞控发出的到达任务航点的通知。               |
|                                                           | `/mavros/home_position/home`                  | 飞控 -> ROS          | 🏠 获取当前设置的“家”的位置。                                 |
|                                                           | `/mavros/rc/in`                               | 飞控 -> ROS          | 📡 **遥控器输入**: 读取人类飞行员的遥控器通道值。             |
|                                                           | `/mavros/actuator_control`                    | 飞控 -> ROS          | ⚙️ **电机指令**: 查看飞控发送给电机/舵机的最终控制指令。      |
|                                                           | `/mavros/esc_status` & `/esc_telemetry`       | 飞控 -> ROS          | ⚡ **电调状态**: 读取数字电调的转速、温度、电压等遥测数据。   |
|                                                           | `/mavros/mount_control/command`               | ROS -> 飞控          | 控制相机云台的指令。                                         |
| **5. 通用、调试与底层** *(Utilities & Low-Level)*         | `/mavlink/from` & `/mavlink/to`               | MAVROS <-> 串口      | 🛠️ **底层数据包**: 用于深度调试，直接查看原始的MAVLink通信数据。 |
|                                                           | `/mavros/param/param_value`                   | ROS <-> 飞控         | 🔧 **参数服务**: 用于读取和设置飞控的所有内部参数（**注意：这是服务，非话题**）。 |
|                                                           | `/mavros/log_transfer/...`                    | 飞控 -> ROS          | 💾 **飞行日志**: 用于下载飞控的“黑匣子”数据。                 |
|                                                           | `/mavros/statustext/recv`                     | 飞控 -> ROS          | 💬 **状态文本**: 接收来自飞控的文本消息。                     |
|                                                           | `/mavros/debug_value/...`                     | ROS <-> 飞控         | 🐞 **调试值**: 一组通用的调试接口。                           |
|                                                           | `/mavros/hil/...`                             | ROS <-> 飞控         | 💻 **硬件在环仿真 (HIL)**: 一整套用于HIL仿真的话题。          |
|                                                           | `/mavros/px4flow/...`                         | 飞控 -> ROS          | 👁️ **光流传感器**: 用于PX4Flow光流传感器的数据。              |
|                                                           | `/mavros/adsb/...`                            | ROS <-> 飞控         | ✈️ **ADS-B**: 用于广播式自动相关监视（ADS-B）的数据。         |
|                                                           | `/mavros/tunnel/in` & `/out`                  | ROS <-> 飞控         | 🚇 **自定义通道**: 用于传输自定义的MAVLink消息。              |

- **3.4.4 关键点**
  - **`/mavros/vision_pose/pose` 何去何从？**
    - **从哪来 (Source)**: 来自于外部定位系统，由 `vins_to_mavros` 节点在对原始VIO/LIO数据进行**坐标校正**后发布出来。
    - **到哪去 (Destination)**:
      1. MAVROS节点订阅这个ROS话题。
      2. MAVROS将收到的 `PoseStamped` 消息**打包**成 MAVLink 协议格式的消息。
      3. MAVROS通过串口或网络将这个MAVLink消息发送给**飞控板**。
  - **飞控板中的内部消息处理逻辑**
    1. 飞控板接收到由 `/vision_pose/pose` 转化而来的 MAVLink 消息。
    2. 飞控内部的核心——**EKF (扩展卡尔曼滤波器)**——获取这个外部位姿数据。
    3. EKF开始进行**数据融合**：
       - **预测步 (Prediction)**: EKF根据内部高频的IMU数据来**预测**无人机下一刻的位姿。
       - **更新/修正步 (Update/Correction)**: 当EKF收到一个外部的、更准确的位姿测量值时（如从 `vision_pose` 发送的数据），它会用这个“外部真值”来**修正**自己的预测。
    4. 融合后的最佳估计结果，成为飞控的“官方”位姿，随后被MAVROS读取，并发布到 `/mavros/local_position/odom` 等话题上。
- **3.4.5 核心话题与节点分析**
  - **`fastlio` 的 `/cloud_registered` 话题**
    - 这个话题是**实时发布，不缓存**的。`fastlio` 在处理完每一帧的LiDAR数据后，就会立即发布一次该话题。
    - 每一次发布的消息都是一个独立的 `sensor_msgs/PointCloud2`，只包含最新处理过的那一帧点云数据，不会累积。
    - 在 RViz 里看起来是“一直在增长的”，是因为 RViz 的 "Decay Time" 参数被设置为 0 或一个很大的值，使 RViz 保留了接收到的每一帧点云。
  - **/offset_odom_pub 节点**
    - **作用**：校正里程计 (odometry) 的初始位置，并将校正后的结果发布到 `/odom2` 话题上。
    - **工作流程**:
      1. **订阅原始里程计**：订阅来自LIO算法输出的话题（如 `/fast_lio/odometry`）。
      2. **计算偏移量**：当收到**第一帧**数据时，记下这个初始的位姿。
      3. **应用偏移**：对于后续每一帧数据，应用变换把第一帧的初始位姿“挪”回到真正的世界原点 (0, 0, 0)。
      4. **发布新里程计**：将这个经过“原点重置”的、新的里程计数据发布出去。
  - **/drone2_ego_planner_node 节点**
    - 它是 EGO-Planner 框架中，负责**无人机进行自主路径规划**的ROS节点。
    - **主要功能**:
      - **🚀 轨迹生成与优化**: 生成一条**平滑**且**动态可行**的飞行轨迹，综合考虑速度、加速度等限制。
      - **🌲 实时避障**: 接收传感器数据构建局部地图，并主动避开障碍物。
      - **🐝 支持无人机集群**: 命名空间 `drone2` 表明它服务于2号无人机，通过节点间通信与其他无人机协同避障。
      - **异步与去中心化**: 每架无人机的规划节点都独立工作，不依赖中央服务器。
  - **ego swarm 工作流程**
    1. **核心规划模块 (`/drone2_ego_planner_node`)**
       - **输入**: 订阅自身里程计话题，以及用于集群避障的 `/broadcast_bspline` 话题（接收来自**其他无人机**的规划轨迹）。
       - **输出**: 最重要的输出是 `/drone2_planning/bspline`，它以 **B样条曲线** 的形式发布计算出的最优轨迹。同时输出用于可视化的 `/grid_map` 等调试话题。
    2. **轨迹执行模块**
       - `/drone2_traj_server`: 这是一个**轨迹服务器**，任务是接收来自规划器的完整轨迹（B样条曲线）。
       - `/position_cmd`: 这是**最终的控制指令话题**。`/drone2_traj_server` 根据当前时间，从B样条曲线上**高频采样**出目标点（位置、速度、加速度），发布到这个话题，由底层控制器执行。
    3. **通信与监控模块**
       - `/gcs_bridge_node`: 一个**桥接节点**，用于在无人机上的 ROS 系统和**地面控制站 (GCS)** 之间建立通信。它订阅规划结果和里程计信息，打包发送给地面站软件。

------



### **四、 注意事项**



**安全是飞行的第一要务。**

- **远离人群**：严禁在人群上方或附近运行飞行器。始终保持安全距离。
- **保持视线**：操作期间，操作员的视线绝不能离开飞行器。
- **手不离控**：随时准备接管，确保您的手始终放在遥控器上。
- **紧急情况处理**：一旦发生任何不可预见的紧急情况，请毫不犹豫地按下遥控器上的**紧急停机 (Emergency Kill Switch)** 开关。
- **低温影响**：在低温环境下飞行，电池性能会显著下降。请确保电池在起飞前已充分预热，并适当减少飞行时间。
- **法律法规**：请严格遵守当地关于无人机飞行的所有法律和规定。