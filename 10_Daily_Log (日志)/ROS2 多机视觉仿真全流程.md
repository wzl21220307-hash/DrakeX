
# 🛠️ BUG 排查记录：ROS2 多机视觉仿真全流程

📅 **时间**: 2026-04-22 ~ 04-24 🏷️ **标签**: #Troubleshooting #PX4 #Gazebo #CycloneDDS #跨机通信

---

## 问题组 1：跨机通信/网络层（已解决）

### 1.1 NX 本机 ROS2 [[节点]] Segmentation fault

> [!ERROR] `Segmentation fault (core dumped)`，DDS 在 `wlan0` 和 `docker0` 之间乱跳

**环境复核**

- [x] `ROS_DOMAIN_ID=42`
    
- [ ] `RMW_IMPLEMENTATION` 未统一（Foxy 默认 Fast DDS 在 aarch64 下崩溃）
    

**解决尝试**

- 尝试 1: `unset ROS_LOCALHOST_ONLY` → 无效
    
- 尝试 2: 绑定 `CYCLONEDDS_URI` 到 `wlan0` → 仍崩
    
- **最终解法**: 安装 `ros-foxy-rmw-cyclonedds-cpp`，强制 `export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp`
    

**根因复盘** Foxy 默认 Fast DDS 在 Jetson aarch64 + JetPack 环境有已知段错误。CycloneDDS 稳定性更好，但后续引发了与 `ros_gz_bridge` 的兼容性问题（见 3.2）。

---

### 1.2 PC-NX 互 ping 不通 / topic 互相看不见

> [!ERROR] `ping 10.181.1.190` 100% packet loss；`ros2 topic list` 互相看不到对方 topic

**环境复核**

- [x] 同一手机热点
    
- [ ] IP 网段不一致（PC: `10.36.x.x` 校园网残留，NX: `10.181.x.x` 热点）
    

**解决尝试**

- 尝试 1: 检查手机热点 AP 隔离 → 实际能 ping 通 `10.181.190.12`
    
- 尝试 2: 多播探测 `ros2 multicast send/receive` → 热点吃多播
    
- **最终解法**: CycloneDDS 单播 XML 硬配置 Peer 地址，两边 `ROS_DOMAIN_ID=42` + `RMW_IMPLEMENTATION=rmw_cyclonedds_cpp` 严格对齐
    

**根因复盘** 手机热点会阻断 DDS 多播发现，必须显式配置 `<Peer address="..."/>`。同时注意热点会重新分配 IP，每次需 `hostname -I` 确认。

## 问题组 2：ROS2 包编译/注册（已解决）

### 2.1 `ros2 pkg list` 找不到 `vision_service`

> [!ERROR] `Package 'vision_service' not found`，但 `install/vision_service/` 目录存在

**环境复核**

- [ ] `AMENT_PREFIX_PATH` 只包含 `my_px4_controller`、`px4_msgs`，漏了 `vision_service`
    
- [ ] `.bashrc` 里有语法错误：`source /opt/ros/foxy/setup.bash source /home/aircraft/.bashrc`（递归 source）
    

**解决尝试**

- 尝试 1: `source install/setup.bash` → 无效（单包编译不更新顶层索引）
    
- **最终解法**: 手动 `export AMENT_PREFIX_PATH="$HOME/.../install/vision_service:$AMENT_PREFIX_PATH"`，并清理 `.bashrc` 语法错误
    

**根因复盘** `colcon build --packages-select` 不会强制重写 `install/setup.bash`，导致新包路径没注册进 `AMENT_PREFIX_PATH`。根治方案是编译整个 ws 不加 `--packages-select`，或写 `.bashrc` 硬编码。


### 2.2 Python 节点 `IndentationError: unexpected indent`

> [!ERROR] `vision_server_pc.py` 第 186 行缩进错乱

**解决尝试**

- **最终解法**: `pip3 install autopep8; autopep8 --in-place ...`
    

