# 排查 cuRobo IK 频繁陷入"无解"

本文是 `7_debug_Ik.md` 和 `8_debug_teleop.md` 之后的第二轮 IK
排查记录。重点解决一个新现象：在 SpaceMouse 遥操作和红方块拖拽
里，cuRobo 经常报：

```text
[right_ee_pose_ik_controller]: IK right failed in 71.7 ms;
  target frame=world pos=(-3.503, -0.732, 1.507);
  reason=cuRobo optimizer failed for this pose.
  The target may be unreachable or over-constrained for the current 6-DOF arm pose.
```

并且会持续连续失败十几到几十帧，TCP 在桌面附近一卡就回不来，
直到把目标拉回到一个明显更"中心"的位姿才恢复。

本文先把"卡住"的几类成因拆开，再给出按风险从低到高的几种调试
路径，最后回答："继续调 cuRobo，还是接 MoveIt？"

## 现象

在 `assets/6_aloha_in_blue_grid_ROS.usda` 上，按照
`8_debug_teleop.md` 启动键盘 + SpaceMouse 调试后：

- 在 TCP 当前位置附近 ±5 cm 范围内的小步目标，IK 大多 50–80 ms
  解出，绿色 `succeeded`。
- 一旦目标 z 超过约 `1.5 m`、或 y 方向贴近 `-0.85 m` 之外，
  IK 会成片地报红色 `failed`，单条 70–100 ms。
- 偶尔会出现"前一帧 succeeded、下一帧同样位姿 failed"的抖动。
- 极端情况比如目标 `z=3.000` 这种明显不可达的，几乎必然失败，
  但失败之后 cuRobo 不会"卡死"，下一帧只要目标合理仍能恢复。

把成功 / 失败位置在 world 系画一下，会看到 cuRobo 实际能稳定
解算的位置呈一个椭球壳，壳的边缘是失败高发区。这本身是 6-DOF
机械臂的物理事实，不是 bug；问题是壳的厚度被姿态约束、
seed 选择、和优化器超参一起进一步压薄了。

## 为什么 cuRobo 容易"陷进去"

cuRobo IK 是 batched GPU 优化器，不是解析 IK。它的成功率取决
于：可达性、姿态约束的紧度、seed 距离最终解的远近、随机种子
和迭代步数。当前实现里几个点同时贴近极限：

### 1. Piper 6-DOF + 严格姿态容忍 = 边界处过约束

`mobile_aloha_joints.yaml`：

```yaml
ee_pose_control:
  position_tolerance_m: 0.005
  rotation_tolerance_rad: 0.10
```

`0.10 rad ≈ 5.7°`。在工作空间中心，这个容忍度宽得很；在边缘
（手臂接近完全伸直）位置，6 个自由度同时要满足"位置 5 mm +
姿态 5.7°"会真的没有解。这不是 cuRobo 的问题，是 6-DOF 串
联臂在工作空间边界本来就只剩 5 自由度（一个奇异方向）甚至
更少。

可观察的特征：失败目标几乎都集中在边缘，把目标拉回中心后立
刻全恢复绿色。

### 2. seed 用"当前关节状态"，边界处会被锁在局部极小

`ee_pose_ik_controller.py` 里的 `_make_curobo_seed`：

```python
positions = [
    self.current_positions.get(command_name, fallback)
    for command_name, fallback in zip(
        self.command_joint_names, self.command_positions
    )
]
```

seed 直接来自 `/joint_states` 的当前角度。这是热启动，正常情
况下让 cuRobo 收敛非常快。但当当前位形已经被"卡"在边界，例
如 `joint2` 或 `joint3` 接近 limit，再加上目标在另一侧，
优化器从这个 seed 一步走出局部极小的概率就会下降。

可观察的特征：failed 之后哪怕目标不再变化，重发也照样失败；
人为把车体后退一小步、或先解一个中心目标让关节重新归位，再
回来就又能解。

### 3. `num_seeds=16` 比官方默认偏少

```python
ik_config = InverseKinematicsCfg.create(
    ...
    num_seeds=16,
)
```

cuRobo 官方 IK 默认 `num_seeds≈30`。`16` 是当时为了把单次解
算压到 50–80 ms 设的。代价是边界附近本来就只有 1–2 个能用的
关节簇，16 个种子全部落到不可行域的概率显著高于 30。

### 4. 没有失败后的退路（fallback）

当前实现失败就直接抛错，下一帧 SpaceMouse 又发了一个新的目
标过来，又解一次。这个流程没有任何缓冲：

