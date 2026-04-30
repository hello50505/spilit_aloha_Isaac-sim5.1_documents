# Isaac Sim 发布相机图像到 ROS2

本文接着 `3_import_camera.md` 和 `4_control_ROS.md`，记录如何把当前 Mobile ALOHA 场景里的四路 Isaac Sim 相机画面发布到 ROS2 Humble 网络。

当前保留的方案是 Isaac Sim ROS2 Bridge raw 图像发布：

```text
Camera prim -> render product -> ROS2CameraHelper -> sensor_msgs/Image
```

之前尝试过 Isaac 侧直接 JPEG 压缩发布 `sensor_msgs/CompressedImage`，但四路同时开启时帧率没有明显收益，相关运行时代码已移除。

## 当前阶段

当前做：

- 发布四路 RGB raw 图像。
- 发布四路 `camera_info`。
- 保留原始 `assets/4.mobile_aloha_in_market.usda`。
- 通过脚本输出带相机 ROS2 graph 的 `assets/5.mobile_aloha_in_market_ros_camera.usda`。
- 不修改 `/cmd_vel`、`/isaac_joint_commands`、`/joint_states` 控制链路。

当前不做：

- 不发布 depth 图像。
- 不发布点云。
- 不发布 compressed image。

## 输入和输出

输入场景：

```text
/home/xiangpc/dataset/isaac-sim/assets/4.mobile_aloha_in_market.usda
```

输出场景：

```text
/home/xiangpc/dataset/isaac-sim/assets/5.mobile_aloha_in_market_ros_camera.usda
```

生成脚本：

```text
/home/xiangpc/dataset/isaac-sim/dev_ws/isaac/scripts/create_mobile_aloha_camera_ros_graph.py
```

## 相机 prim

当前 4 号 USD 中已经有四个真实 `Camera` prim：

```text
/World/split_aloha_rslidar_with_piper_isaac_material/front_camera_link/front_camera_sensor
/World/split_aloha_rslidar_with_piper_isaac_material/top_camera_link/top_camera_sensor
/World/split_aloha_rslidar_with_piper_isaac_material/left_link6/left_camera_sensor
/World/split_aloha_rslidar_with_piper_isaac_material/right_link6/right_camera_sensor
```

它们不是相机外壳 mesh，而是 Isaac Sim 可渲染相机。ROS2 发布 graph 基于这些 Camera prim 创建 render product，再交给 ROS2 Bridge 发布。

## 生成带相机发布 graph 的 USD

在宿主机执行：

```bash
cd /home/xiangpc/dataset/isaac-sim

./isaac-sim-standalone-5.1.0-linux-x86_64/python.sh \
  dev_ws/isaac/scripts/create_mobile_aloha_camera_ros_graph.py
```

脚本默认会：

- 打开 `assets/4.mobile_aloha_in_market.usda`。
- 检查四个 Camera prim 是否存在。
- 创建 `/World/MobileAlohaROSCameraGraph`。
- 为每个相机创建一个 `IsaacCreateRenderProduct`。
- 为每个 render product 创建 `ROS2CameraHelper(type="rgb")`。
- 为每个 render product 创建 `ROS2CameraInfoHelper`。
- 导出 `assets/5.mobile_aloha_in_market_ros_camera.usda`。

如果需要指定分辨率：

```bash
./isaac-sim-standalone-5.1.0-linux-x86_64/python.sh \
  dev_ws/isaac/scripts/create_mobile_aloha_camera_ros_graph.py \
  --width 640 \
  --height 480
```

## ROS2 话题

生成后的场景会发布这些 RGB 图像话题：

| Topic | Type | Frame ID |
| --- | --- | --- |
| `/mobile_aloha/front_camera/rgb` | `sensor_msgs/msg/Image` | `front_camera_link` |
| `/mobile_aloha/top_camera/rgb` | `sensor_msgs/msg/Image` | `top_camera_link` |
| `/mobile_aloha/left_camera/rgb` | `sensor_msgs/msg/Image` | `left_link6` |
| `/mobile_aloha/right_camera/rgb` | `sensor_msgs/msg/Image` | `right_link6` |

对应的 `camera_info` 话题：

| Topic | Type | Frame ID |
| --- | --- | --- |
| `/mobile_aloha/front_camera/camera_info` | `sensor_msgs/msg/CameraInfo` | `front_camera_link` |
| `/mobile_aloha/top_camera/camera_info` | `sensor_msgs/msg/CameraInfo` | `top_camera_link` |
| `/mobile_aloha/left_camera/camera_info` | `sensor_msgs/msg/CameraInfo` | `left_link6` |
| `/mobile_aloha/right_camera/camera_info` | `sensor_msgs/msg/CameraInfo` | `right_link6` |

## 启动流程

### 1. 启动 Isaac Sim

宿主机终端执行：