**根因复盘** 跨平台复制代码时混入了 Tab/空格。以后统一用 `autopep8` 或 `black` 格式化后再保存


## 问题组 3：Gazebo 仿真环境（已解决）

### 3.1 Gazebo 场地道具 `Unable to find uri[model://competition_marker/...]`

> [!ERROR] 启动时大量 `Error Code 14`，然后 `gz_bridge failed to start`

**环境复核**

- [ ] `GZ_SIM_RESOURCE_PATH` 被用户清理掉了，为空
    

**解决尝试**

- **最终解法**: `export GZ_SIM_RESOURCE_PATH="$HOME/competition_ws/PX4-Autopilot/Tools/simulation/gz/models:$HOME/.../worlds"`
    

**根因复盘** `.bashrc` 清理时误删了 Gazebo 模型搜索路径。必须保留，否则 `my_world.sdf` 里的道具全部加载失败。

### 3.2 `ros_gz_bridge` / `ros_gz_image` 在 CycloneDDS 下注册不上 ROS2 节点

> [!ERROR] 日志显示 `Creating GZ->ROS Bridge` 成功，但 `ros2 node list` 为空，`ros2 topic list` 看不到 `/camera`

**环境复核**

- [x] `ROS_DOMAIN_ID=42`
    
- [ ] `RMW_IMPLEMENTATION=rmw_cyclonedds_cpp`（与 `ros_gz_bridge` C++ 节点不兼容）
    

**解决尝试**

- 尝试 1: `ros_gz_image image_bridge /camera` → 参数格式错误
    
- 尝试 2: `ros_gz_bridge parameter_bridge ...` → Lazy 模式不激活 / 节点注册失败
    
- 尝试 3: 切回 Fast DDS `unset RMW_IMPLEMENTATION` → 单机可通
    
- **最终解法**: 单机仿真场景下**全部使用默认 Fast DDS**，不再强制 CycloneDDS
    

**根因复盘** `ros_gz_bridge`（C++）在 CycloneDDS + Humble 下有已知兼容性问题，节点能启动但注册不到 DDS 域。单机跑仿真不需要 CycloneDDS，跨机才需要。

### 3.3 Gazebo 摄像头画面空白 / 不朝下

> [!ERROR] Image display 空白；改 SDF 后画面仍不对

**环境复核**
- [x] OakD-Lite 是远程 Fuel 模型，内部 link 结构复杂，外部 pose 修改无法精确控制相机传感器朝向
    
- [x] `<include>` + `<joint>` 方式只能约束子模型 base_link 的相对位姿，**无法覆盖子模型内部 camera_link 的局部 pose**
    
- [x] 尝试修改 joint/include pose 导致 Roll/Yaw 补偿成 π、穿模白屏等问题
    
- [x] GUI 手动调 Component Inspector 能临时验证，但无法永久化
**解决尝试**

- 尝试 1: 改 `joint` pose → 只改位置不改角度，内部相机仍朝前
    
- 尝试 2: 改 `<include>` pose → 整个子模型旋转后 Z 穿模，内部传感器坐标系混乱
    
- 尝试 3: GUI `Set Pose` → 子模型被 merge 后无此选项
    
- 尝试 4: GUI Component Inspector 手动调 `Pitch: -1.57` + 抬高 Z → 画面验证通过，但重启失效
    
- **最终解法**：**放弃 OakD-Lite，直接在 x500 上自定义 camera_link + camera sensor**
#### 永久化修改步骤
**修改 `x500_depth/model.sdf`，自己定义朝下相机：**
```bash
nano ~/competition_ws/PX4-Autopilot/Tools/simulation/gz/models/x500_depth/model.sdf
```