- 没有"失败时退到只解位置、放弃姿态"的二次尝试。
- 没有"失败时退到上一次成功目标"的回滚。
- 没有"失败时把 seed 微扰再试一次"的随机重试。

所以现象上看到的就是一长串连续 failed，因为 SpaceMouse 60 Hz
推送目标，目标一直在 unreachable 区域内向外爬。

### 5. SpaceMouse "贴当前位姿 + 增量"的策略放大了边界问题

`8_debug_teleop.md` 里写过：

```text
target_position_world = latest_current_tcp_position_world + delta_world_this_tick
```

这个策略避免了 IK 跟不上时目标越积越远，是好的。但反过来，
当 IK 卡住，TCP 不再前进，而 SpaceMouse 还在持续推送：

- 当前 TCP 没动 -> base 不变。
- 增量持续是同一方向 -> 目标越来越深入 unreachable 区。
- 等用户松手时，目标已经在边界外好几厘米，cuRobo 解不了。

这个机制和 cuRobo 的边界敏感度叠加，肉眼看就是"越推越死"。

## 继续调 cuRobo 的几条低成本路径

按风险/工作量从低到高列出来。前 4 条加起来在当前代码里改
动量都不大，可以并联实施。

### 方案 A：放宽 `rotation_tolerance_rad`（首选）

`mobile_aloha_joints.yaml`：

```yaml
ee_pose_control:
  position_tolerance_m: 0.005
  rotation_tolerance_rad: 0.20
```

把姿态容忍度放到 `0.20 rad ≈ 11.5°`。对当前抓瓶/调试任务影
响极小（夹爪本身就有几度容差），但能让边界处的可行域显著扩
大。如果调试结果提示是边界过约束，这一条就能止血。

如果只想做位置控制，临时再放到 `0.40 rad`：

```yaml
rotation_tolerance_rad: 0.40
```

注意：这并不会让 cuRobo 忽略姿态，它只是在优化器内放大姿态
loss 的"容忍区"。

### 方案 B：把 `num_seeds` 拉回官方默认

`ee_pose_ik_controller.py`：

```python
ik_config = InverseKinematicsCfg.create(
    ...
    num_seeds=32,
)
```

`16 -> 32` 在 RTX 30/40 上单次解算从 50–80 ms 涨到 70–110 ms，
边界成功率明显提高。如果只在 SpaceMouse 60 Hz 模式下嫌慢，可
以走方案 D 的"目标降频"。

### 方案 C：`default_joint_position` 改成实际工作姿态

`config/curobo/mobile_aloha_right.yml`：

```yaml
cspace:
  default_joint_position: [0.0, 1.0, -1.0, 0.0, 0.0, 0.0]
```

这个值是 cuRobo 的 null-space anchor，也是 seed 失败时的
fallback。当前值是一个相当极端的"曲臂"位形。在抓瓶场景下机
械臂大概率工作在 `[0, 0.4, -0.6, 0, 0.5, 0]` 附近。把
`default_joint_position` 调到这个区域：

```yaml
default_joint_position: [0.0, 0.4, -0.6, 0.0, 0.5, 0.0]
```

效果是边界附近优化器更倾向"回家"到一个 6-DOF 都没贴 limit
的位形，从而扩大可行域。

`null_space_weight` 也可以稍微抬一点抑制极端解：

```yaml
null_space_weight: [2.0, 2.0, 2.0, 1.0, 1.0, 1.0]
```

权重越大对 default 越靠拢，太大反而会让 IK 没法跟随目标，
取 1.5–2.5 区间合适。

左臂 `mobile_aloha_left.yml` 同步改一份对称的就行。

### 方案 D：失败后做一次"位置-only" retry

只改 `ee_pose_ik_controller.py` 的 `_solve_curobo`：

```python
try:
    result = solver.solve_pose(
        goal_pose,
        current_state=seed_state,
        run_optimizer=True,
    )
    if not self._success_to_bool(result.success):
        raise RuntimeError("first solve failed")
except Exception:
    relaxed_pose = self._make_curobo_pose(
        msg, rotation_tolerance=math.pi  # 等价于忽略姿态
    )
    result = solver.solve_pose(
        relaxed_pose,
        current_state=seed_state,
        run_optimizer=True,
    )
```

cuRobo 真实接口需要看 `InverseKinematicsCfg` 是否支持 per-call
覆盖姿态容忍度；如果不支持，可以预先创建两个 `InverseKinematics`
solver：一个严格姿态、一个仅位置；失败时切到仅位置的那个再试
一次。

