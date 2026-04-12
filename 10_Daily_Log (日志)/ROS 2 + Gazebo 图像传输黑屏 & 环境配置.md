
# 🛠️ BUG 排查记录：ROS 2 + Gazebo 图像传输黑屏 & 环境配置
📅 **时间**: 2026-04-11 19:13 - 20:07  
🏷️ **标签**: #ROS2 #Gazebo #PX4 #SITL #QoS #solved 

## 1. 错误现象 (Log)
> [!ERROR] 终端报错信息
 [INFO] [1775907791.626948753] [ros_gz_bridge]: Creating GZ-&gt;ROS Bridge: [/camera (gz.msgs.Image) -&gt; /camera (sensor_msgs/msg/Image)] (Lazy 0)
 >
WARNING: topic [/camera] does not appear to be published yet

 >[!ERROR] 可视化现象  
 rqt_image_view 窗口全黑，无图像显示  
 >
Gazebo Image display 插件能正常显示彩色方块  
>
 ros2 topic hz /camera 无频率输出（空白等待）
## 2. 环境复核
- [x] Ubuntu 22.04 LTS (Jammy)
- [x] ROS 2 Humble Hawksbill
- [x] Gazebo Sim 7.9 (Garden)
- [x] PX4-Autopilot v1.15.x (gz_x500_depth 模型)
- [x] ROS_DOMAIN_ID=42
- [ ] 初始环境：ROS_LOCALHOST_ONLY=1（已设置，导致后续问题）

## 3. 解决尝试
- **尝试 1**: 使用 `ros_gz_bridge parameter_bridge` 直接桥接  
  → 失败，`Lazy 0` 状态，QoS 不匹配（Best Effort vs Reliable）

- **尝试 2**: 设置 `export ROS_LOCALHOST_ONLY=1` 限制通信  
  → 失败，DDS 发现机制被阻断，bridge 无法发现 Gazebo 节点

- **尝试 3**: 使用 `ros2 run ros_gz_image image_bridge`  
  → 初始失败：Package not found（未安装 ros-gzgarden）

- **最终解决**：
>[!SUCCESS] 安装 ros-gzgarden 并清除环境变量限制
```bash
# 安装 Garden 版本桥接包（包含 image_bridge）
sudo apt install ros-humble-ros-gzgarden

# 关键：清除阻断通信的环境变量
unset ROS_LOCALHOST_ONLY
unset GZ_IP

# 加载环境并启动专用图像桥接
source /opt/ros/humble/setup.bash
ros2 run ros_gz_image image_bridge /camera
```
>[!SUCCESS] 验证数据流（绕开时间戳问题）
```bash
# hz 命令因时间戳基准不匹配（Sim Time vs Wall Time）无法计算频率
# 改用 echo 直接验证数据存在
ros2 topic echo /camera --once
# 输出：header, height: 1080, width: 1920, encoding: rgb8, data: [...]
```
## 4. 根因复盘
| 问题层级    | 具体原因                             | 技术细节                                                                    |
| :------ | :------------------------------- | :---------------------------------------------------------------------- |
| **通信层** | `ROS_LOCALHOST_ONLY=1` 阻断 DDS 发现 | 限制 ROS 2 只在 lo 网卡通信，Gazebo 的 DDS 无法响应/卡住/空白                             |
| **协议层** | **QoS 策略不匹配**（核心问题）              | Gazebo 发布图像使用 **Best Effort**，而参数设为 **Reliable**，两者不兼容导致 `Lazy 0`，数据零传输 |
| **工具层** | `ros2 topic hz` 时间戳基准错误          | Gazebo 使用仿真时间（sec: 2426），hz 使用系统时间，时间差无法计算导致无频率输出                       |
| **日志层** | image_bridge 终端“空白”              | ros_gz_image 默认日志级别为 WARN，成功连接时无输出，造成“假象”                               |
| **依赖层** | `px4_msgs` 模块缺失                  | `competition_node` 依赖 PX4 自定义消息，需单独编译并 source 环境                        |
## 5. 标准启动流程（预防性措施）
```bash
# 终端 1：启动 Gazebo + PX4
cd ~/competition_ws/PX4-Autopilot
make px4_sitl gz_x500_depth

# 终端 2：启动图像桥接（关键：不要设置 ROS_LOCALHOST_ONLY）
source /opt/ros/humble/setup.bash
export ROS_DOMAIN_ID=42
ros2 run ros_gz_image image_bridge /camera  # 自动处理 QoS 转换

# 终端 3：验证（不用 hz，用 echo 或 bw）
ros2 topic echo /camera --once  # 确认数据流通即可开发
```
## 6. 经验总结
- **QoS 是 ROS 2 隐形杀手**：即使话题名正确、节点运行正常，QoS 不匹配 = 零数据传输
- **Wayland/DDS 发现陷阱**：`ROS_LOCALHOST_ONLY` 在单机多节点场景下反而阻断通信，需谨慎使用
- **仿真时间坑**：SITL 中传感器数据使用仿真时间戳，基于时间戳计算的工具（hz、tf）可能失效，改用原始数据验证（echo/bw）
- **日志 ≠ 状态**：终端无输出不代表失败，需用 `ros2 topic list` 二次确认