```bash
cd /home/xiangpc/dataset/isaac-sim
./dev_ws/scripts/run_isaac_mobile_aloha.sh
```

Isaac Sim 打开后：

1. 打开 `assets/5.mobile_aloha_in_market_ros_camera.usda`。
2. 确认 `isaacsim.ros2.bridge` 已启用。
3. 点击 Play。

说明：`run_isaac_mobile_aloha.sh` 会先清理 Conda、系统 CUDA、外部 Python 路径相关环境变量，避免污染 Isaac Sim standalone 自带 Torch/CUDA；然后设置 `ROS_DISTRO=humble`、`RMW_IMPLEMENTATION=rmw_cyclonedds_cpp`、ROS2 Bridge 的 `LD_LIBRARY_PATH` 和 Isaac Sim 自带 Humble `rclpy` 的 `PYTHONPATH`，并通过 `--exec` 自动启动底盘运动学控制器。

### 2. 启动 ROS2 Docker

另开宿主机终端：

```bash
cd /home/xiangpc/dataset/isaac-sim/dev_ws
./scripts/run_ros2_humble_docker.sh
```

如果还需要同时启动底盘和机械臂控制节点：

```bash
./scripts/run_ros2_humble_docker.sh --launch-control
```

Docker 使用 host 网络，因此容器里的 ROS2 节点和宿主机 Isaac Sim ROS2 Bridge 在同一个 ROS2 网络中。

## 验证方法

进入 Docker 容器后执行：

```bash
ros2 topic list -t | grep mobile_aloha
```

应该能看到四路 RGB 和四路 `camera_info`。

检查一路 RGB 图像帧率：

```bash
ros2 topic hz /mobile_aloha/front_camera/rgb
```

检查一路 `camera_info`：

```bash
ros2 topic echo --once /mobile_aloha/front_camera/camera_info
```

用图像工具看画面：

```bash
rqt_image_view
```

然后在界面里选择：

```text
/mobile_aloha/front_camera/rgb
```

如果使用 RViz2，添加 `Image` 显示项，并选择对应 RGB topic。

## 关于 compressed image

本项目暂时不保留 Isaac 侧直接发布 `CompressedImage` 的功能。

原因：

- 四路同时开启时，瓶颈主要在 Isaac 多相机渲染和取图，不在 DDS raw 图像传输。
- Python 侧 JPEG 编码还会增加额外 CPU 开销。
- 实测没有解决四路相机帧率不足的问题。

如果后续仍需要 compressed topic，建议先明确目标是降低下游带宽、降低 rosbag 体积，还是提高源头帧率。这三类目标需要不同方案。

## 常见问题

### ROS2 里看不到相机 topic

检查：

- Isaac Sim 是否打开的是 `assets/5.mobile_aloha_in_market_ros_camera.usda`，不是 4 号 USD。
- Isaac Sim 是否点击了 Play。
- `isaacsim.ros2.bridge` 扩展是否启用。
- 宿主机和 Docker 的 `ROS_DOMAIN_ID` 是否一致。
- 宿主机和 Docker 是否都使用 `RMW_IMPLEMENTATION=rmw_cyclonedds_cpp`。

### 只有 `/joint_states`，没有相机 topic

说明控制 graph 正常，但相机 graph 没有生效。

检查 Stage 里是否存在：

```text
/World/MobileAlohaROSCameraGraph
```

如果不存在，重新生成 5 号 USD：

```bash
cd /home/xiangpc/dataset/isaac-sim

./isaac-sim-standalone-5.1.0-linux-x86_64/python.sh \
  dev_ws/isaac/scripts/create_mobile_aloha_camera_ros_graph.py
```

### 图像方向不对

这属于相机 prim 的 transform 问题，不是 ROS2 Bridge 问题。

处理方法：

- 回到 `3_import_camera.md` 的相机检查方法。
- 在 Isaac Sim viewport 中逐个切换相机。
- 调整对应 Camera prim 的 local rotation。
- 调好后重新生成或保存 5 号 USD。

### 图像太大或帧率太低

四路 raw 图像会增加 Isaac 渲染和 ROS2 传输压力。

可以降低分辨率重新生成：

```bash
./isaac-sim-standalone-5.1.0-linux-x86_64/python.sh \
  dev_ws/isaac/scripts/create_mobile_aloha_camera_ros_graph.py \
  --width 320 \
  --height 240
```

也可以减少相机数量，但当前 graph 生成脚本默认仍创建四路相机发布。

## 当前验收标准

完成后应满足：

- `assets/5.mobile_aloha_in_market_ros_camera.usda` 能正常打开。
- Stage 里存在 `/World/MobileAlohaROSCameraGraph`。
- Docker 内能看到四路 `sensor_msgs/msg/Image`。
- Docker 内能看到四路 `sensor_msgs/msg/CameraInfo`。
- 原有 `/cmd_vel`、`/isaac_joint_commands`、`/joint_states`、`/clock` 不受影响。