收益：连续 failed 的时长从几十帧压到 1–2 帧。代价是失败那帧
姿态会自由变化，调试时肉眼可见 TCP 朝向"突然顺过去"。
配合上方块 marker 时不会引发崩溃，只是视觉上有点抖。

### 方案 E：失败时退到上一次成功目标 + 暂停推送

在 `mobile_aloha_debug_teleop` 里加一个"IK 健康度"窗口：

- 维护最近 N 帧的 `succeeded / failed` 计数。
- 连续失败超过例如 5 帧时，停止根据 SpaceMouse 增量推进目标，
  转而把目标固定到上一次 succeeded 的世界位姿。
- 用户松手 + 把 SpaceMouse 推回相反方向时再恢复正常推进。

这一步不动 cuRobo，纯上层兜底。代价是手感会有一段"墙"的反
馈：到了边界手柄推不动 TCP。

### 方案 F：用 `lock_joints` 把不参与的关节固定

`mobile_aloha_right.yml` 里：

```yaml
kinematics:
  ...
  lock_joints: null
```

可以显式锁住升降关节，避免 cuRobo 把它当作影响 6-DOF 解的
"额外自由度"。当前虽然 cspace 只列了 `joint1~joint6`，但
URDF 里 `lifting_joint` 仍出现在 chain 上：

```yaml
lock_joints:
  lifting_joint: 0.0
```

值会被覆盖为运行时 `/joint_states` 里的 `lifting_joint`，
cuRobo 内部只是把这个关节当作"固定 transform"处理。这可以
显著减少奇异检测里的偶发问题，特别是升降到底之后。

实际值由节点内部根据当前 lifting 位置注入（参考
`6_IMport_IK.md` 里 link6 在 base_link 下的链路推导）。

### 方案 G（仅排查，不留）：打 debug 输出

`ee_pose_ik_controller.py` 失败时把 cuRobo `result` 的关键字
段打出来：

- `result.position_error` 最大值
- `result.rotation_error` 最大值
- `result.success` 矢量（`num_seeds` 维）

如果 `position_error` 很小但 `rotation_error` 大，就是姿态
约束顶住了，方案 A 直接生效。

如果两边都大，且 `success` 全 false，就是物理边界，方案 C/F
更对症。

排查完务必把额外日志删掉，避免污染 `8_debug_teleop.md` 里那
张 READY 表的预期日志格式。

## MoveIt 接入：成本与收益

如果上面 6 条全做完仍然边界失败率不够低，再考虑切 MoveIt。

### MoveIt 能解决什么

- 成熟的 IK 解析后端：默认 `KDL`（数值，类似 cuRobo 但 CPU），
  也可以换 `TRAC-IK`（IK 求解率明显更高，特别是 6-DOF 边
  界）、或 `BioIK`、`PickIK`。
- 自带 RViz 可视化和 MotionPlanning Panel，可以拖动交互式
  marker，姿态/位置目标分开，姿态约束可单独勾选。
- 失败时返回的 error code 更细：`NO_IK_SOLUTION` /
  `GOAL_IN_COLLISION` / `TIMED_OUT` 等等，调试反馈比 cuRobo
  目前一条 "optimizer failed" 友好很多。
- 直接拿到 OMPL / Pilz 路径规划，遥操作做"先规划后执行"也很
  快接上。

### MoveIt 不能直接解决的

- 6-DOF 物理边界仍然是 6-DOF 物理边界。换 TRAC-IK 也只能让
  本可行的姿态多解出来，不会让 0.005 m + 5.7° 的 unreachable
  目标变成 reachable。
- MoveIt 目前主用 ROS2 Humble 的 `moveit2`，需要写一份
  `mobile_aloha_moveit_config` 包：SRDF、joint_limits、
  controllers.yaml、moveit_controllers.yaml。这些不是手写量
  特别大，但要和当前 `joint_command_interpolator` 走的
  `/isaac_joint_commands` 对齐。
- 当前控制链路是 `target_pose -> ee_pose_ik_controller ->
  /isaac_joint_commands`。换 MoveIt 后链路会变成
  `target_pose -> moveit servo / IK service -> JointTrajectory
  controller -> Isaac`。`JointTrajectory` 路径需要一个能消费
  `FollowJointTrajectory` action 的 bridge，目前没有。直接绕
  过 MoveIt 的 controller manager、只调用 IK service 也可以，
  但就是把 MoveIt 当 IK 库用，那不如直接换 `tracikpy`。

