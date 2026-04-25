>[!INFO] PX4 官方二进制，你没有代码，但要知道它干嘛
### 它是干嘛的
**ROS2 和 PX4 之间的「翻译官」**。PX4 飞控内部用 uORB 消息格式，ROS2 用 DDS 消息格式，两者语言不通。MicroXRCEAgent 坐在中间，把两边的话互相翻译。
### 启动命令
```bash
MicroXRCEAgent udp4 -p 8888
```
### 内部逻辑（概念性）
```bash
// 监听 UDP 8888（PX4 飞控连这个端口）
while (true) {
    // 从 PX4 收到 uORB 消息（比如 vehicle_local_position）
    uorb_msg = receive_from_px4_udp(8888);
    
    // 翻译成 ROS2 px4_msgs 格式
    ros_msg = convert_uorb_to_ros2(uorb_msg);
    
    // 发布到 ROS2 DDS
    dds_publish("/fmu/out/vehicle_local_position", ros_msg);
    
    // 反过来：从 ROS2 收到控制指令
    ros_cmd = dds_subscribe("/fmu/in/trajectory_setpoint");
    // 翻译成 PX4 格式,并发送
    uorb_cmd = convert_ros2_to_uorb(ros_cmd);
    send_to_px4_udp(8888, uorb_cmd);
}
```
### 为什么必须先启动它
PX4 SITL 启动时会尝试连接 UDP 8888。如果 Agent 没先开，PX4 报 `gz_bridge failed to start`，整个仿真崩掉。
