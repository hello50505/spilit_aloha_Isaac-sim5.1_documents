# Isaac Sim 导入真实相机记录

本文接着 `2_import_urdf.md`，记录为什么打开当前 USD 后只能看到相机外壳、没有真正的相机画面，以及如何先在 Isaac Sim 内部给机器人加上可渲染的真实相机。

当前阶段只做一件事：

- 在 Isaac Sim/USD 里创建真实 `Camera` prim，并确认视口里能看到相机画面。

暂时不做：

- 不把相机图像发布到 ROS2 网络。
- 不创建 ROS2 topic。
- 不配置 RViz 或 `ros2 topic echo`。

ROS2 Bridge 发布图像放到下一阶段处理。

## 当前现象

打开当前场景：

```text
/home/xiangpc/dataset/isaac-sim/assets/4.mobile_aloha_in_market.usd
```

机器人模型可以显示，相机外壳也能看到，但 Isaac Sim 里没有真正的摄像头画面。

这不是显示 bug，而是因为当前模型里有的是“相机外观 mesh”，不是 Isaac Sim 的“渲染相机”。

## 相机外壳和真实相机的区别

URDF 中这些 link 只是普通机器人 link：

```text
front_camera_link
top_camera_link
left/link6
right/link6
```

它们包含相机外壳 mesh，例如：

```text
front_camera_link.dae
top_camera_link.dae
camera_v3.dae
```

这些 mesh 只负责显示几何外观。Isaac Sim 不会因为一个 link 长得像相机，就自动生成 RGB 图像、depth 图像或相机内参。

真正能出画面的相机，必须是 USD 场景里的 `Camera` prim，也就是 Isaac Sim/Omniverse 的渲染相机。例如：

```text
/World/mobile_aloha/front_camera_link/front_camera_sensor
```

这里的 `front_camera_sensor` 才是能被视口切换、能创建 render product、能渲染图像的真实相机。

## 不能只靠重新导入 URDF

当前 Isaac 专用 URDF：

```text
/home/xiangpc/dataset/isaac-sim/mobile_aloha_sim/split_aloha_mid_360/urdf/split_aloha_rslidar_with_piper_isaac_material.urdf
```

主要作用是描述：

- link。
- joint。
- inertial。
- collision。
- visual mesh。
- 材质保留策略。

这个 URDF 里没有 `<sensor>` / `<camera>` 配置。因此重新用它导入 USD，仍然只会得到机器人结构和相机外壳，不会自动得到真实图像。

理论上 Isaac Sim 的 URDF Importer 有相机相关结构，但对当前项目来说，推荐不要把传感器逻辑继续塞回 URDF。更稳定的路线是：

```text
URDF 负责机器人本体
USD 负责 Isaac Sim 场景和真实传感器
ROS2 Bridge 后续负责通信
```

## 官方提供了什么

Isaac Sim 官方确实提供真实相机能力，但链路是 Isaac Sim 的相机系统，不是 camera mesh 自动驱动。

当前阶段需要用到：

- `UsdGeom.Camera`：USD 原生 Camera prim。
- `isaacsim.sensors.camera.Camera`：Isaac Sim Python API 中的相机封装。
- render product：相机渲染结果的输出对象。
- Isaac Sim viewport：用来临时切换相机视角，确认相机画面是否正确。

本地可参考的官方示例：

```text
/home/xiangpc/dataset/isaac-sim/isaac-sim-standalone-5.1.0-linux-x86_64/standalone_examples/api/isaacsim.sensors.camera/camera_ros.py
```

这个示例虽然名字里有 `ros`，但重点是演示如何根据相机参数创建 Isaac Sim 相机并渲染图像。当前阶段只参考它创建相机、设置分辨率、焦距、视场角和 clipping range 的部分。

后续接 ROS2 时再参考：

```text
/home/xiangpc/dataset/isaac-sim/isaac-sim-standalone-5.1.0-linux-x86_64/standalone_examples/api/isaacsim.ros2.bridge/camera_manual.py
```

当前不要先接 ROS2，避免把“相机本身没有图像”和“ROS2 Bridge 没发布”混在一起排查。

当前内部验证链路是：

```text
Camera prim -> Isaac Sim viewport -> 看到真实渲染画面
```

后续需要程序化采图时，再扩展为：

```text
Camera prim -> render product -> Python/annotator 读取图像
```

再下一阶段才是：

```text
Camera prim -> render product -> ROS2 Bridge -> ROS2 topic
```

## 推荐实施路线

推荐先在 USD 层给已有相机 link 挂真实 Camera prim。

不要优先修改：

```text
split_aloha_rslidar_with_piper_isaac_material.urdf
```

建议流程：