### 实际取舍

短期目标只是"让遥操作不要在边界连续失败"，建议按以下顺序：

1. 先做方案 A（放宽 rotation_tolerance）+ 方案 C（合理的
   default_joint_position）+ 方案 D（位置-only fallback）。
   这一组改动小、风险低，对边界连续失败立竿见影。
2. 仍不够，再做方案 B（`num_seeds=32`）+ 方案 F（`lock_joints`）。
3. 仍不够，且开始关心 self-collision、世界碰撞、和路径规划，
   再迁 MoveIt。这时迁过去能一次性收两件事：IK 求解率 + 路
   径规划，迁的 ROI 才合理。
4. 如果只想换一个更稳的 IK 解析器、不要 MoveIt 全家桶，可以
   先在容器里加 `tracikpy`，作为 cuRobo 失败时的二级 fallback：

   ```python
   try:
       positions = self._solve_curobo(msg)
   except RuntimeError:
       positions = self._solve_trac_ik(msg)
   ```

   TRAC-IK 在 6-DOF 边界处的 success rate 公认比 KDL 高很多，
   且 CPU 求解 1–3 ms，作为 fallback 几乎无感。

## 验证流程

每改完一个方案，按下面顺序复测，确保不引入回归。

1. 重启场景：

   ```bash
   cd /home/xiangpc/dataset/isaac-sim
   ./dev_ws/scripts/run_isaac_mobile_aloha.sh
   ```

   打开 `assets/6_aloha_in_blue_grid_ROS.usda`，Play。

2. 启动 ROS2 控制：

   ```bash
   cd /home/xiangpc/dataset/isaac-sim/dev_ws
   ./scripts/run_ros2_humble_docker.sh --build --launch-control
   ```

   等 `8_debug_teleop.md` 里 `[0]–[5]` 全部 READY。

3. 单步基线复测。先发一个明确"中心可达"的目标：

   ```bash
   ros2 topic pub --once /right_ee_target_pose geometry_msgs/msg/PoseStamped \
     "{header: {frame_id: right/arm_base},
       pose: {position: {x: 0.30, y: 0.0, z: 0.20},
              orientation: {w: 0.521, x: 0.478, y: 0.478, z: 0.521}}}"
   ```

   IK 应稳定 succeeded。

4. 边界扫描复测。沿世界 z 轴每 1 cm 抬一次目标，记录第一次
   连续失败 ≥3 帧时的 z 值。改完前后对比。

5. SpaceMouse 复测。按 `8_debug_teleop.md` 启动键盘 +
   SpaceMouse，做 5 个动作：左右扫、前后扫、上下扫、绕 z
   yaw、绕 x roll。每个动作里点 IK 终端，统计本次 60 s 里
   `failed` 帧数 / 总帧数。

6. 抓瓶任务复测：

   ```bash
   ros2 run mobile_aloha_isaac_control right_bottle_pick_place_debug
   ```

   同样观察日志里红色 failed 比例，并确认 `13_add_friction.md`
   的高摩擦抓持没受姿态容忍度变大的影响。

## 与已有文档的关系

- `7_debug_Ik.md`：第一次 IK 接通的最小流程，重点在"红方块
  发到 cuRobo"。本文是它的"卡住怎么办"补丁。
- `8_debug_teleop.md`：键盘 + SpaceMouse 调试链路。本文里方
  案 D / E / G 的修改点都在这条链路上。
- `9_pub_real_ee_pose.md`：当前 TCP 发布。方案 E 的失败回滚
  目标依赖它发布的 `_in_world` topic。
- `12_debug_collision.md` / `13_add_friction.md` /
  `14_debug_ghost_collision.md`：碰撞和摩擦相关。本文里方案
  F（lock_joints）和 default_joint_position 调整不会影响这
  些场景层 over，可以独立改。
- 所有改动集中在：

  ```text
  dev_ws/ros2_ws/src/mobile_aloha_isaac_control/config/mobile_aloha_joints.yaml
  dev_ws/ros2_ws/src/mobile_aloha_isaac_control/config/curobo/mobile_aloha_left.yml
  dev_ws/ros2_ws/src/mobile_aloha_isaac_control/config/curobo/mobile_aloha_right.yml
  dev_ws/ros2_ws/src/mobile_aloha_isaac_control/mobile_aloha_isaac_control/ee_pose_ik_controller.py
  ```

  恢复时把对应字段回滚到本文之前的版本即可，不需要改场景
  USD。
