# 排查升降台靠近桌子时的幽灵碰撞

本文记录一次"升降台只在桌子附近卡住"的排查和修复流程，作为
`12_debug_collision.md` 中"碰撞可视化"小节之外更深入的步骤，并把修复
统一收敛到场景层 over，不动机器人原始 USD/URDF。

## 现象

测试场景：

```text
assets/6_aloha_in_blue_grid_ROS.usda
```

在抓瓶调试中观察到：

- 车体停在 `table_xpc` 旁边（如 `12_debug_collision.md` 默认抓取位姿
  附近），下放 `lifting_joint` 到底，升降柱在某个高度被顶住，
  joint position 不再下降，PhysX 报较大反作用力。
- 把车体往后挪一段距离，`lifting_joint` 可以正常下放到底，
  无任何接触提示。
- 整个过程中视觉上看不到任何穿模或接触：升降柱、夹爪、桌面之间都
  有可见的空气间隙。

这个就是俗称的**幽灵碰撞**：collider 形状或接触距离比可视几何大，
肉眼看不到接触，PhysX 已经判定接触并产生约束。

## 为什么是 collider，而不是摩擦或 IK

先排除几类容易混淆的情况：

- 不是 IK 求解失败：`lifting_joint` 是 PhysX joint drive 直接控制，不
  经过右臂 IK；并且后退后立刻好转。
- 不是摩擦：`13_add_friction.md` 里加的 high friction 材质只绑在
  `bottle_xpc` 和 `right_link7/right_link8`，对升降柱无影响。
- 不是控制超调：把 `lifting_joint` 的 `drive:linear:physics:targetPosition`
  设成 0，仍然在桌子附近停在某个高度，说明物理层面有 contact 顶住。
- 也不是桌子被弹飞：`table_xpc` 没有 `PhysicsRigidBodyAPI`，是静态
  triangle mesh collider，不会动。

剩下最可能的就是某个 collider 的有效形状超过了可视外壳。

## collider 可视化里能直接看到的线索

按 `12_debug_collision.md` 的方法：

```text
Show by type -> Physics -> Colliders -> All
```

在出问题的位置（不要 Play）观察：

- `table_xpc` 的 magenta wireframe 比可视立方体明显外扩。底部
  延伸到地面以下、侧面比可见桌面外鼓 2~5 cm。
- `bottle_xpc` 的绿色 convex hull 紧贴瓶身，正常。
- 选中机器人后再切到 `Selected`，重点看
  `lifting_link/collisions`、`box_link/collisions`、
  `right/link1 ~ right/link6` 的 wireframe 是否也明显大于可视外壳。

只要有任意一个 wireframe 在水平方向往桌子方向"鼓出"，配合桌子那一圈
外扩，两者在升降下放路径上就可能提前判定为接触。

## 两类底层原因

### 1. URDF collision 用 STL/DAE，导入时默认 convexHull

机器人 URDF 里：

```text
mobile_aloha_sim/split_aloha_mid_360/urdf/split_aloha_rslidar_with_piper_isaac_material.urdf
```

`box_link`、`lifting_link`、`right/link1 ~ right/link6`、`right/link7~link8`
的 collision 全部直接复用视觉 mesh：

```text
box_link        -> meshes/box_link.dae
lifting_link    -> meshes/lifting_link.dae
right/link1~5   -> piper_description/meshes/linkN.STL
right/link6     -> piper_description/meshes/dae/camera_v3.dae
right/link7~8   -> piper_description/meshes/linkN.STL
```

Isaac Sim 从 URDF 导入时，对这种 mesh collider 默认用
`physics:approximation = "convexHull"`。`lifting_link.dae` 这种把整根
柱体、滑轨、连接件、电机外壳都画在一起的几何，convex hull 会包成一个
比可见柱体明显更宽的凸壳，水平方向就会"凸"出去。这是幽灵碰撞最常见的
来源。

### 2. PhysX contactOffset / restOffset

PhysX 在 collider 之间额外维护一个接触距离：

- `physxCollision:contactOffset`：开始解约束的距离
- `physxCollision:restOffset`：稳定接触时两个 collider 表面之间保留的
  距离

默认情况下这两个值会按物体尺寸自适应。`table_xpc` 是 1×1×1 m 的 cube
+ scale=1，PhysX 自适应后 contactOffset 通常落在 0.02~0.05 m 之间。这
就是 magenta wireframe 比可视立方体外扩 2~5 cm 的原因。

机器人侧的 `lifting_link/collisions` 也带各自的 contactOffset，两边相
加，等效 collision 间距很容易达到 5~10 cm。在某些下放高度，肉眼明明
还有 2~3 cm 间距，PhysX 已经判定接触。

只要单独修一边，幽灵感就会显著减小。

## 第一步：脚本枚举所有可疑 collider

在 `Window -> Script Editor` 里跑（沿用 `13_add_friction.md` 风格）：