```xml
<?xml version="1.0" encoding="UTF-8"?>  <!-- XML 文件声明，版本 1.0，UTF-8 编码 -->
<sdf version='1.9'>                        <!-- SDF 格式版本 1.9（对应 Gazebo Garden/Harmonic） -->
  <model name='x500-Depth'>                <!-- 模型定义，名称 x500-Depth（仿真中看到的名字） -->
    <include merge='true'>                 <!-- 引入基础无人机模型，merge=true 表示合并为一个整体 -->
      <uri>x500</uri>                      <!-- 基础无人机模型的名字（PX4 内置） -->
    </include>                             <!-- include 标签结束 -->

    <!-- ========== 自定义朝下相机 ========== -->  <!-- 注释：下面是相机相关定义 -->

    <link name="camera_link">              <!-- 定义一个 link（刚体），名字叫 camera_link -->
      <pose relative_to="base_link">       <!-- 定义该 link 相对于父 link（base_link）的位姿 -->
        0                                  <!-- x=0：在机身正中心（前后不偏移） -->
        0                                  <!-- y=0：在机身正中心（左右不偏移） -->
        0.05                               <!-- z=0.05：在 base_link 上方 5cm（略高于机身底部） -->
        0                                  <!-- roll=0：不绕 X 轴旋转 -->
        1.5708                             <!-- pitch=1.5708（+90°）：绕 Y 轴旋转，让镜头朝下看 -->
        0                                  <!-- yaw=0：不绕 Z 轴旋转 -->
      </pose>                              <!-- pose 标签结束 -->
                                           <!-- 空行：为了代码可读性 -->
      <inertial>                           <!-- 定义 link 的惯性属性（质量和转动惯量） -->
        <mass>0.001</mass>                 <!-- 质量 0.001kg = 1g（相机很轻，近似无质量） -->
        <inertia>                          <!-- 转动惯量矩阵定义开始 -->
          <ixx>1e-6</ixx>                  <!-- 绕 X 轴转动惯量（极小值，近似质点） -->
          <iyy>1e-6</iyy>                  <!-- 绕 Y 轴转动惯量（极小值，近似质点） -->
          <izz>1e-6</izz>                  <!-- 绕 Z 轴转动惯量（极小值，近似质点） -->
        </inertia>                         <!-- 转动惯量矩阵定义结束 -->
      </inertial>                          <!-- 惯性属性定义结束 -->
                                           <!-- 空行：为了代码可读性 -->
      <sensor name="downward_camera" type="camera">  <!-- 定义传感器：名字叫 downward_camera，类型为相机 -->
        <camera>                           <!-- 相机参数配置开始 -->
          <horizontal_fov>1.047</horizontal_fov>  <!-- 水平视场角 1.047 弧度 = 60 度 -->
          <image>                          <!-- 图像输出参数配置 -->
            <width>640</width>             <!-- 图像宽度 640 像素 -->
            <height>480</height>           <!-- 图像高度 480 像素 -->
          </image>                         <!-- 图像参数配置结束 -->
          <clip>                           <!-- 裁剪/可视距离范围配置 -->
            <near>0.1</near>               <!-- 最近可视距离 0.1 米（小于此距离不渲染） -->
            <far>100</far>                 <!-- 最远可视距离 100 米（大于此距离不渲染） -->
          </clip>                          <!-- 可视距离范围配置结束 -->
        </camera>                          <!-- 相机参数配置结束 -->
        <always_on>true</always_on>        <!-- 始终开启传感器，仿真启动后自动运行 -->
        <update_rate>30</update_rate>      <!-- 更新频率 30Hz，每秒输出 30 帧图像 -->
        <visualize>true</visualize>        <!-- 允许在 Gazebo GUI 中可视化预览画面 -->
        <topic>/camera</topic>             <!-- 发布图像数据的话题名为 /camera（ROS2 桥接时订阅这个名字） -->
      </sensor>                            <!-- 传感器定义结束 -->
    </link>                                <!-- camera_link 定义结束 -->
                                           <!-- 空行：为了代码可读性 -->
    <joint name="camera_joint" type="fixed">  <!-- 定义关节：名字叫 camera_joint，固定类型 -->
      <parent>base_link</parent>           <!-- 父 link：无人机机身基座 -->
      <child>camera_link</child>           <!-- 子 link：上面定义的相机（把相机固定在机身上） -->
    </joint>                               <!-- 关节定义结束 -->
  </model>                                 <!-- 模型定义结束 -->
</sdf>                                     <!-- SDF 文件结束 -->
```