```text
1. 打开当前 USD
2. 找到已有相机外壳 link
3. 在相机 link 下创建 Camera prim
4. 调整 Camera 的位置和朝向
5. 切换 viewport 到这个 Camera
6. 确认 Isaac Sim 内部能看到真实画面
7. 保存为新的 USD
```

建议保存为新文件，不覆盖当前场景：

```text
/home/xiangpc/dataset/isaac-sim/assets/4.mobile_aloha_in_market_with_camera.usd
```

这样原始导入结果仍然保留，后续如果相机方向或参数不对，可以反复调整新 USD。

## 推荐相机 prim 命名

建议在已有相机 link 下创建真实相机 prim：

```text
front_camera_link/front_camera_sensor
top_camera_link/top_camera_sensor
left/link6/left_wrist_camera_sensor
right/link6/right_wrist_camera_sensor
```

导入 USD 后，Isaac Sim 可能会把 URDF 里的斜杠改成下划线。例如：

```text
left/link6 -> left_link6
right/link6 -> right_link6
```

因此实际 USD prim 路径要以 Stage 面板里看到的名称为准。不要直接照抄 URDF link 名称。

## GUI 临时验证步骤

先用 GUI 做一次最小验证，确认思路正确。

1. 启动 Isaac Sim：

```bash
cd /home/xiangpc/dataset/isaac-sim

./isaac-sim-standalone-5.1.0-linux-x86_64/isaac-sim.sh --no-ros-env
```

2. 打开场景：

```text
/home/xiangpc/dataset/isaac-sim/assets/4.mobile_aloha_in_market.usd
```

3. 在 Stage 面板里找到前置相机外壳 link。

可能的名称是：

```text
front_camera_link
```

4. 在这个 link 下创建一个 Camera prim。

可以通过菜单创建：

```text
Create -> Camera
```

然后把 Camera prim 移动到 `front_camera_link` 下面，并命名为：

```text
front_camera_sensor
```

5. 调整相机相对 link 的 transform。

如果先做粗调，可以让 Camera prim 的 local translation 接近：

```text
Translate: 0 0 0
Rotate:    0 0 0
```

如果画面方向不对，再按视口结果调整 rotation。不同 mesh 的本体坐标轴不一定等于 Isaac Camera 的光轴方向，所以第一次很可能需要手动校正。

6. 在 viewport 中切换到这个相机。

在视口左上角相机下拉菜单中选择：

```text
front_camera_sensor
```

如果画面切换成功，说明这个 Camera prim 已经是真实可渲染相机。

7. 保存为新 USD：

```text
/home/xiangpc/dataset/isaac-sim/assets/4.mobile_aloha_in_market_with_camera.usd
```

## 判断是否真的有相机画面

正确状态：

- Stage 里有类型为 `Camera` 的 prim。
- viewport 可以切换到这个相机。
- 切换后能看到机器人前方或相机朝向方向的画面。
- 移动机器人或场景物体时，相机画面会随之变化。

错误状态：

- Stage 里只有 `front_camera_link`，没有 `front_camera_sensor`。
- 只能看到相机外壳 mesh，不能在 viewport 相机列表中选择它。
- 视口仍然只能使用 `Perspective`、`Top`、`Front` 等默认相机。

只要没有 `Camera` prim，就还没有真正的 Isaac Sim 相机。

## 当前阶段验收标准

完成当前阶段后，至少满足：

- 新 USD 文件能正常打开。
- Stage 面板中能看到真实 `Camera` prim，例如 `front_camera_sensor`。
- viewport 相机列表中能选择这个相机。
- 切换后能看到场景画面，而不是空白、黑屏或机器人内部。
- 移动相机父 link 或机器人时，相机视角跟随变化。
- 不要求 ROS2 中出现任何 camera topic。

## 当前参数建议

当前先不强求相机和 URDF/真实 RealSense 完全标定对齐，只要求 Isaac Sim 里有可用画面。

建议先保留 Isaac Sim 创建 Camera 时给出的默认投影参数，不要直接把官方示例里的 `fx/fy/cx/cy` 换算值手写进 `.usda`。这些示例值用于 Python API 时还要配合 Isaac Sim 的单位和相机 API，直接写到 USD Camera 属性里可能导致视场角异常，画面看起来被放大。

当前这个左腕相机可以先保持类似下面的参数：

```text
projection:    perspective
focalLength:   18.147562
focusDistance: 400
```

`clippingRange` 按当前能避开相机外壳的值保留即可。后续如果要调画面大小，优先在 Isaac Sim UI 里调 `focalLength` 或 Field of View，并用视口直接观察画面尺度。

当前 `4.mobile_aloha_in_market.usda` 已按同一套基础参数添加四个相机：

```text
front_camera_link/front_camera_sensor
top_camera_link/top_camera_sensor
left_link6/left_camera_sensor
right_link6/right_camera_sensor
```

这四个相机先使用相同的起步配置：

