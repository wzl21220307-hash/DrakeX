## 节点 1：`ros_gz_image` / `image_bridge`

> [!INFO] 官方节点，你没有写它的代码，但必须知道它干嘛

### 它是干嘛的

**Gazebo 的「传话筒」**。Gazebo 内部有一套自己的图像格式（Ignition/Gazebo Sim 的 `gz.msgs.Image`），ROS2 不认识。这个节点把 Gazebo 的图像翻译成 ROS2 能看懂的 `sensor_msgs/Image`，发布到 `/camera`。

### 你启动它的命令
```bash
ros2 run ros_gz_image image_bridge /camera
```
### 内部逻辑（伪代码，官方 C++ 写的）
```cpp
// 订阅 Gazebo 内部话题 /camera
// 收到 gz.msgs.Image {
//   width=640, height=480, 
//   data=[RGB bytes...], 
//   pixel_format=RGB_INT8 
// }

// 翻译成 ROS2 sensor_msgs/Image
ros_image.encoding = "rgb8";        // Gazebo 默认 rgb8
ros_image.data = gz_image.data;     // 直接拷贝字节
ros_image.step = 640 * 3;           // 一行 640 像素 * 3 字节(RGB)

// 发布到 ROS2
publisher.publish(ros_image);       // Topic: /camera
```
### 输入输出

|方向|话题|类型|说明|
|:--|:--|:--|:--|
|输入|`/camera` (Gazebo 内部)|`gz.msgs.Image`|从 Gazebo 的 OakD-Lite 相机来|
|输出|`/camera` (ROS2)|`sensor_msgs/Image`|你的 Python 节点订阅这个|
### 关键注意点

- Gazebo 默认发 `rgb8`，但 OpenCV 默认用 `bgr8`
    
- 所以你的 `vision_server_pc.py` 里必须有 `cv2.cvtColor(img, cv2.COLOR_RGB2BGR)`