**根因复盘**

放弃使用 `<include>` 引入 OakD-Lite 远程模型的方案，因为：

1. 子模型内部有独立的 link/sensor 层级，外部 pose 修改无法穿透到内部 camera 传感器
    
2. `fixed` joint 只约束 parent→child base_link 的相对关系，不覆盖子模型内部坐标系
    
3. 远程模型缓存路径不直观，每次重新下载可能覆盖本地修改
    

**直接在模型中定义 `camera_link` + `camera` sensor** 的优势：

- pose 完全可控：`0 0 0.05 0 1.5708 0` 表示机身中心、略悬空、pitch=90° 垂直朝下
    
- 话题名固定为 `/camera`，ROS2 桥接命令不需要变
    
- 不依赖任何外部模型，代码自包含、可移植、可复现

## 问题组 4：Python 依赖/环境（已解决）

### 4.1 `ModuleNotFoundError: No module named 'gz.transport'`

> [!ERROR] `bridge_simple.py` 导入 Gazebo Python API 失败

**解决尝试**

- 尝试 1: `sudo apt install python3-gz-transport13` → 装了但 Python 路径找不到
    
- **最终解法**: 放弃 Python 桥接，改用 `ros_gz_image` / `ros_gz_bridge`（ROS2 官方桥接）
    

**根因复盘** Gazebo Sim 的 Python 绑定路径经常没对齐，且不同版本命名混乱（`gz` vs `ignition`）。有官方 ROS2 桥接节点时，不需要自己写 Python 桥接。

### 4.2 `numpy` + `scipy` 版本冲突

> [!ERROR] `ValueError: numpy.dtype size changed, may indicate binary incompatibility`

**环境复核**

- [ ] apt 装了 `python3-scipy` (1.8.x)
    
- [ ] pip 装了 `numpy` 2.2.6
    

**解决尝试**

- 尝试 1: `pip3 install "numpy<1.25"` → scipy 1.8 仍不兼容
    
- **最终解法**: `sudo apt remove python3-scipy; pip3 install scipy --upgrade`
    

**根因复盘** apt 的 scipy 是系统编译版，绑定特定 numpy ABI。pip 升级 numpy 后 ABI 断裂。以后 ROS2 Python 节点尽量全用 pip 管理，避免 apt/pip 混用。

### 4.3 `vision_server_pc` 启动报 `best.pt` 不存在

> [!WARN] `[修改3] Classify model load failed... No such file or directory: .../best.pt`

**环境复核**

- [x] `utils/` 目录存在
    
- [ ] 只有 `best.engine`（TensorRT/Jetson），没有 `best.pt`（PyTorch）
    

**解决尝试**

- 尝试 1: 找 NX 上的 `.pt` 文件 → 未找到
    
- **最终解法**: 二维码模式（`mode:=qrcode`）不依赖 YOLO 模型，暂时忽略；classify 模式后续需重新导出 `.pt`
    

**根因复盘** `.engine` 是 TensorRT 序列化文件，平台绑定（Jetson），不能直接在 PC x86_64 上跑。需要在 PC 上用 ultralytics 重新导出 `.pt` 或 `.onnx`。

## 问题组 5：PX4 控制/状态机（已解决）

### 5.1 mission_manager 高频发布导致 PX4 命令风暴

> [!ERROR] `vehicle_command_ack lost` 刷屏 → `Fast CDR exception` → 节点崩溃

**环境复核**

- [x] `create_timer(0.05)` = 20Hz（正常）
    
- [ ] `publish_vehicle_command` 每帧都发，无去重
    

**解决尝试**

- 尝试 1: 降频 timer → 不是根因
    
