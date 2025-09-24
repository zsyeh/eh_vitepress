# ROS2 Humble xrce-dds px4 仿真

 使用1.14版本

以正常方式在 Ubuntu 上设置 PX4 开发环境：

```bash
cd
git clone https://github.com/PX4/PX4-Autopilot.git --recursive
bash ./PX4-Autopilot/Tools/setup/ubuntu.sh
cd PX4-Autopilot/
make px4_sitl
```

输入以下命令以从源获取和构建代理：

```bash
git clone -b v2.4.2 https://github.com/eProsima/Micro-XRCE-DDS-Agent.git
cd Micro-XRCE-DDS-Agent
mkdir build
cd build
cmake ..
make
sudo make install
sudo ldconfig /usr/local/lib/
# 在运行sitl之前先启动代理
MicroXRCEAgent udp4 -p 8888
```

然后运行 sitl

`make px4_sitl gz_x500`





## 构建消息包

```bash
mkdir -p ~/ws_sensor_combined/src/ 
cd ~/ws_sensor_combined/src/
git clone https://github.com/PX4/px4_msgs.git
git clone https://github.com/PX4/px4_ros_com.git

cd ..
source /opt/ros/humble/setup.bash
colcon build

ros2 launch px4_ros_com sensor_combined_listener.launch.py
```

如果这有效，您应该会在启动 ROS 侦听器的终端/控制台上看到打印的数据：

```
RECEIVED DATA FROM SENSOR COMBINED
================================
ts: 870938190
gyro_rad[0]: 0.00341645
gyro_rad[1]: 0.00626475
gyro_rad[2]: -0.000515705
gyro_integral_dt: 4739
accelerometer_timestamp_relative: 0
accelerometer_m_s2[0]: -0.273381
accelerometer_m_s2[1]: 0.0949186
accelerometer_m_s2[2]: -9.76044
accelerometer_integral_dt: 4739
```

**注意固件版本和 消息类型是相关的,请务必保证消息类型合适**

否则需要使用消息转换节点

PX4 v1.16 PX4 v1.16 版本 Experimental 实验的

