> [!INFO] PX4 飞控固件，跑在 PC 上，不是真实无人机

### 它是干嘛的

**虚拟飞行员**。它接收 mission_manager 的航点指令，计算电机转速，通过 Gazebo 的物理引擎让无人机动起来。

### 启动命令
```bash
cd ~/competition_ws/PX4-Autopilot
PX4_GZ_WORLD=my_world make px4_sitl gz_x500_depth
```
### 内部模块（概念性）
```cpp
// 主循环（400Hz 或更高）
while (true) {
    // 1. 接收传感器数据（Gazebo 给的 IMU、气压计、GPS）
    sensors = gazebo_interface.read();
    
    // 2. 状态估计（EKF）：融合传感器，算出当前位置、速度、姿态
    state = ekf_update(sensors);
    
    // 3. 检查控制源
    if (offboard_mode && trajectory_setpoint_valid) {
        // 听 mission_manager 的
        target = receive_ros2_trajectory_setpoint();
    } else {
        // 听遥控器的（手动模式）
        target = read_rc_input();
    }
    
    // 4. 位置控制：当前位置 → 目标位置 → 期望速度
    velocity_cmd = position_controller(state.pos, target.pos);
    
    // 5. 速度控制：期望速度 → 期望姿态
    attitude_cmd = velocity_controller(state.vel, velocity_cmd);
    
    // 6. 姿态控制：期望姿态 → 电机转速
    motor_outputs = attitude_controller(state.att, attitude_cmd);
    
    // 7. 发给 Gazebo 物理引擎
    gazebo_interface.write(motor_outputs);
}
```
### 输入输出（从 PX4 视角）

| 方向 | 来源/去向          | 内容                                                  |
| :- | :------------- | :-------------------------------------------------- |
| 输入 | Gazebo         | IMU、气压计、GPS（虚拟传感器）                                  |
| 输入 | MicroXRCEAgent | `/fmu/in/trajectory_setpoint`（mission\_manager 的指令） |
| 输出 | Gazebo         | 电机推力 → 无人机运动                                        |
| 输出 | MicroXRCEAgent | `/fmu/out/vehicle_local_position`（当前位置反馈）           |

