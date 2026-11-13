# OpenDoge-ROS2

OpenDoge 四足机器人 **opendoge-apx** 的 ROS 2 运控工作区与历史资料归档仓库（ROS 2 Humble）。

原先分散的多个仓库已合并到此仓库统一维护；被合并的原仓库远端已删除，其完整提交历史保留在本仓库的 `archive/<原名>` 分支上。

## 目录结构

```
OpenDoge-ROS2/
├── opendoge_ws/                  # 主 ROS 2 工作区
│   ├── README.md                 # 工作区使用手册
│   ├── LICENSE                   # Apache-2.0（原 opendoge_ros2_core）
│   └── src/
│       ├── opendoge_bringup/     # 启动与 controller 配置
│       ├── opendoge_control/     # ros2_control 配置
│       ├── opendoge_description/ # URDF/Xacro 与 meshes
│       ├── opendoge_rl_node/     # RL 运控节点
│       ├── motor_control/        # 灵足 EDULITE 05 电机 ros2_control 硬件接口
│       └── dm_imu/               # 达妙 IMU 驱动
└── archive/                      # 历史资料归档（不参与构建）
    ├── ROBSTRIDE-EDU05-ROSCONTROL/  # 电机控制早期版本（扁平目录结构）
    └── OpenDogeWorkspace/           # EL05 说明书、MotorStudio、demo 脚本
```

## 构建

```bash
cd opendoge_ws
colcon build --symlink-install
source install/setup.bash   # Windows: call install\setup.bat
ros2 launch opendoge_bringup bringup.launch.py
```

详细的前置条件、配置项与验收步骤见 [opendoge_ws/README.md](opendoge_ws/README.md)。

## 归档来源

| 原仓库 | 现在位置 | 历史分支 |
|---|---|---|
| `OpenDogeRobotics/opendoge_ros2_core` | `opendoge_ws/`（README、LICENSE、`src/` 下 4 个包） | `archive/opendoge_ros2_core` |
| `OpenDogeRobotics/motor_control` | `opendoge_ws/src/motor_control` | `archive/motor_control` |
| `OpenDogeRobotics/DM-IMU-ROS2` | `opendoge_ws/src/dm_imu` | `archive/DM-IMU-ROS2` |
| `OpenDogeRobotics/ROBSTRIDE-EDU05-ROSCONTROL` | `archive/ROBSTRIDE-EDU05-ROSCONTROL`（已被 `motor_control` 取代） | `archive/ROBSTRIDE-EDU05-ROSCONTROL` |
| `OpenDogeRobotics/OpenDogeWorkspace`（Embedded-Team-Workspace 分支） | `archive/OpenDogeWorkspace` | `archive/OpenDogeWorkspace` |

`archive/` 目录带有 `COLCON_IGNORE`，在仓库根目录执行 `colcon` 不会扫描到这些历史包（其中旧版与新版同为 `motor_control_interface` 包名）。

## 许可证

仓库根目录为 MIT。`opendoge_ws/` 源自 Apache-2.0 的 `opendoge_ros2_core`，`opendoge_ws/src/dm_imu` 与 `archive/ROBSTRIDE-EDU05-ROSCONTROL` 为 MIT，各目录内保留原始许可证文件。
