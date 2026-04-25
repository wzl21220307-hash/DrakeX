## 节点 7：Gazebo Sim（物理引擎）

> [!INFO] 仿真世界，不是 ROS2 节点，但和 ROS2 通过插件交互

### 它是干嘛的

**虚拟宇宙**。计算重力、空气阻力、碰撞、电机推力，让无人机在虚拟场地里真实飞行。

### 和你的节点关系

```mermaid
graph LR
    A[Gazebo 物理引擎] -->|IMU/GPS/图像| B[PX4 SITL]
    B -->|电机推力| A
    A -->|图像数据| C[ros_gz_image<br/>image_bridge]
    C -->|/camera| D[vision_server_pc]
    C -->|/camera| E[vision_pnp_pc]
```
### 关键配置（你改过的）

- `x500_depth/model.sdf`：把 OakD-Lite 相机朝下（Pitch -1.57）
    
- `my_world.sdf`：定义了 9m×6m 场地、起飞点、二维码、靶子、障碍物、圆环