This example is built with PX4 and ROS2 versions that use the same message definitions. If you were to use incompatible [message versions](https://docs.px4.io/main/en/middleware/uorb.html#message-versioning) you would need to install and run the [Message Translation Node](https://docs.px4.io/main/en/ros2/px4_ros2_msg_translation_node.html) as well, before running the example:
此示例使用使用相同消息定义的 PX4 和 ROS2 版本构建。如果要使用不兼容的[消息版本 ](https://docs.px4.io/main/en/middleware/uorb.html#message-versioning)，则在运行示例之前，还需要安装并运行[消息转换节点 ](https://docs.px4.io/main/en/ros2/px4_ros2_msg_translation_node.html)：

1. Include the [Message Translation Node](https://docs.px4.io/main/en/ros2/px4_ros2_msg_translation_node.html) into the example workspace or a separate workspace by running the following script:
   通过运行以下脚本，将[消息转换节点](https://docs.px4.io/main/en/ros2/px4_ros2_msg_translation_node.html)包括到示例工作区或单独的工作区中：

   ```
   cd /path/to/ros_ws
   /path/to/PX4-Autopilot/Tools/copy_to_ros_ws.sh .
   ```

2. Build and run the translation node:
   构建并运行翻译节点：

   ```
   colcon build
   source install/local_setup.bash
   ros2 run translation_node translation_node_bin
   ```





## 附录 名词解释

下表解释了在本次交流中我们遇到的几个关键技术名词及其相互关系。

| 名词 (Term)                  | 全称 / 类型 (Full Name / Type)                               | 角色与解释 (Role & Explanation)                              | 与其它名词的关系 (Relationship to Other Terms)               |
| ---------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **SITL**                     | Software-In-The-Loop (软件在环仿真)                          | **一种仿真方法**。它让完整的 PX4 飞控固件代码可以直接在电脑上运行，而无需任何物理硬件。它从 Gazebo 等模拟器获取虚拟的传感器数据，并将计算出的电机指令发回给模拟器，从而实现对虚拟无人机的控制。可以理解为无人机的“**大脑**”运行在一个“**虚拟身体**”里。 | **SITL** 运行 PX4 固件，并通过 **XRCE-DDS** 协议与外部（如ROS 2）通信。 |
| **XRCE-DDS**                 | DDS for eXtremely Resource-Constrained Environments (面向极度资源受限环境的DDS协议) | **一种通信协议**。它是 ROS 2 标准通信中间件 DDS 的轻量化版本，专门设计用于在像无人机飞控这样的微控制器上高效运行。它定义了资源受限设备与完整 ROS 2 网络之间对话的“**语言**”和规则。 | **SITL** 中的客户端和 **Micro-XRCE-DDS-Agent** 之间使用 **XRCE-DDS** 协议进行对话。 |
| **Micro-XRCE-DDS-Agent**     | (程序名称)                                                   | **一个独立的桥接程序**。它扮演着“**翻译官**”和“**中间人**”的角色。它在您的电脑上运行，监听来自 SITL (或真实飞控) 的 **XRCE-DDS** 数据流，并将其“翻译”成标准的 ROS 2 数据流，反之亦然。没有它，ROS 2 节点就无法看到或与 PX4 通信。 | **Agent** 是 **XRCE-DDS** 协议的服务端，它将数据桥接后，以 **px4_msgs** 定义的话题格式发布到 ROS 2 网络中。 |
| **px4_msgs**                 | PX4 ROS 2 Messages (PX4 的 ROS 2 消息包)                     | **一个 ROS 2 消息包**。这个包里包含了所有 PX4 特有话题的数据结构定义（例如 `VehicleStatus.msg`, `TrajectorySetpoint.msg`）。您的 Python 或 C++ 脚本需要 `import` 这些消息定义，才能理解和创建与 PX4 通信的数据。它就像一本“**数据字典**”。 | **px4_msgs** 由 **px4_ros_com** 在编译时根据 PX4 固件源码生成。您的控制脚本（如`fly_to_point.py`）需要依赖这个包。 |
| **px4_ros_com**              | PX4 ROS 2 Communication (PX4 的 ROS 2 通信包)                | **一个 ROS 2 源码仓库/元包**。这是您克隆到 ROS 2 工作空间的核心仓库。它包含了生成 **px4_msgs** 的编译逻辑、一些示例程序（如 `sensor_combined_listener`）和方便使用的 `launch` 文件。 | **px4_ros_com** 是您 `ros2_ws` 中的一个关键源码包，它负责生成 **px4_msgs**。 |
| **sensor_combined_listener** | (ROS 2 节点名称)                                             | **一个用于调试的示例程序**。这个小程序由 **px4_ros_com** 提供，它的唯一作用就是订阅 `/fmu/out/sensor_combined` 这个话题，并将接收到的原始传感器数据（如陀螺仪、加速度计）打印到屏幕上。它是一个完美的“Hello, World!”程序，用来快速**验证整个通信链路是否正常工作**。 | **sensor_combined_listener** 是一个最简单的 ROS 2 节点，它订阅由 **Agent** 桥接过来、并由 **px4_msgs** 定义的传感器话题。 |





### 工作流程总结



## 直接从px4-autopilot拉的代码sitl版本是多少4

如果直接从 `PX4-Autopilot` 仓库拉取代码，没有指定任何分支或标签（即运行 `git clone https://github.com/PX4/PX4-Autopilot.git`），那么获取到的是 **`main` 分支**的代码。

> main 分支意味着什么？
> 它是最新的开发版本: main 分支包含了所有最新的功能、修复和改动。可以把它看作是正在开发中的、未来的 PX4 版本（例如，可能是将来的 v1.16 或更高版本）。
>
> 它没有一个固定的版本号: main 分支是持续更新的，所以它不像 v1.15 或 v1.14 那样有一个明确的版本号。它的版本是“流动的”。
>
> 它可能不稳定: 作为最前沿的开发分支，它可能包含一些未经充分测试的新代码，因此存在不稳定的风险。

您好，这是一个非常重要的问题，能帮助我们理解版本管理的策略。

如果您直接从 `PX4-Autopilot` 仓库拉取代码，没有指定任何分支或标签（即运行 `git clone https://github.com/PX4/PX4-Autopilot.git`），那么您获取到的是 **`main` 分支**的代码。



### `main` 分支意味着什么？



1. **它是最新的开发版本**: `main` 分支包含了所有最新的功能、修复和改动。可以把它看作是**正在开发中的、未来的 PX4 版本**（例如，可能是将来的 v1.16 或更高版本）。
2. **它没有一个固定的版本号**: `main` 分支是持续更新的，所以它不像 `v1.15` 或 `v1.14` 那样有一个明确的版本号。它的版本是“流动的”。
3. **它可能不稳定**: 作为最前沿的开发分支，它可能包含一些未经充分测试的新代码，因此存在不稳定的风险。



### SITL 的版本是多少？



SITL（软件在环仿真）本身没有独立的版本号。**SITL 的版本和您编译它的固件源码版本是完全一致的。**

所以，如果您从 `main` 分支的源码编译 SITL，那么您得到的 SITL 就是 `main` 分支开发版的 SITL。



### 如何查看您当前代码的精确“版本”？



即使 `main` 分支没有固定版本号，您也可以通过 `git` 命令来精确地知道您当前处于哪个代码状态。在 `PX4-Autopilot` 目录下运行：

Bash

```
git describe --always --tags
```

这个命令的输出可能会是像这样的格式： `v1.15.0-rc2-45-g123abcde`

- `v1.15.0-rc2`: 表示您当前的代码是基于 `v1.15.0-rc2` 这个标签的。
- `45`: 表示在那个标签之后，又有 45 个新的提交。
- `g123abcde`: 是您当前代码快照的唯一哈希值缩写。



这直接关系到我们之前遇到的**版本不匹配问题**。

- 如果您使用 `main` 分支的 `PX4-Autopilot`，那么为了保证消息定义一致，您**也必须使用 `main` 分支的 `px4_ros_com`**。
- 这样做的好处是能体验最新功能，但缺点是您可能会遇到像之前那样的、由于两个项目开发进度不完全同步导致的不兼容问题。

这就是为什么在之前的回答中，我建议您切换到 `release/1.15` 或 `v1.14.0` 这样的稳定分支或标签。这能为您提供一个经过验证、保证兼容的“版本快照”。

**总结:**

| 您拉取的分支      | 版本类型          | 稳定性       | 建议的 `px4_ros_com` 版本 |
| ----------------- | ----------------- | ------------ | ------------------------- |
| **`main` (默认)** | **最新开发版**    | **不稳定**   | **`main`**                |
| `release/1.15`    | 1.15 稳定版系列   | 较稳定       | `release/1.15`            |
| `v1.14.0` (标签)  | 1.14.0 稳定发行版 | **非常稳定** | `v1.14.0`                 |

因此，直接拉取的代码版本是**最新的、不稳定的 `main` 分支开发版**。