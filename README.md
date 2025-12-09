# DM-IMU-ROS2

一个基于 ROS 2 的 IMU 采集与发布示例，包含两套实现：

- 传统节点实现：直接打开串口、解析数据帧，并发布 `sensor_msgs/Imu` 与位姿话题
- `ros2_control` 硬件插件：实现 `hardware_interface::SensorInterface`，通过 IMU Sensor Broadcaster 发布标准 IMU 状态

适用于需要快速集成板载 IMU（串口输出，加速度/角速度/欧拉角）的机器人项目。

---

## 功能概览

- 串口管理与非阻塞读写（POSIX `termios`，轮询读取）
- 三段帧结构与 CRC16 校验，安全解析传感器数据
- IMU 四元数、角速度、线加速度发布
- RViz 位姿可视化与 RQT 多曲线绘图配置
- 可选仿真模式：无法打开串口时自动切换，或通过参数显式开启

---

## 目录结构

- `ros2_ws/src/dm_imu/`
  - `include/dm_imu/hardware/imu_hardware.hpp`：`ros2_control` 硬件接口声明
  - `src/hardware/imu_hardware.cpp`：硬件实现与串口读线程，仿真实现
  - `include/dm_imu/uart_port.hpp`、`src/uart_port.cpp`：串口封装（打开/读/写/关闭）
  - `include/dm_imu/imu_driver.h`、`src/imu_driver.cpp`：传统节点的数据帧协议、设备配置命令与发布逻辑
  - `src/test_imu.cpp`：传统节点入口，创建节点并运行
  - `config/imu_broadcaster.yaml`：IMU Broadcaster 配置
  - `urdf/dmbot_imu.urdf.xacro`：传感器 URDF 与 `ros2_control` 插件参数
  - `launch/imu_ros2_control.launch.py`：`ros2_control` 启动文件
  - `resource/dm_imu_hardware_plugin.xml`：插件导出描述
  - `rviz/imu.rviz`：RViz 可视化配置
  - `imu.xml`：RQT MultiPlot 曲线配置

---

## 依赖与环境

- 构建系统：`ament_cmake`（C++14）
- ROS 2 依赖（见 `package.xml`）：`rclcpp`、`sensor_msgs`、`geometry_msgs`、`std_msgs`、`tf2`、`tf2_geometry_msgs`、`hardware_interface`、`pluginlib`、`ros2_control`、`ros2_controllers`、`controller_manager`
- 串口实现依赖 POSIX 接口，仅在 Linux/类 Unix 环境可直接运行；Windows 用户可通过仿真模式或自行适配串口层

---

## 构建与安装

1. 进入工作区：
   
   ```bash
   cd ros2_ws
   ```

2. 构建：
   
   ```bash
   colcon build --packages-select dm_imu
   ```

3. 设置环境：
   
   ```bash
   source install/setup.bash
   ```

---

## 运行方式

### 方式一：传统节点

- 启动节点：
  
  ```bash
  ros2 run dm_imu dm_imu_node
  ```

- 可用参数（通过参数服务器）：
  - `port`：默认 `/dev/ttyACM0`
  - `baud`：默认 `921600`

该节点在初始化时按序下发设备配置命令（打开加速度/陀螺/欧拉角、关闭四元数、设置 1000Hz 输出、保存参数、退出设置），然后进入循环读取与发布。

代码参考：`ros2_ws/src/dm_imu/src/imu_driver.cpp:7`、`ros2_ws/src/dm_imu/src/imu_driver.cpp:19`、`ros2_ws/src/dm_imu/src/imu_driver.cpp:47`、`ros2_ws/src/dm_imu/src/imu_driver.cpp:167`。

发布的主要话题：
- `imu/data`（类型 `sensor_msgs/Imu`）
- `pose`（类型 `geometry_msgs/PoseStamped`）
- `dm/imu`（`std_msgs/Float64MultiArray`，示例中已创建但默认注释掉发布）

### 方式二：ros2_control

