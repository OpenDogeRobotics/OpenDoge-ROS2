# 灵足EDULITE 05 ROS2_CONTROL 示例程序

## 项目概述
本项目基于ROS 2和ros2_control框架实现灵足EDULITE 05电机的硬件控制接口，支持通过CAN总线与电机通信，提供位置、速度和扭矩控制功能，并可通过ROS 2话题进行远程控制。项目包含完整的硬件接口实现、电机控制类、控制器配置和测试程序。

## 核心特性
- **完整的ros2_control集成**：实现标准化的硬件接口，支持ROS 2控制器框架
- **CAN总线通信**：通过CAN总线与灵足EDULITE 05电机进行实时通信
- **多模式控制**：支持位置、速度和扭矩控制模式
- **实时状态反馈**：提供电机位置、速度、扭矩、温度等状态信息
- **PID参数调节**：支持在线调整PID控制参数
- **多电机支持**：可配置和控制多个电机
- **完整的测试套件**：包含硬件接口测试程序和控制示例

## 项目依赖
- ROS 2 Humble Hawksbill (或其他兼容版本)
- ros2_control
- ros2_controllers
- hardware_interface
- controller_manager
- can-utils (用于CAN总线调试)
- Linux CAN总线驱动

## 安装说明
### 1. 安装ROS 2
请按照[ROS 2官方文档](https://docs.ros.org/en/humble/Installation.html)安装对应版本的ROS 2。

### 2. 配置CAN总线
```bash
# 加载CAN内核模块
sudo modprobe can
sudo modprobe can_raw

# 配置CAN接口（假设为can0）
sudo ip link set can0 up type can bitrate 1000000
```

### 3. 创建工作空间
```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws
```

### 4. 克隆仓库
```bash
cd ~/ros2_ws/src
git clone <仓库URL>  # 替换为实际仓库地址
```

### 5. 安装依赖
```bash
cd ~/ros2_ws
rosdep install --from-paths src --ignore-src -r -y
```

## 编译步骤
```bash
cd ~/ros2_ws
colcon build --symlink-install
source install/setup.bash
```

## 项目结构
```
ros2_ws/src/
├── motor_control/
│   ├── CMakeLists.txt
│   ├── package.xml
│   ├── config/
│   │   └── motor_controller_config.yaml    # 控制器配置文件
│   ├── include/
│   │   └── motor_control/
│   │       ├── motor_control_cpp.h         # 电机控制类头文件
│   │       └── motor_hardware_interface.h  # 硬件接口头文件
│   ├── launch/
│   │   └── motor_control_launch.py         # 启动文件
│   ├── src/
│   │   ├── motor_control_cpp.cpp           # 电机控制类实现
│   │   └── motor_hardware_interface.cpp    # 硬件接口实现
│   ├── urdf/
│   │   └── motor_control.urdf              # 机器人模型文件
│   ├── scripts/
│   │   └── control_motor_via_topic.py     # 通过话题控制电机的脚本
│   └── test/
│       └── test_motor_hardware.cpp         # 硬件接口测试程序
└── motor_control_interface/
    ├── CMakeLists.txt
    └── ...
```

## 核心组件

### 1. MotorController类
负责CAN总线通信和电机控制命令发送，支持以下功能：
- CAN总线初始化和配置
- 电机零位设置
- 运动模式配置
- 电机使能/失能控制
- 运动参数设置（扭矩、角度、速度、PID参数）

### 2. MotorHardware类
实现ros2_control硬件接口，连接ROS 2控制器框架与底层电机控制：
- 导出命令接口（position、velocity、effort）
- 导出状态接口（position、velocity、effort）
- 实时读取电机状态
- 执行控制命令

### 3. 控制器配置
支持多种控制器类型：
- `joint_state_broadcaster`：发布关节状态
- `position_controllers/JointGroupPositionController`：位置控制
- `velocity_controllers/JointGroupVelocityController`：速度控制
- `effort_controllers/JointGroupEffortController`：扭矩控制

## 电机控制参数

### 支持的参数范围
- **扭矩范围**：-12.5 ~ 12.5 N·m
- **角度范围**：-12.0 ~ 12.0 rad
- **速度范围**：-50.0 ~ 50.0 rad/s
- **KP范围**：0.0 ~ 500.0
- **KD范围**：0.0 ~ 5.0

### CAN通信参数
- **CAN接口**：can0（可配置）
- **主控设备ID**：253（0xfd）
- **电机ID**：每个电机独立配置（1-252）
- **波特率**：1Mbps

## 配置说明

### 控制器配置
控制器配置文件：`config/motor_controller_config.yaml`
```yaml
controller_manager:
  ros__parameters:
    update_rate: 100  # 控制器更新频率

    joint_state_broadcaster:
      type: joint_state_broadcaster/JointStateBroadcaster

    position_controller:
      type: position_controllers/JointGroupPositionController
      joints:
        - joint1
        - joint2  # 支持多关节
      interface_name_mapping:
        joint1:
          position: position
          velocity: velocity
          effort: effort
        joint2:
          position: position
          velocity: velocity
          effort: effort
      # PID参数配置
      joint1:
        pid:
          p: 100.0
          i: 0.0
          d: 1.0
          i_clamp: 0.0
      joint2:
        pid:
          p: 150.0
          i: 0.0
          d: 1.5
          i_clamp: 0.0
```

### URDF配置
URDF文件定义机器人模型和硬件接口配置：
```xml
<robot name="motor_control_robot">
  <!-- 定义关节 -->
  <joint name="joint1" type="revolute">
    <parent link="base_link"/>
    <child link="link1"/>
    <axis xyz="0 0 1"/>
    <limit lower="-12.0" upper="12.0" effort="12.5" velocity="50.0"/>
  </joint>
  
  <!-- 硬件接口配置 -->
  <ros2_control name="MotorControlHardware" type="system">
    <hardware>
      <plugin>motor_control::MotorControlHardware</plugin>
      <param name="can_interface">can0</param>
      <param name="master_id">253</param>
    </hardware>
    <joint name="joint1">
      <command_interface name="position"/>
      <state_interface name="position"/>
      <state_interface name="velocity"/>
      <state_interface name="effort"/>
    </joint>
  </ros2_control>
</robot>
```

## 启动方法

### 启动电机控制节点
```bash
cd ~/ros2_ws
source install/setup.bash
ros2 launch motor_control motor_control_launch.py
```

启动后会自动：
1. 加载电机硬件接口
2. 启动控制器管理器
3. 激活joint_state_broadcaster
4. 激活配置的控制器（position_controller等）

### 启动参数配置
可以通过命令行参数覆盖默认配置：
```bash
ros2 launch motor_control motor_control_launch.py \
  can_interface:=can1 \
  update_rate:=200
```

## 测试方法

### 1. 运行硬件测试程序
```bash
cd ~/ros2_ws
source install/setup.bash
ros2 run motor_control test_motor_hardware
```

测试程序会执行以下操作：
- 初始化CAN总线
- 测试电机连接
- 设置电机零位
- 测试位置控制
- 读取电机状态

### 2. 通过话题控制电机
使用提供的Python脚本发送位置命令：
```bash
cd ~/ros2_ws
source install/setup.bash
python3 src/motor_control/scripts/control_motor_via_topic.py
```

### 3. 手动发送控制命令
```bash
# 发送位置命令
ros2 topic pub -1 /position_controller/commands std_msgs/msg/Float64MultiArray "data: [0.5, -0.3]"

# 查看关节状态
ros2 topic echo /joint_states

# 列出控制器状态
ros2 service call /controller_manager/list_controllers controller_manager_msgs/srv/ListControllers
```

## 话题和服务接口

### 命令话题
- **话题名**：`/position_controller/commands`
- **消息类型**：`std_msgs/msg/Float64MultiArray`
- **描述**：发送电机位置命令，单位为弧度
- **示例数据**：`data: [0.5, -0.3]` 表示将joint1设置为0.5弧度，joint2设置为-0.3弧度

### 状态话题
- **话题名**：`/joint_states`
- **消息类型**：`sensor_msgs/msg/JointState`
- **描述**：反馈电机当前状态，包括位置、速度和力矩

### 服务接口
- **`/controller_manager/list_controllers`**：列出所有控制器
- **`/controller_manager/switch_controller`**：切换控制器状态
- **`/controller_manager/list_hardware_components`**：列出硬件组件

## 编程接口

### C++接口
```cpp
#include "motor_control/motor_control_cpp.h"

int main() {
    // 创建电机控制器
    MotorController motor("can0", 0xfd);
    
    // 初始化
    if (motor.init_can() != 0) {
        std::cerr << "CAN初始化失败" << std::endl;
        return -1;
    }
    
    // 设置电机参数
    motor.set_motor_angle_zero(1);           // 设置零位
    motor.set_motion_mode(1);                 // 位置模式
    motor.set_motion_enable(1);               // 使能电机
    
    // 设置运动参数（电机ID, 扭矩, 角度, 速度, KP, KD）
    motor.leg_set_motion_parameter(1, 0.0f, 1.0f, 5.0f, 50.0f, 1.0f);
    
    // 关闭
    motor.close_can();
    return 0;
}
```

### Python接口
```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import Float64MultiArray

class MotorController(Node):
    def __init__(self):
        super().__init__('motor_controller')
        self.publisher = self.create_publisher(
            Float64MultiArray, '/position_controller/commands', 10)
        
    def send_position_command(self, positions):
        msg = Float64MultiArray()
        msg.data = positions
        self.publisher.publish(msg)
        self.get_logger().info(f'发送位置命令: {positions}')

# 使用示例
rclpy.init()
controller = MotorController()
controller.send_position_command([1.0, -0.5])  # 设置关节位置
rclpy.shutdown()
```

## 故障排除

### 常见问题及解决方案

#### 1. CAN总线连接失败
**症状**："Bad file descriptor"错误或无法初始化CAN接口
**解决方案**：
```bash
# 检查CAN接口状态
ip link show can0

# 重新配置CAN接口
sudo ip link set can0 down
sudo ip link set can0 up type can bitrate 1000000

# 检查内核模块
lsmod | grep can
```

#### 2. 电机无响应
**症状**：发送命令后电机没有动作
**解决方案**：
- 检查CAN总线硬件连接
- 确认电机ID配置正确
- 验证电机是否已使能：`motor.set_motion_enable(1)`
- 检查电机电源供应

#### 3. 控制器激活失败
**症状**：控制器状态为"inactive"或"finalized"
**解决方案**：
- 检查配置文件中的关节名称是否与URDF一致
- 验证接口映射是否正确
- 查看控制器日志：`ros2 launch motor_control motor_control_launch.py --show-args`

#### 4. 位置控制不准确
**症状**：电机位置与命令值偏差较大
**解决方案**：
- 调整PID参数（增加KP值提高响应速度，增加KD值减少超调）
- 检查电机零位设置：`motor.set_motor_angle_zero(1)`
- 确认电机参数在有效范围内

#### 5. 编译错误
**症状**：colcon build失败
**解决方案**：
```bash
# 清理并重新编译
cd ~/ros2_ws
rm -rf build/ install/ log/
colcon build --symlink-install

# 检查依赖
rosdep check --from-paths src --ignore-src
```

## 性能优化

### 1. 控制频率调优
- 默认更新频率：100Hz
- 可根据需要调整：修改`motor_controller_config.yaml`中的`update_rate`
- 建议范围：50-200Hz

### 2. CAN总线优化
- 使用高质量CAN线缆
- 确保终端电阻正确配置（120Ω）
- 避免长距离传输

### 3. PID参数调优
- **P参数**：控制响应速度，过大会导致振荡
- **I参数**：消除稳态误差，过大会导致积分饱和
- **D参数**：减少超调，提高稳定性

## 扩展开发

### 添加新电机
1. 在`config/motor_controller_config.yaml`中添加新关节配置
2. 在`urdf/motor_control.urdf`中添加新关节定义
3. 更新硬件接口以支持更多电机
4. 重新编译项目

### 自定义控制器
```cpp
// 创建自定义控制器类
class CustomController : public controller_interface::ControllerInterface {
    // 实现控制器逻辑
};

// 在配置文件中注册
controller_manager:
  ros__parameters:
    custom_controller:
      type: my_package/CustomController
```

## 许可证
本项目采用MIT许可证。详见LICENSE文件。

## 贡献指南
欢迎提交Issue和Pull Request来改进项目。请确保：
- 代码符合ROS 2编码规范
- 添加适当的测试用例
- 更新相关文档
