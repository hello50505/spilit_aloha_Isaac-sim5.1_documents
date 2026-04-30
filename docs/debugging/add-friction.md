# 给瓶子和右夹爪增加摩擦

本文记录对 `assets/6_aloha_in_blue_grid_ROS.usda` 的摩擦增强，用于右臂抓取 `bottle_xpc` 时减少打滑。

## 背景

右夹爪两指本身有 collision，来自机器人 URDF：

```text
mobile_aloha_sim/split_aloha_mid_360/urdf/split_aloha_rslidar_with_piper_isaac_material.urdf
```

其中：

```text
right/link7 -> package://piper_description/meshes/link7.STL
right/link8 -> package://piper_description/meshes/link8.STL
```

但在 `assets/6_aloha_in_blue_grid_ROS.usda` 场景层，原来只是空 override：

```text
over "right_link7"
{
}

over "right_link8"
{
}
```

没有显式 physics material，也没有静摩擦/动摩擦参数。`bottle_xpc` 有刚体和碰撞：

```text
PhysicsRigidBodyAPI
PhysicsCollisionAPI
physics:approximation = "convexHull"
physics:collisionEnabled = 1
physics:mass = 0.1
```

但也没有单独的高摩擦材质。

## 已做修改

在 `/World` 下新增 physics material：

```text
/World/high_friction_grasp_physics_material
```

参数：

```text
physics:staticFriction = 2.0
physics:dynamicFriction = 1.5
physics:restitution = 0.0
physxMaterial:frictionCombineMode = "max"
physxMaterial:restitutionCombineMode = "min"
```

绑定对象：

```text
/World/bottle_xpc
/World/split_aloha_rslidar_with_piper_isaac_material/right_link7
/World/split_aloha_rslidar_with_piper_isaac_material/right_link8
/World/split_aloha_rslidar_with_piper_isaac_material/right_link7/collisions
/World/split_aloha_rslidar_with_piper_isaac_material/right_link8/collisions
/World/split_aloha_rslidar_with_piper_isaac_material/right/link7
/World/split_aloha_rslidar_with_piper_isaac_material/right/link8
/World/split_aloha_rslidar_with_piper_isaac_material/right/link7/collisions
/World/split_aloha_rslidar_with_piper_isaac_material/right/link8/collisions
```

这里同时绑定了 `right_link7/right_link8` 和 `right/link7/right/link8` 两套路径。原因是
URDF 原始 link 名是 `right/link7`、`right/link8`，不同导入/显示路径下可能会看到下划线版本
或嵌套版本。为了避免材质绑到空 override 上，两套路径都保留 binding。
同时也把材质绑定到了 `collisions` 子 prim，并给这些 collision prim 显式加了
`MaterialBindingAPI` 和 `strongerThanDescendants`，确保真正参与碰撞的 prim 能拿到高摩擦材质。

这样做的目标是：

- 瓶子和夹爪接触时使用较高摩擦。
- 避免默认材质摩擦太低导致瓶子从指尖滑出。
- `frictionCombineMode = "max"` 让接触双方只要有一边是高摩擦材质，就倾向使用高摩擦组合。
- `restitution = 0` 和 `restitutionCombineMode = "min"` 减少碰撞反弹。

## 如何确认

重新打开场景：

```bash
cd /home/xiangpc/dataset/isaac-sim
./dev_ws/scripts/run_isaac_mobile_aloha.sh
```

打开：

```text
assets/6_aloha_in_blue_grid_ROS.usda
```

在 Stage 中检查：

```text
/World/high_friction_grasp_physics_material
```

选中 `bottle_xpc`，在 Property 面板确认存在 physics material binding：

```text
/World/high_friction_grasp_physics_material
```

再选中：

```text
/World/split_aloha_rslidar_with_piper_isaac_material/right_link7
/World/split_aloha_rslidar_with_piper_isaac_material/right_link8
```

如果 Stage 里显示的是嵌套路径，也检查：

```text
/World/split_aloha_rslidar_with_piper_isaac_material/right/link7
/World/split_aloha_rslidar_with_piper_isaac_material/right/link8
```

还要展开 link，检查：

```text
right_link7/collisions
right_link8/collisions
```

或：

```text
right/link7/collisions
right/link8/collisions
```

确认 collision prim 也绑定同一个 physics material。父 link 的 binding metadata 使用：

```text
bindMaterialAs = "weakerThanDescendants"
```

collision prim 的 binding metadata 使用：

```text
bindMaterialAs = "strongerThanDescendants"
```

这样可以避免只绑到父 link、实际碰撞子 prim 仍然没有高摩擦材质。

## Isaac Sim 5.1 里用脚本确认

Isaac Sim 5.1 的 Property 搜索框不一定能搜到 `material:binding:physics`。
`Materials on selected models` 也主要显示视觉材质，不一定显示 physics material。
最可靠的方式是在 `Window -> Script Editor` 里运行下面的脚本：