```python
from pxr import Usd, UsdGeom
import omni.usd

stage = omni.usd.get_context().get_stage()
robot_root = "/World/split_aloha_rslidar_with_piper_isaac_material"

rows = []
for prim in stage.Traverse():
    path = str(prim.GetPath())
    if not path.startswith(robot_root):
        continue
    if "PhysicsCollisionAPI" not in prim.GetAppliedSchemas():
        continue
    enabled = prim.GetAttribute("physics:collisionEnabled")
    enabled_val = enabled.Get() if enabled else None
    approx = prim.GetAttribute("physics:approximation")
    approx_val = approx.Get() if approx else None
    contact_off = prim.GetAttribute("physxCollision:contactOffset")
    rest_off = prim.GetAttribute("physxCollision:restOffset")
    bbox = UsdGeom.Boundable(prim).ComputeWorldBound(
        Usd.TimeCode.Default(), UsdGeom.Tokens.default_
    ).ComputeAlignedBox()
    size = bbox.GetSize()
    rows.append((
        path,
        enabled_val,
        approx_val,
        contact_off.Get() if contact_off else None,
        rest_off.Get() if rest_off else None,
        tuple(round(s, 3) for s in size),
    ))

for r in rows:
    print(r)
```

重点关注三类输出：

- `approximation = "convexHull"` 且 size 在某个轴上明显大于 link 的
  视觉尺寸的项。
- `approximation = None`：表示走默认值，通常也是 convexHull。
- `contactOffset` 显式设了大值（比如 >0.04）的项。

预期可疑列表：

```text
.../box_link/collisions
.../lifting_link/collisions
.../right_link1/collisions ~ right_link6/collisions
.../right/link1/collisions ~ right/link6/collisions
```

`right_link7/link8` 不要禁用，它们参与抓瓶（参考
`13_add_friction.md`）。

## 第二步：修复方案

按风险从低到高，全部写在 `assets/6_aloha_in_blue_grid_ROS.usda` 的场
景层 over 里，和现有 `arm_base/collisions (active = false)` 保持同样
风格。

### 方案 A：把可疑 link 的 collision approximation 换成更精细的

最小改动、保留碰撞功能。给 `lifting_link/collisions` 和
`box_link/collisions` 改成 `convexDecomposition`：

```usda
over "lifting_link"
{
    over "collisions" (
        prepend apiSchemas = ["PhysxConvexDecompositionCollisionAPI"]
    )
    {
        uniform token physics:approximation = "convexDecomposition"
        int physxConvexDecompositionCollision:maxConvexHulls = 32
        float physxConvexDecompositionCollision:errorPercentage = 0.5
    }
}

over "box_link"
{
    over "collisions" (
        prepend apiSchemas = ["PhysxConvexDecompositionCollisionAPI"]
    )
    {
        uniform token physics:approximation = "convexDecomposition"
        int physxConvexDecompositionCollision:maxConvexHulls = 16
        float physxConvexDecompositionCollision:errorPercentage = 0.5
    }
}
```

`convexDecomposition` 把 mesh 拆成若干凸块组合，能贴合柱体的凹陷处。
代价是仿真稍慢一些，对桌面 PC 调试可忽略。

### 方案 B：抓瓶调试期直接禁用不参与接触的 link

升降下放时只有桌面是可能的接触体，右臂只有 `link7/link8` 真正用于
抓瓶。`link1~link6` 在调试阶段不需要参与碰撞。沿用
`13_add_friction.md` 末尾对 `arm_base/collisions` 用
`(active = false)` 的做法，给两侧 `link1~link6` 也加上：

```usda
over "right_link1"
{
    over "collisions" (
        active = false
    )
    {
    }
}

over "right_link2"
{
    over "collisions" (
        active = false
    )
    {
    }
}

over "right_link3"
{
    over "collisions" (
        active = false
    )
    {
    }
}

over "right_link4"
{
    over "collisions" (
        active = false
    )
    {
    }
}

over "right_link5"
{
    over "collisions" (
        active = false
    )
    {
    }
}

over "right_link6"
{
    over "collisions" (
        active = false
    )
    {
    }
}

over "right"
{
    over "link1"
    {
        over "collisions" (
            active = false
        )
        {
        }
    }

    over "link2"
    {
        over "collisions" (
            active = false
        )
        {
        }
    }

    over "link3"
    {
        over "collisions" (
            active = false
        )
        {
        }
    }

    over "link4"
    {
        over "collisions" (
            active = false
        )
        {
        }
    }

    over "link5"
    {
        over "collisions" (
            active = false
        )
        {
        }
    }

    over "link6"
    {
        over "collisions" (
            active = false
        )
        {
        }
    }
}
```

左臂 `left_link1~6` 和 `left/link1~6` 同理。两套路径都写是因为不同
URDF 导入/显示路径下，下划线版本和带命名空间的嵌套版本有时只活跃一
套，全写避免遗漏。