```text
projection:    perspective
focalLength:   18.147562
focusDistance: 400
translate:     0 0 0.15
```

这样做的目的只是先让四个位置都有可切换的真实 Camera prim。不同相机 link 的坐标轴可能不同，所以前置、顶部、左右腕部相机的朝向不一定一次就对。保存后需要在 Isaac Sim 里逐个切换视角检查，再分别微调各自 Camera prim 的 rotation。

调参原则：

- 画面太近、物体太大：减小 `focalLength` 或增大 FOV。
- 画面太广、物体太小：增大 `focalLength` 或减小 FOV。
- 不要为了修画面大小移动 Camera prim。
- 不要为了修相机外壳遮挡去改内参。

以后如果需要严格按 RealSense D435/D435i 标定，再单独做一版：先拿真实 `camera_info`，再用脚本统一创建 Camera、render product 和 ROS2 `camera_info`。当前文档不展开这部分。

## 画面检查方法

当前阶段只做视觉检查：

1. 在 viewport 里切到目标 Camera。
2. 确认画面方向是相机应该看的方向。
3. 确认画面没有明显被相机外壳遮住。
4. 确认画面尺度大致可用，不要过度放大或过度广角。
5. 移动机器人或机械臂时，相机画面应跟随对应 link 运动。

## 是否需要 render product

如果只是想在 Isaac Sim 里先看到画面，viewport 切换到 Camera prim 就够了。

render product 是下一步更程序化的输出对象，用于：

- Python 中读取图像。
- 保存图像。
- 后续接 synthetic data annotator。
- 后续接 ROS2 Bridge。

当前阶段可以先不手动创建 render product。等确认 Camera prim 朝向、位置和参数正确后，再用脚本创建 render product 会更稳。

## 后续脚本固化思路

GUI 验证通过后，建议用 Python 脚本固化相机创建过程。脚本可以做这些事：

- 打开 `4.mobile_aloha_in_market.usd`。
- 检查目标 link 是否存在。
- 在 link 下创建 `Camera` prim。
- 设置相机 local transform。
- 设置焦距、光圈、分辨率相关参数。
- 保存为 `4.mobile_aloha_in_market_with_camera.usd`。

这样以后重新导入 URDF 或重新生成 USD 时，可以重复执行脚本恢复相机配置。

脚本固化属于下一步工作，本记录先不创建脚本。

## 常见问题

### 能看到相机模型，但 viewport 没有相机可选

说明现在只有 camera mesh，没有真实 `Camera` prim。

处理方法：

- 在 Stage 面板确认是否存在类型为 `Camera` 的 prim。
- 如果没有，手动创建 `front_camera_sensor`。
- 确认它挂在正确的相机 link 下。

### 切换相机后画面是黑的

可能原因：

- 相机朝向地面、机器人内部或空白方向。
- clipping near 太大，近处物体被裁掉。
- 场景光照不足。
- 相机被 mesh 遮挡。

处理方法：

- 先把 Camera prim 临时移动到机器人外部，看是否能看到场景。
- 降低 near clipping，例如 `0.05`。
- 调整 rotation，确认相机光轴朝向外部环境。

### 画面方向不对

URDF link 坐标、mesh 坐标和 USD Camera 光轴约定可能不一致。

处理方法：

- 不要假设 mesh 正面就是 Camera 光轴。
- 在 viewport 里边看边调 Camera prim 的 local rotation。
- 调好后记录最终 rotation，后续写入 Python 固化脚本。

### 相机没有跟着机器人动

通常是 Camera prim 没有挂到机器人 link 下面，而是放在了 `/World` 或其他独立路径。

处理方法：

- 确认 `front_camera_sensor` 是 `front_camera_link` 的子 prim。
- 如果是腕部相机，确认挂在对应腕部 link 下，例如 `left_link6` 或 `right_link6`。

### 重新导入 URDF 后相机没了

这是正常现象。URDF 导入器只会重新生成 URDF 描述的内容，不会自动保留你手动加在旧 USD 上的 Camera prim。

处理方法：

- 不覆盖已经加好相机的新 USD。
- 后续用 Python 脚本把相机添加过程自动化。

## 下一阶段：再接 ROS2

等 Isaac Sim 内部已经能稳定看到相机画面后，再接 ROS2 Bridge。

下一阶段才需要处理：

- `isaacsim.ros2.bridge` 扩展。
- `ROS2CameraHelper`。
- `ROS2CameraInfoHelper`。
- RGB topic。
- depth topic。
- `camera_info` topic。
- ROS2 frame id。
- Docker 内 ROS2 Humble 接收图像。

这样排查顺序更清楚：

```text
先确认 Isaac Sim 里有真实画面
再确认 render product 能输出
最后确认 ROS2 Bridge 能发布
```

当前文档只完成第一步。