- 启动：
  
  ```bash
  ros2 launch dm_imu imu_ros2_control.launch.py \
    port:=/dev/ttyACM0 baud:=921600 frame_id:=imu_link simulate:=false
  ```

- 组成：
  - `robot_state_publisher` 使用 `xacro` 生成的描述
  - `controller_manager/ros2_control_node` 加载硬件与 Broadcaster
  - `controller_manager/spawner` 启动 IMU Sensor Broadcaster

- URDF 中的硬件参数：`urdf/dmbot_imu.urdf.xacro`。
  - `port`、`baud`、`frame_id`、`simulate` 可通过启动参数覆盖

- Broadcaster 配置：`config/imu_broadcaster.yaml`。

- 硬件接口：
  - 初始化与参数解析：`ros2_ws/src/dm_imu/src/hardware/imu_hardware.cpp:13`
  - 激活串口或仿真线程：`ros2_ws/src/dm_imu/src/hardware/imu_hardware.cpp:37`
  - 导出状态接口（四元数/角速度/线加速度）：`ros2_ws/src/dm_imu/src/hardware/imu_hardware.cpp:71`

IMU Sensor Broadcaster 会根据硬件导出的状态接口发布标准 IMU 消息（话题名称依插件/控制器命名空间而定）。

---

## 参数说明

- `port`：串口设备，例如 `/dev/ttyACM0`
- `baud`：串口波特率，例如 `921600`
- `frame_id`：IMU 固定坐标系，默认 `imu_link`
- `simulate`：仿真模式开关。为 `true`/`1`/`True` 时启动仿真；当 `port=mock` 或 `baud=0` 时也会强制仿真

实现参考：`ros2_ws/src/dm_imu/src/hardware/imu_hardware.cpp:17`。

---

## 数据帧与解析

- 帧结构（三段）：加速度/陀螺/欧拉角，每段都有头、数据、CRC16 与尾
- CRC16 校验通过后提取浮点数据，欧拉角转换为四元数

关键结构体与解析：`ros2_ws/src/dm_imu/include/dm_imu/imu_driver.h:23`、`ros2_ws/src/dm_imu/src/imu_driver.cpp:181`、`ros2_ws/src/dm_imu/src/hardware/imu_hardware.cpp:102`。

---

## 可视化与调试

- RViz：
  
  ```bash
  rviz2 -d $(ros2 pkg prefix dm_imu)/share/dm_imu/rviz/imu.rviz
  ```

  使用 `Pose` 显示 `/pose`，固定帧为 `imu_link`。

- RQT MultiPlot：
  
  `imu.xml` 为多曲线配置，针对 `imu/data` 的线加速度与姿态等字段进行绘制。

---

## 仿真模式

- 在硬件接口中，如果无法打开串口（或显式开启仿真），会启动一个周期性生成小幅姿态变化的线程，输出静态加速度/角速度与随时间变化的四元数
- 适用于开发环境无设备、Windows 环境或 CI 测试

参考实现：`ros2_ws/src/dm_imu/src/hardware/imu_hardware.cpp:134`。

---

## 串口说明

- 串口打开与配置：`ros2_ws/src/dm_imu/src/uart_port.cpp:18`
- 非阻塞轮询读取：`ros2_ws/src/dm_imu/src/uart_port.cpp:63`
- 写入与轻微超时重试：`ros2_ws/src/dm_imu/src/uart_port.cpp:81`

注意：当前实现基于 POSIX/`termios`，请在 Linux/类 Unix 环境使用；如需 Windows 支持需替换为相应 API。

---

## 常见问题

- 无法打开串口：检查设备路径与权限；`simulate=true` 可验证管线是否正常
- IMU 话题名称：传统节点发布在 `imu/data` 与 `pose`；`ros2_control` 模式由 IMU Broadcaster 决定具体话题（通常在控制器命名空间下）
- 姿态异常：确认欧拉角单位为度并正确转为弧度后生成四元数

---

## 许可与维护

- 维护者：见 `package.xml`
- 许可：当前占位（`TODO`）；请根据实际项目选择并更新