注意保留：

```text
right_link7 / right_link8 / right/link7 / right/link8
left_link7  / left_link8  / left/link7  / left/link8
```

它们的 `collisions` 不要禁用，否则夹爪不能抓住瓶子，
`13_add_friction.md` 里加的 high friction 也会失去意义。

等需要做避障或自碰撞 benchmark 时再恢复。

### 方案 C：缩小桌子和升降柱的 contactOffset

上面两个方案不动 contact offset，桌子周围那圈 2~5 cm 仍在。如果即使
换了 `convexDecomposition`、关掉了多余 link 的 collision，仍有非常贴
近桌面时的接触，再考虑显式压低 contactOffset。

桌子在场景层直接补：

```usda
def Mesh "table_xpc" (
    prepend apiSchemas = [
        "PhysicsCollisionAPI",
        "PhysxCollisionAPI",
        "PhysxTriangleMeshCollisionAPI",
        "PhysicsMeshCollisionAPI",
    ]
)
{
    float physxCollision:contactOffset = 0.005
    float physxCollision:restOffset = 0.0
}
```

升降柱 over 里也加：

```usda
over "lifting_link"
{
    over "collisions"
    {
        float physxCollision:contactOffset = 0.005
        float physxCollision:restOffset = 0.0
    }
}
```

`contactOffset` 不能小于等于 `restOffset`，也不能为 0；`0.005` 是经
验下限，再小会出现穿透或抖动。改完后桌子那圈 magenta wireframe 会
肉眼变窄。

### 方案 D（仅快速二分）：用 boundingCube 验证假设

只用于排查、不要长期保留。把 `lifting_link/collisions` 临时换成最小
化包围盒：

```usda
over "lifting_link"
{
    over "collisions"
    {
        uniform token physics:approximation = "boundingCube"
    }
}
```

`boundingCube` 是 mesh 的轴对齐 bounding box，通常**比 convexHull 还
要大**。预期：

- 如果在这种情况下问题反而消失，说明原来的 convexHull 因为 mesh 里
  少数几个"飞点"导致了不规则鼓包，应改回方案 A。
- 如果情况更糟（更早卡住），就坐实是 lifting_link 的 collision 范围
  在水平方向超出实际柱体，应直接走方案 B 把不需要的 collision 关
  掉，或换 `convexDecomposition` 配合更紧的参数。

调试结束后必须恢复。

## 验证流程

修完之后按顺序检查：

1. 重新打开场景：

```bash
cd /home/xiangpc/dataset/isaac-sim
./dev_ws/scripts/run_isaac_mobile_aloha.sh
```

打开：

```text
assets/6_aloha_in_blue_grid_ROS.usda
```

2. 重新跑第一步那段脚本，确认每个被修改 prim 的
   `approximation` 已切到目标值，或 `enabled=False`，或父
   prim 的 `(active = false)` 让 collision 子 prim 不再出现在结果里。

3. viewport 打开 `Show by type -> Physics -> Colliders -> All`，肉眼
   对照 wireframe 与可视体积：

   - `table_xpc` 的 magenta wireframe 是否变窄（仅方案 C 后）。
   - `lifting_link/collisions` 是否贴合可见柱体。
   - 关掉 collision 的 link 周围不再有 wireframe。

4. Play，把车停在原来卡住的位置，下放 `lifting_joint` 到底，
   确认 joint position 能贯穿到 limit，不再被顶住。

5. 再开一次 `12_debug_collision.md` 里的
   `right_bottle_pick_place_debug` 脚本：

```bash
cd /home/xiangpc/dataset/isaac-sim/dev_ws
./scripts/run_ros2_humble_docker.sh --build --launch-control -- \
  start_base_control:=false \
  start_ee_pose_ik:=true \
  ik_side:=right \
  start_real_ee_pose:=true \
  start_debug_teleop:=false \
  start_right_ee_target_marker:=false
```

```bash
ros2 run mobile_aloha_isaac_control right_bottle_pick_place_debug
```

观察：

- 抓取时 `right_link7/link8` 仍能稳定夹住瓶子，证明
  `13_add_friction.md` 配置未受影响。
- 抬起、放置阶段升降柱在桌子附近不再被顶住。

## 与已有文档的关系

- `12_debug_collision.md`：抓瓶调试动作和 collision 可视化的入门流
  程。本文是它的进阶补丁。
- `13_add_friction.md`：高摩擦材质和 `arm_base/collisions` 的禁用。
  本文延续了"场景层 over，不动原始 robot USD"的风格，并把
  `(active = false)` 思路推广到 `link1~link6`。
- 修改全部集中在 `assets/6_aloha_in_blue_grid_ROS.usda`，恢复时把对
  应 over 块删掉或改回 `active = true`、移除显式
  `contactOffset` 即可。
