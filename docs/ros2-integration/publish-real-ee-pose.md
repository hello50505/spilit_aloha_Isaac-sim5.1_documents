# 发布真实末端位姿

本文记录新的末端位姿发布方式：不再使用 cuRobo 正运动学从 `/joint_states`
计算当前末端位姿，而是直接使用 Isaac Sim 发布到 `/tf` 的真实 link 位姿。

## 目标

- 订阅 `/tf`。
- 读取左右夹爪两指 `link7` 和 `link8` 的世界位姿。
- 末端 TCP 位置取两指位置中心。
- 末端 TCP 姿态沿用 `link8` 的世界姿态。
- 发布左右末端世界位姿：

```text
/left_ee_current_pose_in_world
/right_ee_current_pose_in_world
```

## 新节点

ROS2 可执行：

```text
real_ee_pose_publisher
```

源码：

```text
dev_ws/ros2_ws/src/mobile_aloha_isaac_control/mobile_aloha_isaac_control/real_ee_pose_publisher.py
```

默认随 `mobile_aloha_control.launch.py` 启动：

```text
start_real_ee_pose:=true
```

旧的 cuRobo FK 位姿发布节点不再默认启动：

```text
start_ee_pose_fk:=false
```

如果临时需要对比旧实现，可以手动打开：

```bash
./scripts/run_ros2_humble_docker.sh --build --launch-control -- \
  start_ee_pose_fk:=true
```

注意：不要同时让两个节点发布同名当前末端位姿 topic，否则下游会收到两个来源的数据。

## 计算逻辑

Isaac Sim 的 `ROS2PublishTransformTree` 会发布机器人 TF。新节点维护一张 TF 表：

```text
child_frame -> parent_frame, translation, rotation
```

每次发布时，节点从目标 finger frame 向上查父节点，直到 `world`，组合得到：

```text
world -> left_link7
world -> left_link8
world -> right_link7
world -> right_link8
```

然后计算：

```text
tcp_position = 0.5 * (link7_world_position + link8_world_position)
tcp_orientation = link8_world_orientation
```

发布的 `PoseStamped.header.frame_id` 为：

```text
world
```

## 配置来源

节点复用 `mobile_aloha_joints.yaml` 中的 frame 配置：

```yaml
ee_pose_control:
  sides:
    left:
      tf_finger_frames:
        - left_link7
        - left_link8
      tf_tcp_orientation_frame: left_link8
    right:
      tf_finger_frames:
        - right_link7
        - right_link8
      tf_tcp_orientation_frame: right_link8
```

发布频率使用：

```yaml
ee_pose_state:
  publish_rate_hz: 50.0
  tf_topic: /tf
  world_frame: world
```

## 检查

启动 Isaac Sim 和 ROS2 控制后检查：

```bash
ros2 topic echo --once /right_ee_current_pose_in_world
ros2 topic echo --once /left_ee_current_pose_in_world
```

场景中黄色小方块会跟随右臂当前真实 TCP：

```text
/World/right_ee_current_pose_marker
```

它由 Isaac 启动脚本订阅 `/right_ee_current_pose_in_world` 后更新，用于和绿色目标方块
`/World/right_ee_target_pose_marker` 对比调试。

检查 TF 是否包含 finger frames：

```bash
ros2 topic echo /tf
```

应能看到：

```text
left_link7
left_link8
right_link7
right_link8
```

如果节点日志提示等待 finger TF，说明 Isaac 场景里的
`ROS2PublishTransformTree` 还没有发布这些 frame，或场景尚未 Play。
