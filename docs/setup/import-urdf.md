# Isaac Sim 导入 URDF 记录

本文记录将 `mobile_aloha_sim` 中的 ALOHA 移动双臂机器人 URDF 导入 Isaac Sim 的过程，以及导入后材质颜色丢失的原因和处理方法。

## 目标模型

原始 URDF：

```text
/home/xiangpc/dataset/isaac-sim/mobile_aloha_sim/split_aloha_mid_360/urdf/split_aloha_rslidar_with_piper.urdf
```

这个文件由 xacro 生成，包含：

- `split_aloha_mid_360` 移动底盘、轮子、升降结构、相机和雷达。
- `piper_description` 左右两套 Piper 机械臂。
- 左右机械臂通过 fixed joint 安装在 `lifting_link` 上。

生成的 Isaac 专用 URDF：

```text
/home/xiangpc/dataset/isaac-sim/mobile_aloha_sim/split_aloha_mid_360/urdf/split_aloha_rslidar_with_piper_isaac_material.urdf
```

导出的 USD 建议保存到：

```text
/home/xiangpc/dataset/isaac-sim/assets
```

## 启动 Isaac Sim

在宿主机启动 Isaac Sim。导入前设置 `ROS_PACKAGE_PATH`，让 URDF Importer 能解析 `package://split_aloha_mid_360` 和 `package://piper_description`。

```bash
cd /home/xiangpc/dataset/isaac-sim

mkdir -p /home/xiangpc/dataset/isaac-sim/assets

export ROS_PACKAGE_PATH=/home/xiangpc/dataset/isaac-sim/mobile_aloha_sim:${ROS_PACKAGE_PATH}

./isaac-sim-standalone-5.1.0-linux-x86_64/isaac-sim.sh --no-ros-env
```

如果后续要同时启用 ROS2 Bridge，需要额外设置 ROS2 RMW 环境。单纯导入 URDF 时可以先不处理 ROS2 Bridge warning。

## GUI 导入步骤

1. 打开 Isaac Sim。
2. 打开 `Window -> Extensions`。
3. 搜索并启用 URDF Importer。
4. 使用 URDF Importer 选择 Isaac 专用 URDF：

```text
/home/xiangpc/dataset/isaac-sim/mobile_aloha_sim/split_aloha_mid_360/urdf/split_aloha_rslidar_with_piper_isaac_material.urdf
```

5. 建议导入选项：

```text
Fix Base Link: 关闭
Merge Fixed Joints: 关闭
Import Inertia Tensor: 开启
Convex Decomposition: 先关闭
```

6. 导出 USD，例如：

```text
/home/xiangpc/dataset/isaac-sim/assets/split_aloha_rslidar_with_piper_isaac_material.usd
```

## 为什么第一次导入没有颜色

在 RViz 或 Robot Viewer 中打开模型时，机器人是有颜色的；但是第一次直接用 Isaac Sim 导入原始 URDF 时，机器人看起来像白模或灰模。

原因不是 mesh 文件不存在，也不是模型没有材质。`meshes` 目录下的 `.dae` 文件确实包含 COLLADA 材质颜色。例如 `ranger_base.dae` 内部有 `library_materials`、`diffuse color` 等材质定义。

问题主要出在 URDF visual 里额外写了统一材质：

```xml
<visual>
  <geometry>
    <mesh filename="package://split_aloha_mid_360/meshes/ranger_base.dae" scale="1000 1000 1000"/>
  </geometry>
  <material name="">
    <color rgba="0.792156862745098 0.819607843137255 0.933333333333333 1"/>
  </material>
</visual>
```

RViz/Robot Viewer 更倾向于保留 DAE 内部的材质分区，因此能看到模型颜色。Isaac Sim 的 URDF Importer 在 URDF 转 USD 时，可能优先使用 URDF 里的统一 `<material><color .../></material>`，从而覆盖 DAE 自带的材质颜色。