- **最终解法**: Kimi Code 在 `publish_vehicle_command` 里加 500ms 去重 + `self.context.ok()` 防护
    

**根因复盘** 20Hz 主循环 × 多路命令（DO_SET_MODE + ARM_DISARM + NAV_LAND）= 命令风暴。PX4 SITL 单线程处理不过来，ack 队列溢出，进而导致 DDS 层序列化崩溃。

### 5.2 `Failsafe activated` + `Matching flight task was not able to run`

> [!ERROR] 起飞后几秒 PX4 切 failsafe，无人机不听控制

**环境复核**

- [ ] `trajectory_setpoint` 发布 `x: 0.0, y: 0.0`（飞场地原点）
    
- [ ] `px4_msgs` 版本 `main` 分支，与 PX4 固件 `v1.14.3` 不匹配
    

**解决尝试**

- 尝试 1: 检查航点坐标 → 发现 TAKEOFF 硬编码 `[0.0, 0.0]`
    
- 尝试 2: 检查 px4_msgs 版本 → `main` vs `v1.14.3` 不匹配
    
- **最终解法**:
    
    1. `git checkout origin/release/1.14` 同步 px4_msgs
        
    2. 清理 `~/.ros/rosidl` DDS 缓存
        
    3. TAKEOFF/IDLE 航点改为 `[self.current_pos_enu[0], self.current_pos_enu[1], z]`
        

**根因复盘** 两个独立问题叠加：

1. **版本不匹配**: px4_msgs IDL 定义和 PX4 uORB 字段顺序不同，导致 Fast CDR 反序列化失败，PX4 拒绝执行 Offboard 指令。
    
2. **航点逻辑错误**: TAKEOFF 状态硬编码 `(0,0)`，无人机试图飞到场地角落，PX4 判定为异常轨迹，自动切 failsafe。

### 5.3 `pnp_node_pc` 日志刷屏导致 Gazebo 卡顿

> [!WARN] 终端被 `Published PnP result: array(...)` 占满，帧率暴跌

**解决尝试**

- **最终解法**: `INFO` → `DEBUG` + 每 5 帧处理一次降频
    

**根因复盘** `get_logger().info()` 每帧打印大量浮点数组，终端渲染成为瓶颈。视觉推理节点应默认静默，只在调试时开 DEBUG。


## 问题组 6：未解决 / 待办

### 6.1 YOLO 分类模型缺失（`best.pt`）

> [!WARN] classify 模式不可用，只有二维码模式能跑

**现状**

- NX 有 `best.engine`，PC 没有 `best.pt`
    
- 比赛需要识别 CIFAR-100 图片靶类别
    

**下一步**

- 在 PC 上用 ultralytics 重新训练/导出 `best.pt`
    
- 或从 NX 拷贝原始 `.pt` 权重（如果存在）

### 6.2 PnP 圆环闭环未接入 mission_manager

> [!WARN] `pnp_callback` 是空壳，`/pnp/result` 数据未用于导航

**现状**

- `vision_pnp_pc` 在发布位姿数据
    
- `mission_manager` 里 `ring_pos_enu` 未从 `/pnp/result` 更新
    

**下一步**

- 在 `mission_manager` 的 `pnp_callback` 里把 `tx_mm, ty_mm, tz_mm` 转为 ENU 偏移，赋值给 `self.ring_pos_enu`，实现视觉闭环穿环。

### .4 真机迁移（Jetson NX）

> [!TODO] 仿真验证通过后，需迁移到真实无人机

**待办清单**

- [ ] NX 上安装 ROS2 Foxy + CycloneDDS（已配置）
    
- [ ] 真实摄像头（USB/CSI）替换 Gazebo `/camera`
    
- [ ] 真实投放机构 GPIO/舵机节点，订阅 `/drop_cmd`
    
- [ ] 场地坐标从 `my_world.sdf` 转为真实 9m×6m 场地测量
    
- [ ] QGroundControl 地面站监控