```python
from pxr import UsdShade
import omni.usd

stage = omni.usd.get_context().get_stage()

paths = [
    "/World/bottle_xpc",
    "/World/split_aloha_rslidar_with_piper_isaac_material/right_link7",
    "/World/split_aloha_rslidar_with_piper_isaac_material/right_link7/collisions",
    "/World/split_aloha_rslidar_with_piper_isaac_material/right_link8",
    "/World/split_aloha_rslidar_with_piper_isaac_material/right_link8/collisions",
]

for path in paths:
    prim = stage.GetPrimAtPath(path)
    print("\n", path, "valid=", prim.IsValid())
    if not prim.IsValid():
        continue
    print("schemas:", list(prim.GetAppliedSchemas()))
    rel = prim.GetRelationship("material:binding:physics")
    print("physics binding:", rel.GetTargets() if rel else None)

material = stage.GetPrimAtPath("/World/high_friction_grasp_physics_material")
print("\nmaterial valid=", material.IsValid())
for name in [
    "physics:staticFriction",
    "physics:dynamicFriction",
    "physics:restitution",
    "physxMaterial:frictionCombineMode",
    "physxMaterial:restitutionCombineMode",
]:
    attr = material.GetAttribute(name)
    print(name, "=", attr.Get() if attr else None)
```

正常情况下应看到：

```text
physics binding: [Sdf.Path('/World/high_friction_grasp_physics_material')]
physics:staticFriction = 2.0
physics:dynamicFriction = 1.5
physics:restitution = 0.0
physxMaterial:frictionCombineMode = max
physxMaterial:restitutionCombineMode = min
```

如果 `right_link7/collisions` 或 `right_link8/collisions` 显示 `valid=False`，
说明当前 Stage 的 collision prim 路径和文档里的不一样。此时在 Stage 树里右键对应
`collisions`，复制 prim path，再把脚本里的路径替换成实际路径。

## 碰撞可视化

优先用 viewport 左上角显示菜单：

```text
Show by type -> Physics -> Colliders -> Selected
```

或：

```text
Show by type -> Physics Mesh -> All
```

观察：

- `bottle_xpc` 是否有绿色 collision outline。
- `right_link7/right_link8` 指尖是否有 collision outline。
- 闭合夹爪时，指尖 collision 是否真正接触瓶子侧面。

## 调参建议

如果瓶子仍然滑出：

```text
physics:staticFriction = 3.0
physics:dynamicFriction = 2.0
```

如果瓶子被夹住后抖动、弹开：

```text
physics:staticFriction = 1.2
physics:dynamicFriction = 1.0
```

同时降低动作速度：

```bash
./scripts/run_ros2_humble_docker.sh --build --launch-control -- \
  start_base_control:=false \
  start_ee_pose_ik:=true \
  ik_side:=right \
  start_real_ee_pose:=true \
  start_debug_teleop:=false \
  joint_command_interpolator_max_step_per_s:=3.0
```

如果还是抓不住，更可能是 collision shape 问题，而不是摩擦数值问题。下一步应给指尖增加简单 box/capsule collider，让接触面更稳定，而不是继续只依赖复杂 STL mesh collider。

## 移除 arm_base 碰撞

抓瓶调试时，`left/arm_base` 和 `right/arm_base` 原始 URDF 使用
`piper_description/meshes/base_link.STL` 作为 collision mesh。这个底座碰撞体比较复杂，
放在升降柱附近时可能和车体/升降结构产生不必要的自碰撞或接触抖动。

之前用 `physics:collisionEnabled = 0` 在 over 上 disable 不生效（引用层里的
`collisions` mesh 仍被合成进了舞台），现在改为直接在场景层把这些 collision 子 prim
**deactivate**，把它们从合成结果中彻底拿掉。

涉及到的路径（4 个，左右各 2 个，分别对应 URDF 默认布局和带 `left/right` 命名空间的布局）：

```text
/World/split_aloha_rslidar_with_piper_isaac_material/left_arm_base/collisions
/World/split_aloha_rslidar_with_piper_isaac_material/right_arm_base/collisions
/World/split_aloha_rslidar_with_piper_isaac_material/left/arm_base/collisions
/World/split_aloha_rslidar_with_piper_isaac_material/right/arm_base/collisions
```

`assets/6_aloha_in_blue_grid_ROS.usda` 里的 over 写法（4 处统一）：

```usda
over "right_arm_base"
{
    over "collisions" (
        active = false
    )
    {
    }
}
```

这样做的好处：

- `(active = false)` 是 USD prim 的属性，会让该 prim 及其后代从 composed stage 中消失，
  无论引用层里 `collisions` 怎么定义、怎么打 `PhysicsCollisionAPI`，都不会再生效。
- 不修改原始 URDF / referenced robot USD，恢复时只需把这四处 over 块删掉或改回 `active = true`。

验证脚本（在 Isaac Sim Script Editor 里跑）：

```python
from pxr import Usd
import omni.usd

stage = omni.usd.get_context().get_stage()
paths = [
    "/World/split_aloha_rslidar_with_piper_isaac_material/left_arm_base/collisions",
    "/World/split_aloha_rslidar_with_piper_isaac_material/right_arm_base/collisions",
    "/World/split_aloha_rslidar_with_piper_isaac_material/left/arm_base/collisions",
    "/World/split_aloha_rslidar_with_piper_isaac_material/right/arm_base/collisions",
]
for p in paths:
    prim = stage.GetPrimAtPath(p)
    print(p, "valid=", prim.IsValid(), "active=", prim.IsActive() if prim.IsValid() else "n/a")
```

预期所有路径要么 `valid=False`，要么 `active=False`，且打开 viewport 的 Show by type ->
Physics -> Colliders 后，`arm_base` 区域不再有红色/绿色 collision wireframe。