## 解决方法

不要直接修改自动生成的原始 URDF，而是生成一个 Isaac 专用 URDF 副本：

```text
split_aloha_rslidar_with_piper_isaac_material.urdf
```

这个文件的处理方式是：

- 保留所有 link、joint、inertial、collision。
- 保留 visual 里的 mesh 引用。
- 删除 mesh visual 下会覆盖 DAE 内嵌材质的 URDF `<material>` 块。
- 不修改原始 `split_aloha_rslidar_with_piper.urdf`。

处理前：

```xml
<visual>
  <origin rpy="0 0 0" xyz="0 0 0.0"/>
  <geometry>
    <mesh filename="package://split_aloha_mid_360/meshes/ranger_base.dae" scale="1000 1000 1000"/>
  </geometry>
  <material name="">
    <color rgba="0.792156862745098 0.819607843137255 0.933333333333333 1"/>
  </material>
</visual>
```

处理后：

```xml
<visual>
  <origin rpy="0 0 0" xyz="0 0 0.0"/>
  <geometry>
    <mesh filename="package://split_aloha_mid_360/meshes/ranger_base.dae" scale="1000 1000 1000"/>
  </geometry>
</visual>
```

这样 Isaac Sim 导入时不会被 URDF 统一颜色覆盖，可以读取 `.dae` 自带的材质颜色。

## 已完成的处理

已经生成：

```text
/home/xiangpc/dataset/isaac-sim/mobile_aloha_sim/split_aloha_mid_360/urdf/split_aloha_rslidar_with_piper_isaac_material.urdf
```

本次生成过程中移除了 26 个 mesh visual 上的 URDF 材质覆盖块。

重新用这个 URDF 导入 Isaac Sim 后，机器人在 Isaac Sim 中可以显示出 mesh 自带的颜色和材质效果。

## 常见 warning

### ROS2 Bridge RMW warning

如果启动 Isaac Sim 时看到：

```text
failed to load any RMW implementations
ROS2 Bridge startup failed
```

这和 URDF 导入本身无关。只是 ROS2 Bridge 没有找到 RMW 库。单纯导入模型时可以忽略。

后续需要 ROS2 联调时，再设置：

```bash
export ROS_DISTRO=humble
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/xiangpc/dataset/isaac-sim/isaac-sim-standalone-5.1.0-linux-x86_64/exts/isaacsim.ros2.bridge/humble/lib
```

### link 或 joint 名称中的斜杠

如果看到：

```text
The path left/link1 is not a valid usd path, modifying to left_link1
```

这是因为 URDF link 名称里有 `/`。USD 里 `/` 是路径分隔符，不能作为 prim 名称的一部分，所以 Isaac Sim 会自动改名。

这通常不影响导入结果，但后续写脚本引用 USD prim 时，要使用 Isaac 转换后的名称，例如 `left_link1`。

### fixed joint axis warning

如果看到：

```text
Joint Axis is not body aligned with X, Y or Z primary axis
```

常见于 fixed joint，例如 `box_joint`、`front_camera_joint`、`lidar_joint`、`arm_and_lifting_left`、`arm_and_lifting_right`。通常不影响模型显示。

后续如果要进一步清理 warning，可以在 xacro 中移除 fixed joint 上无意义的 `<axis>` 和 `<limit>`。

## 导入后检查

导入后重点检查：

- 底盘、四个轮子、升降机构是否显示完整。
- 前相机、顶相机、雷达是否显示。
- 左右 Piper 机械臂是否显示完整。
- 机器人是否落在地面上；如果悬空，调整机器人 root prim 的 `Translate Z`。
- 关节是否保留，尤其是 `lifting_joint`、四个转向关节、四个轮子连续关节、左右机械臂关节。
- 材质颜色是否保留。如果又变成白模，确认导入的是 Isaac 专用 URDF，而不是原始 URDF 或旧 USD。

